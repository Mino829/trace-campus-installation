# インフラ・データベース方針

- 決定日: 2026-08-21
- 状態: 技術方針として採用。提供者と保存地域は大学の情報・個人情報担当の承認後に確定
- 対象: MVP、ステージング、本番

## 決定

1. Web AppはNext.jsをVercelへ配置する
2. Go BackendはOCI互換のコンテナとして構築し、第一候補をGoogle Cloud東京リージョンのCloud Runとする
3. 大学の既存契約、審査、運用体制がAWSを優先する場合は、AWS東京リージョンのコンテナ実行環境へ切り替える
4. 本番の正本DBはMVPからマネージドPostgreSQLを使用する。Google CloudではCloud SQL for PostgreSQL、AWSではAmazon RDS for PostgreSQLを対応先とする
5. 写真バイト列はPostgreSQLへ保存せず、Google Cloud StorageまたはAmazon S3の非公開bucketへ保存する。DBにはオブジェクトキー、状態、ハッシュ、寸法、削除期限だけを保存する
6. 通常GPSの最新表示位置は永続的なPostgreSQL履歴にしない。単一BackendのMVPではGoプロセス内にTTL付きで保持し、複数Backendへ拡張するときはRedis互換の共有TTLストアへ移す
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
| 一時的な共有位置・配信状態 | Memorystore等のRedis互換ストア | ElastiCache等のRedis互換ストア |
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

- セッション、写真、経路等のdomain entityの主キーはPostgreSQLの`uuid`型とし、Goで生成するUUIDv7を基本とする。DB extensionへ依存せず、推測困難性とindexの時系列局所性を両立する
- 外部へ公開しない追加の連番が必要な監査・集計テーブルだけ、`bigint generated always as identity`を使用できる
- 時刻は`timestamp with time zone`としてUTCで保存する
- 状態値は`text`と`check`制約を基本とし、不正な状態をDBでも拒否する
- 外部キー列には参照・削除で使用するインデックスを作る
- 参加トークン、Push endpoint、写真URL、生GPS履歴を主キーや公開識別子へ使用しない

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

## 拡張段階

### Stage 1: MVP

- Go Backendは単一の主インスタンスを基本とする
- PostgreSQLは単一のマネージド主系を正本とする
- 最新表示位置はGoメモリへ30〜60秒TTLで保持し、再起動時は次の位置送信から再構成する
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
