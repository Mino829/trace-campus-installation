# API・WebSocket仕様

## 共通方針

- Base path: `/v1`
- 形式: `application/json`
- 時刻: UTCのISO 8601
- 参加者認証: セッション作成時に発行するBearer token
- 運営・描画認証: 参加者用とは別の固定シークレットまたは短期トークン
- 到着・停止操作は冪等にする
- 位置点はキャンパス範囲と時刻を検証してから保存する

## エラー形式

```json
{
  "error": {
    "code": "INVALID_POSITION",
    "message": "position is outside the allowed area",
    "requestId": "request-id"
  }
}
```

内部情報や生の位置情報をエラーメッセージへ含めない。

## セッション作成

### `POST /v1/sessions`

```json
{
  "consentVersion": "2026-07-01",
  "clientTime": "2026-07-13T12:00:00Z"
}
```

```json
{
  "sessionId": "anonymous-id",
  "participantToken": "one-time-secret",
  "status": "tracking",
  "expiresAt": "2026-07-13T18:00:00Z",
  "positionIntervalSeconds": 5
}
```

`participantToken`はレスポンス時だけ返し、サーバーにはハッシュを保存する。

## 位置送信

### `POST /v1/sessions/{sessionId}/positions`

通信断からの復旧に備え、1〜50件のバッチを受け付ける。

```json
{
  "points": [
    {
      "latitude": 35.000001,
      "longitude": 135.000001,
      "accuracyM": 12.4,
      "recordedAt": "2026-07-13T12:00:05Z"
    }
  ]
}
```

```json
{
  "accepted": 1,
  "rejected": 0,
  "latestRecordedAt": "2026-07-13T12:00:05Z"
}
```

拒否理由は `OUTSIDE_GEOFENCE`、`LOW_ACCURACY`、`TOO_OLD`、`IMPOSSIBLE_JUMP` を想定する。

## 到着

### `POST /v1/sessions/{sessionId}/arrival`

Header: `Idempotency-Key: random-value`

```json
{
  "source": "qr",
  "arrivedAt": "2026-07-13T12:20:00Z"
}
```

```json
{
  "arrivalId": "arrival-id",
  "status": "queued",
  "queuePosition": 1
}
```

二重到着の場合は既存の到着イベントを返す。

## 追跡停止

### `POST /v1/sessions/{sessionId}/stop`

Header: `Idempotency-Key: random-value`

```json
{
  "reason": "participant_request"
}
```

到着後はサーバー側からも停止する。停止済みセッションへ届いた位置点は保存しない。

## 状態取得

### `GET /v1/sessions/{sessionId}`

参加端末が再接続後に状態を復元するために使用する。位置点そのものは返さない。

## 運営API

- `GET /v1/operator/status`: 接続数、追跡中数、到着キュー、描画機状態
- `POST /v1/operator/arrivals/{arrivalId}/skip`: 到着演出をスキップ
- `POST /v1/operator/fallback`: 自律映像へ切り替え
- `GET /health/live`: プロセスの生存確認
- `GET /health/ready`: DBと必要サービスの準備確認

運営APIを参加者向け公開経路から分離する。

## 描画WebSocket

### `GET /v1/display/ws`

接続直後に現在状態のスナップショットを送信し、その後に差分イベントを配信する。

```json
{
  "eventId": "event-id",
  "type": "position.updated",
  "occurredAt": "2026-07-13T12:00:05Z",
  "data": {}
}
```

### `display.snapshot`

```json
{
  "type": "display.snapshot",
  "data": {
    "participants": [
      {
        "sessionId": "anonymous-id",
        "x": 10.2,
        "z": -4.8,
        "accuracyM": 12.4,
        "lastUpdatedAt": "2026-07-13T12:00:05Z"
      }
    ]
  }
}
```

### `position.updated`

表示用のローカル座標だけを送り、緯度経度は描画クライアントへ渡さない。

### `participant.arrived`

```json
{
  "type": "participant.arrived",
  "data": {
    "arrivalId": "arrival-id",
    "sessionId": "anonymous-id",
    "trajectory": [
      {"t": 0, "x": 2.1, "z": 3.4},
      {"t": 1, "x": 2.8, "z": 4.0}
    ],
    "replayDurationSeconds": 15
  }
}
```

### その他のイベント

- `tracking.stopped`
- `arrival.replay.started`
- `arrival.replay.completed`
- `system.fallback.changed`

## 再接続

- 描画クライアントは指数バックオフで再接続する
- 再接続時は必ず `display.snapshot` から復元する
- 参加端末は未送信位置点を最大50件まで保持し、古い順に再送する
- サーバーは `(session_id, recorded_at)` の重複を無視する

