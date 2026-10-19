# データモデル

## 方針

- 本番の正本DBはMVPからマネージドPostgreSQLとする。Google CloudではCloud SQL、AWSではRDSを使用し、標準PostgreSQLの範囲を基本として移植性を保つ
- 写真バイト列はPostgreSQLへ入れず、非公開オブジェクトストレージへ保存する
- ER図は論理モデルを表す。`LIVE_POSITION`は永続テーブルではなく、単一Go BackendプロセスのメモリにLivePresenceSessionごとの最新1件を30〜60秒TTLで保持する。複数Backend化するときだけRedis互換TTLストアへ移す。PostgreSQLの`live_positions`は使用しない
- 個人名、学籍番号、メールアドレス、端末固有IDを保存しない
- 写真用MemorySessionとリアルタイム用LivePresenceSessionを別ID・別トークン・別同意版で管理する
- 両体験を同じ端末で使う場合だけ、ExperienceLinkで色・記号、到着処理、本人向け状態表示を連携する。連携トークンの有効期間は短くし、読み取りのための最小限の関連付けは削除期限まで維持する
- 通常GPSの生緯度経度と点列は永続化せず、量子化済みの最新表示位置だけを30〜60秒保持する
- 写真撮影時GPSはPhotoMomentのアンカーとして写真と同じ期限まで保持する
- 追跡状態と到着演出状態を分離し、一つの状態値に混在させない
- 各セッションの`expires_at`（追跡操作の期限）と`delete_after`（保存データの削除期限）を分離し、追跡失効後も本人の読み取り・自己削除を各`delete_after`まで許可する。期間は大学のプライバシー承認（#9）待ち

## ER図

図中の`string id`は論理識別子の表記である。セッション、写真、経路等の識別子と参照先は既存の推測困難なプレフィックス付き不透明IDを維持し、PostgreSQLでは`text`型で揃える。UUIDv7や`uuid`型への変更は行わない。`LIVE_POSITION`だけはER図内の論理エンティティであり、PostgreSQLのテーブル・外部キーではない。

```mermaid
erDiagram
    MEMORY_SESSION ||--o| EXPERIENCE_LINK : optionally_links
    LIVE_PRESENCE_SESSION ||--o| EXPERIENCE_LINK : optionally_links
    LIVE_PRESENCE_SESSION ||--o| LIVE_POSITION : presents
    MEMORY_SESSION ||--o| ARRIVAL_EVENT : reaches
    MEMORY_SESSION ||--o{ PHOTO_MOMENT : captures
    MEMORY_SESSION ||--o{ ROUTE_SEGMENT : reconstructs
    MEMORY_SESSION ||--o| PUSH_SUBSCRIPTION : enables
    PHOTO_MOMENT ||--|| PHOTO_ASSET : owns
    PHOTO_MOMENT }o--o{ PHOTO_HOTSPOT_BUCKET : contributes_to
    CAMPUS_NODE ||--o{ CAMPUS_EDGE : connects

    MEMORY_SESSION {
        string id PK
        string consent_version
        string consent_language
        string token_hash
        string display_alias
        datetime consented_at
        datetime started_at
        datetime expires_at
        datetime delete_after
        datetime created_at
    }

    LIVE_PRESENCE_SESSION {
        string id PK
        string tracking_status
        string presence_phase
        string consent_version
        string on_site_consent_version
        string token_hash
        string display_alias
        datetime started_at
        datetime stopped_at
        datetime expires_at
        datetime delete_after
        datetime last_seen_at
    }

    EXPERIENCE_LINK {
        string id PK
        string memory_session_id FK
        string live_presence_session_id FK
        string link_token_hash
        datetime link_token_expires_at
        datetime expires_at
    }

    LIVE_POSITION {
        string live_presence_session_id PK
        float accuracy_m
        float local_x
        float local_z
        datetime recorded_at
        datetime received_at
        datetime expires_at
    }

    PUSH_SUBSCRIPTION {
        string memory_session_id PK
        string endpoint_encrypted
        string keys_encrypted
        integer interval_minutes
        string status
        datetime next_send_at
        datetime expires_at
    }

    ARRIVAL_EVENT {
        string id PK
        string memory_session_id FK
        string source
        string checkpoint_id
        string replay_status
        datetime arrived_at
        datetime replay_started_at
        datetime replay_finished_at
    }

    PHOTO_MOMENT {
        string id PK
        string memory_session_id FK
        string photo_asset_id FK
        integer sequence_no
        float latitude
        float longitude
        float accuracy_m
        string location_source
        string snapped_edge_id
        float local_x
        float local_z
        datetime captured_at
        datetime created_at
    }

    PHOTO_ASSET {
        string id PK
        string object_key
        string content_hash
        string mime_type
        integer width
        integer height
        integer size_bytes
        string status
        string visual_features
        datetime delete_after
        datetime created_at
    }

    ROUTE_SEGMENT {
        string id PK
        string memory_session_id FK
        string from_photo_id FK
        string to_anchor_type
        string to_anchor_id
        string graph_version
        string confidence
        string path_local_points
        float length_m
        datetime started_at
        datetime ended_at
    }

    PHOTO_HOTSPOT_BUCKET {
        string id PK
        string coordinate_version
        string three_d_map_version
        string spatial_bucket_id
        integer active_photo_count
        datetime window_started_at
        datetime expires_at
    }

    CAMPUS_NODE {
        string id PK
        string graph_version
        string node_type
        float local_x
        float local_z
    }

    CAMPUS_EDGE {
        string id PK
        string graph_version
        string from_node_id FK
        string to_node_id FK
        string path_points
        float length_m
        string attributes
        boolean enabled
    }

    DISPLAY_CLIENT {
        string id PK
        string name
        string renderer_type
        string status
        string role
        datetime lease_expires_at
        datetime last_seen_at
    }

    IDEMPOTENCY_RECORD {
        string scope PK
        string key_hash PK
        string request_hash
        integer response_status
        string deletion_result
        datetime expires_at
    }

    OPERATOR_AUDIT_LOG {
        integer id PK
        string operator_id
        string action
        string target_type
        string target_id_short
        string result
        datetime created_at
    }
```

