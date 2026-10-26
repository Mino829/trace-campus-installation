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

- [現在計画・会議共有用](current-plan.md): 写真メモリー、写真ピン、到着前後のリアルタイム位置展示をまとめた計画スナップショット
- [企画概要](proposal.md): 背景、目的、対象者、成功指標、公開条件
- [作品コンセプト](concept.md): 体験、表現、展示モード
- [スコープ・要件定義](scope-requirements.md): 対象範囲、機能・非機能要件、受け入れ条件
- [参加者ジャーニー・画面要件](user-journey.md): 画面、正常系、例外系、アクセシビリティ
- [写真メモリー・経路再構成仕様](photo-memory.md): 写真撮影、経路グラフ、ルート推定、本人向けストーリー
- [ロードマップ](roadmap.md): 開発フェーズと完了条件
- [プロジェクト計画・意思決定](project-plan.md): 日程、担当、承認、予算、変更管理
- [リスク台帳](risk-register.md): リスク、軽減策、発生時対応、責任役割
- [試験計画・公開判定基準](test-plan.md): 端末、E2E、負荷、セキュリティ、公開可否

## システム

- [システム構成](architecture.md): コンポーネントと責務
- [インフラ・データベース方針](infrastructure-database-decision.md): Cloudflare Workers + Google Cloud東京、PostgreSQL、環境分離、公開前の承認
- [配置比較の記録](cloudflare-web-backend-option.md): Vercel案とCloudflare案の比較履歴、Web・Go間の通信境界
- [データモデル](data-model.md): 保存データ、関係、削除方針
- [API・WebSocket仕様](api.md): Webアプリ、Go、描画クライアント間の契約
- [参加追跡・到着演出の状態遷移](state-machine.md): 写真、リアルタイム位置、会場内フェーズ、到着演出を分離した状態管理
- [位置座標・3D座標変換](coordinate-mapping.md): GPS検証、平滑化、Unity座標への変換

## 展示運用

- [機材構成](hardware.md): 機材の役割と接続
- [配置・展示運用・障害対応](deployment-operations.md): 配置、監視、復旧、フォールバック
- [プライバシー設計](privacy.md): 同意、匿名化、表示、保存、削除

## 設計上の前提

- Web AppはReact / Next.jsを使用する
- BackendはGoを使用する
- 描画はthree.jsを採用した（2026-10-26、Issue #12）
- MVPから東京のCloud SQL for PostgreSQL 17を正本とする技術方針。大学承認前の本番データ受付は行わない
- Next.js静的WebはCloudflareサイトWorker、Go BackendはGoogle Cloud東京のCloud Runとし、同一オリジン`/api/backend/*`から中継する。AWS東京への変更は別途再決定する
- 写真本体はDBへ保存せず、東京の非公開Cloud Storageへ保存する方針
- スマートフォンはHTTPSでサイトWorkerへ接続する。Cloudflareの転送・ログを含む全処理が東京で行われるとは保証しない
- 描画PCは外向きWebSocketでGo（Cloud Run）へ直接接続する（2026-10-26決定）。Workerは中継せず、描画用資格情報の発行契約はIssue #11で整備する
- 位置取得間隔、精度閾値、同時接続数はキャンパス実地試験で確定する
- 会場PC上のGoは開発・承認済み緊急構成に限る
- 参加追跡状態と到着演出状態は別々に管理する
- 写真を記憶の主役、GPSを写真間の経路再構成に使う補助データとする
- 通常GPSは最新表示だけに使って履歴を保存・再送せず、写真撮影時GPSだけを写真と一緒に再送できる
- 座標変換、CampusGraph、3D Mapを互換バージョンとして同期する
- 公開スクリーンには原写真を表示せず、本人のスマホだけで写真ストーリーを表示する
- 未決事項には暫定値、決定期限、承認者、未決時のフォールバックを設定する

## 未決事項

- [再設定した作業期限](project-plan.md#残り日程の再設定作業案)の承認（展示は2026年10月31日〜11月1日、芝浦工業大学豊洲キャンパス教室棟3F301で両日10:00〜18:00）
- 本番の最大同時参加人数（試験後にIssue #15で決定。予測参加人数50人/日、希望値100人/日とは別）
- 大学の位置情報利用・ネットワーク利用許可
- データ保存期間
- 3Dキャンパスモデルの制作方法
- CloudflareとGoogle Cloudの提供者・処理地域に関する大学承認
- Cloud SQL PostgreSQL 17の可用性、instance size、接続上限とバックアップ/PITR保持
- 写真の保存期間・削除保証、ドメイン/DNS/TLS・監視責任者、予算内の見積もり
- キャンパス経路グラフの作成・更新責任者
- 対応OS・ブラウザの最低バージョン
- 当日の運営担当と新規調達の要否（予算上限5,000円、企画・課金責任者はMino829）
