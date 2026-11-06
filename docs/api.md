# API・WebSocket仕様（完成版）

更新日: 2026-11-06

Backend `6b87961`とFrontend `be94bd2`を基準とする。`/v1`は参加・停止・到着・旧データ本人操作、`/v2`は端末内写真のmetadata用であり、すべてをv2へ変更する設計ではない。コード上の提供と、feature有効化・配備・正式承認・現在の疎通は別である。

## 共通方針

- JSONは`application/json`、通常の入力上限は16KiB、日時はUTCのRFC3339。POST等は空bodyでもサイトWorker経由ではContent-Typeを付ける。
- 本人認証は`Authorization: Bearer <participant-token>`。Memoryと旧Liveは別資格。トークンをURL、QR、通常ログ、描画へ載せない。
- ブラウザの参加者APIは`/api/backend/v1/...`または`/api/backend/v2/...`。以下のAPI pathはGo側の表記である。Workerは許可リスト方式でEdge secretをサーバー側付加する。
- 運営APIの多くは許可Origin＋Edge保護＋個人operator Bearer。例外は直接描画token発行とWSSで、専用Originと独立した描画認証を使う。
- レスポンスは`Cache-Control: no-store`。参加者トークンは端末localStorage、写真はIndexedDB、Google／operator資格はタブメモリで管理する。

### エラー形式

```json
{
  "error": {
    "code": "STATE_CONFLICT",
    "message": "photo state or request content conflicts",
    "requestId": "req_example"
  }
}
```

これはGoの形式。Workerの拒否応答は`{"error":{"code":"..."}}`だけの場合もある。内部DB／GCSエラーや生位置を返さない。

## 公開設定・同意文

### `GET /v1/public/config`

実レスポンスの項目は`registrationOpen`、`livePresenceEnabled:false`、`consent:{version,language,body}`、旧互換用の`livePresenceConsentVersion`／`onSiteConsentVersion`／`positionIntervalSeconds`／`livePositionTtlSeconds`、`spatial`、`photo`、`notification`、`contact`。互換位置設定があっても新規GPS受付を意味しない。

device metadata有効時の`photo`と、通知無効時の`notification`は次の形になる（このJSONは全レスポンスではなく該当プロパティの抜粋）。

```json
{
  "photo": {
    "enabled": true,
    "storageMode": "device",
    "maxCount": 12,
    "maxLocalBytes": 31457280,
    "publicDisplay": false
  },
  "notification": {
    "enabled": false,
    "defaultIntervalMinutes": 30,
    "allowedIntervalsMinutes": [30],
    "exactDeliveryGuaranteed": false,
    "vapidPublicKey": ""
  }
}
```

device時はupload容量／寸法や`photoLocationRetentionHours`を返さない。`notification.vapidPublicKey`は通知無効時に空文字、有効時にVAPID公開鍵を返す。`spatial`は`coordinateVersion`、`campusGraphVersion`、`threeDMapVersion`の3キー。Frontendは取得した同意本文・版・言語を表示し、作成時の版不一致では新本文で再同意する。本文が配信できることは承認済みを意味しない。

## 体験・セッション作成

### `POST /v1/experiences`

```json
{
  "memoryConsentVersion": "2026-10-31-v1",
  "enableLivePresence": false,
  "consentLanguage": "ja",
  "clientTime": "2026-10-31T01:00:00Z"
}
```

同意版・日時は会期中構成の形を示す例であり、現在の設定値として再利用しない。必ず公開設定の版と送信時刻を使う。201の例:

```json
{
  "experienceId": "exp_example",
  "memorySession": {
    "sessionId": "mem_example",
    "participantToken": "memory-participant-secret",
    "status": "main",
    "expiresAt": "2026-10-31T07:00:00Z",
    "deleteAfter": "2026-11-01T07:00:00Z",
    "campusGraphVersion": "toyosu-graph-2026-10-29.1.draft"
  }
}
```

