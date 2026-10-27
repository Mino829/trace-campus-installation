# UML図・システムフロー

この資料は、TRACEの設計書をもとに、利用者、各端末、フロントエンド、バックエンド、DB、描画システムの役割とデータの流れをMermaidで整理したものです。

本資料はGPSを用いない仕様（写真地点は参加者が3Dマップで選ぶ `checkpointId` のみ、MemorySession単独セッション）に沿って改訂した（2026-10-26決定、Issue #23）。通信契約は[API・WebSocket仕様](api.md)、状態の正本は[参加追跡・到着演出の状態遷移](state-machine.md)、保存内容は[データモデル](data-model.md)を優先します。実装中に仕様を変更した場合は、該当する設計書と本資料を同じ変更単位で更新します。

## 図の一覧

| 図 | 答える質問 | 優先度 |
|---|---|---:|
| ユースケース図 | 誰が何を操作するのか | 高 |
| 配置図 | どの端末で何が動くのか | 高 |
| コンポーネント図 | 各ソフトウェアが何を担当するのか | 高 |
| シーケンス図 | 端末間でデータがどの順番に流れるのか | 高 |

## ユースケース図

カメラを許可しない来場者は参加せず、展示を観覧できる前提です。位置情報の同意・位置権限は要求しません。カメラは写真撮影と到着QRの読取のために用います。旧草案版`2026-memory-photo-no-gps-draft-1`を新方式に再利用せず、同意文の正本・新しい版は#9で関係者相談・大学審査後に決めます。

```mermaid
flowchart LR
    Participant["参加者"]
    Visitor["参加しない来場者"]
    Operator["運営者"]
    PrimaryRenderer["主描画PC（Frontend /display）"]
    StandbyRenderer["待機描画PC"]

    subgraph TraceSystem["TRACEシステム"]
        UC01(["同意文（写真・地点・保存期間）を確認する"])
        UC02(["匿名のMemorySessionを開始する"])
        UC03(["写真を撮り3Dマップで地点を選ぶ"])
        UC04(["参加状態と保存期限を確認する"])
        UC05(["撮影通知を停止する"])
        UC06(["保存済みデータを削除する"])
        UC07(["到着QRでチェックインする"])
        UC08(["写真を端末へ保存する"])
        UC11(["展示を観覧する"])

        UC20(["参加キュー・到着・描画状態を監視する"])
        UC21(["新規受付を停止する"])
        UC22(["到着演出を開始・スキップする"])
        UC23(["フォールバックへ切り替える"])

        UC30(["写真地点・経路・到着イベントを受信する"])
        UC31(["写真地点を繋いだ経路を再生する"])
        UC32(["再生開始・完了・失敗を通知する"])
        UC33(["スナップショットから表示を復元する"])
    end

    Participant --> UC01
    Participant --> UC02
    Participant --> UC03
    Participant --> UC04
    Participant --> UC05
    Participant --> UC06
    Participant --> UC07
    Participant --> UC08
    Participant --> UC11

    Visitor --> UC11

    Operator --> UC20
    Operator --> UC21
    Operator --> UC22
    Operator --> UC23

    PrimaryRenderer --> UC30
    PrimaryRenderer --> UC31
    PrimaryRenderer --> UC32
    PrimaryRenderer --> UC33
    StandbyRenderer --> UC30
    StandbyRenderer --> UC33
```

### 権限上のポイント

- 参加者、運営者、描画PCは別々の資格情報を使用する
- 待機描画PCは通常配信とスナップショットを受信できるが、演出完了を確定できない
- 運営操作は運営者だけが実行でき、操作履歴を残す
- 参加に位置情報の同意を要求しない。位置権限を使わない
- 色・記号・`display_alias`・スマホ↔展示の照合表示は廃止した（2026-10-26、Issue #23）

## 配置図

技術方針はCloudflareサイトWorkerでNext.js静的Webと参加者APIの中継、Google Cloud東京でCloud RunとCloud SQL for PostgreSQL 17を採用する。追加の写真画像は端末内IndexedDBだけに置き、正本DBには写真ID・地点・撮影時刻と経路を保存する設計目標（未実装・承認待ち）である。非公開Cloud Storageは旧クラウド写真の本人照会・削除・各`deleteAfter`での清掃が終わるまで保持し、新規写真には使わない。大学の提供者・処理地域・費用承認と公開経路の検証は未完了。Cloudflareの処理・ログは東京保存の保証対象ではなく、会場PC上のBackendは開発・承認済み緊急構成に限定する。

