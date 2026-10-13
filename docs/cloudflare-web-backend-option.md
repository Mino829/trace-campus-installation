# Cloudflare Web + Google Cloud Backend 配置案

- 記録日: 2026-10-13
- 状態: 比較検討中。参加者Webと通信境界は実装用ブランチで検証中。正式な配置決定、大学のデータ配置承認、本番公開を意味しない
- 対象: 参加者向けWeb App、Go Backend、正本DB、写真保存、描画接続
- 既存の採用方針: [インフラ・データベース方針](infrastructure-database-decision.md)のVercel + Google Cloud東京。正式に変更するまでは同文書を正とする

## 検討の背景

`trace_Frontend`の`main`には位置精度調査用の`/log`があり、Cloudflareの静的アセット用`trace-log`と、通知・記録用`trace-push` Workerを使うデモが稼働している。Workerは調査記録をD1、添付写真をKVへ保存する。これらは本番の写真メモリーやリアルタイム存在表現の正本ではない。

同リポジトリの`mino/demo`ブランチにはGo APIに接続する参加者画面がある。ただし、その`/api/backend/[...path]`はNode.jsのNext.js Route Handlerであり、現行の静的書き出し方式へそのまま載せられない。`main`の調査画面と`mino/demo`の参加者画面を統合する際、この通信境界を先に決める。

Go BackendはPostgreSQLを使う通常のHTTPサーバーとして実装され、OCIコンテナとCloud Run / Cloud SQLの配置手順、検証環境向けCI/CDがある。現時点の実装はセッション、位置受付、停止、削除が中心で、写真保存、到着QR、描画WebSocket、運営API、レート制限は未実装である。サーバーを配置してもMVPの全導線が成立したことにはならない。

## 推奨する比較案

```text
参加者スマートフォン
  -> Cloudflare: Web App、必要ならAPI中継Worker
  -> Google Cloud東京: Go API（Cloud Run）
       -> Cloud SQL for PostgreSQL: 本番の正本データ
       -> 非公開Cloud Storage: 加工済み写真
会場のUnity / TouchDesigner
  -> Go Backendへ外向きWSS接続（描画WebSocket実装後）
Cloud Scheduler -> Cloud Run Job: 保存期限の清掃
```

- Web AppはCloudflareでの配信を第一の比較候補とする。画面が静的書き出しで成立する場合は現行デモの配信方式を活用する。サーバー機能が必要な場合はCloudflare上の動的実行方式を別途検証する。
- Go Backend、正本PostgreSQL、写真本体はGoogle Cloud東京にまとめ、既存のGoコンテナとPostgreSQLスキーマを活かす。大学の審査で別構成が必要な場合は既存のAWS代替案と再比較する。
- `trace-push`のD1/KVは位置精度調査の用途に限定する。本番の参加者データへ流用せず、同意、保存期間、削除、運用者、データ配置を調査用と本番用でそれぞれ確認する。
- `local`、`staging`、`production`の設定、データ、資格情報を分離する。本番データをデモや検証環境へコピーしない。

## WebとGoの通信境界

実装用ブランチでは静的書き出しを選び、`trace_Frontend`のサイトWorkerに同一オリジンの`/api/backend/*`中継を追加した。許可したパス・メソッド、JSONサイズ上限、転送ヘッダー、エラー応答を限定する。`trace_Backend`は設定された共有シークレットを`/v1/*`で照合する。参加トークンをURLや公開環境変数へ載せない。これは比較案の技術検証であり、配置方針の正式変更ではない。

Go APIをブラウザから直接呼ぶ方式も比較対象に残す。その場合はCORS、公開エンドポイント、レート制限、Cloud Runの認証境界を確定する。実装した共有シークレットはWorker経由の制限であり、参加者認証やレート制限を代替しない。現行の非公開stagingはIAM認証が必要で、共有シークレットだけではWorkerから到達できない。公開経路とデータ配置の承認が済むまでCloud Runの公開設定は変更しない。描画用WSSの経路はHTTP APIと分けて疎通試験する。

Cloud RunのWebSocketにはリクエスト時間の上限がある。描画クライアントに再接続と状態再同期を実装し、複数インスタンスへ拡張する際は配信状態と主描画機の一意性を検証する。

## 全部Cloudflareに置く案との比較

| 案 | 利点 | 追加の作業・制約 |
|---|---|---|
| Cloudflare Web + Google Cloud Backend | 稼働中のWeb配信を活かし、Go/PostgreSQLの実装と東京配置方針を維持できる | 2サービス間の認証、公開経路、リリース、監視を統合する |
| Cloudflareへ全面移行 | アプリ運用の窓口をまとめられる可能性がある | GoにはContainers等が必要。現時点のContainersには組み込みの自動スケーリングがなく、配置指定はAPAC単位。D1は既存PostgreSQLの置き換えではなく、移行するならスキーマと処理を書き直す。写真保存とWebSocket配信も再設計する |
| 既存のVercel + Google Cloud方針 | 文書と配置手順の変更が少ない | 現在のCloudflareデモとはWebの運用が分かれる |

CloudflareのD1/R2の`apac`指定は東京保存の保証ではない。保存地域が承認条件になる場合は、利用サービスごとに保証範囲を確認する。費用の優劣は利用量、常時接続数、写真容量が決まってから見積もる。

## 決定前に確認すること

1. 大学が承認するWeb、API、DB、写真、調査ログの運用者と保存・処理地域を確認する。
2. 実装用ブランチの静的書き出し＋Worker中継をCIと検証環境で確認し、既存`/log`との共存を試す。
3. Go APIへの公開経路、参加者認証、CORS、レート制限、受付停止、WorkerからGoへの認証を設計する。
4. Cloud Run / Cloud SQLのstagingで、参加開始・位置送信・停止・削除をCloudflare側WebからE2E試験する。写真・到着・描画は各実装後に追加試験する。
5. WebSocketの再接続、期限削除、DB接続上限、監視、ロールバック、50同時追跡と60セッション相当を試す。
6. 技術・個人情報・大学担当が正式な提供者を決めたら、[インフラ・データベース方針](infrastructure-database-decision.md)、[システム構成](architecture.md)、[配置・展示運用](deployment-operations.md)、[プロジェクト計画](project-plan.md)のVercel記述と公開手順を同じ変更単位で更新する。

## 参照した公式資料

- [Next.js: Static Exports](https://nextjs.org/docs/app/guides/static-exports) — リクエストに依存するRoute Handlerは静的書き出しの対象外
- [Cloudflare: Containers FAQ](https://developers.cloudflare.com/containers/faq/)・[Placement](https://developers.cloudflare.com/containers/concepts/placement/) — Goコンテナの運用と配置制約
- [Cloudflare: D1 Data Location](https://developers.cloudflare.com/d1/configuration/data-location/)・[R2 Data Location](https://developers.cloudflare.com/r2/reference/data-location/) — 位置ヒントと保存地域の保証範囲
- [Google Cloud: Cloud Run WebSockets](https://docs.cloud.google.com/run/docs/triggering/websockets) — 接続時間、再接続、複数インスタンス
