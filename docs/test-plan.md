# 試験計画・公開判定基準（完成版）

- 更新日: 2026-11-06
- 対象: Frontend `be94bd2`、Backend `6b87961`の実装確認と、会期終了後の保守・終了判断

## 目的

現行の端末内写真・自己申告地点metadata・three.js展示・本人権利・期限清掃を検証するための項目と、既存ソース証跡の範囲を明確にする。会期は終了し再展示は未定であり、「開場までに旧GPS要件を達成する」工程表ではない。**本改訂ではアプリのbuild/lint/test・実機受入・クラウド停止/清掃を実施していない。以下のシナリオは試験合格記録ではない。** 実行した文書検証は末尾に分けて記録する。

## 証跡の区分

| 区分 | 確認できること | 代替できないもの |
|---|---|---|
| 実装/試験ソース | 担当する機能・拒否条件・試験シナリオが存在する | 当該SHAでの実行成功・実配備・実機適合 |
| stand-in/fixture/emulator | 模擬API/WS/objectでのUIや境界を対象にした試験 | 実Go transaction、実Cloudflare TLS/IAM、実GCS権限 |
| 隔離DB/実Goの試験 | DB・HTTP・冪等性・FIFO・清掃等の実装検証対象 | 本番設定、実Google/スマホ/投影機/Push、正式承認 |
| ソース文書内の過去作業記録 | 当時の配備・UI smokeの記載 | 今回の実行、現在の稼働/停止、全参加データ消去、正式公開受入 |
| 今後必要な実機/運用受入 | 担当者が対象・日時・版・結果を採録する項目 | 未実施のまま合格とすること |

Frontend READMEには`trace-fes-2026`/`trace-production`の過去配備、仮想カメラで参加→写真/地点→実QR到着→展示完了→本人削除の記録がある。arrival-issuer READMEには同一origin bridge切替後の実Chrome・実Google・loopback要求0件・写真0枚到着/完了・資格失効等の記録もある。これらは実スマートフォン・別メンバー物理PC・投影機材・実30分Push・8時間耐久の受入を代替しない。今回の試験結果ではない。

## 試験環境

- 開発/自動試験は合成参加・合成画像・使い捨てDB。共有staging/本番をE2E対象に指定しない。
- 本番・旧調査Worker・閉鎖stagingのorigin、鍵、DB、同意版、Graphを分離する。
- 実機試験は別承認を得たデータと経路で行う。既存参加者の写真/metadataをデモ・負荷試験へ転用しない。
- Backend `go test ./...`はDB環境変数なしで統合試験をskipし得る。成功終了だけでDB試験合格としない。`make integration`は専用PostgreSQLで行う契約。
- migration/cleanupは実データを変更・削除するコマンド。検証名で本番へ実行せず、隔離環境と終了管理操作を分ける。

## 端末・条件マトリクス

従来のiOS 16.4以上/Android Chrome 111以上は対象候補で、全端末適合済みの保証ではない。再展示する場合はiPhoneとメーカーの異なるAndroidを含め実機の機種/OS/ブラウザを記録し、必要な機能を確認する。iPhone Pushのホーム画面追加条件は、通常の作品閲覧/参加の必須条件と混同しない。

| 分類 | 確認条件 |
|---|---|
| 端末/ブラウザ | iPhone Safari/ホーム画面Web App、Android Chrome、対象外/アプリ内ブラウザへの案内 |
| 保存領域 | 通常/プライベート/容量不足/データ消去、IndexedDB・localStorageの分離/復元限界 |
| カメラ | 前後、縦横、高解像度、拒否/取消/過去拒否/故障、撮り直し |
| 回線 | Wi-Fi/モバイル、切替、低速、切断、再読込、複数タブ |
| 通知 | 任意同意・OS拒否・解除・遅延/重複/未達、実providerと実受信 |
| 展示機 | 実Google、直接WSS、主/待機機、再接続、投影QR読取、排熱/スリープ |
| アクセシビリティ | 200%拡大、読み上げ、キーボード、言語切替、カメラを使わない作品閲覧 |

## 主要E2E試験

T番号は旧資料からの参照用に可能な範囲で維持した。旧要件番号の達成を意味しない。各項目の実施状態は今回未実施。

