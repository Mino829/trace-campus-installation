# インフラ・データベース方針

- 決定日: 2026-08-21
- 状態: 技術方針として採用した設計目標。現行Backendへすべて反映済みではなく、提供者と保存地域・保持期間は大学の情報・個人情報担当の承認後に確定
- 対象: MVP、ステージング、本番

Cloudflareで稼働中のWebデモを踏まえた[配置変更案](cloudflare-web-backend-option.md)を比較検討中。本書の採用方針は正式な変更決定まで維持する。

## 決定

1. Web AppはNext.jsをVercelへ配置する
2. Go BackendはOCI互換のコンテナとして構築し、第一候補をGoogle Cloud東京リージョンのCloud Runとする
3. 大学の既存契約、審査、運用体制がAWSを優先する場合は、AWS東京リージョンのコンテナ実行環境へ切り替える
4. 本番の正本DBはMVPからマネージドPostgreSQLを使用する。Google CloudではCloud SQL for PostgreSQL、AWSではAmazon RDS for PostgreSQLを対応先とする
5. 写真バイト列はPostgreSQLへ保存せず、Google Cloud StorageまたはAmazon S3の非公開bucketへ保存する。DBにはオブジェクトキー、状態、ハッシュ、寸法、削除期限だけを保存する
6. 通常GPSの量子化済み最新表示位置は単一Go Backendインスタンスのプロセスメモリに30〜60秒TTLで保持する。PostgreSQLの`live_positions`を使用せず、複数Backendへ拡張するときだけRedis互換の共有TTLストアへ移す
7. 会場のUnity / TouchDesignerはGo Backendへ外向きWSS接続を行い、学内ネットワークから会場PCへの受信ポート開放を前提としない

## 採用理由

- Next.jsの変更ごとにプレビュー環境を作り、本番承認前に端末試験を行いやすい
- Go Backendをコンテナ化することで、Cloud RunとAWSの間でアプリケーションコードを移植しやすい
- 東京リージョンにBackend、DB、写真をまとめ、大学へデータ配置を説明しやすい
- SQLiteの単一ディスク制約を避け、複数Backend、接続プール、バックアップ、復旧へ段階的に拡張できる
- 標準PostgreSQLを使用することで、提供者固有機能への依存と移行コストを抑えられる

## 配置対応

| 領域 | 第一候補: Google Cloud | 代替: AWS |
|---|---|---|
| Next.js | Vercel | Vercel |
| Go API・WebSocket | Cloud Run（東京） | ECS/Fargate等のコンテナ実行環境（東京） |
| 正本DB | Cloud SQL for PostgreSQL（東京） | RDS for PostgreSQL（東京） |
| 写真 | Cloud Storage private bucket（東京） | S3 private bucket（東京） |
| 複数Backend化した場合の共有位置・配信状態（MVPでは未使用） | Memorystore等のRedis互換ストア | ElastiCache等のRedis互換ストア |
| 監視 | Cloud Monitoring / Logging | CloudWatch |

AWS側の具体的なコンテナ実行環境とロードバランサーは、AWSを選択した場合に大学の標準構成へ合わせて確定する。

## 環境分離と公開手順

| 環境 | 用途 | データ |
|---|---|---|
| local | 開発・単体試験 | 合成データだけ |
| staging | 実機、会場、負荷、削除、ロールバック試験 | 合成データを原則とし、承認済みの試験データだけ |
| production | 公開展示 | 参加者データを扱う唯一の環境 |

- feature branchまたはpull requestごとにVercel Previewを作る
- BackendとDB migrationは先にstagingへ配置し、Web、Go、描画機を含むE2E試験後に同じ成果物をproductionへ昇格する
- productionは`main`へのpushだけで自動公開せず、技術責任者の承認を必要とする
- Web、Goコンテナ、DB schema、WebSocket schema、CampusGraph、座標変換、3D Mapの版を一つのrelease manifestへ記録する
- T-1週以降は緊急修正以外を停止し、直前安定版のWeb deployment、Go container image、migration状態、描画ビルドを保持する
- 本番データをlocalまたはstagingへコピーしない

## PostgreSQL設計原則

### 型・制約

- セッション、写真、経路等のdomain entityの主キーと外部キーは既存のプレフィックス付き不透明な`text` IDを保持する。UUIDv7やPostgreSQL `uuid`型への移行は行わない
- 外部へ公開しない追加の連番が必要な監査・集計テーブルだけ、`bigint generated always as identity`を使用できる
- 時刻は`timestamp with time zone`としてUTCで保存する
- 状態値は`text`と`check`制約を基本とし、不正な状態をDBでも拒否する
- 外部キー列には参照・削除で使用するインデックスを作る
- 参加トークン、Push endpoint、写真URL、生GPS履歴を主キーや公開識別子へ使用しない
- `memory_sessions`と`live_presence_sessions`それぞれに`expires_at`と`delete_after`を持ち、期限が異なり得る。`expires_at`後は位置送信を停止しても、本人の状態・閲覧・削除は各`delete_after`まで受け付ける。保持期間は大学のプライバシー審査（#9）待ち

