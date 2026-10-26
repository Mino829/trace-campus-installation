# インフラ・データベース方針

- 決定日: 2026-10-21（Issue #10の技術配置方針を更新）
- 状態: 採用する技術構成。大学の提供者・処理地域・保存期間承認、予算確認、公開経路の実証は未完了。配備・受付開始を許可する決定ではない
- 対象: MVP、ステージング、本番

従来のVercel + Google Cloud方針を、稼働中の静的Webと同一オリジン中継を活かすCloudflare Workers + Google Cloud東京へ変更する。[比較の記録](cloudflare-web-backend-option.md)を参照。

## 決定

1. Next.jsを静的書き出しし、参加者画面、`/log`、`/display`のWeb配信先をCloudflareのサイトWorkerとする。画面機能の実装完了を意味しない
2. Go BackendはOCI互換コンテナをGoogle Cloud東京（`asia-northeast1`）のCloud Runに配置する技術方針とする。AWS東京は大学審査等で変更が必要になった場合の再決定先であり、現行の採用先ではない
3. 本番の正本DBは東京のCloud SQL for PostgreSQL 17、加工済み写真は東京の非公開Cloud Storage bucketとする。DBにはオブジェクトキー、状態、ハッシュ、寸法、削除期限だけを保存する
4. 通常GPSの量子化済み最新表示位置は単一Go Backendインスタンスのプロセスメモリに30〜60秒TTLで保持する。PostgreSQLの`live_positions`を使用せず、複数Backendへ拡張するときだけRedis互換の共有TTLストアへ移す
5. 会場のUnity / TouchDesignerはGo Backendへ外向きWSS接続を行い、学内ネットワークから会場PCへの受信ポート開放を前提としない。接続の認証経路は描画実装・検証前に確定する

## 採用理由

- 稼働中のCloudflare配信とサイトWorkerの同一オリジンAPI中継を再利用でき、Vercel用の別のWeb実行環境を増やさない
- Go Backendをコンテナ化することで、変更が必要になった場合も移植可能性を残す
- 正本DBと写真保存をGoogle Cloud東京にまとめられる。ただしCloudflareを通る通信・ログや調査用D1/KVの処理・保存まで東京と保証するものではない
- SQLiteの単一ディスク制約を避け、複数Backend、接続プール、バックアップ、復旧へ段階的に拡張できる
- 標準PostgreSQLを使用することで、提供者固有機能への依存と移行コストを抑えられる

## 配置対応

| 領域 | 採用構成 | 変更時の再検討先 |
|---|---|---|
| Next.js静的Web・同一オリジン中継 | CloudflareサイトWorker | Vercel（再設計が必要） |
| Go API・WebSocket | Cloud Run（東京） | AWS東京のコンテナ実行環境 |
| 正本DB | Cloud SQL for PostgreSQL 17（東京） | RDS for PostgreSQL（東京） |
| 写真 | Cloud Storage private bucket（東京） | S3 private bucket（東京） |
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
- Web、Goコンテナ、DB schema、WebSocket schema、CampusGraph、座標変換、3D Mapの版を一つのrelease manifestへ記録する
- T-1週以降は緊急修正以外を停止し、直前安定版のWorker配信、Go container image、migration状態、描画ビルドを保持する
- 本番データをlocalまたはstagingへコピーしない

## 公開経路と責任境界

