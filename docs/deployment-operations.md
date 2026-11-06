# 配置・展示運用・障害対応（完成版）

- 更新日: 2026-11-06
- 基準: Frontend `be94bd2`、Backend `6b87961`

## 現在の運用フェーズ

会期（2026-10-31〜11-01）は終了した。再展示は未定。教室棟3F301・両日10:00〜18:00は設計上の会場記録であり、機材調達・現地試験・撤収完了の証跡ではない。**受付/通知停止、期限清掃、クラウド停止・費用、機材撤収、正式承認の実績は未確認**。本改訂では実環境の操作・受入試験を実施していない。

## 採用する配置

コンポーネントと通信経路は[システム構成](architecture.md)を正本とする。参加者APIと限定運営bridgeはサイトWorker経由、描画資格発行・WSSはGo直通である。新規画像は端末内、旧GCS・DB・清掃はクラウド側という境界を維持する。

会期中の独立本番は`trace-fes-2026` / `https://trace-production.async5855.workers.dev`への配備記録がある。旧`trace-log` / `trace-push`、調査D1/KV、`trace-staging-async5855`は別環境である。新規GPS取得/送信・Live作成/位置書込み・新規GCS uploadは停止し、旧Live・旧写真の本人権利と清掃だけを残す。詳しくは[インフラ方針](infrastructure-database-decision.md)。

## 環境・リリース管理

- production/local/stagingと旧調査環境のDB・資格・鍵・データを分離し、本番データを開発や展示デモへ複製しない。
- Frontendはmain pushの「Verify frontend and Workers」成功後、`Production` environmentで検証済みSHAをbuild・既存account/subdomain/Edge binding確認し、`trace-production`だけへ自動配備する。手動は明示`ref`必須、既定検証のみ、公開は`deploy=true`。GitHubの実承認設定・token権限・有効状態は未確認。
- 固定`production.public.json`を使う専用build/validate/deploy経路とし、`.env`、旧一括release、旧Worker helperを本番に流用しない。既存Worker secret bindingは保持し、CIで秘密値を読取・再入力・複製しない。
- Backend自動更新はテスト成功と`STAGING_DEPLOY_ENABLED=true`等を条件とする閉鎖staging専用。WIF・schema baseline・候補ready・清掃job・traffic切替を扱う。本番の自動更新と混同しない。
- 会期終了でCIは自動停止しない。main変更前に配備対象と条件を確認する。Web/Go/schema/Graph本体/座標・Graph・3D Map版/秘密参照版を切替記録に残し、秘密値を記載しない。
- 非互換migrationは受付停止→待機/再生整理→旧API/清掃停止→承認済みbackup→migration→runtime権限→対応版readyの順。現行020に対応したAPI・dispatcher・cleanupを揃え、旧GPS/画像upload版へ戻さない。`migrate`を終了操作として実行しない。
- Cloud Run最大1、新旧revision書込み非並存、DB接続poolと管理枠を維持する。閉鎖stagingのmin0/request-based CPUは本番通知dispatcherの動作保証ではない。

## 機材役割

描画機はブラウザで`/display`を開き、Google本人確認からdisplayTokenと個人operator credentialを別々に取得する。主描画機はleaseで1台、他PCは待機。新PCログインだけで主担当を奪わない。本番はNode.js・gcloud・loopback broker不要。実際の型番・調達・投影・長時間安定性は[機材構成](hardware.md)に従い未確認のまま記録する。

## 起動手順

再展示日は未定であり、以下は保守・承認済み再配備の確認手順。受付再開の指示ではない。

1. 対象環境・公開承認・本人権利・清掃・現行schema/Graphを確認し、受付停止のままGo readyと公開設定を照合する。
2. 同じ本番originの`/display`をブラウザで開く。許可されたGoogleアカウントを対象環境の設定で確認し、描画と運営の認証結果を別々に確認する。
3. 本番運営通信が同一origin `/api/operator`、描画資格とWSSがGo直通であることを確認する。ブラウザへEdge secretを渡さない。旧loopback設定を使用しない。
4. 最初のsnapshot・主leaseを確認する。待機機は正常。配備切替後は`/display`と`/log`の両画面を再読み込みする。
5. 合成/承認済み試験参加で端末写真・metadata、到着、FIFO、完了後新QRによる再視聴、作品閲覧、停止/削除を確認する。作品閲覧だけで到着・予約・通知状態を変えない。
6. 投影、音量、排熱、回線、配線・安全、fallback/復帰と終了経路を実機で確認し、日時・機材・結果を記録する。未実施を成功で補わない。