`DISPLAY_CLIENT`は参加者データと関連付けない。描画機の接続監視だけに使用する。

`IDEMPOTENCY_RECORD`は到着、停止、削除、描画完了の重複適用を防ぐ。キーそのものではなくハッシュ、操作スコープ、リクエスト内容のハッシュ、返却結果、有効期限を最大24時間保存する。同じキーで異なる内容が来た場合は競合として拒否する。両セッション削除の途中結果は確定結果として固定せず、再送ごとに未完了側だけ進める。削除記録はID、資格情報、写真、位置やセッション内容を含めず、両側の`deleted`/`pending`と完了状態を安全に再現できる短期receiptとする。

`OPERATOR_AUDIT_LOG`は受付停止、再生、skip、fallbackなどの運営操作を記録する。生位置、参加トークン、完全な軌跡を含めず、対象IDも運営上必要な短縮表現にする。

## PostgreSQL実装規約

- セッション、写真、経路等のdomain entityの主キー・参照列は既存のプレフィックス付き不透明な`text` IDを保持する。UUIDv7への一括変更はせず、同じIDを使う外部キーの型も`text`で合わせる
- 外部へ公開しない追加の連番が必要な監査・集計テーブルだけ、`bigint generated always as identity`を使用できる
- すべての時刻列は`timestamp with time zone`としてUTCで保存する
- 状態値は`text`と`check`制約を基本とし、アプリだけでなくDBでも未知の状態を拒否する
- 外部キー列には参照・削除で使用するインデックスを作る。PostgreSQLは外部キー列のインデックスを自動作成しないため、migration reviewで漏れを検査する
- Goは上限付き接続プールを共有し、リクエストごとにDB接続を作らない
- migrationはGitで連番管理し、後方互換のあるexpand-contract方式で適用する。production起動時に破壊的migrationを自動実行しない
- 期限削除や写真処理を複数workerで実行する場合は、短いトランザクションと`FOR UPDATE SKIP LOCKED`でjobを取得することを原則とし、オブジェクトストレージのAPI呼び出し中にDB lockを保持しない。ただし、以下の開発者承認済みBackend #6の単一インスタンスMVP例外を適用する
- 日付partition、read replica、provider固有extensionはMVPで先行導入せず、実測した件数、query、VACUUM、復旧要件に基づいて判断する

