# API・WebSocket仕様

この文書は実装目標の契約を含む設計仕様であり、全APIが現行Backendで提供済みであることを示さない。2026-10-26にGPS不使用・写真地点自己申告仕様へ移行した（Issue #23）。位置情報（GPS）は一切使わず、写真地点は参加者が3Dマップで選んだ`checkpointId`のみで表す。到着証明は展示ブースで投影する署名・期限付きQRとする（2026-10-26決定、Issue #11）。大学のプライバシー審査（#9）は未完了。

## 共通方針

- Base path: `/v1`
- 形式: `application/json`
- 時刻: UTCのISO 8601
- 参加者認証: セッション作成時に発行するBearer token
- 運営・描画認証: 参加者用とは別の短期トークン。固定シークレットを使う場合もローテーションと失効手順を持つ
- 到着・削除操作は冪等にする
- GPS（Geolocation API、`watchPosition`、撮影時位置取得）は使用しない。写真地点は参加者が3Dマップで選択した`checkpointId`のみで、座標・対応ノードIDの解決はBackendが行う。クライアントpayloadに座標・ノードID・グラフ版を追加しない（2026-10-26決定、#23）
- セッションは写真用`MemorySession`単独とする。`LivePresenceSession`の新規発行は行わない（2026-10-26決定、#23）
- `ExperienceLink`、色・記号、`display_alias`は廃止した（2026-10-26決定、#23）
- APIとイベントへスキーマバージョンを付け、破壊的変更では`/v2`へ移行する
- 参加者トークンをURL、ログ、描画イベントへ含めない
- CORSは本番Web Appのoriginだけを許可し、各APIにレート制限と最大bodyサイズを設定する

## 暫定入力制限

実地負荷試験で確定するまで、次を初期値とする。学内回線の共有NATを想定し、IP単独ではなくセッション、資格情報、全体負荷を組み合わせて制限する。

| 対象 | 暫定制限 |
|---|---:|
| セッション作成 | 1 IPあたり60回/分、全体10回/秒 |
| 到着・削除 | 1セッションあたり30回/分 |
| 写真メタデータ作成 | 1セッション12件、1分6回、JSON 16KB |
| 写真アップロード | 1枚5MB、許可形式JPEG/WebP、長辺1920px以下 |
| Push購読・設定 | 1セッションあたり10回/分、JSON 16KB |
| 描画WebSocket受信 | 1接続あたり10メッセージ/秒、各256KB |
| 運営API | 1操作者あたり60回/分 |

制限超過は`429`と`Retry-After`を返し、正常な共有回線を遮断しないよう監視しながら調整する。

## 冪等性キー

到着、削除の`Idempotency-Key`は操作スコープ、キーのハッシュ、リクエスト内容のハッシュ、結果を24時間保持する。同じキーで異なる内容は`409 IDEMPOTENCY_CONFLICT`とする。削除以外は同じキーと内容に以前の結果を返す。削除の冪等性記録は写真・チェックポイント地点・経路と同一の保持期間で一緒に消す（2026-10-26決定、#23）。

## 公開設定・同意文

### `GET /v1/public/config`

認証前に、受付状態、対応言語、有効な同意文、保存期間、写真機能情報を返す。クライアントへ同意文を埋め込んだままにせず、サーバーが有効版を決める。

```json
{
  "registrationOpen": true,
  "consent": {
    "version": "2026-memory-photo-no-gps-draft-1",
    "language": "ja",
    "body": "..."
  },
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
  "notification": {
    "enabled": true,
    "intervalMinutes": 30,
    "exactDeliveryGuaranteed": false
  },
  "contact": "approved contact"
}
```

同意文は版`2026-memory-photo-no-gps-draft-1`（jaのみ。enは大学審査後に追加）で、位置情報の項目を含まない。参加条件はカメラ許可のみで、到着QRの読み取りに使う（2026-10-26決定、#23）。`notification`は撮影リマインダー30分間隔のみを表す。現行Go APIの公開設定が位置参加・会場内同意・位置TTL等を返す形から、上記の形へ移行する契約であり、現行レスポンスが上記と同一であることを示さない。

## エラー形式

```json
{
  "error": {
    "code": "INVALID_CHECKPOINT",
    "message": "checkpointId is not enabled in the current graph version",
    "requestId": "request-id"
  }
}
```

内部情報や生の位置情報をエラーメッセージへ含めない。

## 体験・セッション作成

### `POST /v1/experiences`