```mermaid
flowchart TB
    subgraph ParticipantDevice["参加者スマートフォン"]
        MobileBrowser["ブラウザ<br/>Next.js Web App<br/>カメラ・3Dマップ地点選択"]
        MobileStorage["localStorageの参加トークン<br/>IndexedDBの写真・メタデータ再送待ち"]
        MobileBrowser --- MobileStorage
    end

    subgraph Cloudflare["Cloudflare"]
        NextServer["サイトWorker<br/>Next.js静的Web / API中継"]
    end

    subgraph PublicEnvironment["Google Cloud東京（技術方針、公開承認前）"]
        GoBackend["Cloud Run<br/>Go REST API・WebSocket"]
        MainDB[("正本DB<br/>Cloud SQL PostgreSQL 17")]
        PhotoStore[("旧写真専用private bucket<br/>Cloud Storage（期限清掃まで）")]
        NextServer -->|"許可されたAPI + edge secret"| GoBackend
        GoBackend --- MainDB
        GoBackend --- PhotoStore
    end

    subgraph Venue["展示会場"]
        subgraph DisplayComputer["主描画PC<br/>（Frontendリポジトリ /display）"]
            Renderer["three.js描画クライアント"]
        end

        subgraph StandbyComputer["バックアップ描画PC"]
            StandbyRenderer["待機描画または保存済み映像"]
        end

        subgraph OperatorComputer["運営ノートPC"]
            Dashboard["運営ダッシュボード"]
        end

        Pi["Raspberry Pi 4B<br/>死活監視・外部制御候補"]
        Projector["プロジェクター"]
        Audio["音響"]
        Lighting["照明・LED候補"]
    end

    MobileBrowser -->|"HTTPS：画面取得"| NextServer
    MobileBrowser -->|"HTTPS：同一オリジン /api/backend/*"| NextServer
    Dashboard -->|"HTTPS：監視・運営操作"| GoBackend
    Renderer <-->|"WSS：描画PCから外向き接続"| GoBackend
    StandbyRenderer <-->|"WSS：待機接続"| GoBackend
    Pi -.->|"HTTPS：死活監視候補"| GoBackend
    Renderer -->|"HDMI / DisplayPort"| Projector
    Renderer -->|"Audio"| Audio
    Renderer -.->|"Control"| Lighting
    StandbyRenderer -.->|"障害時の代替出力"| Projector
```

### 配置上のポイント

- スマートフォンはCloudflareサイトWorkerの同一オリジンAPIへ外向き接続する。描画WSSは描画PCからGo（Cloud Run）への外向きWebSocket直接接続とし、Workerは中継しない（2026-10-26決定）
- 学内ネットワークから描画PCへの受信ポート開放を前提としない
- Cloud SQL PostgreSQL 17を新規写真メタデータ・経路の正本にする。非公開Cloud Storageは旧写真の本人閲覧・削除・期限清掃だけに使う。大学承認・予算承認・新しい同意文承認まで実データ受付を行わない
- 描画ソフトはthree.jsで決定済み（2026-10-26、Issue #12）。Frontendリポジトリの`/display`がそのまま描画クライアント
- Go BackendはOCI互換コンテナとする。AWS等への変更は別途決定する
- 主描画PC停止時はバックアップ描画または合成映像へ切り替える

## コンポーネント図

