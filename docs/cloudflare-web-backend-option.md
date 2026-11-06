# Cloudflare Web + Google Cloud Backend 配置比較の記録（完成版）

- 更新日: 2026-11-06
- 比較記録: 2026-10-13、技術配置の採用判断: 2026-10-21（設計Issue #10）
- 現行正本: [インフラ・データベース方針](infrastructure-database-decision.md)

会期は終了した。本書は採用に至った比較と完成時点の判断を記録し、未採用案の実装や再展示・公開を指示しない。実装の存在・過去配備記録と、正式承認・現在の稼働/停止・清掃完了は別である。

## 検討の背景

当初のVercel + Google Cloud案に対し、既存のCloudflare静的配信を再利用するためサイトWorker中継を採用した。Next.js静的書き出しへリクエスト依存のNode.js Route Handlerをそのまま配置する案ではなく、Workerが許可リストを持ってGoへ中継する。

旧`trace-log`、通知・調査用`trace-push`とD1/KVは独立した調査系である。完成版の本番Worker `trace-production`へ旧参加資格・鍵・D1/KV・写真を移行するものではない。

## 採用した案

採用した完成構成の図は[システム構成](architecture.md)、配置・資格の契約は[インフラ方針](infrastructure-database-decision.md)を参照する。以下は旧比較から現在の構成へ移った判断の記録である。

- 固定公開設定は`push-worker/production.public.json`。会期中の本番URLは`https://trace-production.async5855.workers.dev`、Goは`trace-fes-2026`の東京Cloud Runへの配備記録がある。現在の稼働確認ではない。
- 新規写真は自己申告の固定Graph地点を選び、Goには`clientPhotoId/checkpointId/capturedAt`だけを送る。新規画像upload、GPS取得/送信、Live作成・位置書込み、Unity/TouchDesigner、写真ピン/フォトスポットは完成版の機能ではない。
- 旧Liveと旧クラウド写真の本人権利・`deleteAfter`清掃、旧署名PUTのfenceを維持する。GCSを「新規写真を保存する現行構成」と説明しない。
- production/local/stagingと旧調査系のデータ・secret・originを分け、本番データを試験へ複製しない。

## WebとGoの通信境界

参加者通信は同一originの`/api/backend/*`で、Workerが許可メソッド・パス・JSONサイズ・headerを限定し、Edge secretを付与する。参加者Bearerは別認証。画像をWorker経由にしたり、GCSへ新規直接PUTしたりしない。

本番`/api/operator/*`は個人運営資格の発行・失効・到着proof取得専用で、status、受付close、通知stop、fallback等を通す一般管理proxyではない。それらの終了操作には、権限・許可Origin・Edge保護を備える別の認可済み運営入口の確認が必要となる。本番展示PCへloopback brokerやEdge secretを配布しない。

描画資格発行とWSSはブラウザからGoへ直接接続する。Google本人確認から得るdisplayTokenとoperator credentialは独立し、署名鍵・Edge secretも兼用しない。Cloud Runの接続時間上限に対し、短命資格の再発行、再接続、snapshot全置換、単一主leaseで整合させる。共有secretはCloud Run IAMや正式公開承認を代替しない。

## 全部Cloudflareに置く案との比較

以下は当時の比較であり、外部サービスの2026-11-06時点の機能を再調査した結果ではない。

| 案 | 当時の利点 | 当時の課題・現在の判断 |
|---|---|---|
| Cloudflare Web + Google Cloud Backend | 静的配信を再利用し、Go/PostgreSQLと東京の正本配置を維持 | 採用。2提供者の認証・リリース・監視・費用を分けて管理 |
| Cloudflare全面移行 | 運用窓口を集約できる可能性 | 未採用。Go実行基盤、PostgreSQL相当への移行、WSS/lease/清掃を再設計する必要がある。D1は既存DBの無変更置換先ではない |
| Vercel + Google Cloud | 初期案との文書差分が小さい | 未採用。別Web実行環境を増やす理由がなくなった |
| AWS東京Backend | 提供者変更時の再検討先 | 現行採用先ではない。承認・移行・費用の別判断が必要 |

Cloudflareの配置ヒントやAPAC指定を東京保存の保証としない。DB/旧bucketが東京でもCloudflareの通信・ログ、認証・Pushを含む全処理が東京とはいえない。大学の審査・保持条件はサービス単位で確認する。

## 公開前に残る確認

見出しは旧参照用に維持する。現在は「開場までの未実装一覧」ではなく、会期後保守・再配備/再展示時に確認すべき事項である。

1. 現在の実資源、CI有効状態、受付/通知停止、期限清掃成功、旧画像/fence残数を確認する。停止は[終了手順](deployment-operations.md#終了手順)の順で判断する。
2. Frontend mainの検証成功後本番自動配備と、Backendの条件付き閉鎖staging自動更新を区別する。会期終了で無効になったとはしない。
3. 本人閲覧・削除期限と清掃が残る間にGo/DB/GCS権限を切らない。端末写真とサーバーmetadataの削除を別に扱う。
4. 実Google、実スマートフォン、投影機材、実Push、長時間運転、復旧・費用、正式同意承認の証跡を確認する。過去UI smokeやテストコードの存在を全項目合格としない。
5. Backend #53の費用停止・共通アカウント資源整理は完了未確認。アラートや最小インスタンス0を課金ゼロとしない。

## 参照した公式資料

比較時に参照した資料。サービスの再選定時には最新条件を再確認する。

- [Next.js: Static Exports](https://nextjs.org/docs/app/guides/static-exports)
- [Cloudflare: Containers FAQ](https://developers.cloudflare.com/containers/faq/)・[Placement](https://developers.cloudflare.com/containers/concepts/placement/)
- [Cloudflare: D1 Data Location](https://developers.cloudflare.com/d1/configuration/data-location/)・[R2 Data Location](https://developers.cloudflare.com/r2/reference/data-location/)
- [Google Cloud: Cloud Run WebSockets](https://docs.cloud.google.com/run/docs/triggering/websockets)

## 現行実装の根拠

[Frontend README](https://github.com/Mino829/TRACE_F/blob/be94bd2/README.md)、[本番固定設定](https://github.com/Mino829/TRACE_F/blob/be94bd2/push-worker/production.public.json)、[Worker許可経路](https://github.com/Mino829/TRACE_F/blob/be94bd2/push-worker/src/site.ts)、[Backend README](https://github.com/Mino829/TRACE_B/blob/6b87961/README.md)。配置比較の採用と実配備・承認を混同しない。