### Backend #6の写真保存実装

対応PR適用後は、論理的なPhotoMomentとPhotoAssetを物理テーブル`photos`の1行で管理する。MemorySessionへの外部キーは`ON DELETE RESTRICT`とし、画像を消す前に親や削除先keyを失わない。`pending`/`ready`/`failed`/`deleting`状態、申告・検証対象メタデータ、任意の撮影時GPS、stage/final object key、受付順とアップロード発行期限を持つ。撮影時刻順の取得と受付順の一意性は区別する。

`memory_sessions.deleting`は写真の新規登録・URL発行を止める内部的な削除フェンスであり、本人認証の`delete_after`を延長しない。無効画像は消去してから`failed`とし、GPSとobject参照を落とす。削除対象の行はcloud消去成功後にだけ削除する。短期の冪等性記録には内容のハッシュと一方向の写真locatorだけを入れ、URL・画像・GPS・平文の写真IDを保持しない。

`photo_upload_fences`にはランダムなアップロード先keyと失効期限だけを保持し、MemorySessionへの外部キーは付けない。画像やGPSではなく、まだ有効なgeneration-zero PUTによる削除後の再作成を拒否するゼロバイトobjectの清掃記録である。写真完了・消去確定と同時にoutboxへ登録し、URL失効後にだけobjectと記録を消す。

### 開発者承認済みMVP例外：写真の外部I/Oと直列化

Backend #6では、写真完了・署名URL発行・消去・アップロードフェンス清掃に限り、PostgreSQL advisory lockを保持したまま必要なCloud Storage/IAM APIを呼び出す方式を開発者が選択・承認した。二重のDB/cloud更新と、消去中にフェンスが再作成されて追跡対象から外れる競合を、単純な直列化で防ぐためである。

ロック順はMemorySession→object fenceとし、フェンス清掃はobject fenceだけを取得する。取得後に状態・期限を再読込し、外部消去とoutbox完了を同じ直列化区間へ含める。外部I/Oは既存のAPI 10秒・定期清掃10秒・起動/清掃コマンド30秒のcontext deadlineで制限し、失敗時は永続化済み`deleting`/outboxから再試行する。

代償として外部I/O待ちがDB接続を占有する。最大1 APIインスタンス、既存の接続プール上限、受付停止の移行窓を維持し、実stagingで写真完了と削除の同時負荷・lock待ち・接続上限・期限超過と復旧を検証してから有効化する。複数APIへの拡張や負荷不成立時はこの例外を拡大せず、generation条件付き更新と処理claim/leaseによるロック外I/O方式を別途承認して移行する。


## MemorySession

写真、写真撮影時GPS、経路、到着演出、本人用ストーリーを管理する。LivePresenceSessionとは別の同意版、トークン、`expires_at`、`delete_after`を持つ。`expires_at`後は新規の追跡操作を受け付けず、本人の状態・写真の閲覧と自己削除は`delete_after`まで許可する。

## LivePresenceSession

通常GPSのベストエフォート表示だけを管理する。`expires_at`後は新規の位置送信を拒否して最新表示位置を破棄し、自己削除はLive自身の`delete_after`まで許可する。状態の読み取りは紐づくMemorySessionの本人用GETで行い、Memoryが先に削除された後はそのGETも提供しない。

- `tracking_status`: `tracking`、`stopped`、`expired`
- `presence_phase`: `approaching`、`arrived`、`on_site`、`ended`
- `on_site_consent_version`: 到着後に用途・範囲が変わる場合だけ必須
- `display_alias`: MemorySessionと共有する場合もExperienceLink経由で短期連携する

## ExperienceLink

同じ端末で両展示へ参加する場合だけ作る関連付け。色・記号と到着処理を連携し、Memory用Bearer tokenで本人向けGETがLiveの状態と期限を取得するための関係を保つが、写真、生GPS、経路、Live用トークンは返さない。連携用トークンの有効期限は`link_token_expires_at`で管理して到着・失効後に無効化し、関連付けは`expires_at`（両セッションのうち早い`delete_after`以下）まで残す。片方の削除時にも関連付けを削除する。追跡用`expires_at`到達だけでは消さない。