## 監視項目

実際の監視設定・通知先・最終成功は未確認。運営status、公開設定、対応ログ/クラウド監視を組み合わせる。

- `registrationOpen`、`notification.enabled/operatorStopped`、fallback mode
- 有効な到着queued/playing、主lease、認証済みWSS接続数（statusの接続数は1プロセス分）
- APIエラー/429、DB接続・lock・容量、CPU/メモリ、描画fps・WSS再接続
- 期限清掃の最終成功、期限超過、旧画像deleting・fence/outbox残数と再消去
- Push購読・dispatcher/provider失敗。endpoint・鍵・通知個人内容はログへ出さない
- schemaとGraph本体/3版、Frontend/Go対応版、バックアップ/PITR保持と復旧時の再削除
- 実資源・課金・残存イメージ/ログ/ストレージ。予算アラートは課金の強制停止ではない

新規GPS更新率・写真upload率・写真ピン件数は現行監視指標ではない。運営statusの値を、機材撤収や全資源停止の証明にしない。

## 障害対応

| 障害 | 現行の対応と注意 |
|---|---|
| 回線/Go/DB停止 | 新しい本人操作の失敗を成功扱いしない。描画は接続とsnapshotを回復し、必要なら合成fallbackへ。IndexedDB写真はサーバーから復旧できない |
| 描画機切断・lease喪失 | 再生中はskip確定し、自動で別機に二重再生しない。接続/主leaseを回復後、既存の明示的運営retry契約で判断 |
| 到着集中 | FIFO・50待機上限を守る。満杯は503/Retry-After。通常待機30秒を理由にskipしない。割込み・任意ID手入力で到着を偽造しない |
| 運営認証失敗 | QRを表示しない。Google/Origin/allowlist/資格を確認し、偽QRや秘密のブラウザ入力で代替しない。描画権限は別確認 |
| 端末カメラ/保存失敗 | 撮影・保存エラーを案内。作品観覧を強制参加にしない。ローカル画像保存成功とGo metadata受付を区別 |
| Graph不一致 | 版文字列だけでなくJSON本体を照合。保存済みSessionを書換えず、旧画像/GPS経路を新規端末経路へ混ぜない |
| 通知誤配信 | 権限のある担当者がGo全通知stopを別操作で実行・確認。provider受理済み通知は撤回できない。旧Workerは別確認 |
| 期限清掃失敗・旧画像漏えい疑い | 新規受付を停止し影響を確認、責任者へ連絡。本人権利・削除再試行・GCS権限を壊す全停止を先行させない |

## フォールバック展示

運営fallbackは`normal/pending/fallback`の永続状態。再生中があればpendingでその1件だけを待ち、30秒固定期限後はoperator skipへ進む。待機列・到着受付・参加受付・通知は消えず、別操作が必要。復帰は明示DELETEで行い、残るFIFO先頭へ戻る。Frontendは合成経路/抽象映像を使い、保持中参加者の実データを撤収後デモへ転用しない。DB停止時や未対応クライアントの実機切替を保証しない。


## 終了手順

この節は現行READMEに記載された操作契約に基づく確認手順であり、実行済みの記録ではない。権限を持つ担当者が対象環境・承認・依存関係を確認して操作し、日時・対象・結果・残件を記録する。会期終了だけで停止成功、全削除、費用ゼロとは判断しない。

### 1. 対象と停止状態を確認する

