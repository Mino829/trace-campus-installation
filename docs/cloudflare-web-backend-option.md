# Cloudflare Web + Google Cloud Backend 配置比較の記録

- 記録日: 2026-10-13、採用判断: 2026-10-21（Issue #10）
- 状態: 比較案のうちCloudflare Web + Google Cloud東京を技術構成として採用。本番配置・大学承認・公開完了ではない
- 対象: 参加者向けWeb App、Go Backend、正本DB、写真保存、描画接続
- 現在の正本: [インフラ・データベース方針](infrastructure-database-decision.md)。以下は採用に至る比較と検証履歴

## 検討の背景

`trace_Frontend`の`main`には位置精度調査用の`/log`があり、Cloudflareの静的アセット用`trace-log`と、通知・記録用`trace-push` Workerを使うデモが稼働している。Workerは調査記録をD1、添付写真をKVへ保存する。これらは本番の写真メモリーやリアルタイム存在表現の正本ではない。

過去の`mino/demo`ブランチの`/api/backend/[...path]`はNode.jsのNext.js Route Handlerであり、静的書き出し方式へそのまま載せられなかった。この制約からサイトWorker中継を実装した。

Go BackendはPostgreSQLを使うHTTPサーバーとして実装され、OCIコンテナとCloud Run / Cloud SQLの配置手順、検証環境向けCI/CDがある。写真・到着QR・描画WebSocket等の実装と本番統合は別途検証が必要で、サーバー配置だけではMVPの全導線は成立しない。

## 採用した案

```text
参加者スマートフォン
  -> Cloudflare: 静的Web App、同一オリジンAPI中継Worker
     -> Google Cloud東京: Go API（Cloud Run）
       -> Cloud SQL for PostgreSQL: 本番の正本データ
       -> 非公開Cloud Storage: 加工済み写真
会場のUnity / TouchDesigner
  -> Go Backendへ外向きWSS接続（描画WebSocket実装後）
Cloud Scheduler -> Cloud Run Job: 保存期限の清掃
```

- Web AppはCloudflareのサイトWorkerで静的書き出しを配信する。動的なNext.js Route Handlerは使わない。追加のサーバー機能が必要な場合は別途設計する
- Go Backend、正本PostgreSQL、写真本体はGoogle Cloud東京にまとめる。大学の審査で別構成が必要な場合はAWS等の代替案を再決定する
- `trace-push`のD1/KVは位置精度調査に限定する。本番の参加者データへ流用せず、通知購読の本番保存先も別途決定する。D1/KVの東京保存・処理を保証しない
- `local`、`staging`、`production`の設定、データ、資格情報を分離する。本番データをデモや検証環境へコピーしない

## WebとGoの通信境界

静的書き出しとサイトWorkerの同一オリジン`/api/backend/*`中継は実装済み。許可したパス・メソッド、JSONサイズ、転送ヘッダー、エラー応答を限定する。Goは共有シークレットを`/v1/*`で照合する。参加者Bearerは別の認証であり、シークレットをブラウザへ渡さない。採用決定は既存実装の本番疎通の証明ではない。

写真メタデータAPIはWorkerの許可リストを拡張する必要がある。画像本体は短期署名URLでCloud Storage東京へ直接PUTする契約であり、Workerの16KB JSON制限を通さない。Cloud Storage側のCORSと署名URL・削除を公開前に試験する。

直接Go APIをブラウザから呼ぶ方式は採用しない。共有シークレットはWorker経由の制限であり、参加者認証やレート制限を代替しない。既存stagingはIAM保護中でWorkerからは共有secretだけでは到達できない。大学承認、直接到達拒否・E2E試験まで公開権限は変更しない。描画用WSSの経路・専用認証は別途決める。

Cloud RunのWebSocketにはリクエスト時間の上限がある。描画クライアントに再接続と状態再同期を実装し、単一インスタンスを維持する。

## 全部Cloudflareに置く案との比較

| 案 | 利点 | 追加の作業・制約 |
|---|---|---|
| Cloudflare Web + Google Cloud Backend | 稼働中のWeb配信を活かし、Go/PostgreSQLの実装と東京配置方針を維持できる | 2サービス間の認証、公開経路、リリース、監視を統合する |
| Cloudflareへ全面移行 | アプリ運用の窓口をまとめられる可能性がある | GoにはContainers等が必要。現時点のContainersには組み込みの自動スケーリングがなく、配置指定はAPAC単位。D1は既存PostgreSQLの置き換えではなく、移行するならスキーマと処理を書き直す。写真保存とWebSocket配信も再設計する |
| 既存のVercel + Google Cloud方針 | 文書と配置手順の変更が少ない | 現在のCloudflareデモとはWebの運用が分かれる |

CloudflareのD1/R2の`apac`指定は東京保存の保証ではない。保存地域が承認条件になる場合は、利用サービスごとに保証範囲を確認する。費用の優劣は利用量、常時接続数、写真容量が決まってから見積もる。

## 公開前に残る確認

1. 大学が承認するWeb、API、DB、写真、調査ログの運用者と保存・処理地域を確認する。東京に置くDBとbucket以外の処理まで東京とは主張しない。
2. 静的書き出し＋Worker中継と既存`/log`の共存をstagingで試す。IAM保護されたstagingを安易に一般公開しない。
3. 独自ドメイン・DNS/TLS・WAF/レート制限・監視の責任分担、Go APIの公開経路、描画WSS、通知購読の保存先を確定する。
4. Cloud Run / Cloud SQLのstagingで、参加開始・位置送信・停止・削除をCloudflare側WebからE2E試験する。写真・到着・描画は実装後に追加試験する。
5. WebSocket再接続、期限削除、DB接続上限、監視、ロールバック、50同時追跡と60セッション相当を試す。予算上限5,000円に対する内訳と大学・個人情報承認を記録する。

## 参照した公式資料

- [Next.js: Static Exports](https://nextjs.org/docs/app/guides/static-exports) — リクエストに依存するRoute Handlerは静的書き出しの対象外
- [Cloudflare: Containers FAQ](https://developers.cloudflare.com/containers/faq/)・[Placement](https://developers.cloudflare.com/containers/concepts/placement/) — Goコンテナの運用と配置制約
- [Cloudflare: D1 Data Location](https://developers.cloudflare.com/d1/configuration/data-location/)・[R2 Data Location](https://developers.cloudflare.com/r2/reference/data-location/) — 位置ヒントと保存地域の保証範囲
- [Google Cloud: Cloud Run WebSockets](https://docs.cloud.google.com/run/docs/triggering/websockets) — 接続時間、再接続、複数インスタンス