例の6時間＋24時間は会期中構成の記録で、承認済み保持期間や現在値を保証しない。Memory期限は作成時刻＋設定TTL、削除期限はその期限＋設定保持時間。新規Live・ExperienceLink・alias・link tokenは返さない。`enableLivePresence:true`は409 `LIVE_PRESENCE_DISABLED`。同意版不一致は409 `CONSENT_VERSION_MISMATCH`、非対応言語は400、clientTimeは24時間前〜5分後を検証する。受付flagまたは永続受付停止で閉じていれば503 `REGISTRATION_CLOSED`。

## 状態取得

### `GET /v1/experiences/{experienceId}`

Memory Bearerで、Memory自身の`deleteAfter`まで使用する。

```json
{
  "memorySession": {
    "sessionId": "mem_example",
    "status": "main",
    "expiresAt": "2026-10-31T07:00:00Z",
    "deleteAfter": "2026-11-01T07:00:00Z"
  },
  "photoSummary": {"ready": 2, "pending": 0, "failed": 0, "maxCount": 12},
  "replayStatus": null
}
```

Memory statusは`main`／`arrived`／`ended`。未到着の`replayStatus`はnull、到着後は`queued`／`playing`／`completed`／`skipped`。演出完了／skipだけではMemoryをendedにしない。旧Liveが残る場合だけ`livePresenceSession:{sessionId,trackingStatus,presencePhase,expiresAt,deleteAfter}`を返す。Live token、位置点、待ち順、推定待時間、alias、`onSiteChoiceRequired`は返さない。

### `GET /v1/live-presence-sessions/{liveSessionId}`

旧Live自身のBearerで`{sessionId,trackingStatus,expiresAt,deleteAfter}`だけを返す。Memory削除後・Live追跡期限後もLive自身の`deleteAfter`まで使用できる。Memory／他人のBearer・保持期限後は401。

## 写真撮影・保存

### 新規device写真

新規画像は端末内のみ。Goへ画像、EXIF、ハッシュ、寸法、容量、URL、緯度経度を送らない。Graph版はMemory作成時に固定し、クライアントが登録payloadへ追加しない。

#### `POST /v2/memory-sessions/{memorySessionId}/photos`

Memory Bearer、`Idempotency-Key`、Content-Type必須。許可キーは次の3つだけで、追加・重複キーも400で拒否する。

```json
{
  "clientPhotoId": "device-photo-001",
  "checkpointId": "grand-stairs-bottom",
  "capturedAt": "2026-10-31T01:05:00Z"
}
```

新規201、同一写真の再送200:

```json
{
  "photoMomentId": "photo_example",
  "clientPhotoId": "device-photo-001",
  "checkpointId": "grand-stairs-bottom",
  "capturedAt": "2026-10-31T01:05:00Z",
  "sequenceNo": 1
}
```

- device機能有効・device Memoryであること、未停止・参加期限内・削除中でないことが必要。
- Graphに存在して`selectable:true`の地点だけを受け付ける。roleだけでは決めない。未選択・未知ID・false／null／属性欠落は400 `INVALID_CHECKPOINT`。
- Memoryの固定Graph版と現行版が違えば新規追加は409 `STATE_CONFLICT`。古い地点を現行座標で読み替えない。
- `capturedAt`はMemory作成以後・参加期限以内・サーバーの5分後以内。最大12件。同一client IDの異内容は409。
- 登録時に撮影順経路を再構成し、次回通知を延期する。`sequenceNo`は受付順、一覧順は`capturedAt,sequenceNo`。
- Frontendは到着前の写真metadataを送る。到着後の追加撮影は端末内のみ。APIは到着だけを理由に写真追加を禁止するわけではない。

#### `GET /v2/memory-sessions/{memorySessionId}/photos`

`{"photos":[...]}`の配列各要素は上の登録応答と同じ5項目。画像URLやupload状態を返さない。

#### `DELETE /v2/memory-sessions/{memorySessionId}/photos/{photoMomentId}`

Memory Bearer、Idempotency-Key、空body。成功204。metadataと前後経路を同じtransactionで削除・再構成し、削除済みclient IDのfenceをMemory削除まで保持する。古い登録再送で復活させない。本人端末の画像消去はFrontend側の別処理である。