- 本番`trace-production` / `trace-fes-2026`、旧`trace-log` / `trace-push`、検証環境、DB・清掃・通知dispatcherの実際の対象と稼働状態を棚卸しする。配備記録は現在の停止状態の代わりにならない。共通アカウント内の他用途を巻き込まない。
- 個人運営資格で`GET /v1/operator/status`を参照し、公開設定と照合する。`registrationOpen`、`notification.enabled` / `operatorStopped`、待機・再生数、描画接続・lease等を確認する。状態照会は停止操作ではなく、接続数はそのAPIプロセスの値、leaseは機材撤収の証明ではない。
- 個人運営資格と許可Origin、サーバー側のEdge保護が必要。参加者・描画資格、Google ID tokenだけ、Edge keyだけでは運営停止を実行しない。本番同一origin `/api/operator` は資格発行/失効とQR proofだけで、status・受付close・通知stop・fallbackを中継しない。これらには別の認可済み運営入口が必要であり、その実配備・利用手段は権限を持つ担当者が確認する。未確認の入口を使えるものとしてコマンドやUIを捏造しない。secretをブラウザ・URL・ログへ載せない。

### 2. 新規受付と撮影通知を別々に停止・確認する

| 対象 | Goの既存操作契約 | 確認すること |
|---|---|---|
| 新規参加 | `POST /v1/operator/registration/close`、個人operator Bearer・許可Origin・空body | 200 `registrationOpen:false`、運営statusと公開設定の受付停止。DBへ永続化され、環境設定をtrueに戻すだけでは再開しない |
| 全Go撮影通知 | `POST /v1/operator/notifications/stop`、同資格・空body | 200 `notificationsStopped:true`、statusで`notification.enabled:false` / `operatorStopped:true`、公開設定でも無効。購読・鍵・予定を消去し、再起動だけでは再開しない |

受付停止は既存参加者の写真・到着・本人閲覧／削除・通知・期限清掃を止めない。通知停止も受付や本人操作を止めない。到着後も通知する新同意方針が実装されているため、会期終了・到着・参加期限の推定だけを全通知停止の代わりにしない。providerが既に受理した通知は取り消せず、停止成功後に届く場合もある。旧`trace-push` WorkerはこのGo操作では停止したと扱わず、別系統として担当者が契約・実状態を確認する。

### 3. 展示の待機・再生と機材を終了する

- 受付停止は到着・再視聴受付や既存参加の一括停止ではない。運営statusと到着一覧で待機・再生を確認し、既存の運営skip等で処理方針と結果を記録する。本人専用のMemory停止を運営一括停止APIと読み替えない。
- 必要な表示終了は既存fallback／skip契約で扱い、fallbackだけで受付・通知・待機列が消えるとは判断しない。参加者の実経路を撤収後のデモ素材へ転用しない。
- 展示クライアントを終了・切断する。共有PCから離れる際は現在の個人運営資格を失効し、Googleからもログアウトする。全資格失効は同アカウントの他PC・タブにも作用するので対象を確認する。資格失効だけでGoogleの再ログインを禁止したとは扱わない。
- プロジェクター・音響・照明・PC・回線、掲示QR、配線・養生、借用物の返却と会場原状復帰を担当者と確認する。一時ファイル・スクリーンショット・デバッグ出力・ダウンロードした写真等は保存目的・本人権利と承認済み方針に従って扱う。撤収・ログアウト済みの証跡は未確認。

### 4. 本人閲覧・削除と期限清掃を維持・確認する

- 参加停止・受付停止・通知停止・全削除は別操作。停止後も各セッションの`deleteAfter`まで本人状態・写真／作品の閲覧・削除を維持する。旧Memory／Liveは各Bearerと各期限を保ち、期限を前倒し・延長しない。旧GPS・画像uploadを再開する旧版へ戻さない。
- 新規写真の原画像は参加端末のIndexedDB内であり、Goが端末画像の保存・消去完了を検証・復旧できるわけではない。サーバーmetadataの削除成功を端末全台の写真消去済みと記載しない。
- 旧クラウド写真が残る場合は各`deleteAfter`まで本人閲覧・削除を提供し、pending、署名付きPUTの残存有効期限、ゼロバイトfenceと期限後の再消去を確認する。旧写真やfenceが必要な間はAPI・清掃双方のGCS設定／権限を外さない。
- 対応版のAPI・期限清掃ジョブ・DBを維持する。Backendには`go run ./cmd/server cleanup`があるが、これは接続先DB等の設定を使う実削除操作であり、確認用のdry-runではない。承認済みの対象・権限・配備版・schema整合を確かめた担当者だけが既存手順で実行する。本更新では実行しない。schema更新の`migrate`を終了処理として実行しない。
- 清掃の成功時刻、期限超過データ、旧写真・派生物・署名URLの再消去、残存購読、ログ・一時物、バックアップ／PITRの保持と復元時の再削除を確認し、残件を記録する。必要以上のバックアップや実データのローカル複製を撤収のために作らない。

