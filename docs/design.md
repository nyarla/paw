# PAW audio wire 設計

Status: Draft

## 0. 名称とコマンド

- プロジェクト名: **PAW audio wire**
- CLI: `paw`
- Host supervisor / worker: `pawd`（`pawd serve`, `pawd --worker`）
- Windows Host: `pawd.exe`
- Client plugin: `paw-audio-wire.clap`
  - 将来: `PAW audio wire.vst3`
- Client 設定: `paw-client.toml`
- Host 設定: `paw-host.toml`
- 環境変数: `PAW_*`
- state magic: `PAWS1`
- audio magic: `PAWA1`
- endpoint: `$XDG_RUNTIME_DIR/paw/<key>.json` など

`pawd` は PAW daemon の略で、Supervisor と Worker を兼ねる同一バイナリ。Worker は `pawd --worker` で起動する。

## 1. 目的

- LAN 内で VST / CLAP プラグインを共有する
- Host がプラグインをロードし、Client は DAW のプラグインとして動作する
- Client は同一ディレクトリの設定ファイルを読み、Host にプラグインのロードを指示する
- Host はネットワーク API / ローカル CLI から操作できる
- Wine prefix 内の Windows Host を Client が自動起動し、Linux DAW から Windows プラグインを使えるようにする（Windows DAW は Phase 3）
- Host はクロスプラットフォーム設計とする。MVP は Linux native + Wine、Windows は Phase 3
- macOS は開発用ハードウェアが無いため、現時点では対象外

## 2. MVP の非目標

- パラメータ / オートメーション
- MIDI / インストゥルメント
- オフラインレンダリング
- プラグイン GUI / GUI streaming
- 32-bit プラグイン
- VST2 / VST3
- WAN / NAT 越え
- セッションごとの完全なサンドボックス

## 3. 用語

| 用語       | 意味                                                                 |
| ---------- | -------------------------------------------------------------------- |
| Client     | DAW にロードされる VST / CLAP プラグイン                             |
| Supervisor | 1 prefix / 1 native instance に 1 つだけ存在する Host の管理プロセス |
| Worker     | ベンダーグループごとに 1 つ起動するプラグイン実行プロセス            |
| Runtime    | native / wine などの起動方式                                         |
| Endpoint   | Supervisor の control port などを記録した JSON ファイル              |
| Session    | Client の 1 プラグインインスタンスに対応する Host 側の実行単位       |
| Group      | 同一プロセスにまとめるプラグインのベンダーグループ                   |

「1 prefix 1 host」は、**Supervisor 1 + vendor worker N** を指す。

### 3.1 リポジトリ構成（予定）

```text
crates/
  paw-protocol
  paw-config
  paw-host-core
  paw-host-clap
  paw-client-core
  paw-client-clap
  paw-launcher
  paw-cli
  paw-test-common
  paw-test-plugins
apps/
  pawd
  paw
```

将来:

- `paw-host-vst3` / `paw-host-vst2`
- `paw-client-vst3`
- `paw-gui-stream`

macOS / AU 関連は現時点の対象外とし、開発環境が用意できた段階で追加する。

## 4. 全体構成

```text
DAW
 └─ Client (CLAP / 将来 VST3)
      ├─ control TCP -> Supervisor
      └─ audio UDP   -> Worker（session.create で返された endpoint）

Supervisor (pawd、1 prefix に 1 つ)
 ├─ control TCP server
 ├─ catalog / allowlist
 ├─ session registry
 ├─ worker manager
 │    ├─ Worker: vendor A (pawd --worker)
 │    │    ├─ plugin instance 1
 │    │    └─ plugin instance 2
 │    └─ Worker: vendor B
 │         └─ plugin instance 3
 └─ endpoint file / pawd.lock
```

- 音声は Client と Worker を直結する。Supervisor は音声パスに入らない
- state / close / stats は Supervisor 経由
- Worker は同一ベンダーのプラグインを同一プロセスにまとめる
- 異なるベンダーは別 Worker に分離する

## 5. 起動とライフサイクル

### 5.1 共通フェーズ

Runtime が native でも Wine でも、起動フェーズは同一とする。

```text
1. resolve_runtime
2. compute_endpoint_key
3. try_connect
4. acquire_spawn_lock
5. recheck
6. spawn
7. await_ready
8. release_lock
9. supervise
```

- `endpoint_key` は `runtime種別 + prefix + pawd exe + protocol major + user` のハッシュ
- spawn 競合は native 側のファイルロックで直列化する
- Supervisor 自身も `pawd.lock` を取り、二重起動を防ぐ
- endpoint は control server の bind 後に原子的に書き、`hello` 成功で ready とする

### 5.2 Runtime ごとの差分

| Runtime      | spawn                                    | endpoint / lock                              | パス変換         |
| ------------ | ---------------------------------------- | -------------------------------------------- | ---------------- |
| Linux native | `pawd serve ...`                         | `$XDG_RUNTIME_DIR/paw/<key>.json`            | なし             |
| Wine (Linux) | `WINEPREFIX=... wine pawd.exe serve ...` | native path を `Z:\...` で Supervisor に渡す | native ↔ Windows |

