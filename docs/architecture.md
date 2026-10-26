# システム構成

本構成はMemorySession単独セッションで動く（2026-10-26決定、Issue #23）。描画はthree.jsを採用し、Frontendリポジトリの`/display`がそのまま描画クライアントである（2026-10-26決定、Issue #12）。旧Unity / TouchDesigner構成は撤去する。

## 全体構成

```text
参加者スマートフォン
  同意・写真撮影（地点選択）・到着・削除
          |
          | HTTPS
          v
Cloudflare サイトWorker
  Next.js静的Web配信 / /api/backend/* 中継
          |
          | HTTPS（参加者API）
          v
Google Cloud東京（採用する配置先、大学承認前）
  Cloud Run: Go Backend
  Cloud SQL for PostgreSQL 17: 正本DB
  非公開Cloud Storage: 加工済み写真
          |
          | WSS（描画PCから外向き接続）
          v
Frontend リポジトリ /display（three.js）
  3Dキャンパス・写真地点・再構成経路・到着演出
          |
          v
プロジェクター / 音響 / 照明
```

本番構成の技術方針はNext.jsの静的書き出しをCloudflareのサイトWorkerで配信し、Go BackendをCloud Run、正本DBをCloud SQL for PostgreSQL 17、加工済み写真を非公開Cloud Storage bucket（いずれも東京）へ置く。大学の提供者・保存/処理地域と費用の承認は未完了。AWS等への変更は別途決定する。

参加者端末は同一オリジンの`/api/backend/*`を通じてGoへ送信する。ただし写真本体はGoが発行する短期署名URLへCloud Storageへ直接PUTする設計で、現行Workerの16KB JSON中継では転送しない。写真APIのWorker許可リストとCloud StorageのCORSは未実装・未検証。描画PCはthree.jsの`/display`クライアントをFrontendリポジトリで運用し、Go（Cloud Run）へ外向きWSS接続する（2026-10-26決定）。Workerは中継せず、描画PCの認証・公開経路は未確定で、現行Worker中継には含まれない。会場PC上のGo Backendは開発または承認済み緊急構成に限り、本番正本へ同時書き込みしない。写真原画像や画像URLは描画機へ送らない。詳細は[インフラ・データベース方針](infrastructure-database-decision.md)を正とする。

## データ配置

| データ | 保存先 | 方針 |
|---|---|---|
| MemorySession、同意、写真メタデータ、写真地点、到着、再構成経路 | PostgreSQL | 本番の単一正本。期限削除とバックアップ保持を管理する |
| 加工済み写真 | 非公開Cloud Storage（東京） | DBにはobject keyだけを保存し、短時間の署名付きアクセスを使う。セッション失効後に`delete_after`まで保持し、写真・地点・経路・削除冪等性記録を同一期間で一括消去する。公開前に処理地域・削除を承認する |
| 生GPS・位置履歴 | 保存しない | リアルタイム位置表現はIssue #23で廃止（2026-10-26）。旧LivePresenceSessionは本人資格での照会・停止・削除と`deleteAfter`期限清掃のみを残す |
| 合成データ | local / stagingと会場PC | 本番参加者データから作らない |

CloudflareのWorker・転送経路と調査用`trace-push`のD1/KVは東京保存・処理の保証対象ではない。後者を本番の参加者データの正本に流用せず、通知購読情報の本番配置先はTRACE_B#12で未決とする。

## 拡張境界

MVPではGo Backendを単一の主インスタンスとして開始し、セッション・写真保存・到着・描画配信を一つの容量計画として検証する。複数Backendへ拡張するときは、WebSocket fan-outと主描画機リースをRedis互換ストアまたはメッセージ基盤へ分離する。PostgreSQLは正本のまま維持し、接続プール、Backend最大インスタンス数、DB接続上限を管理する。

## Web App

- QRコードから匿名参加
- サーバーから同意文（版`2026-memory-photo-no-gps-draft-1`、jaのみ）を取得し、説明と同意
- カメラ権限は到着QR読取のためだけ要求する。位置権限は要求しない
- カメラ撮影、プレビュー、3Dマップでの写真地点（チェックポイント）選択、向き補正、EXIF除去、縮小、再エンコード
- 写真・地点（checkpointId）・撮影時刻の送信と再送状態の表示
- 到着チェックイン（署名・期限付きQR）、参加停止、削除
- 通信切断・権限拒否時の案内
- 到着後の状態と自動削除予定日の表示
- 本人用の写真ストーリー表示、端末保存、写真単位・全体削除

