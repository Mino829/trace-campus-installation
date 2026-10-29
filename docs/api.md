# API・WebSocket仕様

この文書は設計目標を含み、現行Backendで全APIが提供済みであることを示さない。2026-10-26にGPS不使用・写真地点自己申告へ移行した（#23）。追加方針として写真の画像バイト列は参加端末内だけに保持し、Backendへは写真ID・`checkpointId`・撮影時刻など経路に必要なメタデータだけを送る。既存Backendの写真オブジェクト保存契約は移行前の実装であり、移行・新同意文の承認・旧写真の削除保証が済むまで新方式の実データ受付を開始しない。到着証明は展示ブースで投影する署名・期限付きQRとする（#11）。大学のプライバシー審査（#9）は未完了。

## 共通方針

- Base path: 既存セッション・到着・旧写真契約は`/v1`。画像を送らない破壊的な写真契約は`/v2`とし、旧写真の本人照会・削除は旧セッションの`deleteAfter`まで維持する
- 形式: `application/json`
- 時刻: UTCのISO 8601
- 参加者認証: セッション作成時に発行するBearer token
- 描画認証: Googleログイン＋運営メール許可リストを採用。Google IDトークンからOrigin束縛の短期描画資格を発行し、WSの最初のメッセージで資格と版を照合する。詳細は[描画WebSocket](#描画websocket)。長期署名鍵をブラウザ、URL、Workerへ配布しない
- その他の運営認証: Google本人確認から描画とは独立した個人用operator資格を発行する。初期許可者はasync5855@gmail.comのみ。個別/本人全資格の失効、独立Origin/allowlist、個人actorとメール照合hashの監査を採用する（2026-10-29依頼者決定）。実Google設定・本番入口・公開承認は別とする
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
| 端末内の写真 | 暫定で1セッション12枚、合計30MB、1枚5MB、JPEG/WebP、長辺1920px以下。画像バイト列はAPIへ送らない |
| Push購読・設定 | 1セッションあたり10回/分、JSON 16KB |
| 描画WebSocket受信 | 1接続あたり10メッセージ/秒、各256KB |
| 運営API | 個人actorごとに60回/分・burst 30（token bucket）。資格発行は検証前のIP/全体と検証後actorで別制限 |

制限超過は`429`と`Retry-After`を返し、正常な共有回線を遮断しないよう監視しながら調整する。

## 冪等性キー

冪等性キーと要求内容はハッシュで記録する。同じキーの異なる内容は409 IDEMPOTENCY_CONFLICT。到着済み本人の再送は同じArrivalEventと現在の再生状態を返し、過去のqueued応答へ固定しない。写真/停止receiptはMemoryへ紐付け、最大24時間かつdeleteAfter以内、Memory全削除時にcascade消去する。内容なしの全体/一括削除の成功・部分成功receiptだけは従来の最大24時間の再送を維持する。移行で所有者を復元できない内容なし旧receiptは既存期限を延長しない。

## 公開設定・同意文

### `GET /v1/public/config`

認証前に受付状態、対応言語、有効な同意文、保存期間、写真機能情報を返す。次の例は旧クラウド写真契約に対応する開発側の草案であり、新しい端末限定保存の有効な同意文・受付設定ではない。/logのローカル同意文とBackend配信本文・版の不一致、および正本の決定は#9/#23で保留中。承認とFrontend/Backendの実装整合まで`registrationOpen=false`とする。

```json
{
  "registrationOpen": false,
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
    "enabled": false,
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

既存同意候補`2026-memory-photo-no-gps-draft-1`（jaのみ）は写真のサーバー保存を前提とするため、新方式で再利用しない。新方式では同意文の正本・版・本文、端末内写真の消失とサーバー側メタデータの保持・削除を#9で承認してから配信する。新規の`photo`設定は`storageMode: "device"`、`maxLocalBytes`を通知し、アップロードURLや`maxUploadBytes`を返さない。上の旧例や現行Go APIを新契約の提供済みレスポンスと扱わない。`notification`は撮影リマインダー30分間隔のみを表す。

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
    "status": "main",
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  }
}
```

単一の`MemorySession`を発行する。`enableLivePresence`、`livePresenceConsentVersion`、`experienceLinkToken`、レスポンスの`livePresenceSession`は廃止した（2026-10-26決定、#23）。`participantToken`はレスポンス時だけ返し、サーバーにはハッシュを保存する。有効な同意文と異なる版は`409 CONSENT_VERSION_MISMATCH`として拒否する。

`expiresAt`は開発側既定で失効まで6時間、`deleteAfter`は失効後24時間とする。例示日時と保持期間は大学のプライバシー審査（#9）による決定値ではない。`expiresAt`以後は写真メタデータの新規登録などを拒否するが、本人の状態取得・閲覧と削除は`deleteAfter`まで利用できる。`deleteAfter`到達時にサーバーの写真メタデータ・チェックポイント地点・経路・削除冪等性記録を一緒に期限削除する。端末内写真は別の保存領域であり、端末を開かない間の即時削除はサーバーから保証できない。

### `POST /v1/memory-sessions/{memorySessionId}/stop`

本人Memory Bearer、Idempotency-Key、空bodyで200を返す。statusはended、stoppedAt・expiresAt・deleteAfterを返し、最初の停止時刻と既存期限を固定する。Push購読/予定、queued/playing演出、描画除外、fallback進行とreceiptを同じtransactionで確定する。以後の写真/地点入力、初回到着、Push再購読、新規再生は拒否する。通知だけの解除とは異なり再開不可。停止は削除ではなく、本人の状態/ストーリー閲覧と削除はdeleteAfterまで残す。

### `GET /v1/live-presence-sessions/{liveSessionId}`（旧所有者のみ）

旧Live自身のBearerでsessionId・trackingStatus・expiresAt・deleteAfterだけを返す。Memory削除後も、Live自身の失効後もdeleteAfterまで照会・停止・削除を維持する。Memory/他人の資格や保持期限後は401。旧位置やaliasを返さず、停止で期限を変更しない。


## 位置送信・会場内表示・追跡停止（廃止契約）

`POST /v1/live-presence-sessions/{liveSessionId}/position`（位置送信）、`POST /v1/live-presence-sessions/{liveSessionId}/on-site`（会場内`on_site`への移行）、`POST /v1/live-presence-sessions/{liveSessionId}/stop`（追跡停止）は廃止した（2026-10-26、#23）。位置送信・on-siteエンドポイントは完全に廃止し、旧Sessionに対しても受け付けない。停止エンドポイントは廃止契約であり、既存の旧Sessionに対する停止操作だけを残す。`LivePresenceSession`は新規発行せず、既存の旧Sessionは本人資格での状態照会・停止・削除と各`deleteAfter`までの期限清掃のみが有効である。旧GPSデータは前倒し削除せず、各`deleteAfter`まで保持する。旧描画位置イベント・写真の緯度経度列は読み取り契約のみ（TRACE_B#30）。

## 到着

### `POST /v1/memory-sessions/{memorySessionId}/arrival`

Headers: `Authorization: Bearer <memory-participant-token>`、`Idempotency-Key: random-value`

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

到着は展示ブースの署名QR（purpose:"arrival"、keyId、checkpointId、venue、issuedAt/expiresAt、nonce）をBackendが検証して成立する。署名鍵はBackendだけが保持し、個人operator資格で認可された運営入口だけが発行を要求する。技術既定は25秒有効・15秒更新で、現地の読取/不正利用対策は公開前に確認する。参加者はWeb App内のカメラで読み、順位・予想待ち時間・色・記号は返さない。

Backendは本人Bearerと有効なMemorySessionを確認し、サーバー時刻で署名・用途・鍵ID・発行時刻・有効期限・会場・到着用`checkpointId`を検証する。`clientTime`は判定に使用しない。不正な証明は`400 INVALID_ARRIVAL_PROOF`で拒否し、ArrivalEventを作成しない。到着済みの同じMemorySessionの再送は、QR期限後も本人資格で既存のArrivalEventと現在の`replay_status`を返し、二重到着・二重キュー登録をしない。投影QRは複数人が見られるため、QRのランダム値を全参加者で一度限りとみなすことはできない。別セッションによる有効期間内の読み取りや撮影済みQRの遠隔使用を署名だけで防げない点はPS-005・R-05の未解決リスクであり、公開前に責任者と対策を決める。

有効な初回到着は写真枚数によらずArrivalEventを1件作り、通知を停止する。0枚は経路なしの簡易表示、1枚でも写真地点からShibaLabまでの経路を返す（2026-10-29依頼者決定）。FIFOの有効queued上限は50件でplayingを含まず、通常待機30秒ではskipしない。満杯時は503 ARRIVAL_QUEUE_FULLとRetry-After: 5を返し、到着・通知停止・receiptを確定しない。空き回復後は再試行でき、初回には有効QRが必要。停止・失効・本人削除は待機を終了させる。到着APIのstatusは現在のreplay_status、本人状態GETはreplayStatusで返す。ローカル実装/検証と配備・現地承認は別とする。

描画機は参加者の演出中と、その後の終了画面では到着QRを非表示にする。非表示は到着受付の停止やQRの失効を意味しない。直前に読んだQRなど、有効な証明で再生中に届いた到着も検証・受付し、順番を待つ。短期QRでも撮影済みの値の期限内利用は防げないため、非表示だけを不正利用対策とみなさない。

通常の再生開始は自動とする。主描画機は待機中の到着を到着順に選び、前の演出と終了画面が終わって再生可能になったら開始要求を送る。Backendが主リースと到着状態を検証して`playing`を確定した後に再生する。運営者の毎回の開始操作を前提としない。運営による障害後の再試行・強制操作は通常開始とは別の契約として確定する。

正常な再生完了をBackendが確定したら終了画面を4秒表示する（既存Frontendの表示時間を採用）。終了画面はFrontend内の表示状態であり、ArrivalEventに新しい状態を追加しない。終了画面中は次の開始要求を送らない。4秒後に待機があれば次を自動開始し、待機がなければ有効なQRを表示する画面へ戻る。Frontendには終了画面があるが、次の待機再生が表示を中断し得るため、この優先順の実装と実機確認はFrontend #36で必要である。

## データ削除

### `DELETE /v1/memory-sessions/{memorySessionId}`

Header: `Idempotency-Key: random-value`

Memory用トークンで認証し、サーバーの写真メタデータ・チェックポイント地点・経路・到着イベント・Push購読・削除冪等性記録・トークンハッシュを消す。旧方式の写真オブジェクトが残る既存セッションでは先にオブジェクトの消去を確認する。消去成功後204、失敗時503で同じキーを再送できる。新方式の端末内画像はBackendが消せないため、Web Appが同じ参加のローカル画像も削除し、双方の結果を別々に示して失敗時に再試行する。Backendの204を端末内画像の消去完了と表示しない。期限到来時の端末内画像は次回起動時に清掃するが、アプリを開かない間や端末への明示的な書き出し先の削除は保証しない（[プライバシー設計](privacy.md)）。

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

## 写真の端末保存とメタデータ（Backend実装、公開承認前）

Web Appは向き補正・EXIF除去・縮小・再エンコードした画像を同じブラウザのIndexedDBに保持し、Blob・ハッシュ・寸法・画像URLをBackendへ送らない。端末保存失敗時は撮影完了にしない。Backendの/v2は匿名Memoryに属するclientPhotoId・checkpointId・capturedAtだけを保存する。サーバー実装の完了は端末保存の検証や同意/公開承認を意味しない。

### `GET /v2/memory-sessions/{memorySessionId}/photos`

本人Bearerで撮影時刻順に`photoMomentId`、`clientPhotoId`、`checkpointId`、`capturedAt`、`sequenceNo`を返す。画像や画像URL、端末内写真の存在・消失はBackendから判定できない。Web Appは同じ端末のIndexedDBで`clientPhotoId`と画像を照合し、画像が失われていれば「この端末に写真がありません」と示す。別ブラウザ・別端末では画像を復元できない。

### `POST /v2/memory-sessions/{memorySessionId}/photos`

Headers: `Authorization: Bearer <memory-participant-token>`、`Idempotency-Key: random-value`

```json
{
  "clientPhotoId": "client-generated-id",
  "checkpointId": "checkpoint-main-plaza",
  "capturedAt": "2026-07-13T12:05:00Z"
}
```

checkpointIdは固定Graphに存在するselectable:trueの地点が必須で、未知/未選択/非選択可能は400 INVALID_CHECKPOINT。同じclientPhotoId/同内容は同じPhotoMoment、別内容は409 STATE_CONFLICT、同じ冪等性キーの別内容は409 IDEMPOTENCY_CONFLICT。削除済IDは404で復活させない。端末はローカル保存後にメタデータを送信し、オフライン再送は停止/失効前だけ許可する。未送信写真はサーバー経路に現れない。

```json
{
  "photoMomentId": "photo-moment-id",
  "clientPhotoId": "client-generated-id",
  "checkpointId": "checkpoint-main-plaza",
  "capturedAt": "2026-07-13T12:05:00Z",
  "sequenceNo": 1
}
```

Backendの成功応答はメタデータ登録と経路への反映を示すのみで、画像の保存・内容・EXIF除去・ローカル削除を検証した証明ではない。新規写真の署名付きPUT・GET URL、画像完了通知、サーバー画像検査は提供しない。

### `DELETE /v2/memory-sessions/{memorySessionId}/photos/{photoMomentId}`

Headers: `Authorization: Bearer <memory-participant-token>`、`Idempotency-Key: random-value`

本人資格でサーバーのPhotoMoment・チェックポイント地点・経路参照を削除し、残った地点から区間を再生成する。サーバーでの削除成功後204を返す。Web Appは対応する端末内画像も消すが、片側の失敗を完了として隠さず再試行を案内する。サーバーから端末内画像の消去は確認できない。

### 旧`/v1`写真契約の移行

旧`/v1/memory-sessions/{memorySessionId}/photos`の一覧・単体削除と`/v1/.../memory`の本人画像取得は、クラウド写真を既に持つ旧セッションに対し各`deleteAfter`まで残す。旧方式の新規受付、署名URL発行・再発行・`complete`は移行時に停止する。移行時に進行中のアップロードを消去確認なしに放置せず、既発行URLの失効、pending写真と旧オブジェクトの削除確認を行う。旧写真を前倒し削除したり端末内画像が既にあるとみなしたりしない。公開前にBackend/Frontend両方の切替とstagingの旧セッション回帰試験を完了する。

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

## 本人用の写真ストーリー（Backend実装、画像は同じ端末のみ）

### `GET /v2/memory-sessions/{memorySessionId}/memory`

本人Bearerでmomentsとsegmentsを同じ読み取り専用REPEATABLE READスナップショットから返す。画像/取得URLは返さない。写真間を撮影順につなぎ、到着後は最後の写真からShibaLabまでの区間を含める。0枚は経路なし、1枚でも到着区間を持つ。停止後/失効後もdeleteAfterまで閲覧・削除できる。実会場データの正しさや端末内画像の存在はBackendで保証しない。

```json
{
  "moments": [
    {
      "photoMomentId": "photo-moment-id",
      "clientPhotoId": "client-generated-id",
      "capturedAt": "2026-07-13T12:05:00Z",
      "checkpointId": "checkpoint-main-plaza"
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

Web Appは返却された`clientPhotoId`で当該端末のローカル写真を読み出し、本人画面のストーリーを合成する。ローカル画像が消失した写真は地点・経路の記録を残して画像欠落を明示し、サーバーから再取得できるようには見せない。本人が明示操作で写真アプリ/ギャラリーへ書き出した複製はWeb Appの削除対象外であり、サーバーの`deleteAfter`も延長しない。公開スクリーンや描画機はこの本人APIを使わない。

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

photoSummaryは旧クラウド写真のready/pending/failedとmaxCount:12であり、端末画像の存在や内容を検証した値ではない。新方式は/v2のメタデータと同じ端末の画像を照合する。memorySession.statusはmain、arrived、endedで、停止/失効ならended、演出completed/skippedだけならarrivedを維持する。displayAliasとonSiteChoiceRequiredは返さない。
replayStatusは現在のArrivalEvent状態、到着前はnull。本人の到着再送も同じIDと最新状態を返し、QR期限後も回復できる。通常待機だけではskipしない。満杯で未確定の人には再試行を案内する。演出skipでも本人閲覧・削除はdeleteAfterまで維持し、新規画像は同じブラウザに残る場合だけ閲覧できる。

## 運営API

- `GET /v1/operator/status`: 接続数、到着キュー、描画機状態
- `POST /v1/operator/credentials`: Google本人確認から独立した個人運営資格を発行
- `DELETE /v1/operator/credentials/current`: 現在の資格だけを失効
- `DELETE /v1/operator/credentials`: 同じ操作者の保存中の全資格を失効
- `POST /v1/operator/arrival-proofs`: 署名済み到着QRを発行
- `GET /v1/operator/arrivals`: FIFO候補を照会
- `POST /v1/operator/arrivals/{arrivalId}/retry`: 描画障害skipを明示再試行
- `DELETE /v1/operator/fallback`: 通常運転へ復帰
- `POST /v1/operator/arrivals/{arrivalId}/skip`: 到着演出をスキップ
- `POST /v1/operator/fallback`: 自律映像へ切り替え
- `POST /v1/operator/registration/close`: 新規参加受付を停止
- `POST /v1/operator/notifications/stop`: 全撮影通知を停止
- `GET /health/live`: プロセスの生存確認
- `GET /health/ready`: DBと必要サービスの準備確認

運営APIを参加者向け公開経路から分離する。受付停止、skip、再生、fallbackの操作は操作者、時刻、対象、結果を監査ログへ残す。監査ログに位置情報を含めない。写真ピン・フォトスポット停止APIは廃止した（2026-10-26決定、#23）。
描画と一般運営の本人確認には同じGoogle Web identityを使えるが、audience/allowlist/Origin設定と資格は独立する。運営資格で描画認証せず、描画資格でQR発行・受付/通知停止・skip/retry・fallbackを認めない。参加者Workerは運営APIを転送しない。本番入口の有効化承認は別である。

### 個人運営資格の発行・失効・監査

TRACE_OPERATOR_ENABLED=falseを既定とし、有効化には独立のGoogle client ID、メール/Origin許可リスト、サーバー側Edge keyが必須。初期許可者はasync5855@gmail.comのみ。POST /v1/operator/credentialsは単一Google Bearer、完全一致Origin、空body/queryで、下記描画発行と同じ厳格なGoogle署名・claim・所有権検証を行う。成功200はoperatorToken・credentialId・expiresAt・role:"operator"を返す。Google token自体は通常運営APIで使わない。

operatorTokenはランダム256bitの不透明値で2分有効・Origin束縛、ブラウザのメモリだけに保持する。DBはtoken hash、canonical Google subject hash、正規化メールの照合hashだけを保存する。各操作で現在の独立allowlist・期限・失効を検証し、参加者/描画資格と旧HMAC認証を拒否する。Edge keyは会場PCの認証付きloopback broker等のサーバー側だけが保持し、資格の取得/失効・QR要求を固定経路へ中継する。WSSは中継しない。

current削除は現在の資格だけ、全資格削除は同一actorの保存中の資格をtransactionで消し、成功204後の要求は401。認証済みの処理中要求や、その後のGoogle再ログインまで禁止する操作ではない。アカウント禁止は独立allowlistから除去して新設定へ切り替える。UIは失効失敗を成功扱いせず、本人全資格削除に認可がなければ再認証を要求する。

資格発行/失効、proof発行、skip/retry、受付/通知停止、fallbackと個人actor・メール照合hash・時刻・短縮対象・結果の24時間監査を同じtransactionで確定する。生メール/sub・token・画像/地点は監査へ保存しない。hashは匿名化ではなく許可メールから照合可能。旧サービス監査はNULLのメールhashのまま従来期限まで保持し、個人を推測して埋めない。期限切れ資格は通常清掃で削除する。実Google・公開経路・staging・投影機材の確認はローカル実装と分離する。


## 描画WebSocket

### `POST /v1/operator/display-tokens`

運営者の本人確認はGoogleログイン＋メール許可リストを採用する（依頼者選択）。展示ブラウザはGoogle IDトークンを`Authorization: Bearer <Google ID token>`へ設定し、Backendへ直接、空bodyで送る。単一のOriginが展示用許可リストと完全一致することを要求する。GoogleのRS256署名・WebクライアントIDに一致するaud・Google issuer・期限・発行時刻・sub・`email_verified`と、存在する場合のazpを検証する。許可リストは完全なメールアドレス単位で、Googleが現在の所有者を保証するGmailかWorkspace（署名済みhdあり）に限る。外部メールを登録しただけのGoogleアカウントは拒否する（[Googleの指針](https://developers.google.com/identity/sign-in/web/backend-auth)）。

成功は200で`displayToken`と`expiresAt`を返す。描画資格は用途とOriginに束縛し2分有効、Googleのメール・subは含めない。Google IDトークンと描画資格はブラウザのメモリだけに保持し、URL・query・subprotocol・永続ストレージ・ログ・QRへ載せない。署名鍵はBackendのSecret Manager設定だけに置く。参加者/描画資格、旧HMAC発行ヘッダー、複数Authorizationは401、許可外/欠落Originは403、query/bodyは400、機能無効は503、認証前のIP/全体レート制限は429/Retry-Afterとする。

設定は`TRACE_DISPLAY_GOOGLE_CLIENT_ID`、`TRACE_DISPLAY_OPERATOR_EMAILS`、`TRACE_DISPLAY_ALLOWED_ORIGINS`、独立した`TRACE_DISPLAY_TOKEN_KEY`。リストはカンマ区切り・wildcardなし。OriginはHTTPS、開発時だけlocalhost/127.0.0.1のHTTPを認める。Google OAuth側にも展示ページのJavaScript Originを登録する。旧`TRACE_DISPLAY_ISSUER_KEY`は廃止する。

### `GET /v1/display/ws`

短期の描画トークンで認証する。描画PCはthree.jsで描画し、Frontendリポジトリ`/display`がそのまま描画クライアントである（正式採用、Issue #12）。描画PCのブラウザからGo（Cloud Run）へ外向きWebSocketで直接接続し、Worker・会場PC上のagentは中継しない。最初の認証に成功してから現在状態のスナップショットを送信し、その後に差分イベントを配信する。到着演出を進める主描画機は同時に一台だけリースを取得できる。待機描画機はスナップショットと通常表示を受信できるが、演出完了を確定できない。

ブラウザ標準の`WebSocket` APIで接続し、5秒以内に最初のテキストメッセージとして次を送る。独自HTTPヘッダーによる旧native認証は廃止し、Google IDトークンそのものをWSへ送らない。HTTP Upgradeは認証成功を意味せず、最初の`display.snapshot`受信を認証成功として扱う。

```json
{
  "schemaVersion": 1,
  "type": "renderer.authenticate",
  "displayToken": "<issued-display-token>",
  "coordinateVersion": "campus-2026-01",
  "campusGraphVersion": "graph-2026-01",
  "threeDMapVersion": "map-2026-01"
}
```

認証前にsnapshot・差分・リースは渡さない。最初のメッセージは16KiB以下、資格の署名・用途・期限・Originと全版を照合する。未認証/不正資格はclose 4401、版不一致は4409、サイズ超過は1009。欠落/許可外Origin、query、旧Authorizationヘッダー付き接続はHTTP 403で拒否する。接続試行にもIP/全体のレート制限を適用する。

描画資格発行と描画WSの2経路だけはEdge共有secretを要求せず、専用Origin許可リストを用いる。その他の運営APIと参加者の`/v1`・`/v2`は既存のEdge保護を維持する。Cloud Run IAMは別の境界であり、この実装でIAM非公開サービスへ直接接続できるわけではない。

再接続時は有効なGoogle IDトークンで描画資格を再発行し、Googleの資格が期限切れなら再認証する。WS接続自体の上限は50分。許可リスト更新後に旧APIを停止・置換すると新規発行を拒否するが、既発行資格は最大2分、認証済み接続は最大50分残り得る。緊急時は描画停止・署名鍵ローテーションと旧revision/接続の停止を組み合わせる。接続中の個別強制失効APIは今回提供しない。

Backend実装は別PRで提供し、ローカルPostgreSQL・実SDKの署名検証（検証用RSA/JWKS）・実ブラウザの直接WS認証を確認した。Google OAuth設定、実Googleアカウントでのログイン、Frontendのログイン/更新/再生統合、8時間再接続とstagingの承認済み経路は未検証である。本番の描画WSSはそれらと公開承認が済むまで無効を維持する。

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
  "data": {"cursor": 0, "arrivals": []}
}
```

### `participant.arrived`

```json
{
  "schemaVersion": 1,
  "coordinateVersion": "campus-2026-01",
  "campusGraphVersion": "graph-2026-01",
  "threeDMapVersion": "map-2026-01",
  "eventId": "event-id",
  "type": "participant.arrived",
  "occurredAt": "2026-07-13T12:20:00Z",
  "data": {
    "arrivalId": "arrival-id",
    "replayStatus": "queued",
    "pathNodeIds": ["node-1", "node-2", "node-3", "node-4", "node-shibalab"],
    "photoCheckpoints": [
      {"checkpointId": "checkpoint-main-plaza", "capturedAt": "2026-07-13T12:05:00Z"}
    ],
    "replayDurationSeconds": 15
  }
}
```

`pathNodeIds`は写真の撮影順に対応する経路を連結し、最後の写真からShibaLab（到着）までの区間も含む点IDの並びである。再生時間は全体10〜20秒（現Backendの投影値は15秒）で、区間長に比例配分し写真地点で少し停止する描画はFrontend側の未統合事項。`photoCheckpoints`は撮影順。写真0枚、旧方式の写真、Graph版不一致、区間欠損時には描画用の経路を空にし、旧GPS由来の点を新方式へ混ぜない。描画データにはPhotoMoment ID、MemorySession ID、色・記号などの照合情報を含めない。

### サーバーから描画機へのその他のイベント

- `arrival.replay.requested`: 待機到着の通知。主描画機が順序と自身の再生状態を確認して自動開始要求を送るためのもので、受信しただけで再生を始めない
- `photo.moment.created` / `photo.moment.deleted`: 到着が再生待ち・再生中の場合、`arrivalId`、更新後の全`pathNodeIds`、撮影順の全`photoCheckpoints`を渡す。写真単位のIDは送らず、同じ地点・同時刻の写真がある場合も到着単位で全置換する。到着前の写真変更は`participant.arrived`でまとめて通知する
- `system.fallback.changed`（運営切替の実装待ち）

旧`position.updated`、`position.expired`、`tracking.stopped`、Live位相イベント、写真ピン・フォトスポット系イベント（`photo.marker.*`、`photo.hotspot.*`）は廃止した（2026-10-26決定、#23）。旧Liveの本人照会と削除は保持期限まで続けるが、描画用snapshotの`participants`と位置差分は返さない。

### 描画機からサーバーへのメッセージ

描画機はクライアント生成`eventId`、対象`arrivalId`、主描画機リースIDを付けて送る。同じ`eventId`は一度だけ適用する。

- `renderer.heartbeat`: fps、現在モード、主系リース、時刻
- `arrival.replay.started`: 主描画機の自動開始要求。Backendが主リースと到着状態を検証して開始を確定した応答を返してから再生する
- `arrival.replay.completed`: 正常完了
- `arrival.replay.failed`: 失敗理由。再生中の描画障害はskipし、自動で再演しない。運営による再試行の条件は別途決定する

WebSocketのping/pongまたはアプリheartbeatを10秒間隔で送り、30秒応答がなければ切断と判定する。最大メッセージサイズ、許容イベント頻度、未知のschemaVersionの扱いを実装設定で固定する。

## 再接続

- 描画クライアントは指数バックオフで再接続する
- 再接続時は必ず `display.snapshot` から復元する
- スナップショットには生成時点の`eventId`または連番を含め、それ以前の差分を二重適用しない

参加トークンはURLへ置かず、同じブラウザで開き直せるよう端末内に保存する。写真Blobは同じブラウザのIndexedDBに1セッション12枚・計30MB以内で保持し、写真単位・セッション削除時はWeb Appが端末内画像とサーバーメタデータの双方を消す。`deleteAfter`到達をWeb Appが検知したときもローカルデータを消すが、アプリを閉じたままの端末・別ブラウザ・写真アプリへ保存した複製をサーバーは削除できない。ブラウザの容量不足・消去・端末紛失では画像は復元できない。複数タブは既存セッションを検知し、二重のメタデータ送信を開始しない。

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
