# システム構成

## 全体構成

```text
参加者スマートフォン
  同意・位置取得・開始停止・到着・削除
          |
          | HTTPS
          v
公開HTTPS環境
  Next.js Web App
  Go Backend
  単一の正本DB
          |
          | WSS（描画機から外向き接続）
          v
Unity または TouchDesigner
  3Dキャンパス・人物表現・到着演出
          |
          v
プロジェクター / 音響 / 照明
```

本番のGo BackendとDBは公開HTTPS環境に置き、参加者端末と描画機の双方から外向き接続する。会場PC上のGo Backendは開発または公開環境が使用不能な場合の緊急構成とし、同時に二つの正本DBを動かさない。

## Web App

- QRコードから匿名参加
- サーバーから同意文を取得し、位置権限要求前に説明と同意
- `watchPosition` による位置取得
- 緯度、経度、精度、時刻の送信
- 追跡状態、最終更新、未送信状態の表示
- 追跡停止、即時削除、署名付き到着チェックイン
- 権限拒否・通信切断時の案内
- 到着待ちの色・記号、順番、再生スキップの表示

## Go Backend

- 匿名セッション発行
- 位置点の検証とキャンパス範囲判定
- GPSジャンプの検出、表示用データの平滑化
- 軌跡の保存と削除
- 描画クライアントへのリアルタイム配信
- 到着イベントと軌跡再生データの生成
- ヘルスチェックと運営用状態API
- 同意文バージョン、保存期限、受付状態の配信
- レート制限、到着証明検証、期限削除

## 描画

### Unityを選ぶ場合

- 3Dキャンパス、カメラ、到着シーケンスを中心に構築
- GoからWebSocketでイベントを受信
- 完成したアプリケーションとして固定運用

### TouchDesignerを選ぶ場合

- 粒子、残像、音響同期、プロジェクション表現を中心に構築
- WebSocketまたはOSCでデータを受信
- 展示中にパラメータを調整しやすい構成

## 主要イベント案

```json
{
  "schemaVersion": 1,
  "eventId": "event-id",
  "type": "position.updated",
  "occurredAt": "ISO-8601",
  "data": {
    "sessionId": "anonymous-id",
    "x": 0,
    "z": 0,
    "uncertaintyM": 10,
    "displayAlias": {"color": "cyan", "symbol": "circle"}
  }
}
```

```json
{
  "schemaVersion": 1,
  "eventId": "event-id",
  "type": "participant.arrived",
  "occurredAt": "ISO-8601",
  "data": {
    "arrivalId": "arrival-id",
    "sessionId": "anonymous-id"
  }
}
```

描画クライアントへ生の緯度経度を送らない。通信契約の正本は[API・WebSocket仕様](api.md)とする。

## セキュリティ境界

- 参加者、運営、描画の資格情報とAPI権限を分離する
- 参加者トークンをURL、アクセスログ、描画イベントへ含めない
- 運営APIは公開参加経路と分離し、短期資格情報と操作履歴を使用する
- 描画接続は主系を一台に限定し、到着演出の二重処理を防ぐ
- 開発・試験・本番を分離し、本番の位置データを開発へコピーしない

## 最初に検証する項目

- iOS / Androidでの位置取得精度
- 画面状態による位置更新の変化
- 建物付近と屋内の誤差
- 学内Wi-Fiとモバイル回線の切り替え
- 同時参加と複数到着の処理