| ID | 現行シナリオ | 確認する条件 |
|---|---|---|
| T-001 | `/log`で設定・同意・Memory作成 | 本文/版/言語を示し明示同意、成功したMemoryだけを開始。閉鎖/失敗/同意版409で偽成功しない。GPS権限/Live作成なし |
| T-002 | Memory参加停止 | 空body・本人Bearer・固定keyで終端化。Push/待機/再生除外。期限・本人閲覧/削除を維持、端末画像全削除と混同しない |
| T-003 | Memory・旧Live個別/両方削除 | 資格分離・部分成功・同じkey再送、残る資格保持、ExperienceLinkと所有データの整合。サーバー成功だけで端末全消去としない |
| T-004 | 同じoriginの展示QRで初回到着 | 有効proofと本人Memoryで一度だけroot到着、FIFO・現状態復旧。写真0枚でも到着し1枚なら到着区間を作る |
| T-005 | 改変/期限切れ/別環境/使用済みproof | 新規操作を拒否。受理済み同一再送の成功復元は別条件。同じQRを別Memoryが使えることを物理来場保証としない |
| T-006 | metadata送信/到着の応答喪失・回線復旧 | 同じ入力/keyで再送し二重登録せず、画像バイト・旧GPSの再送なし。ローカル画像保存とmetadata受付を分けて観測 |
| T-007 | 再読込/複数タブ/別ブラウザ | 同じブラウザの本人資格・写真・同意・状態を保持。別保存領域で復元できない場合を案内し、偽の新参加へ移行しない |
| T-008 | skipと運営retry | 本人skipは未再生対象、運営retryは描画障害skip等の契約条件だけ、新公開再生IDでFIFO末尾。任意割込みなし |
| T-009 | 主機切断/新資格再接続/待機機 | snapshot全置換、主lease一意、世代/lease不正の完了拒否、切断skipの自動二重再生なし |
| T-010 | fallback/復帰 | 再生中1件だけpending、30秒固定期限、待機列保持、合成映像。受付/通知は別停止、明示復帰でFIFO再開 |
| T-012 | 拡大/読み上げ/日英切替 | 参加・停止・作品閲覧/保存・削除を操作可能。資格/画像を作り直さず、対応版だけ同意参考英訳 |
| T-013 | 撮影・地点選択・端末保存 | IndexedDBにJPEG。Go v2はclientPhotoId/checkpointId/capturedAtだけ、12枚上限、同内容再送/異内容競合、削除ID復活なし |
| T-014 | 写真変換 | 向き補正、長辺1280px以下JPEG、EXIF/GPSを引き継がず端末保存。新規画像のGo/GCS送信0件 |
| T-015 | 固定Graph経路 | JSON本体/3版を一致させる。自己申告地点の最短経路であり実歩行履歴と称さない。削除で再計算、旧版Session/旧GPSを混ぜない |
| T-016 | 到着前/後/停止後の「作品を見る」 | QR・到着status照会・metadata待ちを開始せず本人ローカル作品/JPEG保存。参加・到着・通知を変えず、ホームへ戻れる。0枚は案内 |
| T-017 | 写真単位/Memory削除と再送 | IndexedDBとGo metadataを別確認。失敗を成功扱いせず、client ID fenceで遅延再送が復活しない。旧画像はGCS完了も確認 |
| T-018 | 公開描画/運営/ログ出力 | 原写真・画像URL/bytes・本人ID/Bearer・raw proof/秘密なし。地点/時刻metadataの残余リスクと画面非表示を確認 |
| T-019 | カメラ拒否/写真0枚/到着後撮影 | GPS許可要求なし。0枚到着可能。有効Memoryの到着後撮影は端末内のみで新metadataを送らない |
| T-020 | 旧写真本人アクセス | 未認証・別Bearer・期限切れURL・一覧取得を拒否。停止/失効後もdeleteAfter内は本人閲覧/削除を保持 |
| T-022 | 任意30分通知 | 別同意/OS許可、写真登録で延期、解除/停止/期限/削除/provider無効/運営stopを確認。未達/重複を許容する契約と区別 |
| T-023 | 資格相互利用と失効 | Memory/旧Live/display/operator/Google/Edgeの目的分離。現在/全operator失効、再ログイン禁止とは異なることを確認 |
| T-024 | 3版/Graph本体不一致 | WS拒否・API/清掃起動の版確認、保存済み版を改変しない。名前だけ変えた別JSONを接続しない |
| T-026 | migration・権限・復旧 | 初回/連続/再実行/失敗・不整合schema拒否、runtime権限、API/cleanup対応版、復元後の期限再削除 |
| T-027 | 単一Backend負荷 | DB接続予約枠・pool/lock・プロセス制限を測定。scale-out/新旧revision並存を負荷試験の都合で導入しない |
| T-028 | 環境分離 | local/stagingから本番secret/DB/旧画像へ無断到達しない。本番データcopyなし |
| T-029 | 配備経路・互換切替 | Frontend検証済みSHA本番配備、Backend閉鎖stagingだけ、secret非複製。旧GPS/画像upload版へ単純rollbackしない |
| T-030 | 非対応環境 | 必要機能の制限とブラウザ案内を表示。Push不可を参加全体の拒否条件にしない |
| T-031 | タブ/外部QR/再スキャン | proofをlocalStorageへ保存せず、同画面再送入力を固定、参加前proofを持越さない。同じoriginのみ、失敗でカメラ自動起動なし |
| T-032 | 完了後の新QR本人再視聴 | fresh proofで別replay API、FIFO末尾・新公開再生ID。初回到着/期限/Push方針を不変、queued/playingは既存状態へ合流 |
| T-033 | 到着後通知の新同意 | 新版と一致する新Memoryだけ継続。旧false/停止済み購読を昇格/復元しない。020/API/dispatcher/cleanupを揃える |
| T-034 | 終了操作 | status観測、受付closeと通知stopを別確認、本人権利/旧画像fence/cleanup維持、費用と撤収を別記録 |