## LivePosition

LivePresenceSessionごとに最大1件の最新表示位置を持つ論理エンティティ。受信した生緯度経度はgeofence・異常値検証と座標変換後に破棄する。単一Go Backendインスタンスのプロセスメモリで30〜60秒TTLを設定し、プロセス再起動時は消失してよい。参加端末が次の最新位置を送信した時点で表示を再開する。複数Backend化するときだけRedis互換TTLストアへ移し、PostgreSQLに`live_positions`テーブルや位置履歴を作らない。既存の同名テーブルがある環境ではBackend側の002 migrationで除去する必要がある（本書はmigration適用済みとは主張しない）。

| 項目 | 型 | 内容 |
|---|---|---|
| `accuracy_m` | float | ブラウザが返す推定誤差 |
| `local_x` / `local_z` | float | 量子化・ぼかし済みの表示座標 |
| `recorded_at` | datetime | 端末で取得した時刻 |
| `received_at` | datetime | Backendが受信した時刻 |
| `expires_at` | datetime | 30〜60秒後の失効時刻 |

## ArrivalEvent

到着はセッションにつき最大1件とする。二重送信は同じ結果を返す。

- `source`: `qr`、`nfc`、`operator`
- `replay_status`: `queued`、`playing`、`completed`、`skipped`
- `checkpoint_id`: 到着場所の識別子。署名付き証明そのものは保存しない
- `skip_reason`: `participant_request`、`timeout`、`renderer_failure`、`operator`

到着の受付ではMemorySessionへArrivalEventを作成し、ExperienceLinkが有効な場合だけ対応するLivePresenceSessionの`presence_phase`を`arrived`へ更新する。LivePresenceSessionのGPS送信は自動停止せず、参加者が停止するか、追加同意して`on_site`へ進む。再生状態はArrivalEventだけを正とし、MemorySessionへ`replaying`を重複保存しない。

## PhotoMoment・PhotoAsset

`PhotoMoment`は参加者が残した撮影イベント、`PhotoAsset`は非公開オブジェクトストレージ上の加工済み画像を表す。写真EXIFは保存せず、撮影時位置はブラウザの位置APIから別項目として受け取る。

| 項目 | 内容 |
|---|---|
| `sequence_no` | セッション内の撮影順。unique |
| `location_source` | `captured_gps`、`interpolated`、`unknown` |
| `snapped_edge_id` | map matchingで採用したCampusEdge。未確定ならnull |
| `object_key` | 推測困難な非公開オブジェクトキー。URLではない |
| `content_hash` | アップロード重複・破損検出用。公開しない |
| `status` | `pending`、`ready`、`failed`、`deleting` |
| `visual_features` | 公開表現に使う復元性の低い主要色等。画像内容の人物・物体分類は行わない |
| `delete_after` | セッション削除期限以下。独立して延長しない |

写真は1セッション12枚までを暫定上限とする。原画像をそのまま保存せず、向き補正、EXIF除去、長辺1920px以下への縮小、再エンコード後の画像だけを保存する。写真単位削除では、PhotoMoment、PhotoAsset、派生特徴を削除し、前後のRouteSegmentを再生成または未確定にする。

## RouteSegment

前の写真から次の写真、または最後の写真から到着点までの再構成経路。

- `from_photo_id`: 区間開始のPhotoMoment
- `to_anchor_type`: `photo`または`arrival`
- `to_anchor_id`: 次のPhotoMomentまたはArrivalEvent
- `graph_version`: 使用したCampusGraphの版
- `confidence`: `high`、`medium`、`low`
- `path_local_points`: 表示用のローカル座標列。生緯度経度を含めない

生成時に参照する位置は区間両端の写真撮影時GPSだけとし、通常GPS点列を入力・保存しない。同じ入力と`graph_version`から再生成できるよう、経路生成アルゴリズムの版も設定またはメタデータへ記録する。

## PhotoHotspotBucket