```mermaid
flowchart LR
    subgraph WebApp["Next.js Web App"]
        EntryUI["入口・同意UI<br/>新方式の承認済み文面（未決）"]
        PhotoUI["撮影・3Dマップ地点選択UI"]
        ArrivalUI["到着・完了UI"]
        PhotoQueue["端末画像＋画像なしのメタデータ再送待ち<br/>上限内"]
        SessionCoordinator["セッション復元・複数タブ制御"]
        PhotoUI --> PhotoQueue
        SessionCoordinator --- PhotoUI
        SessionCoordinator --- ArrivalUI
    end

    subgraph Backend["Go Backend"]
        PublicConfig["公開設定・同意文"]
        SessionService["MemorySession発行・認証<br/>旧LivePresenceSessionの照会・停止・削除"]
        PhotoService["写真検証・保存"]
        CheckpointService["checkpointId検証・<br/>座標と対応Graphノードの解決"]
        RouteService["CampusGraph最短経路<br/>写真地点を撮影順に接続・ShibaLab到着区間"]
        ArrivalService["到着証明検証・キュー・再生状態"]
        OperatorService["運営API・監査"]
        RealtimeService["描画WSS<br/>snapshot・点ID差分配信"]
        RetentionService["失効・期限削除"]

        PhotoService --> CheckpointService
        CheckpointService --> RouteService
        RouteService --> RealtimeService
        ArrivalService --> RealtimeService
        OperatorService --> ArrivalService
    end

    subgraph Storage["保存領域"]
        DB[("MemorySession・写真メタデータ・地点・経路<br/>ArrivalEvent・削除冪等性記録")]
        Audit[("OperatorAuditLog")]
    end

    subgraph Display["描画PC Frontend /display（three.js）"]
        WSClient["WebSocketクライアント<br/>heartbeat・再接続"]
        RouteView["写真地点経路の光表現"]
        Replay["到着時の経路再生"]
        Fallback["自律・合成表現"]
        WSClient --> RouteView
        WSClient --> Replay
        WSClient --> Fallback
    end

    subgraph Operation["運営ダッシュボード"]
        Monitor["接続・キュー・描画監視"]
        Controls["受付停止・再生・skip・fallback"]
    end

    EntryUI -->|"GET config・POST session"| PublicConfig
    EntryUI --> SessionService
    PhotoQueue -->|"POST photo + checkpointId"| PhotoService
    EntryUI -->|"GET state・POST stop・DELETE"| SessionService
    ArrivalUI -->|"POST arrival・GET state"| ArrivalService
    SessionCoordinator --> SessionService

    SessionService --> DB
    PhotoService --> DB
    CheckpointService --> DB
    RouteService --> DB
    ArrivalService --> DB
    RetentionService --> DB
    ArrivalService --> Audit
    OperatorService --> Audit

    Monitor -->|"GET operator/status"| OperatorService
    Controls -->|"運営操作"| OperatorService
    RealtimeService <-->|"WSS・グラフ版＋点ID列＋撮影時刻"| WSClient
```

### コンポーネント間の責務

- Web Appは同意、写真画像の端末内保存と3Dマップ地点選択、写真ID・地点・撮影時刻だけの送信、参加者への状態表示、端末とサーバーの削除操作を担当する。座標・ノードID・グラフ版や画像をpayloadへ含めない
- Go Backendは認証、checkpointIdの検証と座標・ノード解決、最短経路再構成、保存、描画配信を担当する
- DBはMemorySession単独の参加状態と到着演出状態を保存する。旧LivePresenceSessionは`deleteAfter`までの期限清掃だけが対象
- 描画PCは点ID列・グラフ版・撮影時刻から表示を生成し、原写真・画像URL・生座標・生GPSを受け取らない
- 運営画面は監視と運営操作だけを担当する

## シーケンス図

### 参加から写真地点送信・到着再生まで

以下のブラウザ↔Go間のAPI矢印は業務フローの省略表現。実通信はすべてブラウザ→CloudflareサイトWorkerの同一オリジン`/api/backend/*`→Goであり、Workerがedge secretを付与する。描画WSSは描画PCからGo（Cloud Run）への外向きWebSocket直接接続とする（2026-10-26決定）。Workerは中継せず、描画用資格情報の発行契約はIssue #11で整備する。

