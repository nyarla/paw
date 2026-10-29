# PAW audio wire 制御プロトコル

Status: Draft

## 1. レイヤ

| レイヤ    | MVP                          | 将来            |
| --------- | ---------------------------- | --------------- |
| Transport | TCP（control）、UDP（audio） | QUIC            |
| Framing   | 長さ付きフレーム             | 同じ            |
| Codec     | CBOR (`ciborium`)            | CBOR + Protobuf |
| RPC       | request / response / event   | 同じ            |

制御と音声は独立している。音声を CBOR / Protobuf に載せない。

## 2. フレーミング

```text
u32 payload_len
u8  frame_type
u8  flags
u16 reserved
u64 request_id
payload
```

- `payload_len` は payload のみの長さ
- `request_id` は request / response / event の相関に使う
- 最大 control frame は 16 MiB
- state はチャンクに分ける

`frame_type`:

| 値  | 意味                       |
| --- | -------------------------- |
| 1   | Control request / response |
| 2   | State chunk                |
| 3   | Ping                       |
| 4   | Pong                       |
| 5   | Event                      |

## 3. Codec と移行

### 3.1 MVP

- CBOR (`ciborium`) のみ
- `hello` で `codecs = ["cbor"]` を通知

### 3.2 将来の Protobuf 移行

- `hello` の `codecs` で両対応を通知し、共通の最上位 codec を選ぶ
- 例: `["protobuf", "cbor"]` を受け取り、Protobuf を選択
- 少なくとも 1 リリースは両 codec をサポートする
- Supervisor ↔ Worker は同時に更新できるため、同じ codec を使ってよい
- Client ↔ Supervisor はバージョンずれを考慮し、`hello` で交渉する

### 3.3 codec 非依存のルール

Protobuf へ移行しやすくするため、CBOR では以下に限定する。

- bool / 整数 / 浮動小数 / 文字列 / bytes / 配列 / map / optional
- CBOR 固有の tag、indefinite length、非文字列キーは使わない
- enum は明示的な文字列タグ、または整数 enum にする
- 未知フィールドは無視する
- 追加フィールドは optional / default にする
- `Value` のような動的型をメッセージに使わない

メッセージ定義は `docs/protocol.md` を source of truth とし、Protobuf 追加時に `.proto` へ 1:1 で移す。

## 4. メッセージ

擬似 Rust 表記。

```rust
Hello {
    protocol_major: u32,
    protocol_minor: u32,
    client_version: String,
    codecs: Vec<String>,
    token: String,
    capabilities: Vec<String>,
}

HelloResult {
    host_version: String,
    protocol_major: u32,
    protocol_minor: u32,
    codec: String,
    capabilities: Vec<String>,
}

CatalogList {}

CatalogEntry {
    id: String,
    format: String,          // clap | vst3 | vst2
    name: String,
    vendor_raw: Option<String>,
    vendor_key: Option<String>,
    group: String,
    group_source: String,
    kind: String,            // effect | instrument
    arch: String,
    path: Option<String>,
    latency_samples: Option<u32>,
}

SessionCreate {
    plugin_id: String,
    kind: String,            // MVP は effect のみ
    audio: AudioSpec,
}

AudioSpec {
    sample_rate: u32,
    max_block_frames: u32,
    channels_in: u32,
    channels_out: u32,
}

SessionCreated {
    session_id: String,
    worker_id: String,
    group: String,
    audio_host: String,
    audio_port: u16,
    session_token: u64,
    plugin_latency_samples: u32,
    audio_ports: Vec<AudioPort>,
}

SessionClose {
    session_id: String,
}

StateGet {
    session_id: String,
}

StateSet {
    session_id: String,
    total_len: u64,
    crc32: u32,
}

Ping {
    nonce: u64,
}

Pong {
    nonce: u64,
}

Event {
    event: String,           // session.lost | worker.started | worker.exited | ...
    session_id: Option<String>,
    worker_id: Option<String>,
    message: Option<String>,
}
```

