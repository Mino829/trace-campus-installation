# 座標系・写真地点の3D座標対応

## 目的

参加者が3Dマップ上で選んだ写真地点（チェックポイント）を、キャンパス3D空間内の安定したローカル座標へ対応付け、展示表現へ渡すこと。

スマートフォンから得る位置データの変換・検証・平滑化・量子化は廃止した。写真地点は参加者が3Dマップで選んだ `checkpointId` のみを入力とし、座標・対応GraphノードIDの解決はBackendが行う。参加者端末のpayloadに座標・ノードID・グラフ版を追加しない（2026-10-26決定、Issue #23）。

処理の流れは次のとおり。

```text
参加者が3Dマップで写真地点（チェックポイント）を選ぶ
  -> 端末内に画像を保存し、写真ID・checkpointId・撮影時刻だけをBackendへ送信
  -> Backendが checkpointId を座標と対応Graphノードへ解決
  -> CampusGraph最短経路で写真地点を撮影順につなぐ
  -> 末尾にShibaLab（到着地）までの最短経路を追加
  -> グラフ版・点ID列・写真ごとの点IDと撮影時刻をWSS配信
  -> three.js描画クライアントが3D座標へ対応付けて再生
```

## 座標系契約と確認できた成果物

以下は設計上の契約であり、Frontendの現行ファイルがそのまま公開用の互換版として承認されたことを意味しない。

