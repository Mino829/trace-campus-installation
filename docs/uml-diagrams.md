# UML図・システムフロー

この資料は、TRACEの設計書をもとに、利用者、各端末、フロントエンド、バックエンド、DB、描画システムの役割とデータの流れをMermaidで整理したものです。

通信契約は[API・WebSocket仕様](api.md)、状態の正本は[参加追跡・到着演出の状態遷移](state-machine.md)、保存内容は[データモデル](data-model.md)を優先します。実装中に仕様を変更した場合は、該当する設計書と本資料を同じ変更単位で更新します。

## 図の一覧

| 図 | 答える質問 | 優先度 |
|---|---|---:|
| ユースケース図 | 誰が何を操作するのか | 高 |
| 配置図 | どの端末で何が動くのか | 高 |
| コンポーネント図 | 各ソフトウェアが何を担当するのか | 高 |
| シーケンス図 | 端末間でデータがどの順番に流れるのか | 高 |

## ユースケース図

位置情報を提供しない来場者も展示を観覧でき、位置取得不能時はQRチェックポイントまたは合成軌跡による代替参加を利用できる前提です。

```mermaid
flowchart LR
    Participant["参加者"]
    Visitor["位置情報を提供しない来場者"]
    Operator["運営者"]
    PrimaryRenderer["主描画機"]
    StandbyRenderer["待機描画機"]

    subgraph TraceSystem["TRACEシステム"]
        UC01(["作品説明・同意内容を確認する"])
        UC02(["匿名セッションを開始する"])
        UC03(["位置情報を送信する"])
        UC04(["追跡状態を確認する"])
        UC05(["追跡を停止する"])
        UC06(["保存済みデータを削除する"])
        UC07(["到着チェックインする"])
        UC08(["待ち順・自分の記号を確認する"])
        UC09(["到着演出をスキップする"])
        UC10(["代替体験へ参加する"])
        UC11(["展示を観覧する"])

        UC20(["追跡数・キュー・描画状態を監視する"])
        UC21(["新規受付を停止する"])
        UC22(["到着演出を開始・スキップする"])
        UC23(["フォールバックへ切り替える"])

        UC30(["現在位置と到着イベントを受信する"])
        UC31(["到着軌跡を再生する"])
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
    Participant --> UC09
    Participant --> UC10
    Participant --> UC11

    Visitor --> UC10
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

- 参加者、運営者、描画機は別々の資格情報を使用する
- 待機描画機は通常表示とスナップショットを受信できるが、演出完了を確定できない
- 運営操作は運営者だけが実行でき、操作履歴を残す
- 観覧や代替体験に位置情報の同意を必須としない

## 配置図

本番のWeb App、Go Backend、正本DBは公開HTTPS環境へ配置します。会場PC上のBackendは開発・承認済み緊急構成に限定し、本番DBと同時に正本として動かしません。

```mermaid
flowchart TB
    subgraph ParticipantDevice["参加者スマートフォン"]
        MobileBrowser["ブラウザ<br/>Next.js Web App<br/>Geolocation API"]
        MobileStorage["sessionStorage<br/>IndexedDB等"]
        MobileBrowser --- MobileStorage
    end

    subgraph PublicEnvironment["公開HTTPS環境"]
        NextServer["Next.js<br/>参加画面の配信"]
        GoBackend["Go Backend<br/>REST API・WebSocket"]
        MainDB[("単一の正本DB<br/>SQLiteまたはPostgreSQL")]
        NextServer --- GoBackend
        GoBackend --- MainDB
    end

    subgraph Venue["展示会場"]
        subgraph DisplayComputer["主描画PC<br/>Mac mini M4またはGPU搭載PC"]
            Renderer["UnityまたはTouchDesigner"]
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
    MobileBrowser -->|"HTTPS：同意・位置・到着・停止・削除"| GoBackend
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