### 4.1 エラー

```rust
Error {
    code: String,
    message: String,
    retryable: bool,
    details: Option<String>,
}
```

主な `code`:

- `auth_failed`
- `protocol_mismatch`
- `codec_unsupported`
- `plugin_not_found`
- `plugin_not_allowed`
- `plugin_load_failed`
- `session_not_found`
- `worker_unavailable`
- `busy`
- `timeout`
- `unsupported`
- `internal`

## 5. state 転送

大きな state を 1 フレームに載せない。

```text
StateBegin { request_id, session_id, total_len, crc32 }
StateChunk { request_id, offset, data: bytes }
StateEnd   { request_id, crc32 }
```

- chunk size は最大 1 MiB
- total state は最大 512 MiB（設定可能）
- 1 セッションにつき同時に 1 つの state 転送のみ
- 転送中も ping / event は処理できる
- タイムアウトは設定可能（例 30 秒）
- 受信側は `total_len` と `offset` を検証し、上限を超える allocation をしない

state envelope:

```text
magic "PAWS1"
version u16
format u8
plugin_id_len u16 + plugin_id
host_version_len u16 + host_version
payload
```

## 6. 音声パケット

UDP。planar f32。制御フレームとは別。

```text
magic u32            "PAWA1"
version u8
flags u8
channels u8
fragment_index u8
sample_rate u32
session_id u64
session_token u64
sample_pos u64
seq u32
frames u16
fragment_count u16
payload_len u16
```

- payload は planar f32
- 1 パケットの payload は MTU 以下（目安 1200 bytes）
- `sample_pos` は Client のサンプル位置
- 1 ブロックが MTU を超える場合は `fragment_index` / `fragment_count` で分割
- Worker は `session_id` + `session_token` を検証する
- 欠落 / 遅延パケットは統計に記録し、出力側で補間する

## 7. Supervisor ↔ Worker

Worker は Supervisor が spawn し、loopback TCP で接続する。同じフレーミング / codec を使う。

```rust
WorkerHello {
    worker_id: String,
    group: String,
    pid: u32,
    arch: String,
    audio_port: u16,
    formats: Vec<String>,
}

WorkerSessionCreate {
    session_id: String,
    plugin_id: String,
    audio: AudioSpec,
}

WorkerSessionCreated {
    audio_port: u16,
    session_token: u64,
    plugin_latency_samples: u32,
}

WorkerSessionClose {
    session_id: String,
}

WorkerStateGet {
    session_id: String,
}

WorkerStateSet {
    session_id: String,
    total_len: u64,
    crc32: u32,
}

WorkerEvent {
    worker_id: String,
    kind: String,
    session_id: Option<String>,
    message: Option<String>,
}
```

- Worker は Supervisor の control 接続が切れたら終了する
- Supervisor は Worker の異常終了を検知し、該当グループのセッションを `session.lost` にする

## 8. バージョニング

- `protocol_major` が異なる場合は接続を拒否する
- `protocol_minor` は後方互換の追加に使う
- `capabilities` で任意機能を通知する（例: `state.chunk`, `audio.fragment`）
- 未知の capability は無視する
- 未知のメッセージは `unsupported` を返す
- フィールド名 / 意味は変更しない。追加は optional にする

## 9. 制限とタイムアウト

| 項目                   | 初期値                          |
| ---------------------- | ------------------------------- |
| control frame 最大     | 16 MiB                          |
| state chunk 最大       | 1 MiB                           |
| state 最大             | 512 MiB                         |
| hello timeout          | 10 秒                           |
| session.create timeout | 30 秒（プラグインロードを含む） |
| state transfer timeout | 30 秒                           |
| ping interval          | 5 秒                            |
| session idle timeout   | 設定値                          |
| worker idle timeout    | 300 秒                          |
