# TRACE — Campus Movement Installation

大学キャンパスを歩く参加者の匿名化された位置と軌跡を、リアルタイム3DCGとして展示するインタラクティブアート・プロジェクトです。

参加者はキャンパス入口のQRコードからWebアプリを開き、位置情報の利用に同意して参加します。展示場では、現在キャンパス内を移動する人々が光・粒子・生命体として表示されます。参加者が展示場へ到着すると、入口から展示場までの軌跡が時間圧縮されて再生され、集合アートへ加わります。

## 技術構成

- Web App: React / Next.js
- Backend: Go
- Realtime: WebSocket
- Rendering: Unity または TouchDesigner
- Storage: SQLite（MVP）、PostgreSQL（拡張時）
- Arrival: QRコード、将来的にNFC

## ドキュメント

- [設計資料一覧](docs/README.md)
- [作品コンセプト](docs/concept.md)
- [システム構成](docs/architecture.md)
- [データモデル](docs/data-model.md)
- [API・WebSocket仕様](docs/api.md)
- [参加セッション状態遷移](docs/state-machine.md)
- [位置座標・3D座標変換](docs/coordinate-mapping.md)
- [配置・展示運用・障害対応](docs/deployment-operations.md)
- [機材構成](docs/hardware.md)
- [プライバシー設計](docs/privacy.md)
- [ロードマップ](docs/roadmap.md)

## 現在のフェーズ

計画・技術検証。最初の重点は、実際のキャンパスにおけるスマートフォンの位置取得精度と通信安定性の検証です。
