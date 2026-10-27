# インフラ・データベース方針

- 決定日: 2026-10-21（Issue #10の技術配置方針）。2026-10-26にIssue #23のGPS不使用・MemorySession単独化と新規写真の端末内保存方針を反映
- 状態: 採用する技術構成と未実装の写真保存設計目標。大学の提供者・処理地域・保存期間承認、予算確認、同意文の正本、公開経路の実証は未完了。配備・受付開始を許可する決定ではない
- 対象: MVP、ステージング、本番。旧LivePresenceSessionと既存のサーバー写真は各`deleteAfter`まで本人照会・削除と期限清掃を維持する

従来のVercel + Google Cloud方針を、稼働中の静的Webと同一オリジン中継を活かすCloudflare Workers + Google Cloud東京へ変更する。[比較の記録](cloudflare-web-backend-option.md)を参照。

## 決定

1. Next.jsを静的書き出しし、参加者画面、`/log`、`/display`のWeb配信先をCloudflareのサイトWorkerとする。画面機能の実装完了を意味しない
2. Go BackendはOCI互換コンテナをGoogle Cloud東京（`asia-northeast1`）のCloud Runに配置する技術方針とする。AWS東京は大学審査等で変更が必要になった場合の再決定先であり、現行の採用先ではない
3. 本番のサーバー側正本DBは東京のCloud SQL for PostgreSQL 17とする。新規写真の画像バイト列は参加者のスマホ内（ブラウザのIndexedDB）のみに保持し、Backendには写真ID・撮影時刻・選択した`checkpointId`等の経路用メタデータを保存する設計目標とする。既存のCloud Storage写真アップロード実装がこの方針に移行済みという意味ではない。既存のサーバー写真とそのobjectキー・本人取得/削除経路は各セッションの`deleteAfter`まで保持・期限清掃する
4. GPSの新規取得・位置送信・リアルタイム位置表示・`LivePresenceSession`の新規発行は行わない。写真地点は参加者が選ぶ`checkpointId`のみを保存し、座標・対応ノードはBackendが解決する。旧Sessionと旧GPSデータは各`deleteAfter`まで保持し、本人資格での照会・停止・削除と期限清掃を継続する。旧データを破棄済みとは扱わない（2026-10-26、#23）
5. 会場のFrontend `/display`（three.js）を描画に使用する。描画PCのブラウザからGo Backendへ外向きWSSで直接接続し、会場PC上のagent・Workerは中継しない。学内ネットワークから描画PCへの受信ポート開放を前提としない。運営者本人確認と短期描画資格情報の発行・保管・更新、ブラウザ用接続認証・版照合・自動再接続は未決であり、展示接続時に決定・実装・検証する

## 採用理由

- 稼働中のCloudflare配信とサイトWorkerの同一オリジンAPI中継を再利用でき、Vercel用の別のWeb実行環境を増やさない
- Go Backendをコンテナ化することで、変更が必要になった場合も移植可能性を残す
- 正本DBをGoogle Cloud東京に置く。新規画像バイト列の保存先は端末内とし、旧サーバー写真のCloud Storageは各`deleteAfter`まで管理する。ただしCloudflareを通る通信・ログや調査用D1/KVの処理・保存まで東京と保証するものではない
- SQLiteの単一ディスク制約を避け、複数Backend、接続プール、バックアップ、復旧へ段階的に拡張できる
- 標準PostgreSQLを使用することで、提供者固有機能への依存と移行コストを抑えられる

## 配置対応

| 領域 | 採用構成 | 変更時の再検討先 |
|---|---|---|
| Next.js静的Web・同一オリジン中継 | CloudflareサイトWorker | Vercel（再設計が必要） |
| Go API・WebSocket | Cloud Run（東京） | AWS東京のコンテナ実行環境 |
| 正本DB | Cloud SQL for PostgreSQL 17（東京） | RDS for PostgreSQL（東京） |
| 新規写真の画像バイト列 / 旧サーバー写真 | 参加者スマホのIndexedDB（移行後の設計目標） / 旧写真用Cloud Storage private bucket（東京、期限到来まで維持） | ブラウザ容量・消失リスクを含め大学承認後に再検討 |
| 複数Backend化時の共有状態（MVPでは未使用） | Redis互換ストア等を再検討 | 同左 |
| 監視 | Cloudflare側のWorker監視とCloud Monitoring / Logging | 再検討 |