#### `GET /v2/memory-sessions/{memorySessionId}/memory`

```json
{
  "moments": [
    {"photoMomentId":"photo_example","clientPhotoId":"device-photo-001","checkpointId":"grand-stairs-bottom","capturedAt":"2026-10-31T01:05:00Z"}
  ],
  "segments": [],
  "deleteAfter": "2026-11-01T07:00:00Z"
}
```

1枚・未到着なら区間は空。区間がある場合は`routeSegmentId`、`fromPhotoMomentId`、`toPhotoMomentId`（到着終端ではnull）、`graphVersion`、`pathNodeIds`を返す。写真一覧と経路を読取専用REPEATABLE READの同一コミット済みsnapshotから読む。画像URL、confidence、ローカルpath点列はこのv2応答には含めない。

### 旧クラウド写真

legacy Memoryの`GET /v1/.../photos`、`GET /v1/.../memory`、`DELETE /v1/.../photos/{photoMomentId}`は本人Bearerと各`deleteAfter`の範囲で維持する。memoryはready画像だけを返し、URL期限は5分またはdeleteAfterの早い方まで。旧写真・GPS由来経路の閲覧・削除用であり、新規device写真をGCSへ送る導線ではない。

旧POST photos、upload-url、completeのハンドラはコードに残るが、現行device構成では503 `PHOTO_DISABLED`、参加者Workerも中継しない。旧設定を有効にすることを本書の運用手順にしない。新旧モードは同時有効化できない。旧オブジェクト／署名PUT fenceが残る間は、upload無効化後もGCS設定・署名権限・cleanupを維持する。削除失敗は`deleting`で再試行し、未失効PUTが画像を再作成しないようゼロバイトobjectとoutboxを期限後に清掃する。

## 到着

### `POST /v1/memory-sessions/{memorySessionId}/arrival`

Memory Bearer、Idempotency-Key:

```json
{"checkpointToken":"signed-short-lived-proof","clientTime":"2026-10-31T01:20:00Z"}
```

成功200:

```json
{"arrivalId":"arrival_example","status":"queued"}
```

旧有効ExperienceLinkがある場合は入力に`experienceLinkToken`を追加して照合し、応答に`livePresencePhase`が入る場合がある。新Memoryはlinkなし・写真0枚でも到着できる。

proofは`base64url(JSON).base64url(HMAC-SHA256)`。payloadは`purpose:"arrival"`、`keyId`、`checkpointId`、`venue`、`issuedAt`／`expiresAt`（Unix秒）、nonceで、25秒有効。サーバー時刻・用途・署名・鍵ID・会場・地点を検証し、不正な新規操作は400 `INVALID_ARRIVAL_PROOF`。QR自体はFrontendで同一origin `/log` URLのfragmentに包む。別Memoryが同じ投影QRを使うことは可能で、物理来場の一回限り保証ではない。

初回到着はMemoryにつき1行。到着、旧Liveの条件付きarrived更新、通知方針に基づく停止、到着区間作成、proof消費、receipt、描画イベントを一つのtransactionで確定する。同じキー・同じbody／別キーでの到着回復は、proof期限や鍵切替後も初回到着IDと**現在のstatus**を返し、再enqueueしない。同じキー・異なるbodyは409。初回APIは再視聴APIの代用ではない。

### `POST /v1/memory-sessions/{memorySessionId}/arrival/replay`

Memory Bearer、Idempotency-Key、上と同形の`checkpointToken`／`clientTime`（link tokenなし）。到着・描画機能有効時に、本人がcompleted後の新しいQRを読み直す操作だけで使用する。200 `{"arrivalId":"arrival_new_generation","status":"queued"}`。