```json
{
  "memoryConsentVersion": "2026-memory-photo-no-gps-draft-1",
  "consentLanguage": "ja",
  "clientTime": "2026-07-13T12:00:00Z"
}
```

```json
{
  "experienceId": "exp_example",
  "memorySession": {
    "sessionId": "mem_example",
    "participantToken": "memory-participant-secret",
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  }
}
```

単一の`MemorySession`を発行する。`enableLivePresence`、`livePresenceConsentVersion`、`experienceLinkToken`、レスポンスの`livePresenceSession`は廃止した（2026-10-26決定、#23）。`participantToken`はレスポンス時だけ返し、サーバーにはハッシュを保存する。有効な同意文と異なる版は`409 CONSENT_VERSION_MISMATCH`として拒否する。

`expiresAt`は開発側既定で失効まで6時間、`deleteAfter`は失効後24時間とする。例示日時と保持期間は大学のプライバシー審査（#9）による決定値ではない。`expiresAt`以後は写真登録などの新規操作を拒否するが、本人の状態取得・閲覧と削除は`deleteAfter`まで利用できる。`deleteAfter`到達時に写真・チェックポイント地点・経路・削除冪等性記録を一緒に期限削除する。

## 位置送信・会場内表示・追跡停止（廃止契約）

`POST /v1/live-presence-sessions/{liveSessionId}/position`（位置送信）、`POST /v1/live-presence-sessions/{liveSessionId}/on-site`（会場内`on_site`への移行）、`POST /v1/live-presence-sessions/{liveSessionId}/stop`（追跡停止）は廃止した（2026-10-26、#23）。位置送信・on-siteエンドポイントは完全に廃止し、旧Sessionに対しても受け付けない。停止エンドポイントは廃止契約であり、既存の旧Sessionに対する停止操作だけを残す。`LivePresenceSession`は新規発行せず、既存の旧Sessionは本人資格での状態照会・停止・削除と各`deleteAfter`までの期限清掃のみが有効である。旧GPSデータは前倒し削除せず、各`deleteAfter`まで保持する。旧描画位置イベント・写真の緯度経度列は読み取り契約のみ（TRACE_B#30）。

## 到着

