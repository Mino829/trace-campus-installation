# システム構成

## 全体構成

```text
参加者スマートフォン
  Next.js Web App
  位置取得・同意・開始停止・到着操作
          |
          | HTTPS / WebSocket
          v
Go Backend
  匿名セッション・位置検証・軌跡保存・イベント配信
          |
          | WebSocket（必要に応じてOSCブリッジ）
          v
Unity または TouchDesigner
  3Dキャンパス・人物表現・到着演出
          |
          v
プロジェクター / 音響 / 照明
```

## Web App

- QRコードから匿名参加
- 位置情報利用の説明と同意
- `watchPosition` による位置取得
- 緯度、経度、精度、時刻の送信
- 追跡停止と到着チェックイン
- 権限拒否・通信切断時の案内

## Go Backend

- 匿名セッション発行
- 位置点の検証とキャンパス範囲判定
- GPSジャンプの検出、表示用データの平滑化
- 軌跡の保存と削除
- 描画クライアントへのリアルタイム配信
- 到着イベントと軌跡再生データの生成
- ヘルスチェックと運営用状態API

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
  "type": "position.updated",
  "participantId": "anonymous-id",
  "latitude": 0,
  "longitude": 0,
  "accuracy": 0,
  "recordedAt": "ISO-8601"
}
```

```json
{
  "type": "participant.arrived",
  "participantId": "anonymous-id",
  "trajectoryId": "trajectory-id"
}
```

## 最初に検証する項目

- iOS / Androidでの位置取得精度
- 画面状態による位置更新の変化
- 建物付近と屋内の誤差
- 学内Wi-Fiとモバイル回線の切り替え
- 同時参加と複数到着の処理