## 位置・表現試験

新規GPS歩行精度・5秒位置送信・on_site点表示は現行の試験対象ではない。代わりに固定Graphの自己申告地点選択・通路接続・到着区間・描画bundleを確認する。本人ローカル作品は到着前からShibaLabまで構成でき、Go到着登録と独立する。誤った地点選択や最短経路が実移動と異なることを説明し、現地の臨時閉鎖/利用時間を自動反映するナビゲーションとは扱わない。Graphの既知経路・禁止辺等の試験コードと現場適合は別。

## 写真試験

写真変換、IndexedDB容量・失敗/再読込/複数タブ、0/1/12枚、作品JPEG/単独写真のOS保存を実機で確認する。参加後の撮影、到着後の端末のみ撮影、metadata受付失敗でも画像がローカルに残る場合を分ける。UIにmetadata送信待ちが常に見えるとは仮定しない。通信監査では新規画像bytes/URL/GPS送信0件を確認する。

削除は端末画像、サーバーmetadata、旧GCS画像/fence、OS保存画像、backupを個別に観測する。閉じた端末の画像が期限ジョブで遠隔消去されるという合格条件を置かない。旧GCSは本人URL・deleting再試行・署名PUT後書込み防止・期限後outbox再消去を別の隔離fixture/承認済み実環境で確認する。

## 負荷・耐久試験

旧50同時GPS追跡/5秒送信/20端末画像uploadの目標は廃止。現行ではmetadata登録/削除、到着・再視聴、待機50件（playing除外）・満杯からの回復、単一主lease・再接続・snapshot、DB pool/lock・清掃と通知dispatcherの競合を測定する。旧60セッションや8時間は旧計画の試験候補であり、実施・合格の証跡は今回未確認。本番参加実績・同時上限の証拠にはしない。

実施する際は同時数/端末数・画像数・到着率・期間・CPU/メモリ・DB接続・APIエラー・待ち時間・描画fps/切断回数を記録し、再展示の目的に合わせて閾値を事前承認する。測定のないp95や成功率を記入しない。実スマホ・実機投影・長時間運転・実Push・backup復旧の受入結果は今回未確認。

## セキュリティ・プライバシー試験

- 同一origin participant中継のメソッド/パス/16KiB制限、追加画像field拒否、Edge共有secret不正、転送IPなりすまし拒否。
- 本番operator bridgeは資格発行/失効/QR proofだけで、status/close/stop/fallbackを通さない。これらは別認可入口を確認し、未配備の運営UIを仮定しない。
- Googleのissuer/audience/email/Origin/allowlist、operator資格2分・独立権限、メモリ保持、現在/全資格失効、監査に生メール・tokenなし。
- 描画は直接Go資格/WSS、初回認証前snapshot禁止、短命資格/不正/版不一致拒否、lease/世代/event再送整合。URL/query/subprotocolに資格なし。
- Memory/旧Live本人資格と各期限、停止後権利、削除部分成功、端末写真/metadata消去の分離。
- 到着後Push継続は新同意版に固定した新規Memoryだけ。既存同意・期限・push_stopped不変、旧cleanup混在禁止。
- Go通知stopのDB永続化・全購読/鍵/予定消去、provider受理済み取消不可。旧Worker通知は別確認。
- 旧bucket/署名URL/fence、backup/PITR/ログの保持・復元時再削除・アクセス権。期限到来だけで消去済みにしない。

## 公開判定