```mermaid
sequenceDiagram
    autonumber
    actor User as 参加者
    participant Browser as スマートフォン<br/>Next.js Web App
    participant Front as Cloudflare<br/>サイトWorker
    participant API as Go Backend<br/>（参加者APIはWorker経由）
    participant DB as 正本DB
    participant Renderer as 描画PC<br/>three.js（/display）
    participant Output as プロジェクター<br/>音響・照明
    actor Operator as 運営端末

    Renderer->>API: WSS接続（描画用の短期トークン）
    API->>DB: 現在の参加・到着状態を取得
    DB-->>API: 現在状態
    API-->>Renderer: display.snapshot<br/>グラフ版＋点ID列

    User->>Browser: 入口の参加QRを読み取る
    Browser->>Front: 参加ページを取得（HTTPS）
    Front-->>Browser: Web Appを配信
    Browser->>API: GET /v1/public/config（Worker中継）
    API-->>Browser: 受付状態・承認済みの新同意文<br/>（版・正本は#9で相談中）
    Browser-->>User: 目的・写真地点・保存期間・停止・削除方法を表示
    User->>Browser: 内容に同意して開始
    Browser->>API: POST /v1/memory-sessions（Worker中継）<br/>同意版と言語
    API->>DB: MemorySessionを作成
    DB-->>API: 保存完了
    API-->>Browser: 201 sessionId・トークン・TTL・deleteAfter

    Note over Browser: 同意完了後に初めてカメラ権限を要求<br/>（地点選択と到着QR読取のため。位置権限は使わない）
    User->>Browser: カメラを許可

    loop 写真を撮るたび
        User->>Browser: 撮影し3Dマップで地点（チェックポイント）を選ぶ
        Browser->>Browser: 向き補正・EXIF除去して<br/>IndexedDBへ画像を保存
        Browser->>API: POST /v2/memory-sessions/{id}/photos<br/>clientPhotoId・checkpointId・capturedAtのみ
        API->>API: checkpointIdを検証<br/>画像バイト列は受け付けない
        API->>DB: PhotoMoment・地点・撮影時刻を保存
        API->>API: CampusGraphで前回地点からの<br/>最短経路を確定
        API-->>Renderer: 経路差分（グラフ版＋点ID列）
        API-->>Browser: メタデータ登録の成否
        Browser-->>User: 端末画像と登録結果を区別して表示
    end

    User->>Browser: 展示ブースに投影された到着QRを読み取る
    Browser->>API: POST /v1/memory-sessions/{id}/arrival<br/>Idempotency-Key・署名付き到着証明
    API->>API: 署名・会場・有効期限・再利用を検証
    API->>DB: ArrivalEventをqueuedで作成
    DB-->>API: arrivalId・キュー状態
    API-->>Renderer: participant.arrived<br/>写真地点の点ID列・撮影時刻・ShibaLab到着区間
    API-->>Browser: arrivalId・到着状態
    Note over API,DB: 写真0枚でも到着は成立する。<br/>再生は簡易表示へ縮小（2026-10-26決定）
    Browser-->>User: 完了状態・自動削除予定日時を表示

    Operator->>API: GET /v1/operator/status
    API->>DB: 到着キューと描画機状態を取得
    DB-->>API: 運営用状態
    API-->>Operator: キュー・描画機状態

    Operator->>API: POST /v1/operator/arrivals/{arrivalId}/play
    API-->>Renderer: arrival.replay.requested<br/>主描画機リース対象のみ
    Renderer->>API: arrival.replay.started<br/>eventId・arrivalId・leaseId
    API->>DB: replay_statusをplayingへ更新
    Renderer->>Output: 写真地点を最短経路で繋いだ経路を<br/>10〜20秒・区間長の比例配分で再生
    Renderer->>API: arrival.replay.completed
    API->>DB: replay_statusをcompletedへ更新

    Browser->>API: GET /v1/memory-sessions/{id}
    API->>DB: 到着・再生状態を取得
    DB-->>API: arrived/stopped・completed・deleteAfter
    API-->>Browser: 位置点を含まない現在状態
    Browser-->>User: 完了状態・削除予定日時・<br/>写真ストーリーの端末保存を表示
```

### 参加者による撮影通知停止・データ削除

到着で撮影通知は自動停止する。ここでは任意の停止と削除を扱います。