macOS native / macOS Wine は現時点では対象外。起動フェーズは Runtime を差し替えて追加できる形にしておく。

### 5.3 自動起動

- DAW のプラグインスキャン時には起動しない
- `activate` をトリガーに非同期で起動する
- `activate` 自体はブロックしない。起動完了までは無音を返す
- `start_processing` / 初回 `process` をフォールバックトリガーにする
- 緊急時は `PAW_DISABLE_AUTOSTART=1` で無効化できる

```toml
[host]
autostart = "on_activate"   # on_activate | on_start_processing | manual
```

### 5.4 Worker のライフサイクル

- 最初のセッション作成時に Supervisor が `pawd --worker` を spawn する
- Worker は control 接続で Supervisor に `worker.hello` を送る
- セッションが `worker_idle_sec` の間 0 なら Worker は終了する
- Worker が落ちた場合、Supervisor は該当ベンダーのセッションだけ `session.lost` にする
- Client はキャッシュした state でセッションを再作成する
- Supervisor が落ちた場合、Worker は control 切断を検知して終了する

## 6. プラグインカタログとベンダーグループ

### 6.1 カタログ

Supervisor はプラグインを scan し、カタログとして保持する。

`PluginMetadata` の主な項目:

- `format`: `clap` / `vst3` / `vst2`
- `id`: CLAP plugin id、VST3 class id、VST2 unique id
- `name`
- `vendor_raw`: プラグイン由来のベンダー文字列
- `vendor_key`: 正規化したベンダーキー
- `group`: 実際に使用する Worker グループ
- `group_source`: `config` / `alias` / `vendor` / `id` / `path` / `default`
- `kind`: `effect` / `instrument`（MVP は effect のみ）
- `arch`
- `path`
- `latency_samples`（取得できれば）

Client はパスを送らず、カタログ ID を送る。Supervisor が allowlist とカタログから解決する。

### 6.2 format ごとの vendor 取得

| Format | 取得元                                                             |
| ------ | ------------------------------------------------------------------ |
| CLAP   | `clap_plugin_descriptor.vendor`、`id`、`name`                      |
| VST3   | `PFactoryInfo.vendor`、`PClassInfo2.vendor`、factory / class info  |
| VST2   | `effGetVendorString`、`effGetProductString`、`effGetVendorVersion` |

`vendor_raw` はそのまま保存し、`vendor_key` は Unicode NFKC、trim、空白正規化、casefold で作る。

### 6.3 グループ解決の優先順位

1. `[[plugin_groups.rules]]` の明示ルール（先勝ち）
2. `[vendor_aliases]`
3. 正規化済み vendor 文字列
4. プラグイン ID の reverse-DNS prefix
5. プラグインの親ディレクトリ名
6. `default`

```toml
[vendor_aliases]
"fabfilter software instruments" = "fabfilter"

[[plugin_groups.rules]]
format = "clap"
id = "com.fabfilter.*"
group = "fabfilter"

[[plugin_groups.rules]]
format = "vst3"
path = "**/FabFilter/*"
group = "fabfilter"
```

- 自動判定が外れた場合は設定で上書きできる
- 同一 vendor を複数グループに分けたい場合もルールで対応する
- 異なる vendor を同一グループにまとめたい場合もルール / alias で対応する
- `group = "default"` または vendor 空の場合は警告をログに出す
- `paw catalog list` で `vendor_raw` / `vendor_key` / `group` / `group_source` を確認できるようにする

### 6.4 Worker key

MVP では `group` を Worker key とする。将来は `group + arch` にする。

同一グループのプラグインは同一 Worker プロセスにロードされるため、以下が成立する。

- プロセス内グローバルによるベンダー IPC
- Windows の名前付き共有メモリ / 名前付きパイプ
- 同一 vendor の複数インスタンス間の状態共有

異なるグループは別プロセスなので、クラッシュやグローバル状態の衝突を隔離できる。

## 7. 音声パスとレイテンシ

### 7.1 DAW のブロックをそのまま転送する

- Client は DAW の `process` 呼び出しごとに 1 ブロックを送る
- ブロックが MTU を超える場合はフラグメント化する
- ヘッダに `sample_pos`, `frames`, `fragment_index`, `fragment_count` を持たせる
- Worker はブロックを再構成し、DAW と同じブロックサイズでプラグインを process する

これにより、プラグインのブロックサイズ依存の挙動をローカル時と揃えやすい。

### 7.2 DAW 設定への追従

ASIO バッファサイズそのものはプラグイン API から直接見えない。  
CLAP の `activate(sample_rate, min_frames, max_frames)`、VST3 の `setupProcessing(maxSamplesPerBlock)` を代理値として使う。

