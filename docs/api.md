# API・WebSocket仕様

この文書はIssue #11で合意した実装目標の契約を含む設計仕様であり、全APIが現行Backendで提供済みであることを示さない。到着証明の実現方法と大学のプライバシー審査（#9）は未完了。

## 共通方針

- Base path: `/v1`
- 形式: `application/json`
- 時刻: UTCのISO 8601
- 参加者認証: セッション作成時に発行するBearer token
- 運営・描画認証: 参加者用とは別の短期トークン。固定シークレットを使う場合もローテーションと失効手順を持つ
- 到着・停止操作は冪等にする
- 写真用`MemorySession`とリアルタイム用`LivePresenceSession`を別ID・別トークン・別同意版で扱う
- 通常GPSは量子化済みの最新表示位置だけを30〜60秒保持し、通信失敗時に蓄積・後送しない
- 写真撮影時GPSだけは写真と一緒に上限付きで再送できる
- APIとイベントへスキーマバージョンを付け、破壊的変更では`/v2`へ移行する
- 参加者トークンをURL、ログ、描画イベントへ含めない
- CORSは本番Web Appのoriginだけを許可し、各APIにレート制限と最大bodyサイズを設定する

## 暫定入力制限

実地負荷試験で確定するまで、次を初期値とする。学内回線の共有NATを想定し、IP単独ではなくセッション、資格情報、全体負荷を組み合わせて制限する。

| 対象 | 暫定制限 |
|---|---:|
| セッション作成 | 1 IPあたり60回/分、全体10回/秒 |
| 位置送信 | 1セッションあたり12回/分、burst 5回 |
| 位置リクエスト | 最新1点、JSON 16KB |
| 状態・到着・停止・削除 | 1セッションあたり30回/分 |
| 写真メタデータ作成 | 1セッション12件、1分6回、JSON 16KB |
| 写真アップロード | 1枚5MB、許可形式JPEG/WebP、長辺1920px以下 |
| Push購読・設定 | 1セッションあたり10回/分、JSON 16KB |
| 描画WebSocket受信 | 1接続あたり10メッセージ/秒、各256KB |
| 運営API | 1操作者あたり60回/分 |

制限超過は`429`と`Retry-After`を返し、正常な共有回線を遮断しないよう監視しながら調整する。

## 冪等性キー