AWSへの切替は別途提供者・保存地域・費用・経路の承認を要する。

## 環境分離と公開手順

| 環境 | 用途 | データ |
|---|---|---|
| local | 開発・単体試験 | 合成データだけ |
| staging | 実機、会場、負荷、削除、ロールバック試験 | 合成データを原則とし、承認済みの試験データだけ |
| production | 公開展示 | 参加者データを扱う唯一の環境 |

- CloudflareのプレビューとサイトWorkerのstagingを本番から分離し、BackendとDB migrationを先にstagingで検証する。既存stagingはIAM保護中で、Workerからの参加者E2Eは未実施
- BackendとDB migrationは先にstagingへ配置し、Web、Go、描画機を含むE2E試験後に同じ成果物をproductionへ昇格する
- productionは`main`へのpushだけで自動公開せず、技術責任者の承認を必要とする
- Web、Goコンテナ、DB schema、WebSocket schema、CampusGraph、3D Mapの版を一つのrelease manifestへ記録する
- T-1週以降は緊急修正以外を停止し、直前安定版のWorker配信、Go container image、migration状態、描画ビルドを保持する
- 本番データをlocalまたはstagingへコピーしない

## 公開経路と責任境界

- ブラウザは同一オリジンの`/api/backend/*`へHTTPSで送信する。サイトWorkerは許可したメソッド・パス・ヘッダーのみをGoへ転送し、`X-Trace-Edge-Secret`をサーバー側で付与する。参加者Bearerは個別API認証として転送し、edge secretとは混同しない
- 新規写真はWorker経由で画像本体を送らず、端末内のIndexedDBに画像バイト列を保持し、Backendへ写真ID・撮影時刻・`checkpointId`等だけを登録する設計目標である。[新しい写真API契約](api.md#写真の端末保存とメタデータ新契約未実装)の`/v2`は未実装であり、旧Cloud Storage短期署名URLへの直接PUT・旧Worker許可リストが変更・検証済みとは扱わない。旧サーバー写真の本人閲覧/削除と期限清掃に要る読取・削除API、bucket・鍵は維持し、既存データの期限到来前に撤去しない
- Cloud Runの`/v1/*`と新方式の`/v2/*`はいずれもedge secretを設定し、不一致を403で拒否する必要がある。現行実装の`/v1/*`保護を`/v2/*`にも拡張・検証するまでは新APIを公開しない。公開用ではSecret Managerの数値版とWorker secretを設定必須とし、参加者Bearer、Go側レート制限、受付停止も維持する。共有secretはIAMではない。Cloud Runを一般到達可能にする変更は大学承認と直接到達拒否試験の後に限る
- CI・運営のCloud Run操作はIAMで制御する。IAM保護中のstagingにWorkerの共有secretだけで到達することはできない。検証用の安全な経路ができるまでstagingは非公開のままにする
- 描画用WSSは現行WorkerのHTTP中継対象外。描画機へedge secretを配らず、`/display`から直接接続する。描画専用資格の発行・保管・失効・更新、ブラウザ用認証、接続先・再接続手順を[Frontend #36](https://github.com/Mino829/TRACE_F/issues/36)と合わせて設計し疎通・認証試験するまで本番公開しない。現Backendの独自ヘッダー認証はブラウザからそのまま利用できない
- CloudflareのWorker・通信経路とアクセスログには参加者のIPやリクエストが触れ得る。Cloud SQL/旧写真用Cloud Storageを東京に置くことと、全処理が東京で行われることは同義でない。`trace-push`のD1/KVは調査デモ用であり本番の写真メタデータ・通知購読の正本には流用しない。各サービスの処理地域・ログ保持は大学の審査対象とする

## PostgreSQL設計原則

### 型・制約

- セッション、写真、経路等のdomain entityの主キーと外部キーは既存のプレフィックス付き不透明な`text` IDを保持する。UUIDv7やPostgreSQL `uuid`型への移行は行わない
- 外部へ公開しない追加の連番が必要な監査・集計テーブルだけ、`bigint generated always as identity`を使用できる
- 時刻は`timestamp with time zone`としてUTCで保存する
- 状態値は`text`と`check`制約を基本とし、不正な状態をDBでも拒否する
- 外部キー列には参照・削除で使用するインデックスを作る
- 参加トークン、Push endpoint、旧写真URL、旧GPSデータを主キーや公開識別子へ使用しない
- 新規セッションは`memory_sessions`のみで、`expires_at`後も本人の状態照会・閲覧・削除を`delete_after`まで受け付ける。旧`live_presence_sessions`は新規発行・位置送信・`on_site`を止め、既存分の本人照会・停止・削除と`delete_after`期限清掃を維持する。旧GPSデータも各`delete_after`まで保持する。保持期間は大学のプライバシー審査（#9）待ち

### 接続管理

- Goはリクエストごとに新規接続せず、上限付きの接続プールを共有する
- DBの最大接続数から運用・migration用の予約枠を除き、`Backend最大インスタンス数 × 1インスタンス当たり最大接続数`が予算を超えないようにする
- Cloud Runの自動拡張には最大インスタンス数を設定する。必要になった場合はPgBouncer、Cloud SQL接続プール、RDS Proxy等を評価する
- connection数、待機接続、長時間query、lock待ちを監視する

### インデックスと期限処理

- すべての外部キー列を索引対象として確認する
- `delete_after`、`expires_at`、到着キュー、新規写真メタデータと旧写真処理キューは実際の抽出条件に合わせたインデックスを作る
- 旧写真の`pending`、`deleting`等の少数状態だけを走査する処理には、状態を限定した部分インデックスを使用する
- 複数workerで期限削除や旧写真処理を行う段階では、短いトランザクションと`FOR UPDATE SKIP LOCKED`で同じjobの二重取得を防ぐ
- 参加者データを日付でpartitionする設計はMVPでは導入せず、実測した件数、削除時間、VACUUM状況から必要性を判断する

### migration

- schema migrationは連番ファイルとしてGitで管理し、適用済みmigrationを変更しない
- production起動時にアプリが破壊的migrationを自動実行しない
- 列追加、両版対応、データ移行、旧列削除の順で原則として後方互換を保つexpand-contract方式を使う
- migration適用、ready確認、アプリ公開を分け、失敗時にWebとGoを直前の互換版へ戻せるようにする
- 大規模な制約追加やindex作成はlock時間をstagingで測り、公開時間外に実行する
- Backendの`002_remove_live_positions.sql`を適用済みと扱わない。旧GPSデータを各`deleteAfter`まで保持し、旧Sessionの本人照会・停止・削除および期限清掃を維持する必要がある。旧契約に依存するテーブル・列の除去は、全対象の期限到来と清掃を確認し、旧本人操作を妨げない移行案・大学承認・staging検証が揃うまで保留する。既存データを破棄したと説明せず、旧バイナリへのrollback可否を移行案ごとに確認する
- 写真移行でも旧Cloud Storage objectキー・状態・本人閲覧/削除・期限清掃を先に外さない。旧データが存在し得る全セッションの`deleteAfter`到来と削除完了、および大学承認・stagingでの移行検証を確認してから旧bucket・鍵の撤去可否を判断する

## 拡張段階

### Stage 1: MVP

- Go BackendのAPIとWebSocketは単一の稼働インスタンスに限定する。Cloud Runの最大インスタンス数を1とし、新旧revision並存にも注意する。当日の最小1は性能試験と費用承認後に設定し、展示時間外の最小数は運用計画で判断する
- PostgreSQLは単一のマネージド主系を正本とする。GPS位置履歴の新規受付は行わないが、旧Sessionと旧GPSデータを各`deleteAfter`まで保持し本人照会・停止・削除と期限清掃に使用する。旧テーブル削除は移行承認待ち
- 写真地点は新規写真のID・撮影時刻・`checkpointId`等のメタデータとしてBackendに写真と同期限で保持し、画像バイト列は端末内にのみ置く設計目標。対応するCampusGraphのノード・座標はBackendで解決する。旧サーバー写真の本人閲覧・削除および期限清掃は維持し、単一Go Backendで旧GPSの短期表示TTLを再構成しない
- 同時MemorySession数は#15の決定と負荷試験で確定する。50の想定値を実証済みとは扱わない

### Stage 2: 複数Backend

- Go Backendを複数インスタンスへ拡張する
- WebSocket配信fan-outと短期leaseをRedis互換ストアまたはメッセージ基盤へ分離する場合は、実測に基づき別途決定する。旧GPS位置用共有TTLストアは導入しない
- 到着演出の主描画機leaseとjob取得を共有状態で一意にする
- PostgreSQLの接続プールと最大インスタンス数を同時に調整する

### Stage 3: 高可用性

- DBの高可用性構成と復旧訓練を費用・可用性要件に応じて評価する。PITRはMVPからの方針（下記バックアップ参照）
- read replicaは読み取り負荷の実測後にだけ導入する
- 複数リージョン化は大学のデータ配置承認、障害目標、費用が必要になった場合に別決定とする

## バックアップ・削除

- PostgreSQLの自動バックアップとpoint-in-time recoveryをMVPから使用する方針とし、保持上限を7日とする。費用・削除保証・大学承認に合わなければ受付開始せず再設計する
- GPS履歴の新規取得・保存は行わない。旧GPSデータは各`deleteAfter`まで保持・期限清掃し、バックアップ設定、リージョン、旧データの削除と復旧試験は公開前に確認する
- 新規画像バイト列はサーバー側でバックアップしない。IndexedDBにはブラウザ・端末ごとの容量制約や消失リスクがあり、別端末に自動移行できない。参加者へ端末での手動保存を案内する。端末を開かない間の画像の自動消去は保証せず、同意文の正本と削除説明は大学承認待ち
- 旧写真bucketの公開アクセスと一覧取得を禁止し、旧画像のversioningと別系統バックアップは原則無効にする。本人閲覧・削除と期限清掃が完了するまでbucket・鍵と必要な権限を保持する
- 新規写真のサーバーメタデータは経路等と`delete_after`で削除する。旧写真ではDB削除とobject削除を冪等なjobとして扱い、片方が失敗した場合は`deleting`状態から再試行する。旧写真objectの削除期限超過・削除失敗、バックアップ世代削除失敗をアラートする

## 残る決定

- CloudflareとGoogle Cloud東京の利用、Cloudflare側の処理・ログ・D1/KVの扱いに関する大学承認。承認されない場合は公開せず代替案を再決定する
- PostgreSQL 17の可用性構成・instance size・接続上限、実測の性能と復旧時間
- 保存期間・バックアップ/PITR保持の個人情報承認と削除保証。端末内写真の容量・消失・他端末移行不可・自動消去非保証・手動保存の説明、および同意文の正本は未承認
- 独自ドメイン購入、Cloudflare DNS/TLS、WAF/レート制限、監視・通知の設定と責任者の確認。企画・課金責任者はMino829、技術・個人情報の承認者もMino829（2026-10-26に兼務で決定）
- Cloud Run、Cloud SQL、旧写真用Cloud Storage、Cloudflare、通信、ドメインの費用見積もり。予算上限5,000円は課金責任者が定めた企画上限であり、予算アラートは課金停止を保証しない。上限内の承認がなければ新規課金・実データ受付を行わない
- 複数Backend化を開始する負荷・接続数・障害要件の閾値
- 描画PCブラウザからの直接WSS接続に必要な運営者本人確認方式、資格情報を誰が発行・保管・失効・更新するか、ブラウザ用認証・版照合、切断後の自動再接続。疎通・認証試験と承認が済むまで公開しない
- 新規写真メタデータ登録と旧写真本人閲覧・削除に使う本人資格の発行・保持・更新経路。既存の旧写真への本人操作を維持しつつ新方式への切替を検証するまで、資格情報や旧読取・削除権限を撤去しない
