# インフラ・データベース方針（完成版）

- 更新日: 2026-11-06
- 対象: 会期終了時点の実装、保守・期限清掃・安全な終了判断
- 基準: Frontend `be94bd2`、Backend `6b87961`。コード・設定・過去の配備記録と、現在の稼働・大学承認・実機受入は区別する

会期（2026-10-31〜11-01）は終了した。再展示は未定。現在の受付・通知・清掃・課金・資源停止の実績は未確認であり、本改訂は実環境を変更しない。[終了手順](deployment-operations.md#終了手順)を運用上の入口とする。

## 決定

1. Next.js静的書き出しをCloudflareサイトWorker `trace-production`で配信する。参加・写真・本人作品は`/log`、展示はthree.jsの`/display`、旧参加の状態・停止・削除は`/`で扱う。
2. Go API・直接WSSはGoogle Cloud東京（`asia-northeast1`）のCloud Run、正本DBはCloud SQL for PostgreSQL 17。会期中の独立本番は`trace-fes-2026`への配備記録がある。これを現在の稼働確認とはしない。
3. 新規写真画像は端末のIndexedDBだけに保存する。Go `/v2`へ送る写真入力は`clientPhotoId`、自己申告の`checkpointId`、`capturedAt`のみ。画像、GPS、画像URL、画像ハッシュ・寸法等を送らない。新規画像のGCS uploadは停止している。
4. 新規GPS取得・送信、LivePresenceSession新規作成、既存Liveの位置書込み・on_site継続は停止。旧Liveと旧GCS写真は各`deleteAfter`まで本人閲覧・停止・削除と期限清掃を維持する。
5. 単一Goインスタンス・新旧revisionの書込み非並存を維持する。到着FIFO・単一主描画機lease・運営制御はDBに永続化する。旧位置キャッシュはGoメモリのTTLのみで、現行描画には配信しない。

## 採用理由

Cloudflareの静的配信と同一origin中継を活かし、Go/PostgreSQLの正本を東京に維持する。端末内写真への切替で新しい画像転送・クラウド画像保存をなくしたが、地点と時刻のmetadataにも行動推測のリスクが残る。DB・旧bucketが東京でも、Cloudflare通信・ログ、Google認証、Push配信の全処理が東京になるわけではない。[比較履歴](cloudflare-web-backend-option.md)は現行の代替配置指示ではない。

## 配置対応

| 領域 | 現行構成・役割 |
|---|---|
| Web・中継 | `https://trace-production.async5855.workers.dev`、サイトWorker |
| Go API・描画WSS | Cloud Run東京。公開設定のGo URLは`https://trace-backend-350621330046.asia-northeast1.run.app` |
| 正本DB | Cloud SQL for PostgreSQL 17東京、参加・同意・metadata・経路・到着・lease・運営制御・暗号化Push購読 |
| 新規写真 | 参加端末IndexedDB。サーバーバックアップも別端末への自動移行もない |
| 旧写真 | 東京の非公開Cloud Storage。旧画像と署名PUTのfence/outboxがなくなるまで保守 |
| 期限清掃 | 対応版APIの清掃、Cloud Run Job＋Cloud Scheduler。実稼働・最終成功は別途確認 |
| 通知 | 任意のGo Web Push、30分のみ。旧`trace-push` Workerとは独立 |
| 認証・秘密 | Google本人確認、Secret Manager、実行サービスアカウント、Worker secret binding |

旧`trace-log`・`trace-push`・調査D1/KV、閉鎖staging `trace-staging-async5855`は独立した対象で、本番配備による更新・移行先ではない。AWS、Vercel、Redisは現行の採用先ではない。

## 環境分離と公開手順

- localは合成データ・使い捨てDB。stagingとproductionの資格・DB・bucketを混ぜず、本番データを開発へコピーしない。
- Frontendはmain pushの検証workflow成功後、`Production` environmentで検証済みSHAをbuild・検証して`trace-production`だけへ自動配備するコードがある。手動workflowは明示`ref`必須、既定は検証だけ、公開は`deploy=true`が必要。会期終了で自動配備が無効になったとは扱わない。
- Backendのmain push後自動更新はテスト成功、対象repository、`STAGING_DEPLOY_ENABLED=true`等を条件とする**閉鎖staging専用**。WIF認証、schema baseline照合、候補ready、清掃job、traffic切替を行う。これを`trace-fes-2026`本番自動配備と読み替えない。
- 現在のGitHub environment承認設定・実権限・CI有効状態は未確認。コードや文書のmain反映前に配備対象を確認する。技術的配備成功は公開承認ではない。
- 固定公開設定、Web SHA、Go image、DB schema、Graph JSON本体、座標・Graph・3D Map版、秘密の参照版を切替記録に残す。秘密値は記録しない。

## 公開経路と責任境界

| 経路 | 認証・制限 |
|---|---|
| 同一origin `/api/backend/*` | サイトWorkerの許可パス・メソッドと16KiB JSON上限。参加Bearerはheader、WorkerがEdge secretを付与 |
| 同一origin `/api/operator/*` | **一般管理proxyではない**。個人運営資格の発行・現在/全資格失効・到着proof取得だけを許可 |
| Go `POST /v1/operator/display-tokens` | Google ID token・描画専用Origin/allowlistで直接認証。Edge secret不要 |
| Go `/v1/display/ws` | ブラウザから直接WSS、最初の`renderer.authenticate`で短命displayToken・3版を検証 |
| status・受付close・通知stop・fallback等 | 個人operator Bearer・許可Origin・Edge保護を備えた別の認可済み運営入口が必要。公開サイトbridgeには追加されていない |

- 本番展示PCはブラウザとGoogleログインを使い、Node.js・gcloud・loopback brokerは不要。loopbackは旧リハーサルの別経路。
- Google ID tokenから得るoperator credentialとdisplayTokenは別資格。参加者Bearer、Edge secret、到着proof鍵、display署名鍵、Push VAPID秘密鍵、購読暗号化鍵を兼用しない。運営資格はタブメモリだけ、秘密鍵はサーバー側だけ。
- Workerは正規の`CF-Connecting-IP`だけを単一`X-Trace-Client-IP`へ変換し、ブラウザの転送IPを捨てる。GoはEdge secret一致時だけ転送IPを信頼する。Edge secretはIAMや本人認証の代わりではない。
- IAM非公開Goへ共有secretだけで到達できない。一般到達性、OAuth Origin、公開承認、TLS/IAMの実確認を設定ファイルの存在から推定しない。

## PostgreSQL設計原則

### 型・制約

- IDは既存のプレフィックス付き不透明`text`、時刻はUTCの`timestamptz`。ランダムIDでもmetadataを完全匿名とは扱わない。
- 新規Memoryには受付時のGraph版と写真modeを固定。保存済み版・同意・期限を後付け更新しない。地点・経路は固定Graphに対する自己申告の再構成であり、GPS実測履歴ではない。
- 同内容再送・競合・写真削除をDB transactionで整合させる。削除済み`clientPhotoId`のfenceはMemoryの`deleteAfter`まで残し、遅延再送で復活させない。
- Memoryと旧Liveは独立した`expiresAt`・`deleteAfter`・Bearerを持つ。参加期限・停止後も削除期限内の本人閲覧・削除を維持する。
- 初回到着と完了後新QRによる本人再視聴は別操作・別receipt。再視聴は新公開再生IDでFIFO末尾へ入り、初回到着時刻・期限・通知方針を書き換えない。

### 接続管理

Goは上限付き接続poolを共有する。Cloud Run最大1とDB管理/migration用接続の予約枠を維持し、接続数・待機・長時間query・lockを確認する。プロセスメモリのレート制限は複数インスタンスで共有されないため、scale-outを構成変更だけで実施しない。

### インデックスと期限処理

期限、FIFO、状態、所有者による照会・削除を索引とtransactionで扱う。待機は有効Memory最大50件（playingを除く）、playingは全体で1件。pending写真の開始待ちは最大15秒、通常待機30秒を理由にskipしない。DBの主leaseを持つ描画機だけが先頭開始を要求できる。切断等で失敗した再生は自動で別機に再実行せず、既存の運営retry契約で扱う。

### migration

- 埋め込み連番SQLを明示実行し、適用済みSQLを変更しない。現行は020まで。API・清掃はDBの連続履歴と対応最終版の一致を要求し、実行ユーザーへschema変更権限を与えない。
- `migrate`は終了・清掃操作ではない。非互換変更は受付停止、キュー整理、旧API/清掃停止、承認済みbackup、migration、runtime権限、対応版readyの順で切り替える。旧版へ単純rollbackしない。
- 019の再視聴ID/FIFO列や020の到着後通知を含め、API・dispatcher・cleanupを同じ対応版にする。020は既存Memoryの通知方針をtrueにしない。新同意版に固定された新規Memoryだけが到着後通知継続対象となる。

## 拡張段階

完成版は単一Backend。複数Backend、共有レート制限・fan-out、可用性拡張は再展示と実測需要が確定してから別設計とする。DB leaseがあることだけでscale-out全体が成立したとはしない。

## バックアップ・削除

- DB自動backup/PITRの上限7日は旧方針の提案値であり、現設定・承認・世代削除実績は未確認。保持上限・復元時再削除・費用を課金/個人情報担当と確認する。
- 旧bucketは非公開、一覧禁止。起動時検査は東京、uniform access、public access prevention、versioning/soft delete/hold等の無効を要求する。これも実bucket設定の今回確認ではない。
- 旧署名PUTは発行後すぐ失効できない。削除ではゼロバイトfenceを置き、URL期限後にoutboxから再消去する。旧画像/fenceが残る間はAPIと清掃のGCS設定・権限を外さない。
- サーバーmetadata・画像清掃成功と、閉じた参加端末のIndexedDB画像消去は別。本人操作・清掃・backups・端末写真を分けて記録する。

## 残る決定

現在の停止状態、期限清掃成功、旧画像/fence残数、backup/PITR保持、正式な同意・大学承認、運用責任者・会期後窓口、実機受入は未確認。費用上限5,000円は企画上の上限で、実費・停止済みの証跡ではない。Backend #53の共通アカウントGCP停止は完了未確認。本人権利と清掃依存を満たす前に資源を消さない。

## 参照実装

- [Frontend本番設定](https://github.com/Mino829/TRACE_F/blob/be94bd2/push-worker/production.public.json)、[サイトWorker](https://github.com/Mino829/TRACE_F/blob/be94bd2/push-worker/src/site.ts)、[Frontend配備workflow](https://github.com/Mino829/TRACE_F/blob/be94bd2/.github/workflows/deploy.yml)
- [Backend README（写真・運営・清掃・migration）](https://github.com/Mino829/TRACE_B/blob/6b87961/README.md)、[Backend staging workflow](https://github.com/Mino829/TRACE_B/blob/6b87961/.github/workflows/test.yaml)
