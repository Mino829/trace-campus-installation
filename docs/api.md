# API・WebSocket仕様

## 共通方針

- Base path: `/v1`
- 形式: `application/json`
- 時刻: UTCのISO 8601
- 参加者認証: セッション作成時に発行するBearer token
- 運営・描画認証: 参加者用とは別の短期トークン。固定シークレットを使う場合もローテーションと失効手順を持つ
- 到着・停止操作は冪等にする
- 位置点はキャンパス範囲と時刻を検証してから保存する
- APIとイベントへスキーマバージョンを付け、破壊的変更では`/v2`へ移行する
- 参加者トークンをURL、ログ、描画イベントへ含めない
- CORSは本番Web Appのoriginだけを許可し、各APIにレート制限と最大bodyサイズを設定する

## 暫定入力制限

実地負荷試験で確定するまで、次を初期値とする。学内回線の共有NATを想定し、IP単独ではなくセッション、資格情報、全体負荷を組み合わせて制限する。

| 対象 | 暫定制限 |
|---|---:|
| セッション作成 | 1 IPあたり60回/分、全体10回/秒 |
| 位置送信 | 1セッションあたり12回/分、burst 5回 |
| 位置バッチ | 50点、JSON 64KB |
| 状態・到着・停止・削除 | 1セッションあたり30回/分 |
| 写真メタデータ作成 | 1セッション12件、1分6回、JSON 16KB |
| 写真アップロード | 1枚5MB、許可形式JPEG/WebP、長辺1920px以下 |
| 描画WebSocket受信 | 1接続あたり10メッセージ/秒、各256KB |
| 運営API | 1操作者あたり60回/分 |

制限超過は`429`と`Retry-After`を返し、正常な共有回線を遮断しないよう監視しながら調整する。

## 冪等性キー

到着、停止、削除、描画完了の`Idempotency-Key`は操作スコープ、キーのハッシュ、リクエスト内容のハッシュ、結果を24時間保持する。同じキーと同じ内容には以前の結果を返し、同じキーで異なる内容は`409 IDEMPOTENCY_CONFLICT`とする。

## 公開設定・同意文

### `GET /v1/public/config`

認証前に、受付状態、対応言語、有効な同意文、保存期間、位置取得間隔を返す。クライアントへ同意文を埋め込んだままにせず、サーバーが有効版を決める。