到着、停止、削除、描画完了の`Idempotency-Key`は操作スコープ、キーのハッシュ、リクエスト内容のハッシュ、結果を24時間保持する。同じキーで異なる内容は`409 IDEMPOTENCY_CONFLICT`とする。削除以外は同じキーと内容に以前の結果を返す。両セッション削除の`pending`結果は中間状態であり、同じキーと両資格情報で再送すると未完了側を再実行し、進捗結果を更新する。完了した側は再作成しない。

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
  "livePositionTtlSeconds": 60,
  "photoLocationRetentionHours": 24,
  "spatial": {
    "coordinateVersion": "campus-2026-01",
    "campusGraphVersion": "graph-2026-01",
    "threeDMapVersion": "map-2026-01"
  },
  "photo": {
    "enabled": true,
    "maxCount": 12,
    "maxUploadBytes": 5242880,
    "maxLongEdgePixels": 1920,
    "publicDisplay": false
  },
  "photoMarker": {
    "ttlMinutes": 30,
    "hotspotMinimumCount": 3,
    "quantizationMeters": 10,
    "sparseMode": "momentary"
  },
  "notification": {
    "enabled": true,
    "defaultIntervalMinutes": 30,
    "allowedIntervalsMinutes": [15, 30],
    "exactDeliveryGuaranteed": false
  },
  "contact": "approved contact"
}
```

現行Go APIの公開設定は位置参加の同意版・会場内同意版・位置TTL・空の写真機能情報までを返す。上記の`photoMarker`と`notification`は各機能が実装された後の契約例であり、現行レスポンスには含まれない。現行の`photo.enabled`は`false`である。参加用の独立したMemory/Liveの期限は`POST /v1/experiences`と本人用GETの各セッション内で返す。未提供機能の値を固定で返して「有効」と誤認させない。

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

## 体験・セッション作成

### `POST /v1/experiences`

```json
{
  "memoryConsentVersion": "2026-memory-01",
  "livePresenceConsentVersion": "2026-live-01",
  "enableLivePresence": true,
  "consentLanguage": "ja",
  "clientTime": "2026-07-13T12:00:00Z"
}
```

```json
{
  "experienceId": "exp_example",
  "experienceLinkToken": "short-lived-link-secret",
  "memorySession": {
    "sessionId": "mem_example",
    "participantToken": "memory-participant-secret",
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  },
  "livePresenceSession": {
    "sessionId": "live_example",
    "participantToken": "live-participant-secret",
    "trackingStatus": "tracking",
    "presencePhase": "approaching",
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  },
  "displayAlias": {"color": "cyan", "symbol": "circle"},
  "positionIntervalSeconds": 5
}
```

`enableLivePresence: false`なら`livePresenceSession`は省略する。各`participantToken`はレスポンス時だけ返し、サーバーには別々のハッシュを保存する。一方のトークンで他方の写真・位置を操作できない。`experienceLinkToken`は色・記号と到着連携だけに使う短期トークンで、写真、生GPS、経路を返さない。有効な同意文と異なる版は`409 CONSENT_VERSION_MISMATCH`として拒否する。

両セッションには別々の`expiresAt`と`deleteAfter`を返す。例示日時と保持期間は大学のプライバシー審査（#9）による決定値ではない。`expiresAt`以後は新規位置送信などの追跡操作を拒否するが、本人の状態取得・閲覧と削除は各セッションの`deleteAfter`まで利用できる。`deleteAfter`到達時に保存データを期限削除する。

## 位置送信

### `POST /v1/live-presence-sessions/{liveSessionId}/position`

最後に取得した現在位置1件だけを受け付ける。失敗した通常GPSは端末へ保持せず、復旧後の最新位置から再開する。

```json
{
  "point": {
    "latitude": 35.000001,
    "longitude": 135.000001,
    "accuracyM": 12.4,
    "recordedAt": "2026-07-13T12:00:05Z"
  }
}
```

```json
{
  "accepted": true,
  "latestRecordedAt": "2026-07-13T12:00:05Z",
  "expiresAt": "2026-07-13T12:01:05Z"
}
```

検証後、生の緯度経度は永続化せず、量子化済みローカル座標の最新値へ置き換える。拒否理由は`OUTSIDE_GEOFENCE`、`LOW_ACCURACY`、`TOO_OLD`、`FUTURE_TIME`、`IMPOSSIBLE_JUMP`を想定する。

通常GPSの最新表示位置は単一Go Backendプロセスのメモリで30〜60秒TTLとし、再起動で失われる。PostgreSQLの`live_positions`テーブルを使わない。複数インスタンス化はRedis互換TTLストアへの移行後に行う。

## 到着

### `POST /v1/memory-sessions/{memorySessionId}/arrival`

Header: `Idempotency-Key: random-value`

```json
{
  "checkpointToken": "signed-short-lived-proof",
  "experienceLinkToken": "short-lived-link-secret",
  "clientTime": "2026-07-13T12:20:00Z"
}
```

```json
{
  "arrivalId": "arrival-id",
  "status": "queued",
  "queuePosition": 1,
  "estimatedWaitSeconds": 15,
  "livePresencePhase": "arrived",
  "onSiteChoiceRequired": true,
  "displayAlias": {"color": "cyan", "symbol": "circle"}
}
```

到着証明として展示場入口で定期更新する署名付きQR（`checkpointId`、会場、発行時刻、有効期限、ランダム値）を検討しているが、Issue #11の到着機構は未確定で、このリクエスト・レスポンスは提案であって提供済みAPIではない。採用時は期限切れ、改変、別会場の証明を拒否し、MemorySessionへ到着イベントを作り、有効なExperienceLinkがある場合だけ対応するLivePresenceSessionを`arrived`へ変更する。到着時に撮影通知を停止・削除する設計だが、通知自体も未実装なら利用可能と案内しない。通常GPSは本人が継続可否を選ぶまで自動停止しない。

### `POST /v1/live-presence-sessions/{liveSessionId}/on-site`

```json
{
  "continue": true,
  "consentVersion": "2026-on-site-01"
}
```

`continue: true`は既に`arrived`であるLivePresenceSessionだけが要求でき、本人が用途・表示粒度を理解して明示的に追加同意した版を検証してから`on_site`へ移行する。`approaching`から直接移行しない。`continue: false`では追跡を停止し、最新表示位置を消去して`ended`へ移行する。会場内GPSが不十分な場合、サーバー設定により`point`、`area`、`interaction_only`の表示粒度を返す。到着証明の具体的な仕組みは未確定であり、到着を経ずに会場内表示を許可しない。

## 追跡停止

### `POST /v1/live-presence-sessions/{liveSessionId}/stop`

Header: `Idempotency-Key: random-value`

```json
{
  "reason": "participant_request"
}
```

停止済みLivePresenceSessionへ届いた通常GPSは拒否し、最新表示位置を消去する。停止はMemorySessionの写真・思い出データの削除を意味しない。

## データ削除

### `DELETE /v1/memory-sessions/{memorySessionId}`

Header: `Idempotency-Key: random-value`

Memory用トークンで認証し、写真、撮影時GPS、派生特徴、再構成経路、到着イベント、Push購読、MemorySessionのトークンハッシュを削除する。写真ピンを消去し、フォトスポットへの寄与を減算する。LivePresenceSessionは削除しないが、ExperienceLinkは削除する。

### `DELETE /v1/live-presence-sessions/{liveSessionId}`

Header: `Idempotency-Key: random-value`

Live用トークンで認証し、最新表示位置、LivePresenceSession、トークンハッシュ、ExperienceLinkを削除する。MemorySessionは削除しない。

### `DELETE /v1/experiences/{experienceId}`

Headers: `Authorization: Bearer <memory-participant-token>`、`X-Trace-Live-Token: <live-participant-token>`、`Idempotency-Key: <same-key-for-retry>`。リクエストbodyなし。両方のセッションとトークンが揃っている場合にだけ使用する。最初からLiveがない場合、または片側を個別DELETEで削除済みの場合は残る側の個別DELETEを使用する。

```json
{
  "memory": "pending",
  "live": "deleted",
  "complete": false
}
```

成功または部分成功をHTTP `200`で返す。Live側を先に削除・削除記録と一緒に確定し、続いてMemory側を別トランザクションで削除する。最初のLive削除が失敗した場合はエラーとして返し、同じキーで再送する。Live削除後にMemory削除が失敗した場合は`memory: "pending"`、`live: "deleted"`、`complete: false`を返す。一方の失敗で完了側を復活させない。`pending`や通信失敗の場合、端末は両方の資格情報と同じ冪等性キーを保持して同じリクエストを再送する。完了済み側は24時間の削除記録から結果を再現し、未完了側だけを再試行する。両方が`deleted`の場合だけ`complete: true`を返す。片方のトークンだけで両方を削除しない。写真オブジェクト削除の確認とインフラログ・バックアップの扱いは[プライバシー設計](privacy.md)に従う。

## 写真撮影・保存

ブラウザで向き補正、EXIF除去、縮小、再エンコードを行った後、非公開オブジェクトストレージへアップロードする。Backendは申告値だけを信用せず、保存後に形式、寸法、容量、EXIF不在、ハッシュを検証して`ready`へ変更する。

Backend Issue #6の以下の記述は対応PR適用後の契約であり、mainへの反映・配備済みを示さない。非公開Cloud Storageを明示設定した場合だけ新規写真を有効にし、既定では無効とする。大学承認・実データ利用の許可を意味しない。経路・写真ピン・フォトスポットは別Issueであり、イベントや経路の記述は引き続き実装目標を含む。

### `GET /v1/memory-sessions/{memorySessionId}/photos`

再読込・再接続時にPhotoMomentのID、`clientPhotoId`、撮影時刻、受付順の`sequenceNo`、`pending`、`ready`、`failed`、`deleting`状態を撮影時刻順に返す。画像URLは返さない。期限切れの`pending`には新しいアップロードURLを要求できるが、端末内に対応する加工済み写真がない場合は再アップロード不能として削除を案内する。

### `POST /v1/memory-sessions/{memorySessionId}/photos`

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
  "uploadHeaders": {
    "Content-Type": "image/jpeg",
    "Content-Length": "820000",
    "x-goog-if-generation-match": "0",
    "Cache-Control": "private, no-store, max-age=0"
  },
  "uploadExpiresAt": "2026-07-13T12:10:00Z"
}
```