- スマートフォンと描画PCは、どちらも公開環境へ外向きに接続する
- 学内ネットワークから描画PCへの受信ポート開放を前提としない
- SQLiteは単一Backendと永続ディスクを使用できる場合のMVP候補とする
- 複数Backendや一時ディスク、無停止切替が必要ならPostgreSQLを使用する
- 主描画PC停止時はバックアップ描画または合成映像へ切り替える

## コンポーネント図

```mermaid
flowchart LR
    subgraph WebApp["Next.js Web App"]
        EntryUI["入口・説明・同意UI"]
        TrackingUI["追跡状態UI"]
        ArrivalUI["到着・待機・完了UI"]
        PositionWatcher["watchPosition<br/>位置取得"]
        LocalQueue["未送信位置キュー<br/>最大50件"]
        SessionCoordinator["セッション復元・複数タブ制御"]
        EntryUI --> PositionWatcher
        PositionWatcher --> LocalQueue
        SessionCoordinator --- TrackingUI
        SessionCoordinator --- ArrivalUI
    end

    subgraph Backend["Go Backend"]
        PublicConfig["公開設定・同意文"]
        SessionService["匿名セッション・認証"]
        PositionService["位置検証・geofence"]
        MappingService["座標変換・平滑化・量子化"]
        ArrivalService["到着証明・キュー・再生状態"]
        OperatorService["運営API・監査"]
        RealtimeService["描画WebSocket<br/>snapshot・差分配信"]
        RetentionService["失効・期限削除"]

        PositionService --> MappingService
        MappingService --> RealtimeService
        ArrivalService --> RealtimeService
        OperatorService --> ArrivalService
    end

    subgraph Storage["保存領域"]
        DB[("ParticipantSession<br/>PositionPoint<br/>ArrivalEvent")]
        Audit[("IdempotencyRecord<br/>OperatorAuditLog")]
    end

    subgraph Display["Unity / TouchDesigner"]
        WSClient["WebSocketクライアント<br/>heartbeat・再接続"]
        LiveView["現在位置の抽象表現"]
        Replay["到着軌跡の再生"]
        Fallback["自律・合成表現"]
        WSClient --> LiveView
        WSClient --> Replay
        WSClient --> Fallback
    end

    subgraph Operation["運営ダッシュボード"]
        Monitor["接続・追跡・キュー監視"]
        Controls["受付停止・再生・skip・fallback"]
    end

    EntryUI -->|"GET config・POST sessions"| PublicConfig
    EntryUI --> SessionService
    LocalQueue -->|"POST positions"| PositionService
    TrackingUI -->|"GET state・POST stop・DELETE"| SessionService
    ArrivalUI -->|"POST arrival・GET state"| ArrivalService
    SessionCoordinator --> SessionService

    SessionService --> DB
    PositionService --> DB
    MappingService --> DB
    ArrivalService --> DB
    RetentionService --> DB
    SessionService --> Audit
    ArrivalService --> Audit
    OperatorService --> Audit

    Monitor -->|"GET operator/status"| OperatorService
    Controls -->|"運営操作"| OperatorService
    RealtimeService <-->|"WSS・加工済み座標と演出イベント"| WSClient
```

### コンポーネント間の責務

- Web Appは同意、位置取得、参加者への状態表示、停止・削除操作を担当する
- Go Backendは認証、位置検証、座標加工、状態管理、保存、描画配信を担当する
- DBは参加追跡状態と到着演出状態を分けて保存する
- Unity / TouchDesignerは生の緯度経度を受け取らず、表示用データから表現を生成する
- 運営画面は監視と運営操作だけを担当し、生位置を表示しない

## シーケンス図

### 通常参加から到着演出まで

