# API・WebSocket仕様

この文書は設計目標を含み、現行Backendで全APIが提供済みであることを示さない。2026-10-26にGPS不使用・写真地点自己申告へ移行した（#23）。追加方針として写真の画像バイト列は参加端末内だけに保持し、Backendへは写真ID・`checkpointId`・撮影時刻など経路に必要なメタデータだけを送る。既存Backendの写真オブジェクト保存契約は移行前の実装であり、移行・新同意文の承認・旧写真の削除保証が済むまで新方式の実データ受付を開始しない。到着証明は展示ブースで投影する署名・期限付きQRとする（#11）。大学のプライバシー審査（#9）は未完了。

## 共通方針

- Base path: 既存セッション・到着・旧写真契約は`/v1`。画像を送らない破壊的な写真契約は`/v2`とし、旧写真の本人照会・削除は旧セッションの`deleteAfter`まで維持する
- 形式: `application/json`
- 時刻: UTCのISO 8601
- 参加者認証: セッション作成時に発行するBearer token
- 運営・描画認証: 参加者用とは別の短期トークンを使う。発行者認証、ブラウザへの受け渡し、失効・ローテーションと公開経路は未決（設計Issue #11）。長期鍵をブラウザ、URL、Workerへ配布しない。発行契約が確定するまで運営API・描画WSSを本番公開しない
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
| 運営API | 1操作者あたり60回/分 |

制限超過は`429`と`Retry-After`を返し、正常な共有回線を遮断しないよう監視しながら調整する。

## 冪等性キー