クライアントは`uploadHeaders`を使用して申告したサイズの画像だけを`PUT uploadUrl`で送る。ブラウザの`Content-Length`はBlobサイズから自動設定し、その他の返却ヘッダーは変更しない。URLの期限は5分とMemorySessionの`expiresAt`の早い方までとし、新規作成専用のgeneration条件で上書きを禁止する。読み取り・一覧取得には使えない。同じ登録要求は同じPhotoMomentへ集約し、内容が異なる再送は409、削除済み写真の同じ登録再送は404とする。登録再送時にURL期限が切れていれば`uploadUrl`を省略し、次の再発行APIを使用する。

### `POST /v1/memory-sessions/{memorySessionId}/photos/{photoMomentId}/upload-url`

`pending`かつ有効なPhotoMomentに対してだけ、新しい短期アップロードURLを発行する。初回を含む発行回数は5回までとし、上限は409、`ready`、`failed`、`deleting`には発行しない。URL再発行と`complete`のbodyは空とする。

### `POST /v1/memory-sessions/{memorySessionId}/photos/{photoMomentId}/complete`

アップロード完了を通知する。Backendはオブジェクトを検証し、成功時は`ready`、不正形式やEXIF残存時は削除して`failed`にする。同じ`clientPhotoId`、ハッシュ、冪等性キーの再送は同じPhotoMomentを返す。`ready`確定後に量子化済みの`photo.marker.created`を配信し、3件以上の集計だけ`photo.hotspot.updated`を配信する。