- 新しい公開再生IDを発行し、`queued_at`を現在時刻としてFIFO末尾へ入れる。初回のroot到着ID／arrived_at／経路終端／Memory期限／Push停止方針を変更しない。
- 新規受付はcompletedかつ有効・未停止・非削除Memoryが必要。queued／playing／skipped・失効は409 `STATE_CONFLICT`。競合時は本人GETへ合流し追加予約を作らない。
- 初回も含め、同じproofはMemoryごと1度だけ新規消費できる。DB保存はSHA-256 fingerprintのみ。再利用は400 `INVALID_ARRIVAL_PROOF`。
- 再視聴receiptは初回と別。受理済み同キー・同bodyはproof期限後・状態変更後も保存済み成功応答を返すだけで、再enqueueしない。異bodyは409。receiptはMemory削除まで所有される。
- 満杯・競合はtransactionを確定しないのでproof／receiptを消費しない。

### FIFOとスキップ

有効Memoryのqueuedは最大50件（playingを除く）。満杯は503 `ARRIVAL_QUEUE_FULL`＋`Retry-After: 5`、副作用なし。FIFOは`queued_at,id`順で、通常待機30秒を理由にskipしない。旧pending写真だけ最大15秒の開始待機とし、後続は追い越さない。再生中は全体1件。主描画機が自動開始要求し、Backendがlease・先頭・写真待機・単一playingを検証する。手動play APIはない。

`POST /v1/.../arrival/skip`はMemory本人のqueuedだけをskipする（空body、200 `{"replayStatus":"skipped"}`）。運営skip／renderer障害retryは別権限。skipは本人の閲覧・削除期限を短縮しない。

## 参加停止と旧Live

`POST /v1/memory-sessions/{memorySessionId}/stop`はMemory Bearer、Idempotency-Key、空body。200で`{status:"ended",stoppedAt,expiresAt,deleteAfter}`を返す。最初のstoppedAtを固定し、Push購読・予定削除、queued／playingのparticipant_request skip、描画除外・fallback進行を確定する。新規写真・初回到着・Push再購読・新規再生は拒否し、再開APIはない。停止は削除でなく、期限は変更しない。

旧Liveは自身のBearerとIdempotency-Keyで`POST /v1/live-presence-sessions/{id}/stop`へ`{"reason":"participant_request"}`を送り、`{"trackingStatus":"stopped","presencePhase":"ended"}`を得る。Memory写真を削除しない。位置POSTは本人認証後410 `POSITION_DISABLED`。`on-site`の`{"continue":true,...}`は410 `ON_SITE_DISABLED`、`continue:false`は停止側の互換操作として残る。Workerは位置POSTを中継しない。

## データ削除

| API | 資格／成功 | 範囲 |
|---|---|---|
| `DELETE /v1/memory-sessions/{id}` | Memory Bearer、Idempotency-Key、204 | device metadata・経路・到着・receipt・proof消費・通知・Memory、旧画像は消去確認後。旧Liveは削除しない |
| `DELETE /v1/live-presence-sessions/{id}` | Live Bearer、Idempotency-Key、204 | 旧Live・最新位置・link。Memoryは削除しない |
| `DELETE /v1/experiences/{id}` | Memory Bearer＋`X-Trace-Live-Token`＋Idempotency-Key、空body、200 | 両方の資格が揃う旧体験だけ。Liveを先に確定、Memoryを別transactionで確定 |

両方削除は`{"memory":"pending","live":"deleted","complete":false}`または両方deleted／complete trueを返す。pending／通信失敗では両資格と同じキーを保持して再送し、完了側を復活させない。元からLiveがない場合・片側を個別削除済みの場合は残る側の個別DELETEを使う。両方削除のHTTP 200だけを全削除完了と判断しない。

片側削除でExperienceLinkを除去する。写真metadataと端末画像、GCS旧画像、バックアップ／インフラログは同じ物ではない。Frontendは対象の端末写真・資格を消去するが、サーバーの期限清掃だけでは端末画像や本人が書き出したJPEGは消えない。

## 撮影通知

`PUT /v1/memory-sessions/{id}/push-subscription`、Memory Bearer:

```json
{"consent":true,"subscription":{"endpoint":"https://push-provider.example/subscription","keys":{"p256dh":"base64url-key","auth":"base64url-key"}},"intervalMinutes":30}
```