写真撮影地点を5〜10m単位で量子化し、一定時間窓で集計した公開用の非個人データ。個別写真ピンは永続テーブルを持たず、PhotoMomentの準備完了イベントから短時間だけ生成する。

- 個別ピンのペイロードには写真、画像URL、生GPS、セッションID、正確な時刻を含めない
- 個別ピンは15〜30分以内に失効し、少人数時は遅延、追加量子化、短時間表示、非表示のいずれかを適用する
- `active_photo_count`が3件以上のbucketだけをフォトスポットとして公開する
- 写真単位またはMemorySession全体の削除時は寄与数を減算し、3件未満になれば直ちに公開を止める
- `coordinate_version`、`three_d_map_version`が公開中の組み合わせと一致しないイベントは描画しない

## PushSubscription

MemorySessionに紐づく任意参加の撮影リマインダー。ブラウザの購読エンドポイントと鍵は暗号化し、通知本文に場所、写真、参加記号、経路を含めない。到着、停止、購読解除、期限切れの最も早い時点で無効化・削除する。通知時刻は厳密な保証をせず、端末内タイマーを補助表示として使える。

## CampusNode・CampusEdge

キャンパス内の歩行可能経路を表すバージョン付きグラフ。CampusEdgeは距離、屋内外、階段、スロープ、エレベーター、利用時間、通行止め、バリアフリー可否を属性として持つ。公開中のグラフを直接上書きせず、新しい版を検証してから切り替える。

## インデックス

- `memory_sessions(delete_after)`
- `live_presence_sessions(tracking_status, presence_phase, expires_at)`
- `live_presence_sessions(delete_after)`
- `experience_links(memory_session_id)` unique
- `experience_links(live_presence_session_id)` unique
- `experience_links(expires_at)`
- `arrival_events(replay_status, arrived_at)`
- `photo_moments(memory_session_id, sequence_no)` unique
- `photo_moments(photo_asset_id)` unique
- `photo_assets(delete_after, status)`
- `route_segments(memory_session_id, started_at)`
- `photo_hotspot_buckets(coordinate_version, three_d_map_version, spatial_bucket_id, window_started_at)` unique
- `photo_hotspot_buckets(expires_at)`
- `push_subscriptions(status, next_send_at)`
- `campus_nodes(graph_version)`
- `campus_edges(graph_version, enabled)`
- `display_clients(last_seen_at)`
- `idempotency_records(scope, key_hash)` unique
- `idempotency_records(expires_at)`
- `operator_audit_logs(created_at)`

実装時は、期限・queue走査のqueryに合わせて次の部分インデックスを優先する。

- `photo_assets(delete_after) where status in ('pending', 'deleting')`
- `arrival_events(arrived_at) where replay_status = 'queued'`
- `push_subscriptions(next_send_at) where status = 'active'`
- `memory_sessions(delete_after) where delete_after is not null`
- `live_presence_sessions(expires_at) where tracking_status = 'tracking'`

同じ列へfull indexとpartial indexを重複して作らず、`EXPLAIN`と実測で採用する索引を決める。

## 削除

1. LivePresenceSessionの`expires_at`到来時は`expired`にし、LivePositionを削除し、位置送信を拒否する。状態取得・自己削除用トークンとExperienceLinkの関連付けは`delete_after`まで維持する
2. 各Sessionの`delete_after`到来時は、該当Sessionの写真・経路・到着・PushSubscription等の関連データ、トークン、対応するExperienceLinkを削除対象にする
3. 写真が寄与したPhotoHotspotBucketを減算し、公開条件を下回ったbucketを非公開にする
4. 非公開オブジェクトストレージ上の写真を削除し、成功を確認する
5. DBレコードをトランザクション内で削除する。オブジェクト削除失敗時は`deleting`として再試行し、期限超過をアラートする
6. 参加者画面は「写真の思い出を削除」「リアルタイム参加を削除」「両方を削除」を分け、両方の場合も各削除を独立に完了・再試行できるようにする
7. 個人を復元できない集計値だけを残す場合は、大学への説明と同意文へ明記する

インフラのセキュリティログとバックアップはDB削除と同時に個別レコードを消せない場合があるため、原則として生位置を含めず、別途定めた短い保持期限で世代ごと削除する。