```mermaid
sequenceDiagram
    autonumber
    actor User as 参加者
    participant Browser as スマートフォン<br/>Next.js Web App
    participant Front as 公開HTTPS環境<br/>Next.js
    participant API as Go Backend
    participant DB as 正本DB
    participant Renderer as 描画PC<br/>Unity / TouchDesigner
    participant Output as プロジェクター<br/>音響・照明
    actor Operator as 運営端末

    Renderer->>API: WSS接続（描画用の短期トークン）
    API->>DB: 現在の参加・到着状態を取得
    DB-->>API: 現在状態
    API-->>Renderer: display.snapshot

    User->>Browser: 入口の参加QRを読み取る
    Browser->>Front: 参加ページを取得（HTTPS）
    Front-->>Browser: Web Appを配信
    Browser->>API: GET /v1/public/config
    API-->>Browser: 受付状態・同意文・保存期間・取得間隔
    Browser-->>User: 目的・取得項目・停止・削除方法を表示
    User->>Browser: 内容に同意して開始
    Browser->>API: POST /v1/sessions<br/>同意版・言語・端末時刻
    API->>DB: 匿名セッションを保存<br/>参加トークンはハッシュのみ保存
    DB-->>API: 保存完了
    API-->>Browser: 201 sessionId・参加トークン・alias・期限

    Note over Browser: 同意完了後に初めて位置権限を要求
    Browser-->>User: ブラウザの位置権限を要求
    User->>Browser: 位置利用を許可
    Browser->>Browser: watchPositionを開始

    loop 設定された取得間隔ごと
        Browser->>API: POST /v1/sessions/{sessionId}/positions<br/>Bearer token・位置点バッチ
        API->>API: 時刻・精度・geofence・ジャンプを検証<br/>座標変換・平滑化・量子化
        alt 有効な位置点
            API->>DB: 生位置と表示用ローカル座標を保存
            DB-->>API: 保存完了
            API-->>Renderer: position.updated<br/>ローカル座標・不確実性・aliasのみ
            Renderer->>Output: 現在位置を抽象表現として描画
        else 無効な位置点
            API->>API: 保存・描画配信を行わない
        end
        API-->>Browser: accepted・rejected・拒否理由
        Browser-->>User: 追跡状態と最終送信時刻を表示
    end

    User->>Browser: 展示場入口の到着QRを読み取る
    Browser->>API: POST /v1/sessions/{sessionId}/arrival<br/>Idempotency-Key・署名付き到着証明
    API->>API: 署名・会場・有効期限・再利用を検証
    API->>DB: 同一トランザクションで<br/>追跡停止＋ArrivalEventをqueuedで作成
    DB-->>API: arrivalId・キュー状態
    API-->>Renderer: participant.arrived<br/>arrivalId・alias・加工済み軌跡
    API-->>Browser: queuePosition・待ち時間・alias
    Browser->>Browser: watchPositionと未送信点を終了
    Browser-->>User: 到着待ち状態と自分の色・記号を表示

    Operator->>API: GET /v1/operator/status
    API->>DB: 到着キューと描画機状態を取得
    DB-->>API: 運営用状態
    API-->>Operator: キュー・追跡数・描画機状態
    Operator->>API: POST /v1/operator/arrivals/{arrivalId}/play
    API-->>Renderer: arrival.replay.requested<br/>主描画機リース対象のみ
    Renderer->>API: arrival.replay.started<br/>eventId・arrivalId・leaseId
    API->>DB: replay_statusをplayingへ更新
    Renderer->>Output: 10〜20秒の到着軌跡を再生
    Renderer->>API: arrival.replay.completed
    API->>DB: replay_statusをcompletedへ更新

    Browser->>API: GET /v1/sessions/{sessionId}
    API->>DB: 追跡・再生状態を取得
    DB-->>API: stopped・completed・deleteAfter
    API-->>Browser: 位置点を含まない現在状態
    Browser-->>User: 完了状態と削除予定日時を表示
```

### 参加者による追跡停止・データ削除

追跡停止は新しい位置点を止める操作で、保存済みデータの削除とは分けて扱います。