endpoint例は構造の説明で、実際は許可HTTPS providerと妥当な鍵だけを受け付ける。説明後の本人操作・別途のブラウザ通知許可が必要。PUT／PATCHの応答は`status`、`intervalMinutes`、`nextSendAt`、`expiresAt`だけ。PATCHは`{"status":"paused"}`／`{"status":"active"}`／`{"intervalMinutes":30}`。DELETE空bodyは204で購読と未送信予定を削除する。30分以外は受け付けず、配信時刻・exactly-onceを保証しない。

既存・既定Memoryは初回到着時に`push_stopped=true`と購読削除を確定する。schema 020以降、新しい継続用同意版・本文と完全一致する新規Memoryにだけ`push_after_arrival=true`を固定できる。既存の同意・期限・停止は変更せず、PUT／PATCH／後付け設定で昇格しない。継続対象も参加停止、旧Live停止、解除、失効、削除、provider 404/410、運営の全通知停止で止まる。到着だけで購読を作成・復元しない。解除は再購読可能だが、終端停止の解除はない。

通知機能・送信サービス無効／運営停止時は新規PUT／PATCHを503 `PUSH_UNAVAILABLE`とし、本人DELETEは残す。endpoint／鍵は暗号化保存。通知payloadは固定一般文面と相対URLだけで、写真、位置、参加IDを入れない。

## 運営API

Google ID tokenをBearerにして空body・queryなしで`POST /v1/operator/credentials`へ送る。署名、issuer、audience、sub、時刻、verified email、必要なazp／Workspace所有権と独立許可リストを検証する。200 `{operatorToken,credentialId,expiresAt,role:"operator"}`。資格は2分・Origin束縛の不透明Bearerで、DBはhashだけ。以降の操作はGoogle token自体でなくoperatorTokenを使う。

| API | 用途／応答 |
|---|---|
| `DELETE /v1/operator/credentials/current` | 現在の資格を失効、204 |
| `DELETE /v1/operator/credentials` | 本人の保存中全資格を失効、204。新しいGoogleログインを禁止する操作ではない |
| `POST /v1/operator/arrival-proofs` | 空body、200 `{checkpointToken,expiresAt}` |
| `GET /v1/operator/status` | 受付、キュー件数、通知停止、接続数、主lease有効性、fallback。参加者・資格・写真を返さない |
| `GET /v1/operator/arrivals` | `replayCandidates:[{arrivalId,readyPhotoCount}]`、FIFOの有効候補 |
| `POST /v1/operator/arrivals/{arrivalId}/skip` | 200 `{replayStatus:"skipped"}` |
| `POST /v1/operator/arrivals/{arrivalId}/retry` | renderer_failure skipのみ、有効Memory・空枠を検証し新公開IDで末尾へ、200 `{replayStatus:"queued"}` |
| `POST /v1/operator/registration/close` | 空body、永続受付停止。既存本人操作・到着・通知は別 |
| `POST /v1/operator/notifications/stop` | 空body、200 `{notificationsStopped:true}`。全購読削除と永続停止。再起動で解除せず、再開APIなし |
| `POST / DELETE /v1/operator/fallback` | 空body、200 `{fallback:{mode,switchAt,changedAt}}`。POSTでpending／fallback、DELETEでnormal |

本番サイトの限定bridgeは`POST /api/operator/operator/credentials`、`GET /api/operator/arrival-token`（GoのPOST proofへ変換）、`DELETE /api/operator/operator/credentials/current`、`DELETE /api/operator/operator/credentials`のみ。他の管理APIは中継しないため、停止等は別の認可済み運営経路を使用する。旧loopback brokerと本番Workerを混同しない。

全通知停止はprovider要求との排他・監査を同transactionで確定するが、すでにproviderが受理した通知の端末到着は取り消せない。fallbackは受付／通知停止とは独立で、待機列を維持する。運営操作・資格発行失効を個人actorとメール照合hash・時刻・短縮対象・結果で24時間監査する。

## 描画WebSocket