```text
daw_block_frames = max_frames_count / maxSamplesPerBlock
（取得できない場合は設定値 block_size を使う）

daw_block_ms = daw_block_frames / sample_rate * 1000
jitter_ms = max(10.0, daw_block_ms)      # auto のデフォルト
jitter_samples = ceil(jitter_ms * sample_rate / 1000)

report_latency_samples =
    jitter_samples
  + plugin_latency_samples
  + safety_samples
```

- **DAW 自身の ASIO バッファをプラグイン遅延に加算しない**。PDC が二重補正になる
- DAW のブロックサイズはジッター量の基準として使う
- DAW のブロックが小さい場合にジッターが不足しないよう、最低 10ms を持たせる
- DAW のブロックサイズ / サンプルレートが変わったら再 activate が来るので再計算する
- Host のプラグイン遅延がロード後に判明したら `request_restart` で報告値を更新する
- 起動完了前は `plugin_latency_samples = 0` として暫定値を報告する

```toml
[audio]
latency_policy = "follow_daw"   # follow_daw | fixed
jitter_ms = "auto"              # auto | number
jitter_blocks = "auto"          # auto | number
block_size = 256                # フォールバック
safety_samples = 0
```

### 7.3 ジッタバッファ

- Client は `jitter_samples` を目標に出力を遅延させる
- 欠落パケットはフェード付き無音で補間する
- アンダーラン / レイトパケットは統計に記録する
- RTT は ping / pong で計測するが、PDC には設定したジッター値を使う

## 8. 制御と state

- 制御は TCP + 長さ付きフレーミング
- MVP の codec は CBOR（`ciborium`）
- 将来的に Protobuf へ移行する。フレーミングとメッセージ意味論は codec 非依存にする
- 詳細は `docs/protocol.md`

### 8.1 state proxy

- Client の `save` で Host のプラグイン state を取得し、DAW プロジェクトに埋め込む
- Client の `load` で Host のセッション作成後に state を送る
- state はチャンク転送する
- 保存時に Host が応答しない場合は、Client が保持する直近のキャッシュを保存する
- Host にプラグインが無い場合は state を保持したまま無音 + エラー表示にする

state の envelope:

```text
magic "PAWS1" | version | format | plugin_id | host_version | payload
```

### 8.2 エラー / 再接続

- control 接続が切れた場合はセッションを猶予付きで保持する
- Worker が落ちた場合は `session.lost` を Client に通知する
- Client は指数バックオフで再接続し、state を復元する
- 再起動ループを避けるため、連続失敗回数に上限を設ける

## 9. セキュリティ

- Supervisor / Worker はデフォルトで `127.0.0.1` に bind する
- endpoint ファイルはユーザー権限 0700 のディレクトリに置く
- `hello` で token を検証する
- Worker の control 接続にも別 token を使う
- 音声パケットには `session_token` を含め、Worker が検証する
- Client はプラグインのパスを送れない。カタログ ID + allowlist のみ
- 将来のリモート LAN 利用時は bind アドレスと firewall 設定を明示する

## 10. テスト

### 10.1 テスト用プラグイン

以下の CLAP テストプラグインを用意する。Linux / Windows の両方にビルドする。

- `paw.test.gain`: 音声パス確認用
- `paw.test.latency`: 既知の遅延（例 128 samples）を報告
- `paw.test.state`: state save/load 確認用
- `paw.test.same_a` / `paw.test.same_b`: 同一 vendor。共通 DLL / SO のグローバル変数で同一プロセス判定
- `paw.test.other`: 別 vendor。別 Worker になることの確認
- `paw.test.crash`: 任意のタイミングで abort し、Worker 分離を確認

同一 vendor のテストには、両プラグインがリンクする共通ライブラリを使う。同一 Worker なら共通ライブラリのグローバルを共有し、別 Worker なら共有しない。

### 10.2 テスト項目

- 音声の sine パススルー
- ジッター / パケットロス / アンダーラン
- state save/load
- レイテンシ報告
- 同一 vendor が同一 Worker に入ること
- 別 vendor が別 Worker に入ること
- Worker クラッシュ時に他 vendor のセッションが継続すること
- Wine prefix の自動起動 / endpoint / 再接続
- 32-bit は MVP 後

## 11. ロードマップ

詳細は `docs/roadmap.md`。

1. Linux DAW + Linux native Host（CLAP、エフェクトのみ）
2. Linux DAW + Wine Host（Windows CLAP、エフェクトのみ）
3. Windows DAW 対応（VST3 クライアント、VST3 Host backend、可能なら VST2）

macOS / AU は開発環境が無いため現時点の対象外。

## 12. 未決事項

- VST3 SDK のライセンスとプロジェクト全体のライセンス
- GUI streaming のキャプチャ方式（Windows Graphics Capture / Wine X11 / Wayland）
- Windows Client で VST2 クライアントまで必要か
- Worker の再起動ポリシー（自動再作成の上限、state 復元のタイミング）