会期終了により今後の開場期限は設定しない。保守変更は影響する本人権利・清掃・秘密・対応schemaを確認して行う。再展示/受付再開を決定する場合にのみ、現行Must相当の試験結果、実機/実Google/回線/投影、負荷/耐久/復旧、本文/保持/問い合わせ/責任者の承認、fallback・終了操作の実行可能性を証跡付きで判定する。

コード存在、過去smoke、文書の完成を正式公開受入の合格としない。再公開の承認が不明なまま受付を開かない。受付/通知停止済みや課金ゼロを推定せず、[終了手順](deployment-operations.md#終了手順)で個別に確認する。

## 2週間の技術成立判定

旧G-01〜G-06は会期前のGPS・画像upload・写真ピン等を含む検討履歴であり、完成版の工程/未達TODOではない。完成時の変更（端末写真、GPSなし、固定Graph、three.js、任意30分通知）を上表で検証する。旧Gate合格や縮小案の正式承認を本書から推定しない。

## 既存ソース証跡

以下は存在を確認した試験ソース/文書。実行済み一覧ではない。

| 範囲 | 主な参照ソース |
|---|---|
| 参加/端末写真/作品・日英・再スキャン | [log-keepsake.spec.ts](https://github.com/Mino829/TRACE_F/blob/be94bd2/tests/log-keepsake.spec.ts)、[log-tabs.spec.ts](https://github.com/Mino829/TRACE_F/blob/be94bd2/tests/log-tabs.spec.ts)、[upload-privacy.spec.ts](https://github.com/Mino829/TRACE_F/blob/be94bd2/tests/upload-privacy.spec.ts) |
| 描画/QR/Google・資格/再視聴 | [display-link.spec.ts](https://github.com/Mino829/TRACE_F/blob/be94bd2/tests/display-link.spec.ts)、[display.spec.ts](https://github.com/Mino829/TRACE_F/blob/be94bd2/tests/display.spec.ts)、[環境別確認記録](https://github.com/Mino829/TRACE_F/blob/be94bd2/arrival-issuer/README.md) |
| Go写真metadata・削除 | [api/photo_moments_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/photo_moments_test.go)、[store/photo_moments_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/photo_moments_test.go) |
| 旧Live・Memory停止/保持 | [api/legacy_live_retention_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/legacy_live_retention_test.go)、[api/memory_sessions_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/memory_sessions_test.go) |
| FIFO/再視聴・lease | [store/arrival_fifo_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/arrival_fifo_test.go)、[store/arrival_replay_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/arrival_replay_test.go)、[api/display_queue_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/display_queue_test.go) |
| 運営/停止・fallback | [api/operator_status_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/operator_status_test.go)、[api/registration_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/registration_test.go)、[api/notifications_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/notifications_test.go)、[api/fallback_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/api/fallback_test.go) |
| 到着後通知/移行 | [store/push_after_arrival_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/push_after_arrival_test.go)、[store/push_after_arrival_migration_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/push_after_arrival_migration_test.go)、[store/migrate_test.go](https://github.com/Mino829/TRACE_B/blob/6b87961/internal/store/migrate_test.go) |
| 配備条件と検証の境界 | [Frontend README](https://github.com/Mino829/TRACE_F/blob/be94bd2/README.md)、[Frontend workflow](https://github.com/Mino829/TRACE_F/blob/be94bd2/.github/workflows/deploy.yml)、[Backend README](https://github.com/Mino829/TRACE_B/blob/6b87961/README.md)、[Backend workflow](https://github.com/Mino829/TRACE_B/blob/6b87961/.github/workflows/test.yaml) |

## 本改訂の文書検証

2026-11-06、文書だけを対象に次を実行した。Frontend／Backendの動作試験や実環境の受入とは別である。

| 対象 | 実行内容・結果 |
|---|---|
| Markdown 24ファイル | Markedで解析し、内部リンク130件のファイル・見出し参照をGitHub形式のslugで解決。参照不整合0件、閉じ忘れたコードfence0件 |
| JSON例15件 | JSONとして解析成功。実APIへの送信試験ではない |
| Mermaid図12件 | Mermaid 12.3.0で構文解析成功。図を含む6ページでSVG計12件の生成を確認 |
| 実装参照先65パス | 固定版に対応するローカルFrontend／Backendでファイル・ディレクトリの存在を確認。外部サイト全体のリンク生存確認ではない |
| 読者導線 | 一時ローカルHTMLでREADMEから現在状況へ実際に遷移し、構成図・UML図のブラウザ表示を確認 |

検証用rendererと補助処理はリポジトリへ追加していない。GitHub側のMermaidバージョンや実際のテーマ・表示幅まで保証するものではない。