### 5. 保持義務と分離してクラウド費用停止を確認する

[Backend #53](https://github.com/Mino829/TRACE_B/issues/53)は共通アカウントのGCP停止・費用対応の参照事項であり、実施・完了は未確認。受付・通知停止成功だけではCloud Run、Cloud SQL、ストレージ、Scheduler、イメージ、ログ等の課金停止を保証しない。課金責任者Mino829と対象資源・他用途との共有・保持期限・本人アクセス・清掃依存を確認し、維持する資源と停止可能な資源、実施者、停止日時・結果、残存費用を記録する。

本人権利と期限清掃を必要な期限まで維持できることを確認する前にAPI・DB・清掃・必要なbucket／secretを停止・削除しない。設計資料だけでは実資源と安全な停止対象を確定できないため、ここに一括停止・削除コマンドは追加しない。予算アラート、最小インスタンス0、サイトの閉鎖表示を「費用ゼロ」の証跡にしない。

### 6. 事後記録と残件をまとめる

受付・通知停止、本人権利と清掃、クラウド費用、撤収をそれぞれ「確認済み／未確認／未実施」と対象・日時・担当・根拠付きで記録する。当時の公開判断・同意承認・現地試験、実参加数・障害を証跡から整理し、見つからない記録を合格・承認済みとして補完しない。未実装・未統合機能は[将来バックログ](roadmap.md#将来バックログ)へ移し、再展示日は未定のままとする。

## リハーサル受け入れ条件

再展示時は[試験計画・公開判定基準](test-plan.md)で実機、実Google、端末保存、metadata限定送信、直接WSS/主lease、FIFO/再視聴、本人権利・削除、実Push、長時間運転を別々に確認する。会期前のGPS・新規GCS・写真ピン・on_site位置展示を受入条件として復活させない。本書の記載は合格実績ではない。

## 会場安全・導線

- 電源容量・接地・排熱・転倒防止を確認し、ケーブル養生で避難/車椅子導線を塞がない。
- 投射光・音量・強い点滅・暗所への注意と代替観覧を用意する。
- OS更新・スリープ・個人通知・スクリーンセーバー、共有PCのGoogleログアウトと資格失効を確認する。
- 機材・回線・借用物・原状復帰の現物記録を残す。コード上のlease失効を撤収済みとはしない。

## 参照実装と証跡

- [Frontend本番配備/本人導線・過去UI確認記録](https://github.com/Mino829/TRACE_F/blob/be94bd2/README.md)、[環境別展示ログイン](https://github.com/Mino829/TRACE_F/blob/be94bd2/arrival-issuer/README.md)
- [本番bridge許可リスト](https://github.com/Mino829/TRACE_F/blob/be94bd2/push-worker/src/site.ts)、[Frontend配備workflow](https://github.com/Mino829/TRACE_F/blob/be94bd2/.github/workflows/deploy.yml)
- [Backend運営停止/本人権利/清掃](https://github.com/Mino829/TRACE_B/blob/6b87961/README.md)、[閉鎖staging自動配備](https://github.com/Mino829/TRACE_B/blob/6b87961/.github/workflows/test.yaml)

過去の仮想カメラUI smoke・ブリッジ切替後Chrome確認はソース文書にある当時の記録であり、今回の実行結果、全参加者清掃、実スマートフォン・投影機材・正式公開受入の合格ではない。