```mermaid
sequenceDiagram
    autonumber
    actor User as 参加者
    participant Browser as スマートフォン<br/>Next.js Web App
    participant API as Go Backend
    participant DB as 正本DB
    participant Renderer as 描画PC<br/>Unity / TouchDesigner

    alt 追跡だけを停止する
        User->>Browser: 「追跡停止」を選択
        Browser->>Browser: watchPositionを停止<br/>未送信点を消去
        Browser->>API: POST /v1/sessions/{sessionId}/stop<br/>Idempotency-Key・reason
        API->>DB: tracking_statusをstoppedへ更新<br/>delete_afterを前倒し
        DB-->>API: 現在状態
        API-->>Renderer: tracking.stopped
        API-->>Browser: 停止済みの現在状態
        Browser-->>User: 追跡停止済み・保存期限・削除操作を表示
        Note over API,DB: 停止後に届いた位置点は保存しない
    else 保存済みデータを即時削除する
        User->>Browser: 「今すぐ削除」を選択して確認
        Browser->>Browser: watchPositionを停止<br/>未送信点を消去
        Browser->>API: DELETE /v1/sessions/{sessionId}<br/>Idempotency-Key・Bearer token
        API->>DB: セッション・位置点・到着イベント・<br/>トークンハッシュをトランザクションで削除
        API->>DB: 内容を持たない短期の削除冪等性記録を保存
        DB-->>API: 削除完了
        API-->>Renderer: tracking.stopped
        API-->>Browser: 204 No Content
        Browser->>Browser: sessionStorageのトークンを消去
        Browser-->>User: 削除完了を表示
        Note over Browser,DB: 削除後は同じトークンで状態を復元できない
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
    participant Renderer as 描画PC<br/>Unity / TouchDesigner
    participant Output as プロジェクター<br/>音響・照明
    actor Operator as 運営端末

    alt スマートフォンの通信が切断
        Browser-xAPI: 位置送信失敗
        Browser->>Local: 未送信点を上限50件で一時保存
        Browser-->>User: 未送信件数と再送予定を表示
        Note over Browser,Local: 上限超過・期限超過の点は破棄する
        Browser->>API: 通信復旧後に古い順で位置バッチを再送
        API->>API: clientPointIdで重複排除・入力検証
        API->>DB: 有効かつ追跡中の点だけ保存
        DB-->>API: 保存完了
        API-->>Browser: accepted・rejected
        Browser->>Local: 送信済みの点を消去
    else スマートフォンを再読込
        Browser->>API: GET /v1/sessions/{sessionId}<br/>sessionStorageのBearer token
        API->>DB: 状態を取得
        DB-->>API: 追跡・再生・期限・alias
        API-->>Browser: 位置点を含まない現在状態
        Browser-->>User: 追跡中・到着待ち・完了の画面を復元
    end

    alt 描画機またはWSSが切断
        Renderer-xAPI: heartbeat・WSS切断
        Renderer->>Output: 最後の状態から自律表現へ移行
        Operator->>API: GET /v1/operator/status
        API-->>Operator: 描画機をdegradedとして表示
        Renderer->>API: 指数バックオフでWSS再接続
        API->>DB: 現在の参加・到着状態を取得
        DB-->>API: 現在状態
        API-->>Renderer: display.snapshot<br/>生成時点のeventIdまたは連番付き
        Renderer->>Renderer: snapshot以前の差分を二重適用せず復元
        Renderer->>Output: リアルタイム表示へ自然に復帰
        opt 再生中に主描画機が切断した場合
            Note over API,Renderer: 別端末で自動再開しない
            Operator->>API: skipまたは承認付き再試行
            API->>DB: replay_statusと理由を更新
        end
    end
```

## 図と実装で共通に守ること

- 同意完了前にブラウザの位置情報APIを呼び出さない
- 参加トークンをURL、ログ、描画イベントへ含めず、サーバーにはハッシュだけを保存する
- 描画機へ渡すのは加工済みローカル座標であり、生の緯度経度は渡さない
- 到着受付と追跡停止は同じDBトランザクションで処理する
- 追跡状態と到着演出状態は別々に管理する
- 到着、停止、削除、描画完了は冪等に処理する
- 主描画機だけが到着演出の開始・完了を確定できる
- 通信断時は端末内の上限付き保持、描画の自律表現、復旧時のスナップショットを使用する
- 本番の正本DBは一つだけとし、会場PC上のBackendと同時書き込みしない

