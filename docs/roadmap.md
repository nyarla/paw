# PAW audio wire ロードマップ

Status: Draft

## Phase 1: Linux native MVP

目的: Linux DAW から Linux native Host 上の CLAP エフェクトを利用できるようにする。

- Client: Linux CLAP プラグイン
- Host: Linux native `pawd`
- Supervisor + vendor worker
- CLAP backend
- エフェクトのみ
- UDP 音声、ジッタバッファ、パケット補間
- state save/load proxy
- レイテンシ報告
- 同一プラグインの複数セッション
- vendor グループ / 同一 vendor の同一 Worker
- Worker クラッシュ分離
- CLI: `paw` による catalog / session / debug
- テスト用 CLAP プラグイン
- token 認証、localhost bind

## Phase 2: Wine Host

目的: Linux DAW から Wine prefix 内の Windows Host 上の Windows CLAP エフェクトを利用できるようにする。

- `pawd.exe` を `x86_64-pc-windows-gnu` でビルド
- Wine prefix の自動起動
- endpoint / `pawd.lock` / spawn lock
- Windows CLAP のロード
- Windows テストプラグイン
- Wine のログ / エラー表示
- native と Wine で同じ起動フェーズを使う

## Phase 3: Windows

目的: Windows DAW から Host を利用できるようにする。Wine の実用性を高めるため VST3 / VST2 も扱う。

- VST3 Client
- VST3 Host backend
- 可能なら VST2 Host backend
- native Windows Host
- Windows DAW の音声設定への追従
- VST3 の vendor 取得（`PFactoryInfo.vendor` / `PClassInfo2.vendor`）
- VST2 の vendor 取得（`effGetVendorString` など）
- VST2 SDK / VST3 SDK のライセンス確認

Windows DAW の CLAP 対応は限定的なため、VST3 Client は必須と考える。VST2 Client は必要になった段階で検討する。

## Post-MVP

### パラメータ / オートメーション

- Host のプラグインパラメータを Client に proxy する
- CLAP の dynamic params / VST3 の parameter changes を扱う
- DAW のオートメーションに反映する

### MIDI / インストゥルメント

- CLAP note ports / VST3 note events の転送
- MIDI クロック / transport の転送
- インストゥルメントのテスト

### オフラインレンダリング

- DAW の offline / faster-than-realtime render への対応
- ブロッキング送受信または大きなバッファでのモード
- 非対応時の明示的なエラー / 無音

### 32-bit

- 32-bit Windows Host のビルド
- `group + arch` の Worker key
- 32-bit VST2 の検証

### GUI streaming（任意機能）

MVP では実装しないが、アーキテクチャ上は Host の GUI extension と Client の editor を追加できるようにしておく。

想定するモード:

```toml
[gui]
mode = "none"        # none | host_window | stream
```

- `none`: GUI なし
- `host_window`: Host 側のディスプレイにプラグイン GUI を表示する
  - 同一マシンで Wine prefix 間ブリッジする場合など
  - GUI streaming は off
- `stream`: プラグインウィンドウをキャプチャして Client に配信する
  - LAN 経由で別マシンの Host を使う場合
  - マウス / キーボード入力を Host に転送する

実装上の検討事項:

- キャプチャ
  - Windows: Windows Graphics Capture / DWM / PrintWindow
  - Wine: X11 (XComposite など)
  - Linux native: X11 / Wayland (PipeWire)
- エンコード
  - H.264 / VP9 / AV1
  - 低遅延優先。必要なら JPEG / zstd の簡易モード
- 入力転送
  - 座標変換、フォーカス、修飾キー、ホイール
  - セキュリティ / 誤操作対策
- Client 側
  - CLAP GUI / VST3 editor に映像サーフェスを埋め込む
  - `raw-window-handle` とデコーダを利用
  - 音声スレッドをブロックしない
- 転送
  - 音声 / 制御とは別の TCP / QUIC ストリーム
  - GUI streaming の有無は `hello` の capability で通知

注意:

- GPU アクセラレーションされたウィンドウは単純な BitBlt では取得できないことがある
- Wine の GPU 対応は環境依存
- GUI streaming はパラメータ proxy / オートメーションの代替にはならない

### その他

- WAN / TLS / NAT 越え
- Client ↔ Host 間の共有メモリ音声パス
- セッションごとのサンドボックス
- Worker のグループ化ポリシーの高度化
- 圧縮 state 転送