```mermaid
sequenceDiagram
    autonumber
    actor User as 参加者
    participant Browser as スマートフォン<br/>Next.js Web App
    participant API as Go Backend
    participant DB as 正本DB
    participant Renderer as 描画PC<br/>three.js（/display）

    alt 撮影通知だけを停止する
        User->>Browser: 「通知を停止」を選択
        Browser->>API: Web Push購読の解除<br/>Idempotency-Key
        API->>DB: 購読を解除し送信予定を削除
        API-->>Browser: 解除済みの現在状態
        Browser-->>User: 通知停止済み・写真メタデータ登録継続を表示
    else 保存済みデータを削除する
        User->>Browser: 「今すぐ削除」を選択して確認
        Browser->>Browser: 同じブラウザの写真画像と<br/>再送待ちを消去
        Browser->>API: DELETE MemorySession<br/>Memory用Bearer token・Idempotency-Key
        API->>DB: 写真メタデータ・地点・経路・到着と<br/>削除冪等性記録を削除
        API->>DB: 内容を持たない短期の削除冪等性記録を保存
        DB-->>API: サーバー側の削除完了
        API-->>Renderer: tracking.stopped
        API-->>Browser: 204 No Content
        Browser->>Browser: localStorageのトークンを消去
        Browser-->>User: 端末・サーバー別の削除成否を表示
        Note over Browser,DB: サーバーの204だけで端末画像の消去は証明できない
    end
```

### 通信断と再接続

```mermaid
sequenceDiagram
    autonumber
    actor User as 参加者
    participant Browser as スマートフォン<br/>Next.js Web App
    participant Local as 端末内一時領域<br/>IndexedDB等
    participant API as Go Backend
    participant DB as 正本DB
    participant Renderer as 描画PC<br/>three.js（/display）
    actor Operator as 運営端末

    alt スマートフォンの通信が切断
        Browser-xAPI: 写真ID・checkpointId・撮影時刻の登録失敗
        Browser->>Local: 加工済み画像とメタデータ再送待ちを上限内で保持
        Browser-->>User: 端末画像保存済み・サーバー未受理を区別して表示
        Browser->>API: 通信復旧後、画像なしのメタデータだけ冪等再送
        Browser->>Local: 再送待ちだけ消去し、本人用画像は保持
    else スマートフォンを再読込
        Browser->>API: GET /v1/memory-sessions/{id}<br/>端末内のBearer token
        API->>DB: 状態を取得
        DB-->>API: 到着・再生・期限
        API-->>Browser: 到着・再生・期限の現在状態
        Browser->>API: GET /v2/memory-sessions/{id}/photos
        API-->>Browser: 写真ID・checkpointId・撮影時刻のみ
        Browser->>Local: 写真IDに対応する画像を照合
        Browser-->>User: 状態と画像有無を別に表示し、消失画像は復元不可と案内
    end

    alt 描画機またはWSSが切断
        Renderer-xAPI: heartbeat・WSS切断
        Renderer->>Renderer: 最後の状態から自律表現へ移行
        Operator->>API: GET /v1/operator/status
        API-->>Operator: 描画機をdegradedとして表示
        Renderer->>API: 指数バックオフでWSS再接続
        API->>DB: 現在の参加・到着状態を取得
        DB-->>API: 現在状態
        API-->>Renderer: display.snapshot<br/>生成時点のeventId付き
        Renderer->>Renderer: snapshot以前の差分を二重適用せず復元
        Note over API,Renderer: 版不一致イベントは破棄し、<br/>直前の互換版表示へ戻す
        opt 再生中に主描画機が切断した場合
            Note over API,Renderer: 別端末で自動再開しない
            Operator->>API: skipまたは承認付き再試行
            API->>DB: replay_statusと理由を更新
        end
    end
```

## 図と実装で共通に守ること