### `POST /v1/memory-sessions/{memorySessionId}/arrival`

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
  "status": "queued"
}
```

到着は、展示ブースのプロジェクター映像に表示し定期更新する署名・期限付きQR（`checkpointId`、会場、発行時刻、有効期限、ランダム値）のBackend検証でのみ成立する（2026-10-26決定、#11）。参加者は展示ブースの列に並び、自分の番でWeb App内のカメラからこのQRを読み取る。Web App内で読み取るため、到着を読み取った端末のMemorySessionへ結び付けられる。待ち順は人の列で扱い、参加端末へ順番、予想待ち時間、色・記号を返さない。

期限切れ、改変、別会場の証明を拒否し、MemorySessionへArrivalEventを作成し、`status`を`arrived`へ変更して撮影通知を停止する。写真0枚・1枚でも到着は成立し、再生は簡易表示（スポット名や写真の有無の案内）に縮小する（2026-10-26決定、#23）。到着済みセッションへの再送は既存のArrivalEventを返す。

## データ削除

### `DELETE /v1/memory-sessions/{memorySessionId}`

Header: `Idempotency-Key: random-value`

Memory用トークンで認証し、写真、チェックポイント地点、経路、到着イベント、Push購読、削除冪等性記録、MemorySessionのトークンハッシュを削除する。削除は消去成功後204、消去失敗時は503とし、同じ冪等性キーで再試行できる。写真・チェックポイント地点・経路・削除冪等性記録は同一期間で一緒に消す（2026-10-26決定、#23）。写真オブジェクト削除の確認とインフラログ・バックアップの扱いは[プライバシー設計](privacy.md)に従う。

### `DELETE /v1/live-presence-sessions/{liveSessionId}`（廃止契約）

Header: `Idempotency-Key: random-value`

新規発行は廃止（2026-10-26、#23）。旧Session向けの読み取り/削除契約として残す。Live用トークンで認証し、旧GPSデータ、旧LivePresenceSession、トークンハッシュを削除する。旧Sessionの削除は各`deleteAfter`まで可能で、MemorySessionは削除しない。

### `DELETE /v1/experiences/{experienceId}`（廃止契約）

Headers: `Authorization: Bearer <memory-participant-token>`、`X-Trace-Live-Token: <live-participant-token>`、`Idempotency-Key: <same-key-for-retry>`。リクエストbodyなし。

新規発行は廃止したため、単一資格情報の削除は上記の`DELETE /v1/memory-sessions/{memorySessionId}`を使う（2026-10-26決定、#23）。旧両資格情報DELETEは旧Session向け読み取り/削除契約として残す。両方のセッションとトークンが揃っている場合にだけ使用する。最初からLiveがない場合、または片側を個別DELETEで削除済みの場合は残る側の個別DELETEを使用する。

```json
{
  "memory": "pending",
  "live": "deleted",
  "complete": false
}
```

成功または部分成功をHTTP `200`で返す。Live側を先に削除・削除記録と一緒に確定し、続いてMemory側を別トランザクションで削除する。Live削除後にMemory削除が失敗した場合は`memory: "pending"`、`live: "deleted"`、`complete: false`を返す。一方の失敗で完了側を復活させない。`pending`や通信失敗の場合、端末は両方の資格情報と同じ冪等性キーを保持して同じリクエストを再送する。完了済み側は24時間の削除記録から結果を再現し、未完了側だけを再試行する。両方が`deleted`の場合だけ`complete: true`を返す。

## 写真撮影・保存

ブラウザで向き補正、EXIF除去、縮小、再エンコードを行った後、非公開オブジェクトストレージへアップロードする。Backendは申告値だけを信用せず、保存後に形式、寸法、容量、EXIF不在、ハッシュを検証して`ready`へ変更する。撮影時GPSは取得しない。写真地点は参加者が3Dマップで選んだ`checkpointId`のみである（2026-10-26決定、#23）。

Backend Issue #6の以下の記述は対応PR適用後の契約であり、mainへの反映・配備済みを示さない。非公開Cloud Storageを明示設定した場合だけ新規写真を有効にし、既定では無効とする。大学承認・実データ利用の許可を意味しない。経路は別Issueであり、経路の記述は引き続き実装目標を含む。

### `GET /v1/memory-sessions/{memorySessionId}/photos`

再読込・再接続時にPhotoMomentのID、`clientPhotoId`、`checkpointId`、撮影時刻、受付順の`sequenceNo`、`pending`、`ready`、`failed`、`deleting`状態を撮影時刻順に返す。画像URLは返さない。期限切れの`pending`には新しいアップロードURLを要求できるが、端末内に対応する加工済み写真がない場合は再アップロード不能として削除を案内する。

### `POST /v1/memory-sessions/{memorySessionId}/photos`

Header: `Idempotency-Key: random-value`

```json
{
  "clientPhotoId": "client-generated-id",
  "checkpointId": "checkpoint-main-plaza",
  "capturedAt": "2026-07-13T12:05:00Z",
  "mimeType": "image/jpeg",
  "sizeBytes": 820000,
  "width": 1440,
  "height": 1920,
  "sha256": "hex-encoded-hash"
}
```

`checkpointId`は必須である。参加者が3Dマップで地点を選ばずに写真を送ることはできず、未選択・無効な地点は`400 INVALID_CHECKPOINT`で拒否する（現行実装準拠、TRACE_F#45）。`checkpointId`が有効であるかは、セッション作成時に返した`campusGraphVersion`のチェックポイント一覧で検証する。成功時に短時間の一回限りアップロードURLを返す。

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

アップロード完了を通知する。Backendはオブジェクトを検証し、成功時は`ready`、不正形式やEXIF残存時は削除して`failed`にする。同じ`clientPhotoId`、ハッシュ、冪等性キーの再送は同じPhotoMomentを返す。写真ピン・フォトスポットの集計は廃止したため、配信イベントは行わない（2026-10-26決定、#23）。

Backend #6の保存処理では、不正画像の消去に失敗した場合は`deleting`を永続化し、再試行して消去できてから`failed`へ進める。未アップロードの`complete`は409、成功・再送は同じ`ready`または`failed`状態を200で返す。

### `DELETE /v1/memory-sessions/{memorySessionId}/photos/{photoMomentId}`

Header: `Idempotency-Key: random-value`

写真オブジェクト、PhotoMomentを削除する。前後のRouteSegmentは残った写真のチェックポイント地点から再生成し、再生成できない場合は未確定区間にする。

写真削除は消去成功後204、消去失敗時は503とし、同じ冪等性キーで再試行できる。先に`deleting`を確定して新規閲覧を止め、オブジェクト消去前にPhotoMomentやMemorySessionを失わない。未失効のPUTが画像を復活させないよう、アップロード先はゼロバイトで占有し、最後のURL期限後にランダムkey・期限だけのoutboxから消去する。画像・PhotoMoment参照をこのoutboxへ残さない。派生特徴・別サムネイルを生成しないため、それらの再生成は各機能実装時の契約である。

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

継続する通知は30分ごとの撮影リマインダーのみである。精度テスト用10分通知は廃止した（2026-10-26決定、#23）。通知なしでも参加できる。endpointと鍵は暗号化保存し、通常ログ、描画、運営画面へ出さない。購読情報の配置（D1維持 vs Go移管）はTRACE_B#12で未決であり、決定として扱わない。

### `PATCH /v1/memory-sessions/{memorySessionId}/push-subscription`

`active`・`paused`の切り替えを行う。写真撮影後は`nextSendAt`を延期する。正確な分刻みの配信を保証しない。

### `DELETE /v1/memory-sessions/{memorySessionId}/push-subscription`

購読と未送信予定を削除する。到着、参加停止、期限切れ、Push providerの失効応答でも同じ処理を使う。通知payloadには写真、セッションIDを含めず、一般的な文面と撮影画面への相対URLだけを含める。文面は現行のまま維持する。

## 写真ストーリー

### `GET /v1/memory-sessions/{memorySessionId}/memory`

本人用の写真、写真間の再構成経路、写真ごとの地点を時系列に返す。画像取得URLは参加認証後に発行する5分以内の署名付きURLとし、レスポンス・プロキシ・CDNログに参加トークンを含めない。信頼度・確からしさ（accuracy/uncertainty）表現は廃止した（2026-10-26決定、#23）。

経路は写真を撮った順に各写真の地点（チェックポイント）をCampusGraph最短経路でつなぎ、最後の写真からShibaLab（到着）までの区間も最短経路・同じ線で描く。再生は全体10〜20秒で、区間の長さに比例配分し、写真地点で少し停止する。撮影時刻間隔の比例配分は使わない。Backend #7の経路生成まで`segments`は空配列であり、下の非空例を現行提供済みとは扱わない。

```json
{
  "status": "ready",
  "moments": [
    {
      "photoMomentId": "photo-moment-id",
      "capturedAt": "2026-07-13T12:05:00Z",
      "checkpointId": "checkpoint-main-plaza",
      "imageUrl": "short-lived-private-view-url"
    }
  ],
  "segments": [
    {
      "routeSegmentId": "route-segment-id",
      "fromPhotoMomentId": "photo-1",
      "toPhotoMomentId": "photo-2",
      "graphVersion": "graph-2026-01",
      "pathNodeIds": ["node-1", "node-2", "node-3"]
    }
  ],
  "deleteAfter": "2026-07-14T12:20:00Z"
}
```

公開スクリーンや描画機はこのAPIを使用できない。本人端末への保存は、端末のcanvasで作った写真ストーリーの画像または写真を明示操作で保存し、サーバーの`deleteAfter`を延長しない（Mino829/TRACE_F#14）。canvasで写真を描くため、写真の短命URLの配信は参加者サイトのオリジンからのGETをCORSで許可する。

## 状態取得

### `GET /v1/experiences/{experienceId}`

参加端末が再接続後に状態を復元するために使用する。`Authorization: Bearer <memory-participant-token>`で本人認証し、MemorySessionの`deleteAfter`まで取得できる。到着後の完了状態、削除予定日時、写真ストーリーの端末保存のための情報を返す。順位、待ち時間、集合表現への参加表示は返さない。`on_site`概念は廃止した。

```json
{
  "memorySession": {
    "sessionId": "mem_example",
    "status": "arrived",
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  },
  "replayStatus": "queued",
  "photoSummary": {"ready": 4, "pending": 1, "failed": 0, "maxCount": 12}
}
```

Backend #6の写真実装では、`photoSummary`の`ready`/`pending`/`failed`をMemorySessionの実件数から算出し、`maxCount`は12とする。`deleting`は一覧から確認し、保持中の上限には含める。`status`は`main`、`arrived`、`ended`のいずれかである（[状態遷移](state-machine.md)参照）。写真機能の有効化・配備は別操作であり、通知・到着キュー等の未実装項目はこの例に登場するだけで提供済みとみなさない。

## 運営API

- `GET /v1/operator/status`: 接続数、到着キュー、描画機状態
- `POST /v1/operator/arrivals/{arrivalId}/skip`: 到着演出をスキップ
- `POST /v1/operator/fallback`: 自律映像へ切り替え
- `POST /v1/operator/registration/close`: 新規参加受付を停止
- `POST /v1/operator/arrivals/{arrivalId}/play`: 主描画機へ再生を指示
- `POST /v1/operator/notifications/stop`: 全撮影通知を停止
- `GET /health/live`: プロセスの生存確認
- `GET /health/ready`: DBと必要サービスの準備確認

運営APIを参加者向け公開経路から分離する。受付停止、skip、再生、fallbackの操作は操作者、時刻、対象、結果を監査ログへ残す。監査ログに位置情報を含めない。写真ピン・フォトスポット停止APIは廃止した（2026-10-26決定、#23）。

## 描画WebSocket

### `GET /v1/display/ws`

短期の描画トークンで認証する。描画PCはthree.jsで描画し、Frontendリポジトリ`/display`がそのまま描画クライアントである（正式採用、Issue #12）。描画PCはGo（Cloud Run）へ外向きWebSocketで直接接続し、Workerは中継しない。接続直後に現在状態のスナップショットを送信し、その後に差分イベントを配信する。到着演出を進める主描画機は同時に一台だけリースを取得できる。待機描画機はスナップショットと通常表示を受信できるが、演出完了を確定できない。

描画へ渡すデータは、グラフの点IDの並び（経路）+グラフ版+写真ごとの点IDと撮影時刻のみである。原写真、画像URL、生座標、生GPSは渡さない（2026-10-26決定、#23）。`campusGraphVersion`等の版が不一致のイベントは破棄し、直前互換版へ戻す。

```json
{
  "schemaVersion": 1,
  "coordinateVersion": "campus-2026-01",
  "campusGraphVersion": "graph-2026-01",
  "threeDMapVersion": "map-2026-01",
  "eventId": "event-id",
  "type": "display.snapshot",
  "occurredAt": "2026-07-13T12:00:05Z",
  "data": {}
}
```

### `participant.arrived`

```json
{
  "schemaVersion": 1,
  "campusGraphVersion": "graph-2026-01",
  "eventId": "event-id",
  "type": "participant.arrived",
  "occurredAt": "2026-07-13T12:20:00Z",
  "data": {
    "arrivalId": "arrival-id",
    "sessionId": "anonymous-id",
    "pathNodeIds": ["node-1", "node-2", "node-3", "node-4", "node-shibalab"],
    "photoCheckpoints": [
      {"photoMomentId": "photo-moment-id", "checkpointId": "checkpoint-main-plaza", "capturedAt": "2026-07-13T12:05:00Z"}
    ],
    "replayDurationSeconds": 15
  }
}
```

`pathNodeIds`は最後の写真からShibaLab（到着）までを含む点IDの並びである。再生時間は全体10〜20秒で区間長に比例配分し、写真地点で少し停止する。`photoCheckpoints`は撮影順である。

### サーバーから描画機へのその他のイベント

- `arrival.replay.requested`
- `photo.moment.created`: 写真ごとの`checkpointId`と撮影時刻だけを含む
- `photo.moment.deleted`
- `system.fallback.changed`

旧`position.updated`、`tracking.stopped`、写真ピン・フォトスポット系イベント（`photo.marker.*`、`photo.hotspot.*`）は廃止した（2026-10-26決定、#23）。

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
- スナップショットには生成時点の`eventId`または連番を含め、それ以前の差分を二重適用しない

参加トークンはURLへ置かず、`localStorage`へ保存する。通知や別タブから開き直しても同じセッションを続けるためである。本人による削除と`deleteAfter`の到達を検知したら、端末から消去する。加工済み写真だけはIndexedDB等へ上限付きで保存し、送信成功、写真削除、セッション削除、失効時に消去する。写真は12枚・合計30MBを超えて端末へ保持しない。複数タブはBroadcastChannel等で既存セッションを検知し、二重の写真送信を開始しない。

## HTTPステータス方針

| Status | 用途 |
|---:|---|
| 200 / 201 / 204 | 取得・作成・削除成功 |
| 400 | JSON、時刻、`checkpointId`、必須項目が不正 |
| 401 / 403 | 資格情報なし、無効、権限不足 |
| 404 | 対象が存在しない。削除済みの詳細は明かさない |
| 409 | 状態競合、同意版不一致、主描画機リース競合 |
| 413 | bodyまたは点数上限超過 |
| 429 | レート制限 |
| 503 | 受付停止、DB利用不能、公開準備未完了 |