到着、削除の`Idempotency-Key`は操作スコープ、キーのハッシュ、リクエスト内容のハッシュ、結果を24時間保持する。同じキーで異なる内容は`409 IDEMPOTENCY_CONFLICT`とする。到着は一度だけ作成したArrivalEventの**現在の**再生状態を返す（以前の`queued`応答を固定して返さない）。削除以外の他の冪等操作は同じキーと内容に以前の結果を返す。削除の冪等性記録は写真・チェックポイント地点・経路と同一の保持期間で一緒に消す（2026-10-26決定、#23）。

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
    "expiresAt": "2026-07-13T18:00:00Z",
    "deleteAfter": "2026-07-14T18:00:00Z"
  }
}
```

単一の`MemorySession`を発行する。`enableLivePresence`、`livePresenceConsentVersion`、`experienceLinkToken`、レスポンスの`livePresenceSession`は廃止した（2026-10-26決定、#23）。`participantToken`はレスポンス時だけ返し、サーバーにはハッシュを保存する。有効な同意文と異なる版は`409 CONSENT_VERSION_MISMATCH`として拒否する。

`expiresAt`は開発側既定で失効まで6時間、`deleteAfter`は失効後24時間とする。例示日時と保持期間は大学のプライバシー審査（#9）による決定値ではない。`expiresAt`以後は写真メタデータの新規登録などを拒否するが、本人の状態取得・閲覧と削除は`deleteAfter`まで利用できる。`deleteAfter`到達時にサーバーの写真メタデータ・チェックポイント地点・経路・削除冪等性記録を一緒に期限削除する。端末内写真は別の保存領域であり、端末を開かない間の即時削除はサーバーから保証できない。

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

到着は、展示ブースのプロジェクター映像に表示し定期更新する署名・期限付きQR（用途、`checkpointId`、会場、発行時刻、有効期限、ランダム値、署名鍵ID）のBackend検証でのみ成立する（2026-10-26決定、#11）。Backendだけが署名鍵を保持して到着証明を発行し、会場の投影装置は発行済みQRを表示する。発行要求元の認証経路は運営・描画資格の発行契約とともに未決であり、公開APIから無認証で発行してはならない。QRの有効期間と更新間隔は実機試験で決める。参加者は展示ブースの列に並び、自分の番でWeb App内のカメラから読む。待ち順は人の列で扱い、参加端末へ順位・予想待ち時間・色・記号を返さない。

Backendは本人Bearerと有効なMemorySessionを確認し、サーバー時刻で署名・用途・鍵ID・発行時刻・有効期限・会場・到着用`checkpointId`を検証する。`clientTime`は判定に使用しない。不正な証明は`400 INVALID_ARRIVAL_PROOF`で拒否し、ArrivalEventを作成しない。到着済みの同じMemorySessionの再送は、QR期限後も本人資格で既存のArrivalEventと現在の`replay_status`を返し、二重到着・二重キュー登録をしない。投影QRは複数人が見られるため、QRのランダム値を全参加者で一度限りとみなすことはできない。別セッションによる有効期間内の読み取りや撮影済みQRの遠隔使用を署名だけで防げない点はPS-005・R-05の未解決リスクであり、公開前に責任者と対策を決める。

有効な初回到着は写真枚数にかかわらずArrivalEventを1件作り、MemorySessionを`arrived`へ変更して撮影通知を停止する。写真0枚・1枚は簡易表示とする（#23）。演出の待機キューは同時最大10件（`queued`のみを数え、`playing`は含まない）、受付から再生開始まで最大30秒とし、到着受付・容量判定・キュー登録を競合下でも原子的に確定する。満杯時も到着は成功し、ArrivalEventの`replay_status=skipped`、`skip_reason=queue_full`を返す。30秒以内に再生を開始できなければ`skip_reason=timeout`で`skipped`へ進める。`pending`写真の短い待機もこの30秒に含め、写真の有無で到着を拒否しない。キューに並べられなかった参加者にも本人用ストーリーと削除は提供する。`status`はArrivalEventの現在の`replay_status`（`queued`、`playing`、`completed`、`skipped`）であり、再読込時は状態取得APIから復元する。これらは実装目標であり、実機・負荷試験（#15）の合格を意味しない。

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

## 写真の端末保存とメタデータ（新契約、未実装）

参加者のWeb Appは撮影画像の向き補正・EXIF除去・縮小・再エンコードを端末内で行い、同じブラウザのIndexedDBに保持する。画像Blob、ハッシュ、寸法・画像URLはBackendへ送らない。写真を端末へ保存できない場合は写真の登録を完了したと表示せず、容量・権限の確認または保存失敗を案内する。Backendへ送るのは匿名セッションに属する写真ID・`checkpointId`・撮影時刻だけである。現行Backendのクラウド写真APIはこの新契約の実装ではない。

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

`checkpointId`は必須とし、セッション作成時に固定した`campusGraphVersion`でBackendが検証する。未選択・無効な地点は`400 INVALID_CHECKPOINT`。同じセッションの`clientPhotoId`と同内容の再送は同じPhotoMomentを返し、異なる内容は409、削除済みIDの再送は404として復活させない。端末は加工済み画像のローカル保存後にメタデータを送信する。オフラインならメタデータだけを端末内で再送待ちにし、`expiresAt`まで同じIDで再送する。期限切れで未送信ならサーバーの経路に現れず、画像を端末から保存または削除できるよう案内する。

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

## 本人用の写真ストーリー（新契約、未実装）

### `GET /v2/memory-sessions/{memorySessionId}/memory`

本人Bearerで写真メタデータ・写真地点・経路を時系列に返す。画像とその取得URLは返さない。Backend #7の経路生成まで`segments`は空配列であり、下の非空例を現行提供済みとは扱わない。写真間の地点を撮影順につなぎ、最後の写真からShibaLabまで同じ線で再生する。

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

旧Backend #6の写真実装では、`photoSummary`の`ready`/`pending`/`failed`を旧クラウド写真のMemorySession実件数から算出し、`maxCount`は12とする。`deleting`は旧写真一覧から確認し、保持中の上限には含める。この旧例の`ready`は新しい端末限定写真の存在・内容をBackendが検証したことを意味しない。新方式の本人画面は`GET /v2/.../photos`でメタデータを照会し、画像の有無は同じ端末内で判定する。`status`は`main`、`arrived`、`ended`のいずれか（[状態遷移](state-machine.md)参照）。通知・到着キュー・新写真APIの未実装項目はこの例に登場するだけで提供済みとみなさない。
`replayStatus`はArrivalEventの現在状態であり、キュー満杯・30秒の待機超過時も`skipped`を返す。到着がまだ成立していない場合は`null`とする。写真ストーリーと本人削除は演出スキップ後も`deleteAfter`まで利用できる。ただし新方式の画像は同じブラウザ内に残る場合に限り閲覧でき、旧写真の本人取得・削除は各`deleteAfter`まで維持する。

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
描画・運営資格情報の発行経路は設計Issue #11で保留中。既存の「短期トークン」「外向きWSS」は資格情報をブラウザで安全に発行・保管できることを保証しない。発行者・配送経路・スコープ・有効期限・失効・監査・参加者経路との分離がレビューで確定するまで、本番の運営/描画入口を開放しない（Backend #9/#11）。到着QRの発行入口も同じ判断を要する。

## 描画WebSocket

### `GET /v1/display/ws`

短期の描画トークンで認証する。描画PCはthree.jsで描画し、Frontendリポジトリ`/display`がそのまま描画クライアントである（正式採用、Issue #12）。描画PCのブラウザからGo（Cloud Run）へ外向きWebSocketで直接接続し、Worker・会場PC上のagentは中継しない。接続直後に現在状態のスナップショットを送信し、その後に差分イベントを配信する。到着演出を進める主描画機は同時に一台だけリースを取得できる。待機描画機はスナップショットと通常表示を受信できるが、演出完了を確定できない。

ブラウザの標準`WebSocket` APIは現Backendが接続時に要求する`Authorization`と版の独自HTTPヘッダーを設定できない。直接接続を選んだことは、ブラウザ認証が実装済みという意味ではない。運営者本人確認、短期資格の発行・失効・更新、ブラウザで使える接続時認証、版照合、接続元制限、自動再接続の契約は設計Issue #11とFrontend #36で確定し、疎通・拒否試験まで描画WSSを一般公開しない。本人確認方式は展示接続時まで保留する。

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

- `arrival.replay.requested`
- `photo.moment.created` / `photo.moment.deleted`: 到着が再生待ち・再生中の場合、`arrivalId`、更新後の全`pathNodeIds`、撮影順の全`photoCheckpoints`を渡す。写真単位のIDは送らず、同じ地点・同時刻の写真がある場合も到着単位で全置換する。到着前の写真変更は`participant.arrived`でまとめて通知する
- `system.fallback.changed`（運営切替の実装待ち）

旧`position.updated`、`position.expired`、`tracking.stopped`、Live位相イベント、写真ピン・フォトスポット系イベント（`photo.marker.*`、`photo.hotspot.*`）は廃止した（2026-10-26決定、#23）。旧Liveの本人照会と削除は保持期限まで続けるが、描画用snapshotの`participants`と位置差分は返さない。

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