### 直接描画資格と接続

`POST /v1/operator/display-tokens`は許可された展示Originから直接Goへ、Google ID tokenのBearer・空body・queryなしで送る。200 `{displayToken,expiresAt}`、有効2分・Origin束縛・独立署名鍵。個人operatorTokenやEdge secretは代用不可。

`GET /v1/display/ws`へ直接WSS接続する。query・Authorization headerは不可。5秒以内の最初のtext frame:

```json
{
  "schemaVersion": 1,
  "type": "renderer.authenticate",
  "displayToken": "short-lived-display-token",
  "coordinateVersion": "toyosu-2026-08-28.1",
  "campusGraphVersion": "toyosu-graph-2026-10-29.1.draft",
  "threeDMapVersion": "toyosu-3dmap-2026-10-30"
}
```

認証失敗4401、版不一致4409でclose。認証前にsnapshotやleaseを渡さない。接続は最大50分で更新し、新しいGoogle由来tokenで再接続する。

### snapshot・差分

サーバーフレームは`schemaVersion:1`、3種の版、`eventId`、`type`、`occurredAt`、`data`。接続直後の実在形（空キュー例）:

```json
{
  "schemaVersion": 1,
  "coordinateVersion": "toyosu-2026-08-28.1",
  "campusGraphVersion": "toyosu-graph-2026-10-29.1.draft",
  "threeDMapVersion": "toyosu-3dmap-2026-10-30",
  "eventId": "0",
  "type": "display.snapshot",
  "occurredAt": "2026-10-31T01:00:00Z",
  "data": {
    "cursor": 0,
    "arrivals": [],
    "fallback": {"mode":"normal","switchAt":null,"changedAt":"2026-10-31T00:00:00Z"}
  }
}
```

arrivalsの各要素と`participant.arrived`／`photo.moment.created`／`photo.moment.deleted`のdataは`{arrivalId,replayStatus,pathNodeIds,photoCheckpoints,replayDurationSeconds}`。`photoCheckpoints`は撮影順の`{checkpointId,capturedAt}`のみ、durationは15。写真変更イベントも**全投影置換**であり、削除した写真IDだけを渡す形式ではない。Graph不一致・不完全区間は空経路とし、推測補完しない。

`arrival.replay.requested`は`{arrivalId}`。再視聴／運営retryでは全metadataの`participant.arrived`を先に、requestedを後に同transactionで確定する。描画のarrivalIdは現世代`replay_id`で、初回だけroot到着IDと同一。旧世代への完了・失敗は拒否しrootへフォールバックしない。

その他は`arrival.replay.started/completed/failed`（`{arrivalId,replayStatus}`）、`arrival.replay.skipped`（`{arrivalId,reason}`）、`system.fallback.changed`。新規Live position、写真pin／hotspot、写真画像／URLを配信しない。

DB連番イベントのeventIdは数字文字列、snapshotの`data.cursor`は数値。制御応答のeventIdは別のcontrol IDであり、差分cursorに使わない。snapshotと永続状態は同じcursor lockで切り出し、その後の差分だけを読む。欠番・保持切れ・再接続はsnapshotで全置換し、過去の差分を再適用しない。

### 主leaseと描画機からのメッセージ

クライアントのメッセージは`schemaVersion:1`、3種の版、`type`を共通に持つ。認証後`renderer.primary.acquire`を送り、`renderer.primary.granted`の`data.leaseId`を保持する。別接続は`renderer.primary.conflict`。leaseは30秒、10秒heartbeat等で更新し、失効／切断時はそのleaseのplayingをrenderer_failure skipにする。

再生メッセージの例（フィールドはdata内でなくトップレベル）:

```json
{
  "schemaVersion": 1,
  "coordinateVersion": "toyosu-2026-08-28.1",
  "campusGraphVersion": "toyosu-graph-2026-10-29.1.draft",
  "threeDMapVersion": "toyosu-3dmap-2026-10-30",
  "type": "arrival.replay.started",
  "eventId": "renderer-event-001",
  "arrivalId": "arrival_example",
  "leaseId": "lease_example"
}
```

