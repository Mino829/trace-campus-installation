# 設計資料一覧

学園祭会期終了後の**完成版設計書**。2026-11-06改訂。詳細本文は完成時点の実装を基準とし、旧GPS案・新規クラウド写真保存・描画方式未選定を現行要件として扱わない。

最初に[現在状況・完成版の参照基準](current-plan.md)を読む。コードに実装があること、過去の配備・検証記録があること、現在の実環境で確認できたことを分ける。終了後の停止・全削除・精算・撤収・正式承認・実機受入の結果は未確認。

## 文書の優先関係

| 確認したいこと | 正本となる文書 |
|---|---|
| 対象版・現状・旧案との差分 | [現在状況](current-plan.md) |
| 作品の目的・完成版の範囲 | [企画概要](proposal.md)、[要件定義](scope-requirements.md) |
| 参加者の操作と画面 | [参加者ジャーニー](user-journey.md)、[写真仕様](photo-memory.md) |
| API・描画通信の契約 | [API・WebSocket仕様](api.md) |
| 参加・到着・再視聴の状態 | [状態遷移](state-machine.md) |
| 保存対象・同意・保持・削除 | [プライバシー](privacy.md)、[データモデル](data-model.md) |
| 配置・資格・終了後の管理 | [インフラ方針](infrastructure-database-decision.md)、[配置・展示運用](deployment-operations.md) |
| 検証範囲と証跡の扱い | [試験計画](test-plan.md) |

矛盾が見つかった場合は、[固定したソース版](current-plan.md#現行実装の参照先)で確認し、影響する文書を合わせて修正する。本文の仕様改訂を、実装変更や実データ受付・削除の承認として扱わない。

## プロダクト

- [企画概要](proposal.md): 背景、対象、作品の目的、実績との区別
- [作品コンセプト](concept.md): 写真の節目と「記憶の道」、本人作品と会場展示
- [スコープ・要件定義](scope-requirements.md): 完成版で提供すること、提供しないこと、境界条件
- [参加者ジャーニー・画面要件](user-journey.md): 同意、撮影、地点、作品閲覧、QR、停止、削除
- [写真メモリー・経路再構成仕様](photo-memory.md): 端末写真とmetadataの分離、固定Graph、作品画像

## システム

- [システム構成](architecture.md): Web、Worker、Go、DB、描画、旧ストレージの責務
- [API・WebSocket仕様](api.md): 本人・運営・描画の経路と資格、metadata、到着・再視聴
- [データモデル](data-model.md): 現行Memoryとmetadata、旧データ、関連付け、期限清掃
- [状態遷移](state-machine.md): 参加と展示再生を分離した状態管理
- [座標・Graph・3D座標変換](coordinate-mapping.md): 固定地点と3Dモデルの互換版
- [UML図・システムフロー](uml-diagrams.md): 主要な処理とコンポーネント間の流れ
- [インフラ・データベース方針](infrastructure-database-decision.md): Cloudflare＋Google Cloud東京、PostgreSQL、環境分離
- [Cloudflare配置比較の記録](cloudflare-web-backend-option.md): 旧候補比較と完成構成の関係

## 運用・検証

- [配置・展示運用・障害対応](deployment-operations.md): 完成構成の保守、障害時判断、[終了手順](deployment-operations.md#終了手順)
- [機材構成](hardware.md): ブラウザ描画機・投影・回線の役割、撤収の確認境界
- [プライバシー設計](privacy.md): 同意、端末／サーバーの保存境界、本人権利、旧データ
- [リスク台帳](risk-register.md): 完成構成と終了後管理に残るリスク
- [試験計画・検証記録の扱い](test-plan.md): 現行仕様の検証項目、過去の記録、未確認の実機受入

## 計画・判断履歴

- [ロードマップ](roadmap.md): 完成版の到達点、会期終了後の確認、再展示の判断境界
- [プロジェクト計画・意思決定](project-plan.md): 担当・予算の記録、主要判断、事後記録方針、設計改訂記録
- [初期意見書](initial-discussion0810.md): 初期の開発方針に関する記録。現行仕様ではない

旧仕様本文と旧納期は[改訂前の設計記録](https://github.com/Mino829/trace-campus-installation/tree/56267484c825764288152ed87cd47c0b5fb73feb)およびGit履歴で追跡する。完成仕様に戻さない廃止案と、採否未決の将来案は区別する。
