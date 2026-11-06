# TRACE — Campus Movement Installation

キャンパスで撮った写真と、写真の節目をつなぐ経路を3DCGで振り返るインタラクティブアート作品です。写真は参加者の端末内に保存し、自分で選んだ撮影地点を固定のキャンパスGraphでつなぎます。新しいGPSの取得・送信は行いません。

**学園祭会期終了後の完成版設計書（2026-11-06改訂）**。ここでいう完成版は、完成時点のコードに合わせた設計記録です。運用終了・全データ削除・費用精算・正式承認がすべて完了したという意味ではありません。

## 完成版の構成

| 領域 | 完成時点の仕様 |
|---|---|
| 参加者Web | React / Next.js。`/log`で同意・参加・撮影・作品閲覧・停止・削除。日本語／英語表示 |
| 写真 | 新規画像は端末内IndexedDBのみ。Goには`clientPhotoId`・`checkpointId`・`capturedAt`のmetadataを送る |
| 経路・本人作品 | 固定Graphのチェックポイント間を再構成。到着前から作品を閲覧し、作品画像を端末へ保存できる |
| 会場展示 | three.jsの`/display`。Google認証、直接描画WSS、単一主描画機とFIFO再生キュー |
| 到着・再視聴 | 会場QRの短命proofで到着。完了後の有効な参加者は、新しいQR操作で待機列末尾へ再視聴を申し込める |
| 撮影通知 | 本番は任意のGo Web Push、30分間隔。到着後の継続は新規参加時の同意方針で分離 |
| API・DB | Go / PostgreSQL 17。参加停止と削除を分離し、各`deleteAfter`まで本人の権利を維持 |
| 配置 | CloudflareサイトWorker＋Google Cloud東京。参加者・運営APIは同一origin中継、描画は認証付き直接WSS |
| 旧データ | 旧Live・旧クラウド写真の本人操作と期限清掃を維持。新しいGPS・画像アップロードは受け付けない |

完成仕様の基準は[Frontend `be94bd2`](https://github.com/Mino829/TRACE_F/tree/be94bd270bcf99628c0af556e2f60f3dc78e4785)と[Backend `6b87961`](https://github.com/Mino829/TRACE_B/tree/6b879613518745ba9d4541b83eb2a3dbe0dfa697)。実装契約と運用実績の区別は[現在状況・完成版の参照基準](docs/current-plan.md)にまとめています。

## 読む順序

1. [企画概要](docs/proposal.md)・[作品コンセプト](docs/concept.md): 作品の目的と体験
2. [スコープ・要件定義](docs/scope-requirements.md)・[参加者ジャーニー](docs/user-journey.md): 実装した範囲と画面の振る舞い
3. [写真メモリー・経路再構成](docs/photo-memory.md): 端末写真・地点・作品閲覧
4. [システム構成](docs/architecture.md)・[API](docs/api.md)・[データモデル](docs/data-model.md)・[状態遷移](docs/state-machine.md): 実装間の契約
5. [プライバシー](docs/privacy.md)・[配置・展示運用](docs/deployment-operations.md): 保持・削除と終了後の管理

[設計資料一覧](docs/README.md)に、座標・3Dモデル、インフラ、UML、機材、リスク、検証、意思決定の資料を掲載しています。

## 会期終了後の扱い

会期は2026年10月31日〜11月1日で、終了はユーザー確認済みです。教室棟3F301・両日10:00〜18:00は設計記録上の会場・時間です。再展示日程は未定です。

受付・全通知の停止、本人閲覧・削除と期限清掃、クラウド費用、機材撤収の実施結果は未確認です。[終了手順](docs/deployment-operations.md#終了手順)に従って対象環境ごとの証跡を確認します。本改訂は設計文書の更新であり、クラウド操作、実データ削除、受付再開や実環境への配備を行うものではありません。
