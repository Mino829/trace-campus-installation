# TRACE — Campus Movement Installation

キャンパスで撮った写真と、その節目をつなぐ経路を3DCGで振り返るインタラクティブアート・プロジェクトです。**2026年11月2日現在、文化祭の会期は終了し、撤収・データ保護・費用停止確認・振り返りのフェーズです。再展示の日程は未定です。**

まず[現在状況と設計・実装の差分](docs/current-plan.md)を確認し、[会期終了後の手順](docs/deployment-operations.md#終了手順)へ進んでください。会期終了は確認済みですが、受付・通知・クラウドの停止、期限削除、機材撤収、公開承認・現地試験の実績は未確認です。この文書更新では実環境の停止・削除を実施していません。

## 技術構成

- Web App: React / Next.js。新しいGPSの取得・送信は行わない
- 写真: 新規写真は端末内だけに保存。Goへ送るのは自己申告チェックポイント等のmetadataで、画像は送信しない
- Backend: Go / PostgreSQL。旧Live・旧クラウド写真の本人閲覧・停止・削除と期限清掃を維持する
- Rendering: ブラウザのthree.js `/display` と認証付き描画WSS
- Deployment: CloudflareサイトWorkerから参加者・運営APIを同一originでGoへ中継。実環境の稼働・停止状態や正式承認は別途確認する
- 撮影通知: 本番は任意のGo Web Push（30分）。旧Worker通知とは別系統
- 到着・再視聴: 会場QR。再視聴は有効な本人の新しいQR操作に限り、待機列末尾へ追加

確認元: [Frontend README](https://github.com/Mino829/TRACE_F/blob/be94bd2/README.md)、[Backend README](https://github.com/Mino829/TRACE_B/blob/6b87961/README.md)。これはコード・READMEの契約であり、実配備・現地受入・停止成功の証明ではありません。GPS、サーバーへの新規画像保存、Unity / TouchDesignerの比較を含む詳細設計には旧計画が残っています。未mergeの設計PRも含め、[差分の扱い](docs/current-plan.md#設計資料と未merge契約の扱い)を確認してください。

## ドキュメント

- [現在状況・会期終了後の優先作業](docs/current-plan.md)
- [設計資料一覧](docs/README.md)
- [企画概要](docs/proposal.md)
- [作品コンセプト](docs/concept.md)
- [スコープ・要件定義](docs/scope-requirements.md)
- [参加者ジャーニー・画面要件](docs/user-journey.md)
- [写真メモリー・経路再構成仕様](docs/photo-memory.md)
- [システム構成](docs/architecture.md)
- [インフラ・データベース方針](docs/infrastructure-database-decision.md)
- [データモデル](docs/data-model.md)
- [API・WebSocket仕様](docs/api.md)
- [参加追跡・到着演出の状態遷移](docs/state-machine.md)
- [位置座標・3D座標変換](docs/coordinate-mapping.md)
- [配置・展示運用・障害対応](docs/deployment-operations.md)
- [機材構成](docs/hardware.md)
- [プライバシー設計](docs/privacy.md)
- [ロードマップ](docs/roadmap.md)
- [プロジェクト計画・意思決定](docs/project-plan.md)
- [リスク台帳](docs/risk-register.md)
- [試験計画・公開判定基準](docs/test-plan.md)

## 現在のフェーズ

会期終了・撤収／振り返り。設計記録上の会期は2026年10月31日〜11月1日、会場は教室棟3F301、両日10:00〜18:00です。今後の開場予定や、旧計画の達成・承認済みを意味しません。終了後は受付・通知停止の確認と本人の閲覧・削除／期限清掃の維持を優先します。
