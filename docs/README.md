# 設計資料一覧

TRACE MVPの仕様と設計判断をまとめる。実装中に仕様を変更した場合は、コードと同じ変更単位で該当文書を更新する。

## 文書の優先関係

- 目的・公開条件: [企画概要](proposal.md)
- 実装する要件と受け入れ条件: [スコープ・要件定義](scope-requirements.md)
- 写真と経路再構成: [写真メモリー・経路再構成仕様](photo-memory.md)
- 通信契約: [API・WebSocket仕様](api.md)
- 状態の正本: [参加追跡・到着演出の状態遷移](state-machine.md)
- データ保持・削除: [プライバシー設計](privacy.md)と[データモデル](data-model.md)
- 配置・DBの技術方針: [インフラ・データベース方針](infrastructure-database-decision.md)
- 公開可否: [試験計画・公開判定基準](test-plan.md)

矛盾が見つかった場合はこの優先関係に従って修正し、未決事項を暗黙に実装へ持ち込まない。

## プロダクト

- [現在計画・会議共有用](current-plan.md): GPS不使用、参加者選択の写真地点（`checkpointId`）、CampusGraph最短経路と到着時再生をまとめた計画スナップショット
- [企画概要](proposal.md): 背景、目的、対象者、成功指標、公開条件
- [作品コンセプト](concept.md): 体験、表現、展示モード
- [スコープ・要件定義](scope-requirements.md): 対象範囲、機能・非機能要件、受け入れ条件
- [参加者ジャーニー・画面要件](user-journey.md): 画面、正常系、例外系、アクセシビリティ
- [写真メモリー・経路再構成仕様](photo-memory.md): 新規写真の端末内画像保存、経路グラフ、ルート推定、本人向けストーリーと旧サーバー写真
- [ロードマップ](roadmap.md): 開発フェーズと完了条件
- [プロジェクト計画・意思決定](project-plan.md): 日程、担当、承認、予算、変更管理
- [リスク台帳](risk-register.md): リスク、軽減策、発生時対応、責任役割
- [試験計画・公開判定基準](test-plan.md): 端末、E2E、負荷、セキュリティ、公開可否

## システム

- [システム構成](architecture.md): コンポーネントと責務
- [インフラ・データベース方針](infrastructure-database-decision.md): Cloudflare Workers + Google Cloud東京、PostgreSQL、新規画像の端末内保存と旧写真bucket維持、公開前の承認
- [配置比較の記録](cloudflare-web-backend-option.md): Vercel案とCloudflare案の比較履歴、Web・Go間の通信境界
- [データモデル](data-model.md): 保存データ、関係、削除方針
- [API・WebSocket仕様](api.md): Webアプリ、Go、描画クライアント間の契約
- [参加と到着演出の状態遷移](state-machine.md): MemorySession単独、署名・期限付き到着QRと旧Sessionの本人操作
- [チェックポイント・3D座標対応](coordinate-mapping.md): 参加者選択地点のBackendでの解決とCampusGraph・3D Mapの版対応（旧GPS/Unity変換は廃止）

## 展示運用

- [機材構成](hardware.md): 機材の役割と接続
- [配置・展示運用・障害対応](deployment-operations.md): 配置、新旧写真の監視・期限清掃、復旧、フォールバック
- [プライバシー設計](privacy.md): 同意、匿名化、表示、新旧写真の保存・削除

## 設計上の前提

- Web AppはReact / Next.jsを使用する
- BackendはGoを使用する
- 描画はthree.jsを採用した（2026-10-26、Issue #12）
- MVPから東京のCloud SQL for PostgreSQL 17を正本とする技術方針。大学承認前の本番データ受付は行わない
- Next.js静的WebはCloudflareサイトWorker、Go BackendはGoogle Cloud東京のCloud Runとし、同一オリジン`/api/backend/*`から中継する。AWS東京への変更は別途再決定する
- 新規写真の画像バイト列は本人のスマホ内（ブラウザのIndexedDB）のみに保持し、Backendには写真ID・撮影時刻・`checkpointId`等の経路用メタデータのみ保存する設計目標。現行のCloud Storage写真アップロード実装が移行済みという意味ではない
- 旧サーバー写真は各セッションの`deleteAfter`まで本人閲覧・削除と期限清掃を維持し、そのための非公開Cloud Storage bucket・鍵・経路を前倒し撤去しない
- IndexedDBには容量制限・消失リスクがあり、写真は他端末へ自動移行しない。端末を開かない間の自動消去は保証せず、本人による手動保存を案内する
- スマートフォンはHTTPSでサイトWorkerへ接続する。Cloudflareの転送・ログを含む全処理が東京で行われるとは保証しない
- 描画PCではFrontend `/display`（three.js）を使用する。描画PC側からGo（Cloud Run）への外向きWSSを用い、Workerは中継しない。ブラウザからの直接接続か会場PC上のagent中継か、描画資格情報の発行・保管・更新は未決。疎通・認証試験と承認までは本番公開しない
- 参加者が選ぶ写真地点は`checkpointId`のみ。座標・対応ノードはBackendが解決する。GPS・リアルタイム位置表示・新規LivePresenceSession・`on_site`は廃止（2026-10-26、Issue #23）
- 会場PC上のGoは開発・承認済み緊急構成に限る
- 参加状態と到着演出状態は別々に管理する
- 写真のチェックポイント地点をCampusGraph最短経路で結び、ShibaLabへの到着区間も同じ線で再生する。写真ピン・フォトスポット集計は廃止
- 旧LivePresenceSessionと旧GPSデータは廃棄済みではない。新規位置送信・`on_site`を受け付けず、各`deleteAfter`までは本人資格での照会・停止・削除と期限清掃を維持する
- CampusGraph・3D Mapの互換バージョンを同期する
- 公開スクリーンには原写真を表示せず、本人のスマホだけで写真ストーリーを表示する設計目標。旧写真の本人取得も各`deleteAfter`まで維持する
- 新しい写真保存方式は未実装・未承認。同意文の正本、資格情報の発行経路、大学承認、公開条件・疎通試験が揃うまで実データ受付を行わない

## 未決事項

- [再設定した作業期限](project-plan.md#残り日程の再設定作業案)の承認（展示は2026年10月31日〜11月1日、芝浦工業大学豊洲キャンパス教室棟3F301で両日10:00〜18:00）
- 本番の最大同時参加人数（試験後にIssue #15で決定。予測参加人数50人/日、希望値100人/日とは別）
- 大学の新規写真メタデータ・旧サーバー写真・チェックポイント地点・旧GPSデータの保持と学内ネットワーク利用の承認
- データ保存期間（開発側既定はMemorySession失効まで6時間、`deleteAfter`は失効後24時間。大学承認値ではない）
- 3Dキャンパスモデルの制作方法
- CloudflareとGoogle Cloudの提供者・処理地域に関する大学承認
- Cloud SQL PostgreSQL 17の可用性、instance size、接続上限とバックアップ/PITR保持
- 新規写真メタデータ・旧サーバー写真の保存期間・削除保証、端末内保存のリスク説明と同意文の正本、ドメイン/DNS/TLS・監視責任者、予算内の見積もり
- キャンパス経路グラフの作成・更新責任者
- 対応OS・ブラウザの最低バージョン
- 当日の運営担当と新規調達の要否（予算上限5,000円、企画・課金責任者はMino829）
- 描画WSSの直接接続かagent中継か、資格情報の発行・保管・更新方法と疎通・認証試験。写真の新旧経路で本人資格を発行・使用する経路も未決
