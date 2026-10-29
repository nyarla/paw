# PAW audio wire

PAW audio wire は、LAN 内で VST / CLAP プラグインを共有するためのツールです。Host がプラグインをロードし、DAW にロードされた Client がネットワーク経由で音声を送受信します。Wine prefix 内の Host を Client が自動起動することで、Linux DAW から Windows プラグインを利用できます（Windows DAW は後続フェーズ）。

現在は設計フェーズです。実装はまだありません。

## ドキュメント

- [設計](docs/design.md)
- [制御プロトコル](docs/protocol.md)
- [ロードマップ](docs/roadmap.md)
- [既知の制約](docs/limitations.md)

## コマンド（予定）

- `pawd` — Host supervisor / worker
  - `pawd serve` — Supervisor を起動
  - `pawd --worker` — ベンダーグループ用 Worker を起動
- `paw` — CLI
  - `paw catalog list`
  - `paw session ...`
  - `paw debug ...`

## 主要な構成

- 1 prefix / 1 native instance につき Supervisor は 1 つ
- Supervisor がベンダーグループごとに Worker を起動
- Client は Supervisor と制御通信し、Worker と直接音声通信する
- 同一ベンダーのプラグインは同一 Worker に集約し、プラグイン間 IPC を成立させる
- MVP は CLAP / エフェクトのみ。VST2 / VST3、MIDI、パラメータ、GUI streaming は後続
- macOS / AU は開発用ハードウェアが無いため現時点の対象外