## Go Backend

- MemorySession単独の発行・認証。LivePresenceSessionは新規発行せず、既存分は本人資格での状態照会・停止・削除と`deleteAfter`期限清掃のみ残す
- 同意文版管理。位置情報項目は同意文へ含めない
- 写真保存（EXIF除去検証、縮小、形式・容量検証）
- チェックポイント地点検証：`checkpointId`の妥当性検証とCampusGraphノード・ローカル座標への解決。参加者端末のpayloadには座標・ノードID・グラフ版を含めない
- CampusGraph最短経路の再構成（撮影順の地点＋最後からShibaLab到着区間）
- 再構成経路の保存と削除（写真・派生データと同一期間）
- Web Push撮影リマインダー（30分ごと、通知なし参加可）と到着時の停止
- 到着検証（署名・期限付きQRの到着証明）とArrivalEvent作成
- CampusGraph・3D Mapの互換版管理と配信切替・ロールバック
- 描画PCへのWSS配信（グラフの点ID列、グラフ版、写真ごとの点IDと撮影時刻のスナップショットと差分）
- 到着イベントと軌跡再生データ（点ID列・撮影時刻を含む）
- ヘルスチェックと運営用状態API
- 同意文バージョン、保存期限、受付状態の配信
- レート制限、到着証明検証、期限削除
- DBとオブジェクトストレージを横断する写真・派生データの削除

## 写真保存境界

- ブラウザは画像の向き、EXIF除去、縮小、再エンコードを行ってから送信する
- Backendも画像形式、寸法、容量を再検証し、信頼できないメタデータを破棄する
- オブジェクトストレージのbucketは非公開とし、推測困難なキーと短時間の署名付き取得を使う
- DBにはオブジェクトキー、状態、ハッシュ、寸法、削除期限を保存し、公開URLを保存しない
- 描画イベントには点ID・撮影時刻だけを含め、写真・PhotoMoment ID・MemorySession ID・画像URL・生GPSを含めない
- 本人用写真ストーリーの取得は参加トークンで認証し、署名付きURLを短時間だけ発行する

## 描画

- Frontendリポジトリの`/display`をthree.jsで実装し、主描画PCのプロジェクターへ投影する
- Goへ外向きWSS接続し、グラフ（CampusGraph）の点ID列・グラフ版・写真ごとの点IDと撮影時刻を受信する
- 3D座標への対応付けは3D Map設定から描画クライアントが行う
- 接続断時はスナップショットから復元する

## 主要イベント

描画PCへ送るイベントは、スナップショットと差分でpointId列の変化とphotosの追加を配信する。

```json
{
  "schemaVersion": 2,
  "eventId": "event-id",
  "type": "participant.arrived",
  "occurredAt": "ISO-8601",
  "campusGraphVersion": "graph-2026-01",
  "threeDMapVersion": "map-2026-01",
  "data": {
    "arrivalId": "arrival-id",
    "pointIds": ["checkpoint-…", "…"],
    "photos": [
      {"pointId": "checkpoint-…", "capturedAt": "ISO-8601"}
    ]
  }
}
```

描画クライアントへ原写真、画像URL、生の座標・GPS、MemorySession IDを送らない。通信契約の正本は[API・WebSocket仕様](api.md)とする。

## セキュリティ境界

- サイトWorkerは許可したAPIだけを中継し、サーバー側のedge secretでGoの`/v1/*`への到達を制限する。Bearerは参加者の個別認証であり、edge secretをブラウザ・描画機へ公開しない。CI・運営のCloud Run操作はIAMを使う
- 参加者、運営、描画の資格情報とAPI権限を分離する
- 参加者トークンをURL、アクセスログ、描画イベントへ含めない
- 運営APIは公開参加経路と分離し、短期資格情報と操作履歴を使用する
- 描画接続は主系を一台に限定し、到着演出の二重処理を防ぐ
- 開発・試験・本番を分離し、本番の位置データを開発へコピーしない

## 最初に検証する項目

- iOS / Androidでの写真保存成功率とEXIF除去
- 展示ブースのQR読み取りと到着送信の成否（定周期発行と有効期限）
- 3Dマップ地点選択操作のユーザビリティ、誤選択時の扱い（再選択の可否は未決）
- 学内Wi-Fiとモバイル回線の切り替え、写真送信の再送
- 同時到着・同時再生の処理とp95
- 30分ごとの撮影リマインダーの配信・停止（到着時停止含む）
- CampusGraph・3D Mapの版不一致の破棄とロールバック