- GPS・位置情報API（Geolocation、watchPosition）を参加フローで使用しない
- 同意文はカメラ許可のみを参加条件とし、位置情報の項目を含めない
- 参加者が送るpayloadに座標・対応ノードID・グラフ版を含めない（MemorySession単独方針）
- 参加者の資格情報はMemorySession単独とする。旧LivePresenceSessionは本人照会・停止・削除と期限清掃以外の用途を持たない
- 参加トークンをURL、ログ、描画イベントへ含めず、サーバーにはハッシュだけを保存する
- 描画機へ渡すのは点ID・グラフ版・撮影時刻であり、原写真・画像URL・生座標・生GPSは渡さない
- 到着は展示ブースの署名・期限付きQRのBackend検証でのみ成立し、ArrivalEvent作成と同じトランザクションで撮影通知の停止を登録する
- 到着、削除、描画完了は冪等に処理する
- 主描画機だけが到着演出の開始・完了を確定できる
- 通信断時は端末に写真画像と写真ID・`checkpointId`・撮影時刻のメタデータ再送待ちを上限付きで保持する。画像をネットワークへ再送しない。描画の自律表現と復旧時のスナップショットを使用する
- 本番の正本DBは一つだけとし、会場PC上のBackendと同時書き込みしない

## 統合前に確認が必要な点

以下は、図の作成時に設計書だけでは一意に判断できなかった点です。図では開発を進められるよう暫定的な流れを置いていますが、担当機能を統合する前に関係者で決定し、API仕様と本資料へ反映してください。

| 確認事項 | 現在の図での暫定解釈 | 統合前に決めること |
|---|---|---|
| 到着演出の開始方法 | 運営者が`POST /v1/operator/arrivals/{arrivalId}/play`を実行して開始する | 毎回手動で開始するか、Backendがキューから自動開始するか |
| `participant.arrived`と`arrival.replay.requested`の役割 | 到着時に`participant.arrived`で点ID列・撮影時刻を渡し、再生時に`arrival.replay.requested`を送る | 軌跡を受けた描画機が何をするか、差分とスナップショットの分担 |
| 参加者画面の状態更新 | `GET /v1/memory-sessions/{id}`で到着状態・削除予定日時を、`GET /v2/memory-sessions/{id}/photos`でサーバーメタデータを取得する。端末写真の有無はIndexedDBで判定 | ポーリング間隔、画面更新方法、参加者向け通知を追加するか |
| チェックポイント選択UIの粒度 | 図では3Dマップ上の一覧選択（実装PRは82点）と表記 | ポケモンGO風8か所スポット案との採否、写真地点の再選択（選び直し）の可否。uni-bit・code-yutoと相談中（Mino829/TRACE_F#38・#45） |
| 主描画機リース | 主描画機だけが再生開始・完了を送れるものとしている | リースの取得方法、有効時間、更新、切断時の解放、待機機への切替手順 |
| 描画WebSocketの差分復元 | `display.snapshot`のeventIdを基準に古い差分を捨てる | 採用するカーソル方式、順序逆転・欠落・再送の処理 |
| クラウド提供者 | CloudflareサイトWorker + Google Cloud東京（Cloud Run / Cloud SQL PostgreSQL 17）を新方式の技術方針として採用。非公開Cloud Storageは旧写真の期限清掃まで維持 | 大学の提供者・処理地域・個人情報承認と費用/公開経路の確認。Cloudflare側の東京処理は保証しない |
| PostgreSQL構成 | Cloud SQL for PostgreSQL 17を正本とする | 可用性、instance size、接続上限、7日上限のバックアップ/PITRと削除保証 |
| 描画ソフト | three.js採用済み（2026-10-26、Issue #12）。Frontendリポジトリ`/display` | 残る決定は主描画機リースとWSS認証（Issue #11） |
| Raspberry Pi・音響・照明 | 候補または任意接続として点線で表している | MVPに含める範囲、接続方式、障害時に切り離せる条件 |
| 写真0枚・1枚の到着 | 到着は成立し、再生は簡易表示（スポット名や写真の有無の案内）へ縮小する（2026-10-26決定） | 簡易表示の具体的な絵作り・文言 |

旧契約の位置系の暫定解釈（`on_site`追加同意、使用する精度、ExperienceLink、両セッションDELETEの応答など）は2026-10-26の決定（Issue #23）で廃止したため、本表から削除した。

### 統合時の更新ルール

- 決定した項目は「暫定解釈」のまま残さず、該当する図と設計書へ反映する
- APIまたはイベントを変更した場合は、送信側と受信側の担当者が同じ内容で合意する
- 未決の機能はインターフェースをモック化し、統合時に暗黙の仕様を持ち込まない。GPSに紐づく旧表現は復活させない
