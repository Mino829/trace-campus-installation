# UML図・システムフロー（完成版）

更新日: 2026-11-06

Frontend `be94bd2`／Backend `6b87961`（schema 20）の構造と主要シーケンスを示す。存在しないGPS送信、写真pin／hotspot、Unity／TouchDesignerを完成機能として描かない。実装存在と配備・承認・実機試験の完了を区別する。

配置境界は[システム構成](architecture.md#全体構成)、物理FKは[データモデル](data-model.md#er図)、状態図は[状態遷移](state-machine.md)、wire形式は[API仕様](api.md)に集約する。本書で同じ配置図・ER図・状態図を重複定義しない。

## 図の一覧

| 図 | 答える質問 |
|---|---|
| コンポーネント図 | 実在するどのモジュールが入出力・正本更新・描画を担当するか |
| 写真保存・作品閲覧シーケンス | 画像本体を送らず、ローカル作品を到着と独立して開く流れは何か |
| Google認証・初回到着・本人再視聴シーケンス | 資格・proof・初回到着・現再生世代・FIFOをどう分離するか |
| 停止・削除シーケンス | 停止で本人権利を残し、旧画像を安全に消す流れは何か |

## コンポーネント図

実在ファイルの責務依存を表す。矢印はモジュール間の利用・データ受渡しで、すべてが直接ネットワーク接続するという意味ではない。実通信はWorker／直接WSSの境界を通る。

```mermaid
flowchart LR
    subgraph Frontend[Frontend]
        Screen["ping-log-screen.tsx\n参加・撮影・metadata連携"]
        Local["ping-log/store.ts\nIndexedDB entries"]
        Arrival["ray/arrival.tsx\n作品閲覧とQR意図の分離"]
        Client["trace-api/client.ts\n本人REST呼出し"]
        Operator["display/operator-session.ts\n個人運営資格"]
        Renderer["display/renderer-link.ts\nWSS・cursor・lease"]
        Player["display/player.ts\nthree.js描画"]
        Scene["display/campus-graph.ts\nroute-path.ts・固定Graph"]
        Screen --> Local
        Screen --> Client
        Screen --> Arrival
        Arrival --> Client
        Renderer --> Player
        Player --> Scene
    end
    subgraph Backend[Backend]
        API["api/server.go・photo_moments.go\n入力・本人認証・応答"]
        Auth["api/operator_auth.go\napi/display_auth.go"]
        WS["api/display.go\nWSS認証・message処理"]
        Moments["store/photo_moments.go\nmetadata・経路・fence"]
        Arrivals["store/arrivals.go\narrival_replay.go・FIFO・proof消費"]
        Display["store/display.go\nsnapshot・連番・主lease"]
        Routes["route/route.go\n固定Graph検証・経路探索"]
        API --> Moments
        API --> Arrivals
        Moments --> Routes
        Arrivals --> Display
        Moments --> Display
        WS --> Display
    end
    Client --> API
    Operator --> Auth
    Renderer --> WS
```

運営用operatorTokenとWSS用displayTokenの発行は独立。Graphは同梱JSONで、DBのCampusNode／CampusEdge表ではない。DBの行・所有関係はデータモデルを参照する。

## シーケンス図

### 写真保存・作品閲覧

```mermaid
sequenceDiagram
    autonumber
    actor Person as 参加者
    participant UI as /log
    participant Local as 端末IndexedDB
    participant Edge as サイトWorker /api/backend
    participant API as Go本人API
    participant DB as PostgreSQL

    UI->>Edge: GET /v1/public/config
    Edge->>API: Edge secret付き中継
    API-->>UI: 同意本文・版・機能・固定Graph版
    UI-->>Person: 同意本文と保存境界を表示
    Person->>UI: 明示同意して参加
    UI->>Edge: POST /v1/experiences（Live=false）
    Edge->>API: 参加作成
    API->>DB: Memory・Experience、同意版・Graph版を固定
    API-->>UI: Memory Bearer・参加期限・削除期限
    Person->>UI: 撮影・建物／階／地点を選ぶ
    UI->>Local: 再エンコードJPEG・PhotoPlaceを保存
    alt 到着前の送信対象写真
        UI->>Edge: POST /v2/.../photos（3項目だけ）
        Edge->>API: clientPhotoId・checkpointId・capturedAt
        API->>DB: metadata・経路・receiptを同transactionで確定
        API-->>UI: photoMomentId・sequenceNo等（画像URLなし）
        UI->>Local: metadata対応IDを保持（JPEGは残す）
    else 到着後に撮った写真
        UI->>Local: photoMomentId=null、画像と地点は端末内のみ
    end
    Person->>UI: 作品を見る
    UI->>Local: 本人の写真と地点を読む
    UI->>UI: 同梱Graphで3D経路・作品画像を構成
    UI-->>Person: 到着前から閲覧・JPEG保存
    Note over UI,DB: 作品閲覧はカメラ・到着登録・再予約・通知停止を伴わない
```

写真本体はEdge、Go、GCS、展示PCへ送らない。サーバー到着終端と独立したローカルShibaLabへの作品経路を利用できる。写真0枚では作成できない案内を出す。API失敗を端末だけの参加成功で代替しない。

### Google認証・初回到着・本人再視聴

```mermaid
sequenceDiagram
    autonumber
    actor Operator as 許可された展示担当者
    participant Display as /display three.js
    participant Bridge as 本番 /api/operator
    participant API as Go API / WSS
    participant DB as PostgreSQL
    participant Phone as 参加済み /log
    participant Edge as /api/backend

    Operator->>Display: Googleログイン
    Display->>API: 直接POST display-tokens（Google ID token）
    API-->>Display: 独立displayToken（2分）
    Display->>Bridge: POST operator/credentials（Google ID token）
    Bridge->>API: POST /v1/operator/credentials + Edge secret
    API->>DB: 個人資格hash・actor／メール照合hash
    API-->>Display: operatorToken（2分・Origin束縛）
    Display->>API: 直接WSS接続・renderer.authenticate
    API-->>Display: display.snapshot（cursor・arrivals・fallback）
    Display->>API: renderer.primary.acquire
    API->>DB: 単一主lease取得（30秒）
    API-->>Display: renderer.primary.granted（leaseId）
    Display->>Bridge: GET arrival-token（operator Bearer）
    Bridge->>API: POST arrival-proofs + Edge secret
    API-->>Display: checkpointToken（25秒）
    Display-->>Phone: 同一origin /log URLのfragmentをQR表示
    Phone->>Edge: POST /v1/.../arrival（Memory Bearer・key・proof）
    Edge->>API: Edge secret付き中継
    API->>DB: 署名検証後、到着・proof消費・receipt・経路・通知方針を確定
    API-->>Phone: 初回root arrivalId・queued
    API-->>Display: participant.arrived（現世代全metadata）
    API-->>Display: arrival.replay.requested
    Display->>API: arrival.replay.started（eventId・現世代ID・leaseId）
    API->>DB: 有効lease・FIFO先頭・写真待機・単一playingを検証
    API-->>Display: renderer.ack / 差分started
    Display->>Display: pathNodeIdsと撮影地点metadataで再生
    Display->>API: arrival.replay.completed
    API->>DB: 現世代completed・receipt・差分確定
    API-->>Phone: 本人GETでcompletedを取得
    Note over Display,Phone: 終了画面4秒とQR非表示はFrontendの制御
    Phone->>Phone: 本人が新しいQRの読取を明示
    Phone->>Edge: POST /v1/.../arrival/replay（別scopeのkey・新proof）
    Edge->>API: 再視聴要求
    API->>DB: completed・Memory有効・空枠・未使用proofを確認
    API->>DB: 新replay_id・queued_atでFIFO末尾へ、receiptを保存
    API-->>Phone: 新公開arrivalId・queued
    API-->>Display: 新世代participant.arrived → requested
    Note over API,DB: 初回root ID・arrived_at・経路終端・Memory期限・通知方針は不変
```

上図のHTTP応答はWorker経由を含む論理応答の省略表現。直接描画token／WSSにEdge secretを渡さない。運営資格を拒否されても描画だけ許可される場合はその権限範囲で継続し、偽QRを作らない。

同じ初回arrivalの回復は初回IDと現在状態を返すだけ。本人再視聴receiptの同キー・同body再送は保存済み成功応答で、新規予約しない。使用済みproofはMemoryごと再消費できず、別Memoryの同QR利用は可能。キュー満杯／競合はproof消費もreceiptもロールバックする。

待機接続は主leaseを持たないため、再生確定できない。playing中切断・lease失効はrenderer_failure skipで、別機への自動再生引継ぎはしない。運営障害retryだけが新世代を末尾へ入れる。

### 参加者による停止・データ削除

```mermaid
sequenceDiagram
    autonumber
    actor Person as 本人
    participant UI as 本人ブラウザ
    participant Edge as /api/backend
    participant API as Go API
    participant DB as PostgreSQL
    participant GCS as 旧非公開GCS
    participant Local as 端末保存

    Person->>UI: 参加を終了
    UI->>Edge: POST /v1/memory-sessions/id/stop（空body・key）
    Edge->>API: Memory本人認証
    API->>DB: stopped_at固定・Push消去・再生skip・描画除外
    API-->>UI: ended・stoppedAt・元のexpiresAt/deleteAfter
    Note over UI,DB: 参加再開なし、期限内の本人閲覧・削除は維持
    Person->>UI: 本人データを削除
    UI->>Edge: DELETE Memory（本人Bearer・key）
    Edge->>API: 削除要求
    alt 旧GCS画像あり
        API->>DB: deletingフェンス確定
        API->>GCS: 旧画像消去・upload fence維持
        alt 消去失敗
            API-->>UI: 失敗、同じkeyで再試行
            Note over API,GCS: 対象行・keyを失わずdeletingから再試行
        else 消去確認成功
            API->>DB: Memoryと所有データを削除・短期削除receipt
            API-->>UI: 204
            UI->>Local: 対象端末写真・本人資格を消去
        end
    else device metadataのみ
        API->>DB: Memory・metadata・経路・到着・receipt・proof消費等を削除
        API-->>UI: 204
        UI->>Local: 対象端末写真・本人資格を消去
    end
    Note over DB,Local: サーバー期限清掃だけでは端末画像・本人書出しJPEGを消せない
```

旧Liveは別本人Bearerで照会・停止・削除する。Memoryを削除しても旧Live自身のdeleteAfterまでの本人権利を失わせない。一括削除だけはMemory＋Live両資格を要求し、Live成功後にMemoryを別確定する。HTTP 200のpendingは未完了で、同じキー・両資格で未完了側を再試行する。

## 図と実装で共通に守ること

- GPSを新規取得・送信せず、旧データの本人権利と清掃を残す。
- 参加・描画・個人運営の資格を混ぜず、Edge secret・proof署名鍵はサーバー側だけに置く。
- 本番operator bridgeは資格発行・失効・proof取得だけ。status／受付停止／全通知停止／fallback／skip／retryはこのbridgeに存在せず、別の認可済み運営経路を必要とする。
- 画像本体は端末内、展示は版付きnode ID列と撮影地点・時刻metadata。公開差分に本人資格・Memory ID・画像URL・生GPSを載せない。
- 初回到着rootと公開再生世代を分離し、現世代以外の開始・完了を受理しない。通常待機30秒skipや毎回手動playは使わない。
- snapshotで全置換し、DB連番差分と制御応答IDを区別する。欠番を推測で埋めない。
- 新同意版で作ったMemoryの到着後通知方針を固定し、既存Memoryの同意・停止・期限を変更しない。
- 会期終了から現在の全停止・削除完了・撤収を推定しない。図は実行済み試験結果ではない。

## 実装参照

- [Frontend画面・本番境界](https://github.com/Mino829/TRACE_F/blob/be94bd2/README.md)、[Workerの限定経路](https://github.com/Mino829/TRACE_F/blob/be94bd2/push-worker/src/site.ts)
- [端末写真保存](https://github.com/Mino829/TRACE_F/blob/be94bd2/src/lib/ping-log/store.ts)、[作品閲覧／到着意図](https://github.com/Mino829/TRACE_F/blob/be94bd2/src/components/ping-log/ray/arrival.tsx)
- [本人API](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/server.go)、[v2 metadata](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/photo_moments.go)
- [初回到着](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/arrivals.go)、[再視聴](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/arrival_replay.go)、[主lease・FIFO・snapshot](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/display.go)
- [Memory停止](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/memory_sessions.go)、[旧画像削除・fence](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/photo/service.go)