- 目標：CampusGraphと3D Mapの対応を同一のメートル単位で管理し、原点・軸・版を固定する。公開版は上書きせず、互換組み合わせを定めて切り替え・ロールバックできるようにする。旧緯度経度→ローカル座標変換、geofence、量子化、セッション固定オフセット、平滑化は参加者の新しい写真地点処理から廃止した（2026-10-26、Issue #23）。
- 実体：Frontendリポジトリの [`models/キャンパス全体.blend`](https://github.com/Mino829/TRACE_F/blob/main/models/%E3%82%AD%E3%83%A3%E3%83%B3%E3%83%91%E3%82%B9%E5%85%A8%E4%BD%93.blend)、[`models/trace/campus_graph.json`](https://github.com/Mino829/TRACE_F/blob/main/models/trace/campus_graph.json)、展示用 [`public/display/campus.glb`](https://github.com/Mino829/TRACE_F/blob/main/public/display/campus.glb) がある。Graph JSONの確認時点の値は `campusGraphVersion=toyosu-graph-2026-10-26.1.draft`、`coordinateVersion=toyosu-2026-08-28.1`、`threeDMapVersion=キャンパス全体.blend 2026-10-26`（464 node、502 edge、`selectable=true` は82点）。これはファイルに記載された版であり、本番公開版の承認ではない。
- 実体：Graphの `threeD` はBlender座標（1単位=1m、Z上、地面=0）。[`models/scripts/export_web.py`](https://github.com/Mino829/TRACE_F/blob/main/models/scripts/export_web.py) は4棟の中心付近の地面を原点としてY上のDraco圧縮GLBを書き出す。描画側 [`src/lib/display/campus-graph.ts`](https://github.com/Mino829/TRACE_F/blob/main/src/lib/display/campus-graph.ts) は `origin=(-160.48048400878906,-14.865501403808594)` を使い、Blender `(x,y,z)` をthree.js `(x-ox,z,-(y-oy))` に変換する。Graphの別フィールド `local` は旧アプリ座標（教室棟1F南面の自動ドア原点のENU）であり、展示GLBの原点と同一ではない。新しい参加者payloadにこの座標を含めない。
- 未確認：実際の展示用GLBとGraphの版付き配布・公開互換組み合わせ、原点/軸/単位の成果物横断検査と現地基準点による精度、公開版への昇格・直前互換版への復帰手順は未承認。モデルの寸法は写真・案内図・航空写真・PLATEAU等からの推定で実測値ではない（[`models/readme.md`](https://github.com/Mino829/TRACE_F/blob/main/models/readme.md)）。実装Graphには階ごとの高さがある一方、旧MVP方針は高さ固定であり、階層表示と経路で使う高さ・通行条件の採否はGraph担当と照合する。

3Dモデル更新時にはGraphとの対応を確認し、互換版の組み合わせを変更する（設計目標）。単にJSONの版文字列が存在するだけで、切替やロールバックを検証済みとは扱わない。

## チェックポイント点とGraphノードの対応

- 設計目標：参加者が選択する `checkpointId` をBackendが公開中のCampusGraphの対応ノードと座標へ解決し、非選択の経路用ノードと区別する。対応版の保存・版不一致時の扱いは [TRACE_B#25](https://github.com/Mino829/TRACE_B/issues/25)・[TRACE_B#7](https://github.com/Mino829/TRACE_B/issues/7) と揃える
- 現行のGraph JSONには `selectable=true` が82点あり、Frontendの [`src/lib/places/places.json`](https://github.com/Mino829/TRACE_F/blob/main/src/lib/places/places.json) も `toyosu-graph-2026-10-26.1.draft` を記載する。選択地点のUIは [TRACE_F#38](https://github.com/Mino829/TRACE_F/issues/38)・[TRACE_F#45](https://github.com/Mino829/TRACE_F/pull/45) の実装を参照する。8か所のスポット案は82点の代替一覧ではなく、上に重ねて見せる任意の印という提案であり、表示方法は未決
- 写真地点の選び直し、展示期間中に版を固定するか・どの一覧をBackendと `/log` と `/display` に配布するかは未決。[PR #25のレビュー](https://github.com/Mino829/trace-campus-installation/pull/25#issuecomment-6410151662) で指摘された版ずれ時の `INVALID_CHECKPOINT` リスクを、公開版契約の決定前に解消した扱いにしない

## 経路の解決

- 写真を撮った順に、各写真の地点（チェックポイントの対応ノード）をCampusGraph最短経路でつなぐ
- 最後の写真からShibaLab（到着地）までの区間も最短経路でつなぎ、同じ線として描く
- 通行止め設定は描画だけでなく経路探索からも除外する
- 経路は距離・通行制約のみで判定する

## CampusGraph / 3D Mapの互換版同期（旧FR-035）

設計目標として、描画イベントとスナップショットには `campusGraphVersion` と `threeDMapVersion` を付け、Backendと描画PCは公開設定の互換組み合わせを照合し、版不一致を破棄する。切替順序（設定の公開、描画PCの事前読込、ヘルス確認、Backendの配信切替）と直前互換版への復帰は未検証の運用要件である。現行Graph JSONの `coordinateVersion`、`campusGraphVersion`、`threeDMapVersion` の値と、Frontend描画コード内の照合は実装の証跡だが、Backend配信と公開設定との統合確認ではない。[TRACE_B#9](https://github.com/Mino829/TRACE_B/issues/9)・[TRACE_B#7](https://github.com/Mino829/TRACE_B/issues/7) と[TRACE_F#36](https://github.com/Mino829/TRACE_F/issues/36) で整合させる。

### 制作物と責任分界

| 制作物 | 確認できた実体・設計目標 | 担当・未確認 |
|---|---|---|
| CampusGraph | Frontend `models/trace/campus_graph.json` に版・node・edgeと `lengthMetres` / `passable` がある。公開契約では階段・屋内外・通行止めの扱いも揃える | Graphの制作は[TRACE_F#8](https://github.com/Mino829/TRACE_F/issues/8)（code-yuto）。Backendへの読み込み、通行制約・版整合は[TRACE_B#7](https://github.com/Mino829/TRACE_B/issues/7)。現地通行可否は未確認 |
| 3D Map/座標系 | Blender原本とGLB成果物、Frontendの `toScene` 変換がある。原点・軸・単位は上記の実体と区別して公開契約に固定する | 3D/展示画面の担当はcode-yuto（[PR #24](https://github.com/Mino829/trace-campus-installation/pull/24)）。表現責任者による承認・実測は未確認 |
| 通行止め設定 | edge ID、開始・終了、理由、承認者を含む日時付きJSONは設計目標 | 運営側の公開・承認とBackend反映は未確認 |

### 描画へ渡すデータと実装境界

描画へ渡すデータの決定は、経路のグラフ点ID列・グラフ版・写真ごとの点IDと撮影時刻のみ（原写真・画像URL・生座標・生GPSは渡さない、Issue #23）。3D座標への対応付けは `/display` 側のGraph/3D Map設定から行う。three.js採用、Frontend `/display` と担当code-yuto は[PR #24](https://github.com/Mino829/trace-campus-installation/pull/24)の決定記録・担当者コメントで確認済み。ただしFrontendの現行 [`src/lib/display/bridge.ts`](https://github.com/Mino829/TRACE_F/blob/main/src/lib/display/bridge.ts) は旧 `displayAlias`・`photoMomentId` を含む旧イベント形とローカル `ws://127.0.0.1:8765/ws` を使用しており、Issue #23決定後の配信契約と統合済みではない。`/display` のthree.js再生コードの存在は、本番到着イベントの接続試験や新契約への移行完了を意味しない。

WSSの短期描画資格・スナップショット/差分・リース・版照合・停止/TTL/削除反映は[TRACE_B#9](https://github.com/Mino829/TRACE_B/issues/9)、経路と地点の削除再計算は[TRACE_B#7](https://github.com/Mino829/TRACE_B/issues/7)・[TRACE_B#25](https://github.com/Mino829/TRACE_B/issues/25)、展示画面からの再生接続は[TRACE_F#36](https://github.com/Mino829/TRACE_F/issues/36) の実装・契約対象。[TRACE_F PR #46](https://github.com/Mino829/TRACE_F/pull/46) は展示PCの `arrival-issuer` がブラウザのWebSocketをBackendへ中継し、資格情報をブラウザに置かない構成を提示している。ブラウザからGoへの直接接続と同一視せず、資格発行経路・運営API入口は[設計Issue #11](https://github.com/Mino829/trace-campus-installation/issues/11)で未決として扱う。

イベントの具体的なスキーマは未統一。[architecture.md](architecture.md) の例（`schemaVersion: 2` / `pointIds`）と [api.md](api.md) の例（`schemaVersion: 1` / `pathNodeIds` / `photoCheckpoints`、旧個別IDを含む）は食い違っており、どちらかを実装済みの新契約として採用しない。写真ピン・経路再生・通常展示・合成フォールバックおよび音響/照明の表現範囲、技術・表現責任者の承認もIssue #12の未完了項目である。

## 表示上のプライバシー

- 生の緯度経度は保存・演算しないため、ログ、WebSocket、スクリーンショットへ出さない設計は維持する
- 描画へ渡すデータは点IDと撮影時刻のみであり、写真・画像URL・参加者の座標を含めない
- セッションや個人を推測できる表示はしない（#23で廃止した色・記号・照合表示方針は採用しない）
- 同時参加者が少なく個人の行動を推測されやすい場合の保護方針が必要になった場合は、別途決定する

## 実地試験項目

- 主な通路・建物沿い・屋内・展示場入口で写真地点を選択する操作（ユーザビリティ）
- 学内Wi-Fiとモバイル回線
- 同じ地点選択から生成した経路の再現性
- 全体10〜20秒の比例配分で再生できること
- 複数経路の距離差、写真地点が少ない・多すぎる場合の挙動
- 通行止め・立入禁止・階段を経路へ含めないこと
- CampusGraph・3D Mapの版不一致の破棄、切替、直前版へのロールバック
- 写真保存・送信・再送の通信不安定時の動作

具体的な合格値は[試験計画・公開判定基準](test-plan.md)で管理する。