## 統合前に確認が必要な点

以下は、図の作成時に設計書だけでは一意に判断できなかった点です。図では開発を進められるよう暫定的な流れを置いていますが、担当機能を統合する前に関係者で決定し、API仕様と本資料へ反映してください。

| 確認事項 | 現在の図での暫定解釈 | 統合前に決めること |
|---|---|---|
| 到着演出の開始方法 | 運営者が`POST /v1/operator/arrivals/{arrivalId}/play`を実行して開始する | 毎回手動で開始するか、Backendがキューから自動開始するか |
| `participant.arrived`と`arrival.replay.requested`の役割 | 到着時に`participant.arrived`で加工済み軌跡を渡し、再生時に`arrival.replay.requested`を送る | 軌跡をどちらのイベントへ含めるか、各イベントを受けた描画機が何をするか |
| 参加者画面の状態更新 | 必要なタイミングで`GET /v1/sessions/{sessionId}`を呼び、待ち順・再生完了を取得する | ポーリング間隔、画面更新方法、参加者向けリアルタイム通知を追加するか |
| 追跡停止APIの成功応答 | 状態遷移資料に合わせて、停止済みの現在状態を返す | HTTP statusとresponse bodyの正式な形式 |
| 削除時の描画機通知 | `tracking.stopped`で表示停止とキュー除外を伝える | 削除専用イベントや再生キャンセルイベントが必要か、描画側の消去範囲 |
| 主描画機リース | 主描画機だけが再生開始・完了を送れるものとしている | リースの取得方法、有効時間、更新、切断時の解放、待機機への切替手順 |
| 描画WebSocketの差分復元 | `display.snapshot`のeventIdまたは連番を基準に古い差分を捨てる | 採用するカーソル方式、順序逆転・欠落・再送の処理 |
| 未来時刻の位置点 | 図では「入力検証」とだけ記載し、処理を確定していない | `FUTURE_TIME`として拒否するか、受信時刻へ補正するか。API仕様と座標変換資料で記述が異なる |
| 描画へ渡す誤差情報 | 図では「不確実性」と表記している | `accuracyM`と`uncertaintyM`のどちらを正式名称にするか、生の精度か加工後の値か |
| 到着後の端末内未送信点 | 到着成功後に`watchPosition`と未送信点を終了する | 到着要求と同時に届いた位置バッチの順序、到着失敗時に追跡を継続するか |
| 代替参加の処理 | ユースケースだけに記載し、端末間フローには含めていない | QRチェックポイントのAPI、合成軌跡の生成主体、到着キューへ入れる条件 |
| フロントエンドの配置 | Next.jsとGo Backendを同じ「公開HTTPS環境」に置く | 同一ホスト、別サービス、リバースプロキシ構成のどれにするか |
| 正本DB | 条件に応じてSQLiteまたはPostgreSQLと表記している | 本番で使用するDB、永続ディスク、バックアップ、復旧方法 |
| 描画ソフト | UnityまたはTouchDesignerと併記している | 最終的に使用するソフトと、WebSocketイベントを受け取る実装担当 |
| Raspberry Pi・音響・照明 | 候補または任意接続として点線で表している | MVPに含める範囲、接続方式、障害時に切り離せる条件 |

### 統合時の更新ルール

- 決定した項目は「暫定解釈」のまま残さず、該当する図と設計書へ反映する
- APIまたはイベントを変更した場合は、送信側と受信側の担当者が同じ内容で合意する
- 未決の機能はインターフェースをモック化し、統合時に暗黙の仕様を持ち込まない
- リアルタイム位置の本番利用に関わる未決事項が残る場合は、合成データ表示をフォールバックとする