### 接続管理

- Goはリクエストごとに新規接続せず、上限付きの接続プールを共有する
- DBの最大接続数から運用・migration用の予約枠を除き、`Backend最大インスタンス数 × 1インスタンス当たり最大接続数`が予算を超えないようにする
- Cloud Runの自動拡張には最大インスタンス数を設定する。必要になった場合はPgBouncer、Cloud SQL接続プール、RDS Proxy等を評価する
- connection数、待機接続、長時間query、lock待ちを監視する

### インデックスと期限処理

- すべての外部キー列を索引対象として確認する
- `delete_after`、`expires_at`、到着キュー、写真処理キューは実際の抽出条件に合わせたインデックスを作る
- `pending`、`deleting`等の少数状態だけを走査する処理には、状態を限定した部分インデックスを使用する
- 複数workerで期限削除や写真処理を行う段階では、短いトランザクションと`FOR UPDATE SKIP LOCKED`で同じjobの二重取得を防ぐ
- 参加者データを日付でpartitionする設計はMVPでは導入せず、実測した件数、削除時間、VACUUM状況から必要性を判断する

### migration

- schema migrationは連番ファイルとしてGitで管理し、適用済みmigrationを変更しない
- production起動時にアプリが破壊的migrationを自動実行しない
- 列追加、両版対応、データ移行、旧列削除の順で後方互換を保つexpand-contract方式を使う
- migration適用、ready確認、アプリ公開を分け、失敗時にWebとGoを直前の互換版へ戻せるようにする
- 大規模な制約追加やindex作成はlock時間をstagingで測り、公開時間外に実行する
- Backendの`002_remove_live_positions.sql`は旧APIのテーブル参照と非互換の例外である。位置受付を止め、旧API・清掃ジョブを停止した上で適用し、同じschema版の新APIのみ起動する。旧バイナリへの単純なrollbackはできず、既存表示位置は破棄する。承認とstagingでの初回・差分移行検証なしに公開中のDBへ適用しない

## 拡張段階

### Stage 1: MVP

- Go BackendのAPIとWebSocketは単一の稼働インスタンスに限定する。Cloud Run等の自動拡張設定も最大1インスタンスとし、別プロセスへ最新位置リクエストや描画接続を分散させない
- PostgreSQLは単一のマネージド主系を正本とするが、最新表示位置を保存しない
- 最新表示位置はGoメモリへ30〜60秒TTLで保持し、再起動時は次の位置送信から再構成する。既存のPostgreSQL `live_positions`はBackendの002 migrationで除去する必要がある。設計記載はmigration実施を意味しない
- 50同時追跡、5秒間隔、60セッション相当を負荷試験する

### Stage 2: 複数Backend

- Go Backendを複数インスタンスへ拡張する
- 最新表示位置、WebSocket配信fan-out、短期leaseをRedis互換ストアまたはメッセージ基盤へ分離する
- 到着演出の主描画機leaseとjob取得を共有状態で一意にする
- PostgreSQLの接続プールと最大インスタンス数を同時に調整する

### Stage 3: 高可用性

- DBの高可用性構成、point-in-time recovery、復旧訓練を有効化する
- read replicaは読み取り負荷の実測後にだけ導入する
- 複数リージョン化は大学のデータ配置承認、障害目標、費用が必要になった場合に別決定とする

## バックアップ・削除

- PostgreSQLの自動バックアップとpoint-in-time recoveryを使用するが、保持期間はプライバシー設計と大学承認の範囲内に制限する
- 暫定上限はDBバックアップ7日とし、生GPS履歴をバックアップ対象へ入れない
- 写真bucketの公開アクセスと一覧取得を禁止し、加工済み写真のversioningと別系統バックアップは原則無効にする
- DB削除とオブジェクト削除を冪等なjobとして扱い、片方が失敗した場合は`deleting`状態から再試行する
- 削除期限超過、object削除失敗、バックアップ世代削除失敗をアラートする

## 残る決定

- Google CloudまたはAWSの最終選択と大学承認
- PostgreSQLのversion、可用性構成、instance size、接続上限
- 正式な保存期間とバックアップ保持期間
- 独自domain、DNS、証明書、WAF、rate limitの責任境界
- 複数Backend化を開始する負荷・接続数・障害要件の閾値
