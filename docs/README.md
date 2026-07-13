# 設計資料一覧

TRACE MVPの仕様と設計判断をまとめる。実装中に仕様を変更した場合は、コードと同じ変更単位で該当文書を更新する。

## プロダクト

- [作品コンセプト](concept.md): 体験、表現、展示モード
- [ロードマップ](roadmap.md): 開発フェーズと完了条件

## システム

- [システム構成](architecture.md): コンポーネントと責務
- [データモデル](data-model.md): 保存データ、関係、削除方針
- [API・WebSocket仕様](api.md): Webアプリ、Go、描画クライアント間の契約
- [参加セッション状態遷移](state-machine.md): 開始、到着、再生、停止、期限切れ
- [位置座標・3D座標変換](coordinate-mapping.md): GPS検証、平滑化、Unity座標への変換

## 展示運用

- [機材構成](hardware.md): 機材の役割と接続
- [配置・展示運用・障害対応](deployment-operations.md): 配置、監視、復旧、フォールバック
- [プライバシー設計](privacy.md): 同意、匿名化、表示、保存、削除

## 設計上の前提

- Web AppはReact / Next.jsを使用する
- BackendはGoを使用する
- 描画はUnityまたはTouchDesignerを比較後に決定する
- MVPの保存先はSQLiteとし、必要になった時点でPostgreSQLを検討する
- スマートフォンはHTTPSで公開エンドポイントへ接続する
- 描画PCは外向きWebSocket接続を使用する
- 位置取得間隔、精度閾値、同時接続数はキャンパス実地試験で確定する

## 未決事項

- 展示日と制作期限
- 想定する最大同時参加人数
- 大学の位置情報利用・ネットワーク利用許可
- データ保存期間
- UnityまたはTouchDesignerの最終選択
- 3Dキャンパスモデルの制作方法