Backend #6の保存処理では、不正画像の消去に失敗した場合は`deleting`を永続化し、再試行して消去できてから`failed`へ進める。未アップロードの`complete`は409、成功・再送は同じ`ready`または`failed`状態を200で返す。写真ピン・フォトスポットのイベント配信はBackend #10/#9の実装後とし、写真保存完了だけで配信済みとは扱わない。

### `DELETE /v1/memory-sessions/{memorySessionId}/photos/{photoMomentId}`

Header: `Idempotency-Key: random-value`

写真オブジェクト、PhotoMoment、派生特徴を削除する。前後のRouteSegmentは残った写真撮影時GPSから再生成し、再生成できない場合は未確定区間にする。個別ピンを消し、対応するフォトスポット集計を減算する。3件未満になれば`photo.hotspot.hidden`を配信する。

写真削除は消去成功後204、消去失敗時は503とし、同じ冪等性キーで再試行できる。先に`deleting`を確定して新規閲覧を止め、オブジェクト消去前にPhotoMomentやMemorySessionを失わない。未失効のPUTが画像を復活させないよう、アップロード先はゼロバイトで占有し、最後のURL期限後にランダムkey・期限だけのoutboxから消去する。画像・撮影時GPS・PhotoMoment参照をこのoutboxへ残さない。現段階では派生特徴・別サムネイル・RouteSegment・ピンを生成しないため、それらの再生成・減算は各機能実装時の契約である。

## 撮影通知

### `PUT /v1/memory-sessions/{memorySessionId}/push-subscription`

```json
{
  "subscription": {
    "endpoint": "https://push-provider.example/subscription",
    "keys": {"p256dh": "base64url-key", "auth": "base64url-key"}
  },
  "intervalMinutes": 30
}
```

通知許可は説明後の参加者操作内で要求する。endpointと鍵は暗号化保存し、通常ログ、描画、運営画面へ出さない。許可間隔は公開設定の値だけを受け付ける。

### `PATCH /v1/memory-sessions/{memorySessionId}/push-subscription`

`intervalMinutes`の変更と`active`・`paused`の切り替えを行う。写真撮影後は`nextSendAt`を延期する。正確な分刻みの配信を保証しない。

### `DELETE /v1/memory-sessions/{memorySessionId}/push-subscription`

購読と未送信予定を削除する。到着、参加停止、期限切れ、Push providerの失効応答でも同じ処理を使う。通知payloadには位置、写真、セッションIDを含めず、一般的な文面と撮影画面への相対URLだけを含める。

## 写真ストーリー

### `GET /v1/memory-sessions/{memorySessionId}/memory`

本人用の写真、写真間の再構成経路、信頼度を時系列に返す。画像取得URLは参加認証後に発行する5分以内の署名付きURLとし、レスポンス・プロキシ・CDNログに参加トークンを含めない。

Backend #6の本人memoryは`ready`写真だけを返し、URL期限は5分とMemorySessionの`deleteAfter`の早い方までとする。`locationConfidence`は撮影時GPSなし/精度100m超を`low`、30m超〜100mを`medium`、30m以下を`high`とする暫定分類で、経路の信頼度とは異なる。Backend #7の経路生成まで`segments`は空配列であり、下の非空例を現行提供済みとは扱わない。

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

### `GET /v1/experiences/{experienceId}`

参加端末が再接続後に状態を復元するために使用する。従来どおり`Authorization: Bearer <memory-participant-token>`で本人認証し、MemorySessionの`deleteAfter`まで取得できる。ExperienceLinkで結ばれたLivePresenceSessionが残る場合、その状態・期限と共有aliasだけを返し、Live用トークンも位置点そのものも返さない。MemorySessionが先に削除された場合、このGETは使えないが、Live本人用トークンでの個別削除はLive自身の`deleteAfter`まで可能である。両トークンを必須とするのは両セッションDELETEだけである。