- ブラウザは同一オリジンの`/api/backend/*`へHTTPSで送信する。サイトWorkerは許可したメソッド・パス・ヘッダーのみをGoへ転送し、`X-Trace-Edge-Secret`をサーバー側で付与する。参加者Bearerは個別API認証として転送し、edge secretとは混同しない
- 写真のメタデータ登録・完了APIはWorker中継の対象へ追加する必要がある。画像本体は[写真API契約](api.md#写真撮影保存)どおり短期署名URLへの直接PUTであり、現行Workerの16KB JSON中継で5MB画像を転送しない。Cloud StorageのCORS、署名URLの権限・期限、削除・地域、Workerの許可リストを写真公開前に検証する
- Cloud Runの`/v1/*`はedge secretが設定されたとき不一致を403で拒否する。公開用ではSecret Managerの数値版とWorker secretを設定必須とし、参加者Bearer、Go側レート制限、受付停止も維持する。共有secretはIAMではない。Cloud Runを一般到達可能にする変更は大学承認と直接到達拒否試験の後に限る
- CI・運営のCloud Run操作はIAMで制御する。IAM保護中のstagingにWorkerの共有secretだけで到達することはできない。検証用の安全な経路ができるまでstagingは非公開のままにする
- 描画用WSSは現行WorkerのHTTP中継対象外。描画機へedge secretを配らず、描画専用資格情報・接続先・再接続手順を[Frontend #36](https://github.com/Mino829/TRACE_F/issues/36)と合わせて設計し疎通試験するまで本番公開しない
- CloudflareのWorker・通信経路とアクセスログには参加者のIPやリクエストが触れ得る。Cloud SQL/Cloud Storageを東京に置くことと、全処理が東京で行われることは同義でない。`trace-push`のD1/KVは調査デモ用であり本番の写真・通知購読の正本には流用しない。各サービスの処理地域・ログ保持は大学の審査対象とする

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

- Go BackendのAPIとWebSocketは単一の稼働インスタンスに限定する。Cloud Runの最大インスタンス数を1とし、新旧revision並存にも注意する。当日の最小1は性能試験と費用承認後に設定し、展示時間外の最小数は運用計画で判断する
- PostgreSQLは単一のマネージド主系を正本とするが、最新表示位置を保存しない
- 最新表示位置はGoメモリへ30〜60秒TTLで保持し、再起動時は次の位置送信から再構成する。既存のPostgreSQL `live_positions`はBackendの002 migrationで除去する必要がある。設計記載はmigration実施を意味しない
- 50同時追跡、5秒間隔、60セッション相当を負荷試験する

### Stage 2: 複数Backend

- Go Backendを複数インスタンスへ拡張する
- 最新表示位置、WebSocket配信fan-out、短期leaseをRedis互換ストアまたはメッセージ基盤へ分離する
- 到着演出の主描画機leaseとjob取得を共有状態で一意にする
- PostgreSQLの接続プールと最大インスタンス数を同時に調整する

### Stage 3: 高可用性

- DBの高可用性構成と復旧訓練を費用・可用性要件に応じて評価する。PITRはMVPからの方針（下記バックアップ参照）
- read replicaは読み取り負荷の実測後にだけ導入する
- 複数リージョン化は大学のデータ配置承認、障害目標、費用が必要になった場合に別決定とする

## バックアップ・削除

- PostgreSQLの自動バックアップとpoint-in-time recoveryをMVPから使用する方針とし、保持上限を7日とする。費用・削除保証・大学承認に合わなければ受付開始せず再設計する
- 生GPS履歴をバックアップ対象へ入れない。バックアップ設定、リージョン、削除と復旧試験は公開前に確認する
- 写真bucketの公開アクセスと一覧取得を禁止し、加工済み写真のversioningと別系統バックアップは原則無効にする
- DB削除とオブジェクト削除を冪等なjobとして扱い、片方が失敗した場合は`deleting`状態から再試行する
- 削除期限超過、object削除失敗、バックアップ世代削除失敗をアラートする

## 残る決定

- CloudflareとGoogle Cloud東京の利用、Cloudflare側の処理・ログ・D1/KVの扱いに関する大学承認。承認されない場合は公開せず代替案を再決定する
- PostgreSQL 17の可用性構成・instance size・接続上限、実測の性能と復旧時間
- 保存期間・バックアップ/PITR保持の個人情報承認と削除保証
- 独自ドメイン購入、Cloudflare DNS/TLS、WAF/レート制限、監視・通知の設定と責任者の確認。企画・課金責任者はMino829、技術・個人情報の承認者もMino829（2026-10-26に兼務で決定）
- Cloud Run、Cloud SQL、Cloud Storage、Cloudflare、通信、ドメインの費用見積もり。予算上限5,000円は課金責任者が定めた企画上限であり、予算アラートは課金停止を保証しない。上限内の承認がなければ新規課金・実データ受付を行わない
- 複数Backend化を開始する負荷・接続数・障害要件の閾値