`arrival.replay.completed`／`arrival.replay.failed`も同形。同じeventIdのlease／対象／操作を照合して一度だけ適用する。成功は`renderer.ack`のdata `{eventId,replayStatus}`、競合は`renderer.rejected`のdata `{eventId,code}`。別世代・失効lease・非FIFO先頭・再生重複を拒否する。`renderer.heartbeat`はトップレベル`leaseId`、`fps`、`mode`を使用できる。版不一致は`renderer.version.rejected`後切断。WSは256KiB・1接続10 message/秒、ping10秒・応答期限30秒。

## 冪等性キー

Idempotency-Keyは空でなく200文字以内。機能ごとのscopeを混ぜない。

| 操作 | 保持／再送 |
|---|---|
| 停止・写真mutation・個別／一括削除 | 一般receiptは最大24時間。Memory所有分はdeleteAfter以下・Memory削除で消去。削除済み写真の登録再送で復活しない |
| 初回到着 | 24時間receiptは入力hashとroot locator。応答は現在の再生状態から再構成 |
| 本人再視聴 | 専用receiptをMemory削除まで保持。受理済み成功応答をそのまま返す |
| 描画完了等 | renderer eventId専用receiptでlease・現世代・操作を照合 |
| 一括削除 | 進捗receiptから未完了側だけ再試行。pendingを確定成功として固定しない |

「全APIが24時間同じJSONを返す」とはしない。写真POSTは3項目の正規化入力hash、到着／再視聴はbody bytesのhashを使うため、通信再送ではendpoint・body・キーを固定する。Frontendはproof本文を永続化せず、同画面内再送だけを固定する。

## 入力制限・HTTPステータス

| 対象 | 実装上のtoken bucket（単一プロセス） |
|---|---|
| 参加作成・公開設定 | 各IP 60回/分 burst60、各全体10回/秒 burst10 |
| 本人状態・到着・停止・削除 | 資格ごと30回/分 burst30 |
| 写真metadata POST | Memoryごと6回/分 burst6、保持12件 |
| Push | Memoryごと10回/分 burst10 |
| 運営操作 | actorごと60回/分 burst30、全体20回/秒 burst100 |
| 公開API追加上限 | 認証付きIP 1200回/分 burst120、全体100回/秒 burst100 |

超過は429 `RATE_LIMITED`＋Retry-After。一括削除は両資格の枠を同時に確認する。カウンター・接続数は複数API間で共有せず最大1インスタンスを維持する。資格発行／接続にも検証前制限がある（実装README参照）。

200／201／204は成功、400は入力・proof不正、401／403は資格・Origin・Edge拒否、404は対象なし、409は状態／同意／冪等性競合、410は廃止済み位置操作、413はbody超過、Workerの415はJSON形式不正、429は制限、503は受付／機能／DB／キューなど利用不能。`GET /health/live`はプロセス、`GET /health/ready`はDB・埋込schemaとの整合を確認するだけで、Google・カメラ・投影・実Pushや公開承認を証明しない。

## 実装参照

- [API登録・設定・参加・旧Live・削除](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/server.go)、[Memory停止・Live本人GET](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/memory_sessions.go)
- [v2入出力](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/photo_moments.go)、[同一snapshot・経路・fence](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/photo_moments.go)
- [到着proof／API](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/arrivals.go)、[初回到着](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/arrivals.go)、[再視聴](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/arrival_replay.go)
- [描画wire形式・認証・heartbeat](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/display.go)、[snapshot・lease・FIFO](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/display.go)
- [Google描画認証](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/display_auth.go)、[個人運営認証](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/operator_auth.go)
- [本番Worker経路](https://github.com/Mino829/TRACE_F/blob/be94bd2/push-worker/src/site.ts)、[Frontend本人導線・proof再送](https://github.com/Mino829/TRACE_F/blob/be94bd2/README.md)、[通知・運営・旧画像・レート制限](https://github.com/Mino829/TRACE_B/blob/6b87961/README.md)