```json
{
  "memorySession": {
    "sessionId": "mem_example",
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  },
  "livePresenceSession": {
    "sessionId": "live_example",
    "trackingStatus": "tracking",
    "presencePhase": "arrived",
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  },
  "replayStatus": "queued",
  "stopReason": null,
  "displayAlias": {"color": "cyan", "symbol": "circle"},
  "photoSummary": {"ready": 4, "pending": 1, "failed": 0, "maxCount": 12},
  "queuePosition": 1,
  "estimatedWaitSeconds": 15
}
```

Backend #6の写真実装では、`photoSummary`の`ready`/`pending`/`failed`をMemorySessionの実件数から算出し、`maxCount`は12とする。`deleting`は一覧から確認し、保持中の上限には含める。LivePresenceSessionが存在しない場合は`livePresenceSession`を省略する。写真機能の有効化・配備は別操作であり、通知・到着キュー等の未実装項目はこの例に登場するだけで提供済みとみなさない。

## 運営API

- `GET /v1/operator/status`: 接続数、追跡中数、到着キュー、描画機状態
- `POST /v1/operator/arrivals/{arrivalId}/skip`: 到着演出をスキップ
- `POST /v1/operator/fallback`: 自律映像へ切り替え
- `POST /v1/operator/registration/close`: 新規参加受付を停止
- `POST /v1/operator/arrivals/{arrivalId}/play`: 主描画機へ再生を指示
- `POST /v1/operator/photo-markers/stop`: 個別写真ピンを停止
- `POST /v1/operator/photo-hotspots/stop`: フォトスポット表示を停止
- `POST /v1/operator/notifications/stop`: 全撮影通知を停止
- `GET /health/live`: プロセスの生存確認
- `GET /health/ready`: DBと必要サービスの準備確認

運営APIを参加者向け公開経路から分離する。受付停止、skip、再生、fallbackの操作は操作者、時刻、対象、結果を監査ログへ残す。監査ログに生位置を含めない。

## 描画WebSocket

### `GET /v1/display/ws`

短期の描画トークンで認証する。接続直後に現在状態のスナップショットを送信し、その後に差分イベントを配信する。到着演出を進める主描画機は同時に一台だけリースを取得できる。待機描画機はスナップショットと通常表示を受信できるが、演出完了を確定できない。

```json
{
  "schemaVersion": 1,
  "coordinateVersion": "campus-2026-01",
  "campusGraphVersion": "graph-2026-01",
  "threeDMapVersion": "map-2026-01",
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
  "coordinateVersion": "campus-2026-01",
  "campusGraphVersion": "graph-2026-01",
  "threeDMapVersion": "map-2026-01",
  "eventId": "event-id",
  "type": "display.snapshot",
  "occurredAt": "2026-07-13T12:00:05Z",
  "data": {
    "participants": [
      {
        "livePresenceId": "display-scoped-id",
        "x": 10.2,
        "z": -4.8,
        "accuracyM": 12.4,
        "displayAlias": {"color": "cyan", "symbol": "circle"},
        "lastUpdatedAt": "2026-07-13T12:00:05Z"
      }
    ],
    "photoMarkers": [
      {"markerId": "short-lived-marker-id", "x": 12.0, "z": -3.0, "expiresAt": "2026-07-13T12:35:00Z"}
    ],
    "photoHotspots": [
      {"spatialKey": "node-main-plaza", "x": 15.0, "z": -5.0, "photoCountBand": "3-9"}
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
- `photo.marker.created`: 量子化済み座標、抽象色、TTLだけを含む
- `photo.marker.expired` / `photo.marker.deleted`
- `photo.hotspot.updated`: 3件以上の集計、量子化済み座標、件数帯だけを含む
- `photo.hotspot.hidden`: 閾値未満になった集計を非表示にする
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
- 参加端末は未送信の通常GPSを保持せず、失敗した点を破棄する
- 復旧後は新しく取得した最新位置から送信を再開する
- スナップショットには生成時点の`eventId`または連番を含め、それ以前の差分を二重適用しない

参加トークンはURLや永続的な`localStorage`へ置かず、同じタブの再読込に必要な範囲で`sessionStorage`へ保存する。加工済み写真と写真撮影時GPSだけはIndexedDB等へ上限付きで保存し、送信成功、写真削除、セッション削除、失効時に消去する。通常GPSは保存しない。写真は12枚・合計30MBを超えて端末へ保持しない。複数タブはBroadcastChannel等で既存セッションを検知し、二重の位置監視や写真送信を開始しない。

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