```json
{
  "registrationOpen": true,
  "consent": {
    "version": "2026-07-01",
    "language": "ja",
    "body": "..."
  },
  "positionIntervalSeconds": 5,
  "rawPositionRetentionHours": 24,
  "photo": {
    "enabled": true,
    "maxCount": 12,
    "maxUploadBytes": 5242880,
    "maxLongEdgePixels": 1920,
    "publicDisplay": false
  },
  "contact": "approved contact"
}
```

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
  "consentLanguage": "ja",
  "clientTime": "2026-07-13T12:00:00Z"
}
```

```json
{
  "sessionId": "anonymous-id",
  "participantToken": "participant-secret",
  "trackingStatus": "tracking",
  "replayStatus": "none",
  "displayAlias": {"color": "cyan", "symbol": "circle"},
  "expiresAt": "2026-07-13T18:00:00Z",
  "deleteAfter": "2026-07-14T18:00:00Z",
  "positionIntervalSeconds": 5
}
```

`participantToken`はレスポンス時だけ返し、サーバーにはハッシュを保存する。セッション中の複数APIで使用するためワンタイムではない。有効な同意文と異なる`consentVersion`は`409 CONSENT_VERSION_MISMATCH`として拒否する。

作成時の`deleteAfter`は最長追跡時間を含む削除上限である。到着または停止時に、承認済み保持期間を`stoppedAt`へ加えた時刻へ前倒しする。参加者画面には常に最新値を表示する。

## 位置送信

### `POST /v1/sessions/{sessionId}/positions`

通信断からの復旧に備え、1〜50件のバッチを受け付ける。

```json
{
  "points": [
    {
      "clientPointId": "client-generated-id",
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
  "latestRecordedAt": "2026-07-13T12:00:05Z",
  "rejectedItems": []
}
```

部分成功は`200`とし、拒否点には`clientPointId`と理由を返す。拒否理由は `OUTSIDE_GEOFENCE`、`LOW_ACCURACY`、`TOO_OLD`、`FUTURE_TIME`、`IMPOSSIBLE_JUMP`、`DUPLICATE` を想定する。認証失敗やセッション終了時はバッチ全体を拒否する。

## 到着

### `POST /v1/sessions/{sessionId}/arrival`

Header: `Idempotency-Key: random-value`

```json
{
  "checkpointToken": "signed-short-lived-proof",
  "clientTime": "2026-07-13T12:20:00Z"
}
```

```json
{
  "arrivalId": "arrival-id",
  "status": "queued",
  "queuePosition": 1,
  "estimatedWaitSeconds": 15,
  "displayAlias": {"color": "cyan", "symbol": "circle"}
}
```

到着証明は展示場入口の画面で定期更新するQRを基本とし、`checkpointId`、会場、発行時刻、有効期限、ランダム値を署名で保護する。期限切れ、改変、別会場の証明を拒否する。印刷した固定QRは撮影共有を防げないため、障害時に受付確認と組み合わせる代替手段とする。`source`は検証結果からサーバーが決め、参加端末の自己申告を信用しない。二重到着の場合は既存の到着イベントを返す。到着時刻はサーバー受信時刻を正とし、到着受付と同じトランザクションで追跡を停止する。

## 追跡停止

### `POST /v1/sessions/{sessionId}/stop`

Header: `Idempotency-Key: random-value`

```json
{
  "reason": "participant_request"
}
```

停止済みセッションへ届いた位置点は保存しない。停止は保存済みデータの削除を意味しない。

## データ削除

### `DELETE /v1/sessions/{sessionId}`

Header: `Idempotency-Key: random-value`

参加トークンで認証し、セッション、位置点、写真、派生特徴、再構成経路、到着イベント、参加トークンハッシュを削除する。非公開オブジェクトストレージ上の写真削除を確認してから`204 No Content`を返す。同じ`Idempotency-Key`による再送だけは、位置やセッション内容を持たない24時間以内の削除済み記録から同じ`204`を返す。それ以外の状態取得や送信は拒否する。インフラのセキュリティログとバックアップの削除期限は[プライバシー設計](privacy.md)に従う。

## 写真撮影・保存

ブラウザで向き補正、EXIF除去、縮小、再エンコードを行った後、非公開オブジェクトストレージへアップロードする。Backendは申告値だけを信用せず、保存後に形式、寸法、容量、EXIF不在、ハッシュを検証して`ready`へ変更する。

### `GET /v1/sessions/{sessionId}/photos`

再読込・再接続時にPhotoMomentのID、撮影順、`pending`、`ready`、`failed`状態を返す。画像URLは返さない。期限切れの`pending`には新しいアップロードURLを要求できるが、端末内に対応する加工済み写真がない場合は再アップロード不能として削除を案内する。

### `POST /v1/sessions/{sessionId}/photos`

Header: `Idempotency-Key: random-value`

```json
{
  "clientPhotoId": "client-generated-id",
  "capturedAt": "2026-07-13T12:05:00Z",
  "location": {
    "latitude": 35.000001,
    "longitude": 135.000001,
    "accuracyM": 18.2,
    "recordedAt": "2026-07-13T12:04:59Z"
  },
  "mimeType": "image/jpeg",
  "sizeBytes": 820000,
  "width": 1440,
  "height": 1920,
  "sha256": "hex-encoded-hash"
}
```

`location`は取得不能なら省略できる。成功時に短時間の一回限りアップロードURLを返す。

```json
{
  "photoMomentId": "photo-moment-id",
  "status": "pending",
  "uploadUrl": "short-lived-private-upload-url",
  "uploadExpiresAt": "2026-07-13T12:10:00Z"
}
```

クライアントは指定された`Content-Type`と申告したサイズの画像だけを`PUT uploadUrl`で送る。アップロードURLは対象オブジェクトへの書き込みだけを許可し、読み取り、上書き、一覧取得を許可しない。

### `POST /v1/sessions/{sessionId}/photos/{photoMomentId}/upload-url`

`pending`かつ有効なPhotoMomentに対してだけ、新しい短期アップロードURLを発行する。発行回数を制限し、`ready`、`failed`、`deleting`には発行しない。

### `POST /v1/sessions/{sessionId}/photos/{photoMomentId}/complete`

アップロード完了を通知する。Backendはオブジェクトを検証し、成功時は`ready`、不正形式やEXIF残存時は削除して`failed`にする。同じ`clientPhotoId`、ハッシュ、冪等性キーの再送は同じPhotoMomentを返す。

### `DELETE /v1/sessions/{sessionId}/photos/{photoMomentId}`

Header: `Idempotency-Key: random-value`

写真オブジェクト、PhotoMoment、派生特徴を削除する。前後のRouteSegmentは残った写真とGPSから再生成し、再生成できない場合は未確定区間にする。削除確認後に`204 No Content`を返す。

## 写真ストーリー

### `GET /v1/sessions/{sessionId}/memory`

本人用の写真、写真間の再構成経路、信頼度を時系列に返す。画像取得URLは参加認証後に発行する5分以内の署名付きURLとし、レスポンス・プロキシ・CDNログに参加トークンを含めない。

```json
{
  "status": "ready",
  "moments": [
    {
      "photoMomentId": "photo-moment-id",
      "capturedAt": "2026-07-13T12:05:00Z",
      "imageUrl": "short-lived-private-view-url",
      "locationConfidence": "high"
    }
  ],
  "segments": [
    {
      "routeSegmentId": "route-segment-id",
      "fromPhotoMomentId": "photo-1",
      "toPhotoMomentId": "photo-2",
      "confidence": "medium",
      "path": [
        {"t": 0, "x": 2.1, "z": 3.4},
        {"t": 1, "x": 2.8, "z": 4.0}
      ]
    }
  ],
  "deleteAfter": "2026-07-14T12:20:00Z"
}
```

公開スクリーンや描画機はこのAPIを使用できない。本人端末への保存は画像を明示操作でダウンロードし、サーバーの`deleteAfter`を延長しない。

## 状態取得

### `GET /v1/sessions/{sessionId}`

参加端末が再接続後に状態を復元するために使用する。位置点そのものは返さない。

```json
{
  "sessionId": "anonymous-id",
  "trackingStatus": "stopped",
  "replayStatus": "queued",
  "stopReason": "arrival",
  "displayAlias": {"color": "cyan", "symbol": "circle"},
  "photoSummary": {"ready": 4, "pending": 1, "failed": 0, "maxCount": 12},
  "queuePosition": 1,
  "estimatedWaitSeconds": 15,
  "expiresAt": "2026-07-13T18:00:00Z",
  "deleteAfter": "2026-07-14T12:20:00Z"
}
```

## 運営API

- `GET /v1/operator/status`: 接続数、追跡中数、到着キュー、描画機状態
- `POST /v1/operator/arrivals/{arrivalId}/skip`: 到着演出をスキップ
- `POST /v1/operator/fallback`: 自律映像へ切り替え
- `POST /v1/operator/registration/close`: 新規参加受付を停止
- `POST /v1/operator/arrivals/{arrivalId}/play`: 主描画機へ再生を指示
- `GET /health/live`: プロセスの生存確認
- `GET /health/ready`: DBと必要サービスの準備確認

運営APIを参加者向け公開経路から分離する。受付停止、skip、再生、fallbackの操作は操作者、時刻、対象、結果を監査ログへ残す。監査ログに生位置を含めない。

## 描画WebSocket

### `GET /v1/display/ws`

短期の描画トークンで認証する。接続直後に現在状態のスナップショットを送信し、その後に差分イベントを配信する。到着演出を進める主描画機は同時に一台だけリースを取得できる。待機描画機はスナップショットと通常表示を受信できるが、演出完了を確定できない。

```json
{
  "schemaVersion": 1,
  "eventId": "event-id",
  "type": "position.updated",
  "occurredAt": "2026-07-13T12:00:05Z",
  "data": {}
}
```

### `display.snapshot`

```json
{
  "schemaVersion": 1,
  "eventId": "event-id",
  "type": "display.snapshot",
  "occurredAt": "2026-07-13T12:00:05Z",
  "data": {
    "participants": [
      {
        "sessionId": "anonymous-id",
        "x": 10.2,
        "z": -4.8,
        "accuracyM": 12.4,
        "displayAlias": {"color": "cyan", "symbol": "circle"},
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
  "schemaVersion": 1,
  "eventId": "event-id",
  "type": "participant.arrived",
  "occurredAt": "2026-07-13T12:20:00Z",
  "data": {
    "arrivalId": "arrival-id",
    "sessionId": "anonymous-id",
    "displayAlias": {"color": "cyan", "symbol": "circle"},
    "trajectory": [
      {"t": 0, "x": 2.1, "z": 3.4},
      {"t": 1, "x": 2.8, "z": 4.0}
    ],
    "photoMoments": [
      {
        "photoMomentId": "photo-moment-id",
        "t": 0.4,
        "x": 2.5,
        "z": 3.8,
        "confidence": "medium",
        "abstractColor": "#72D6E8"
      }
    ],
    "replayDurationSeconds": 15
  }
}
```

### サーバーから描画機へのその他のイベント

- `tracking.stopped`
- `arrival.replay.requested`
- `photo.moment.created`: ID、抽象色、ローカル座標、信頼度だけを含む
- `photo.moment.deleted`
- `system.fallback.changed`

### 描画機からサーバーへのメッセージ

描画機はクライアント生成`eventId`、対象`arrivalId`、主描画機リースIDを付けて送る。同じ`eventId`は一度だけ適用する。

- `renderer.heartbeat`: fps、現在モード、主系リース、時刻
- `arrival.replay.started`: 再生開始の確定
- `arrival.replay.completed`: 正常完了
- `arrival.replay.failed`: 失敗理由。サーバーは再試行またはskipを判断する

WebSocketのping/pongまたはアプリheartbeatを10秒間隔で送り、30秒応答がなければ切断と判定する。最大メッセージサイズ、許容イベント頻度、未知のschemaVersionの扱いを実装設定で固定する。

## 再接続

- 描画クライアントは指数バックオフで再接続する
- 再接続時は必ず `display.snapshot` から復元する
- 参加端末は未送信位置点を最大50件まで保持し、古い順に再送する
- 5秒間隔で50件を超えると約4分10秒より古い点が失われることを参加画面の状態へ反映する
- サーバーは `(session_id, client_point_id)` の重複を無視する
- スナップショットには生成時点の`eventId`または連番を含め、それ以前の差分を二重適用しない

参加トークンはURLや永続的な`localStorage`へ置かず、同じタブの再読込に必要な範囲で`sessionStorage`へ保存する。未送信位置点と加工済み写真はIndexedDB等へ上限付きで保存し、送信成功、写真削除、セッション削除、失効時に消去する。写真は12枚・合計30MBを超えて端末へ保持しない。複数タブはBroadcastChannel等で既存セッションを検知し、二重の位置監視や写真送信を開始しない。

## HTTPステータス方針

| Status | 用途 |
|---:|---|
| 200 / 201 / 204 | 取得・作成・削除成功 |
| 400 | JSON、時刻、座標、必須項目が不正 |
| 401 / 403 | 資格情報なし、無効、権限不足 |
| 404 | 対象が存在しない。削除済みの詳細は明かさない |
| 409 | 状態競合、同意版不一致、主描画機リース競合 |
| 413 | bodyまたは点数上限超過 |
| 429 | レート制限 |
| 503 | 受付停止、DB利用不能、公開準備未完了 |
