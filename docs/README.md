# 設計資料一覧

TRACEの設計判断と履歴をまとめる。**2026-11-02現在は会期終了・撤収／振り返り**。[現在状況](current-plan.md)から[終了手順](deployment-operations.md#終了手順)へ進む。会期終了以外の停止・削除・承認・試験・撤収実績は未確認であり、終了日到来や実装の存在から完了扱いにしない。

## 文書の優先関係

- 現在状況と設計／実装差分: [現在計画](current-plan.md)。現行実装は同書から参照するFrontend / Backend READMEを確認する
- 終了後の確認と操作境界: [終了手順](deployment-operations.md#終了手順)。実停止・実削除の記録ではない
- 以下は既存設計内の優先関係。GPS・画像保存・描画方式等の旧設計を現行実装へ適用する根拠にはしない
- 目的・公開条件: [企画概要](proposal.md)
- 実装する要件と受け入れ条件: [スコープ・要件定義](scope-requirements.md)
- 写真と経路再構成: [写真メモリー・経路再構成仕様](photo-memory.md)
- 通信契約: [API・WebSocket仕様](api.md)
- 状態の正本: [参加追跡・到着演出の状態遷移](state-machine.md)
- データ保持・削除: [プライバシー設計](privacy.md)と[データモデル](data-model.md)
- 配置・DBの技術方針: [インフラ・データベース方針](infrastructure-database-decision.md)
- 公開可否: [試験計画・公開判定基準](test-plan.md)

矛盾が見つかった場合は、現行実装・未merge提案・旧設計のどれかを先に特定する。実装済みを承認済みと同一視せず、旧要件IDや試験条件は事後照合のため維持する。未決事項を暗黙の決定にせず、実施記録のない項目は未確認とする。

## プロダクト

- [現在状況・会期終了後の優先作業](current-plan.md): 実装READMEの契約、設計との差分、未merge提案、会期前計画の折りたたみ履歴
- [企画概要](proposal.md): 背景、目的、対象者、成功指標、公開条件
- [作品コンセプト](concept.md): 体験、表現、展示モード
- [スコープ・要件定義](scope-requirements.md): 対象範囲、機能・非機能要件、受け入れ条件
- [参加者ジャーニー・画面要件](user-journey.md): 画面、正常系、例外系、アクセシビリティ
- [写真メモリー・経路再構成仕様](photo-memory.md): 写真撮影、経路グラフ、ルート推定、本人向けストーリー
- [ロードマップ](roadmap.md): 現在の撤収・振り返りと、旧開発フェーズの履歴／将来バックログ
- [プロジェクト計画・意思決定](project-plan.md): 会期終了後の記録方針、過去の日程・担当・承認・予算の履歴
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
- [配置・展示運用・障害対応](deployment-operations.md): 現行契約に基づく終了手順と、旧配置・開場・障害対応の設計履歴
- [プライバシー設計](privacy.md): 同意、匿名化、表示、保存、削除

## 設計上の前提

**この節以下は会期前の設計前提・未決事項の履歴**。新規GPS取得・送信、新規画像のサーバー保存、写真ピン、Unity / TouchDesigner未選定等は現行実装ではない。現行は端末内写真＋チェックポイントmetadata、three.js `/display`、Goの任意30分Pushで、旧データの本人権利と期限清掃を維持する。詳細な差分と未merge契約は[現在状況](current-plan.md#設計資料と未merge契約の扱い)を参照する。未実装項目を今回会期の納期にせず、採否未決の将来バックログとして扱う。

- Web AppはReact / Next.jsを使用する
- BackendはGoを使用する
- 描画はUnityまたはTouchDesignerを比較する既存方針にthree.jsを候補として追加し、採用後に構成・描画クライアントの配置を更新する
- MVPから東京のCloud SQL for PostgreSQL 17を正本とする技術方針。大学承認前の本番データ受付は行わない
- Next.js静的WebはCloudflareサイトWorker、Go BackendはGoogle Cloud東京のCloud Runとし、同一オリジン`/api/backend/*`から中継する。AWS東京への変更は別途再決定する
- 写真本体はDBへ保存せず、東京の非公開Cloud Storageへ保存する方針
- スマートフォンはHTTPSでサイトWorkerへ接続する。Cloudflareの転送・ログを含む全処理が東京で行われるとは保証しない
- 描画PCは外向きWebSocket接続を使用する設計。経路・資格情報は未決であり現行Workerでは未中継
- 位置取得間隔、精度閾値、同時接続数はキャンパス実地試験で確定する
- 会場PC上のGoは開発・承認済み緊急構成に限る
- 参加追跡状態と到着演出状態は別々に管理する
- 写真を記憶の主役、GPSを写真間の経路再構成に使う補助データとする
- 通常GPSは最新表示だけに使って履歴を保存・再送せず、写真撮影時GPSだけを写真と一緒に再送できる
- 座標変換、CampusGraph、3D Mapを互換バージョンとして同期する
- 公開スクリーンには原写真を表示せず、本人のスマホだけで写真ストーリーを表示する
- 未決事項には暫定値、決定期限、承認者、未決時のフォールバックを設定する

## 未決事項

- [会期前の再設定期限案](project-plan.md#残り日程の再設定作業案)の履歴と当時の判断証跡（今後の期限ではない。再展示日程は未定）
- 本番の最大同時参加人数（試験後にIssue #15で決定。予測参加人数50人/日、希望値100人/日とは別）
- 大学の位置情報利用・ネットワーク利用許可
- データ保存期間
- TouchDesigner / Unity / three.js の比較と最終選択
- 3Dキャンパスモデルの制作方法
- CloudflareとGoogle Cloudの提供者・処理地域に関する大学承認
- Cloud SQL PostgreSQL 17の可用性、instance size、接続上限とバックアップ/PITR保持
- 写真の保存期間・削除保証、ドメイン/DNS/TLS・監視責任者、予算内の見積もり
- キャンパス経路グラフの作成・更新責任者
- 対応OS・ブラウザの最低バージョン
- 当日の運営担当と新規調達の要否（予算上限5,000円、企画・課金責任者はMino829）
