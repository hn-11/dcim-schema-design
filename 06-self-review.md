# 06. セルフレビュー — SQL アンチパターン & データセンター一般化

確定案 **Semantic-Typed DCIM**（[03-finalists.md](./03-finalists.md) / ER 図 [05-er-diagram.md](./05-er-diagram.md)）を
**SQL アンチパターン**（主に Bill Karwin『SQL Antipatterns』の分類）と
**データセンターへの一般化（多数 DC へ展開するパッケージとしての汎用性）** の観点で批判的に点検する。

本設計の通奏低音は「**ゆるい IoT スキーマにせず、ドメイン知識を DB 制約（FK / CHECK / UNIQUE / 排他アーク）で担保する**」こと。
このレビューは、その宣言目標を **どこで満たし / どこで（移植性 LCD のために）サービス層へ委ねているか** を明示することを主眼とする。
横断制約は **LCD（最小公倍数SQL・[09章](./09-portability.md)）** と **マルチテナント／コロケーション（[08章](./08-tenancy-colocation.md)）**。

重大度: 🔴 要修正（設計の前提を崩す） / 🟡 要改善（運用で効く） / 🟢 設計判断（許容だが明示すべき）

> **一段落サマリ**: [03章](./03-finalists.md)確定案は、旧案で最重要だった多態スコープキー（`capacity_budget`/`threshold`）を排他アーク FK で解決し、3軸 `point_type` を廃止してフラット `metric` カタログに一本化し、タグレイヤを全廃したことで EAV リスクを排除した。`equip_kind` は単一親ツリーに確定し、汎用 `def` エンジン・多重継承 DAG は持ち込まれていない。残る要修正は空間タイプ語彙のハードコードのみ（🔴 P1）。新規リスクとして `v_equip_flow` マテビュー鮮度と冗長グループの意図—実態乖離がサービス層で担保すべき課題として浮上する（🟡 P2）。VA/W 誤称・float 課金・CHECK 列挙拡張・三相力率は従来どおり改善対象（🟡）。U 重なり・ツリー階層・hypertable FK は引き続き健全（🟢）。

---

## 1. ✅ 解決済 — 多態スコープキー → 排他アーク FK（旧最重要弱点）

旧案で **最大の弱点** だった `(scope_type, scope_id)` 多態キー（`scope_id` に実 FK を張れず孤児・誤参照を DB が防げない）は、
[03章 L8/L9](./03-finalists.md) で **排他アーク FK** に置き換え済み。

- `capacity_budget`: `location_id / rack_id / breaker_id` をそれぞれ nullable 実 FK とし
  `CHECK ( num_nonnull(location_id, rack_id, breaker_id) = 1 )` で「ちょうど1つ非NULL」を強制。
- `threshold`: `metric_id / equipment_id / location_id` を排他アーク化（同上）。

**残差（🟡・後述 item 10 参照）**: `CHECK` は行内整合のみ。同一スコープ対象に対する重複行（例: `location_id=5` の同一 `dimension` が 2 行）は防げない。アーク別部分インデックスで補完:

```sql
CREATE UNIQUE INDEX ON capacity_budget (location_id, dimension) WHERE location_id IS NOT NULL;
CREATE UNIQUE INDEX ON capacity_budget (rack_id,     dimension) WHERE rack_id     IS NOT NULL;
CREATE UNIQUE INDEX ON capacity_budget (breaker_id,  dimension) WHERE breaker_id  IS NOT NULL;
```

---

## 2. ✅ 解決済/消滅 — 3軸 point_type → フラット metric カタログ

旧設計の `point_type(quantity×phenomenon×func×duct)` と `quantity_unit` 複合 FK は全廃。
代わりに [03章 L4](./03-finalists.md) の **フラット `metric` カタログ**が単位を 1 行に持つ。

- `metric` 行が `unit` を 1 つ保持 → **「`active_power` に `°C`」は物理的に起こり得ない**
  （読み値は単位を持たず、参照する `metric` 行が単位を確定する）。
- `data_point UNIQUE(equipment_id, metric_id, role, phase)` で「1 機器に同義の点は 1 つ」を保証。
  全列 NOT NULL（`phase` は `'none'` センチネル）なので UNIQUE が確実に効く。
- 旧 `point_type.code` 無し問題・NULL 混じり UNIQUE・quantity FD 問題は **すべて解消**
  （`metric` は `code UK` を持つ・[03章 L4](./03-finalists.md)）。

量と単位の整合は **`metric` カタログが構造的に保証する**。残課題なし。

---

## 3. ✅ 解決済/消滅 — タグレイヤ全廃（旧 EAV 残差）

`tag_def` / `entity_tag` レイヤは確定案から **スコープ外として削除済み**（[03章](./03-finalists.md) に登場しない）。

- **EAV 面が存在しない**: 任意キー・型なし値という EAV の危険面が、そもそも不在。
- 旧 `entity_tag.value_type` vs. `value_*` 列の整合問題（value_type='number' のタグに value_str を挿入しても
  CHECK が通る、という行間条件）は **問題ごと消滅**。
- 長尾属性（カスタム属性）は現版では**持たない**。将来追加するなら `(entity_type, entity_id)` の多態参照ではなく、
  各エンティティの**リレーショナル派生スーパータイプ**（型別テーブル）で表現する方針（EAV 化しない）。

---

## 4. ✅ 解決済（オプション DAG 有効化時のみ caveat） — equip_kind ツリー

`equip_kind` は **単一親ツリー**（`parent_id` 自己 FK）に確定（[03章 L1](./03-finalists.md)）。
汎用 `def` エンジン・多重継承 DAG は持ち込まない。

- 「全 HVAC / 全 UPS」集約は `equip_kind.parent_id` ツリーの再帰 CTE か、
  （読み多なら）木の閉包への単純 JOIN で足りる。
- 木の閉包は DAG 閉包と異なり: `path_count` 不要・辺削除 = 単純 DELETE・
  サイクルは `parent_id` の構造上起き得ない。

**オプションの equip_kind DAG を将来有効化する場合のみ以下が必須**（現状は不要）:

- `path_count`（経路数）列 — DAG では 1 子孫が複数経路で同一祖先に到達しうる。
  安易な DELETE は他経路で生き残るべき行まで消す。
- 辺追加は O(祖先数 × 子孫数) の upsert + `path_count` 増分。
  辺削除は減算・0 になった行だけ物理削除。
- サイクル禁止: 辺 `u→v` 追加前に `(v, u)` が閉包に存在すれば拒否。
- 増分・減算・サイクル検査はサービス層に集約（再帰トリガの方言差を避ける・[09章](./09-portability.md)）。
  ダイヤモンド継承の単体テストを伴う。

実需が出るまで DAG の複雑性を持ち込まない判断は健全。

---

## 5. 🟡 v_equip_flow — マテビュー鮮度・更新タイミング（NEW）

[03章 L7](./03-finalists.md) の connectivity（電力フロー / A系B系 / dual-cord）は、
物理 path（`power_panel→breaker→power_feed→port→cable`）を真実源とし、
**導出ビュー `v_equip_flow`** として実装する（並行グラフを手で維持しない）。

**リスク**:
- 通常の `VIEW` は毎回再帰 CTE を実行するため、大規模 DC では問い合わせコストが高い。
- **`MATERIALIZED VIEW`** にすると `REFRESH MATERIALIZED VIEW` が必要（デフォルトで自動更新されない）。
- REFRESH のタイミングを誤ると **古い経路でのキャパシティ/冗長検証が通る** という誤判断が生じる。

**対応**:
- 変更系 DML（`power_feed` / `breaker` / ケーブル行への `INSERT/UPDATE/DELETE`）後に
  **明示的 REFRESH** を呼ぶサービス層フローを定義。小規模であれば非マテ `VIEW` のまま
  使い、クエリコストを計測してから判断する。
- REFRESH が遅い場合は `CONCURRENTLY` オプションで参照を止めずに更新（一意インデックス必要）。
- `v_equip_flow` の「最終 REFRESH 時刻」を保持し、UX 上でフレッシュネスを明示することを推奨（[09章](./09-portability.md)）。

---

## 6. 🟡 redundancy_group — 意図（intent）と物理実態（reality）の乖離（NEW）

[03章 L7](./03-finalists.md) の `redundancy_group`（`topology`, `n_required`, `leg`/`role`）は
**冗長の意図を第一級で持つ**。しかし FK だけでは物理実態を保証できない。

- `topology='2N'` と宣言しても、`redundancy_member.leg` が実際に独立した電力系統（A フィード / B フィード）を
  辿るかは `v_equip_flow` との突合が必要（SPOF 検証・[04章 UC-5](./04-validation-queries.md)）。
- `topology='N+1'` と宣言しても、1 台落ちたときに `capacity_budget` が充足するかは
  `equipment_demand` との集計が必要。
- `n_required` の整合（メンバが N 台に満たない）も FK では防げない。

**対応**:
- SPOF 検証（2N: leg=A/B が独立 root か）と容量充足検証（N+1: Σ `estimated` ≤ `redundancy_reserve` 除きの `rated`）は
  **サービス層 / 監視ビュー** に置く（行間集約のため DB `CHECK` 不可・[09章 原則3](./09-portability.md)）。
- 検証クエリは [04章](./04-validation-queries.md) に定義し、デプロイ後の定期バリデーションとして組み込む。

---

## 7. 🔴 空間タイプ語彙のハードコード（汎用性の最重要課題）

`loc_type IN ('region','campus','building','floor','room','dataCenter','cage','zone','row')`
（[03章 L2](./03-finalists.md)）は語彙を固定している。実 DC は **"data hall" / "suite" / "pod" /
"white space" / "MMR" / "電気室"** など呼称・段数が多様。

**改善**: `loc_type` を **ルックアップテーブル化**（`location_type(code, label, rank)`）し、
親子規則（`loc_type_rule` 相当）と整合させる。これで「建屋なし DC」「ケージ階層が深いコロ」
「ゾーン主体のハイパースケール」をデータで吸収できる（後述 item 8 の 31 Flavors 修正と同根）。

---

## 8. 🟡 31 Flavors（CHECK-IN 列挙のハードコード）

`equipment.status` / `conn.protocol` / `value_kind` / `load_strategy` 等多数が `CHECK (col IN ('a','b',...))` で固定。
**パッケージを多数 DB に配布する場合、列挙値の追加に全 DB への `ALTER TABLE` が必要**になり汎用性を阻害（item 7 と同根）。

**判断軸**:
- **不変の語彙**（`data_point.role`=sensor/sp/cmd/derived、`phase`=L1/L2/L3/none、
  `equip_kind.power_class`=it/cooling/facility/other、`redundancy_group.topology`=N/N+1/2N 等）は
  CHECK enum で良い（固定・閉じた集合・[03章 L1](./03-finalists.md)）。
- **増えうる語彙**（`equipment.status` ライフサイクル、`loc_type`、`conn.protocol`）は
  **ルックアップテーブル化**してデータ駆動で拡張可能にする（[09章 9.2](./09-portability.md)）。

固定集合の CHECK は残し、増える集合だけテーブル化するのがバランスの取れた対処。

---

## 9. 🟡 VA/W 誤称（`available_power_w` は実は VA）

`power_feed.available_power_w = round(V × A × util/100 × √3)::bigint`（[03章 L7](./03-finalists.md)）は
**皮相電力(VA)** であり有効電力(W)ではない（力率未考慮）。W と VA の混同はドメイン上の誤り。

**改善**:
- **`available_power_va` に改名**し、必要なら `power_factor` 列を足して
  `available_power_w = available_power_va × power_factor` を別途用意。
- 三相係数 `1.732` は **`sqrt(3.0)`** を使い、マジックナンバーの丸め誤差を避ける。

**overflow 確認（🟢）**: 生成列の型は `bigint`。`voltage_v numeric × amperage_a numeric` の積は
feed 単位で最大 ~10^7 オーダ → `bigint`（~9.2×10^18）に対し溢れる余地は無い。現設計の型幅は妥当。確認済み。

---

## 10. 🟡 課金・kWh は float でなく numeric

[04章](./04-validation-queries.md) の電気代計算は `measurement.value double precision` から kWh・通貨額を算出する。
テレメトリ実測値の `double` は妥当だが、**通貨・課金額は `numeric` を使う**のが鉄則
（浮動小数の丸め誤差は課金で許されない）。派生する金額・kWh は `numeric` にキャストして計算・保存すること。
コロ契約電力（`equipment_demand.contracted`・[08章](./08-tenancy-colocation.md)）の課金も同様。

---

## 11. 🟡 三相電力・力率モデルの浅さ

`power_feed` は単一の `voltage_v`/`amperage_a` のみ（[03章 L7](./03-finalists.md)）。
実 3 相は L1/L2/L3 の相別電流を持ち、相不平衡が容量計算に効く。`data_point.phase` は点側にあるが、
**feed 側の相別メータリング**が無い。キャパシティ算定（`available_power_va`）も相不平衡・力率を無視（item 9 と連動）。

**改善**: `power_feed_phase(feed_id, leg, voltage_v, amperage_a)` を追加、`power_factor` を導入。

---

## 12. 🟡 タイムゾーン非対応

`location` に時間帯が無い。多リージョン DC で **ローカル日次の電気代/PUE 集計**や
CAGG の `time_bucket` はタイムゾーンを要する。サイト相当の `location` に **`time_zone text`** を持たせ、
課金・日次ロールアップ（[10章](./10-room-group-derived-metrics.md)）で使用する
（`time_bucket('1 day', ts, 'Asia/Tokyo')`）。

---

## 13. 🟡 運用パラメータが DDL にベタ書き

`create_hypertable(... by_range('time', INTERVAL '1 day'))`、`compress_segmentby`、retention、
`conn.poll_time_ms` の既定などが固定気味。DC 規模で最適値が大きく変わる。
パッケージとしては **デプロイ別の設定（config テーブル or デプロイ変数）から流し込み**、
DDL に焼き込まない方が良い（[09章](./09-portability.md) の時系列アダプタ境界に置く）。

---

## 14. 🟡 カスタムメトリック拡張のガバナンス

`metric.code` は UNIQUE であり、複数 DC・複数顧客で **`metric.code` の名前空間衝突**が起きうる。
パッケージ同梱（builtin）行と顧客追加（custom）行の**ガバナンス／区分**を明文化すべき
（`local:` プレフィクスや顧客プレフィクスの命名規約）。タグレイヤが存在しないため EAV 的な拡張面はなく、
長尾属性のリスクは item 3 で消滅しているが、メトリックカタログの運用管理（誰が追加できるか・命名衝突）は別途必要。

---

## 15. 🟢 Keyless Entry（hypertable に FK 無し）— 設計判断

`measurement` hypertable に FK を張らないのは正しい（チャンク DROP/圧縮の阻害回避・[09章 原則1](./09-portability.md)）。
`series_id` だけが TSDB へ越境する。

**副作用**: `equipment` 削除で `data_point`/`series` は CASCADE 削除されるが、`measurement` 行は FK が無いので残る（孤児時系列）。
retention で自然消滅するが、即時整合が要るなら **series 退役（`series.retired_at`）時に `drop_chunks` を
組み合わせたバックグラウンド物理削除**ジョブを運用に組み込む、と明記すべき。

---

## 16. 🟢 U 物理重なり禁止（confirm: 良）

ラックの U 重なり禁止は **占有 U 行 ＋ 複合 UNIQUE**
（`rack_unit_occupancy(rack_id, rack_side, face, u)`・[03章 L2](./03-finalists.md) / [09章 9.2](./09-portability.md)）で実装。
full-depth 機器は front/rear 両面に占有行を入れて片面機器と確実に衝突させる。
PostgreSQL 固有の `EXCLUDE`/`btree_gist`/range 型を使わず **全エンジンで同形・原子的・拡張ゼロ**。
設計目標と移植性を両立。良。

---

## 17. 🟢 マルチテナント / コロケーション（[08章](./08-tenancy-colocation.md) で対応済）

`tenant` 表 ＋ 全エンティティ `tenant_id`、`cage` 境界、`equipment_demand.contracted`（コロ契約電力）で
所有・境界・再課金の骨格は [08章](./08-tenancy-colocation.md) に分離モジュール化済み。運用プロファイルで切替。
割当（allocation）と所有（ownership）の分離・テナント階層・テナント別キャパシティ予算の深さは 08 章の範囲に委ねる。

---

## 18. 🟢 スコープの明示（意図的な割り切り）

以下は「DCIM コア」に絞った結果。汎用化の観点でスコープ判断として明示すべき:

- **IPAM 不在**: インターフェースはあるが IP/VLAN/VRF は範囲外（純 DCIM の割り切り）。
- **冷却トポロジー**: `equip_logical_link`（medium=`chilled_water`/`air`）・`capacity_budget` の cooling 次元で
  電力と冷却を同一グラフで辿れる。ただし気流・封じ込め・RDHx ループの詳細は薄い。
- **0U / バスウェイ**: U 位置前提（`rack_unit_occupancy`）で、0U 縦型 PDU・busway tap-off の表現が弱い
  （`rack_side` で部分対応）。
- **ディスカバリ来歴**: `source(discovered/manual/import)` / `last_seen` / merge 来歴が無く、
  自動投入運用には弱い。
- **DC/直流給電・OCP**: `medium='elec'` は媒体として持つが、48V 通信局給電・OCP busbar(51/54V) のモデルは薄い。

これらは「初版スコープ」として意図的非対象と明記し、拡張ロードマップに置くのが誠実。

---

## 優先度付き是正リスト

| 優先 | 項目 | 対応 |
|------|------|------|
| 🔴 P1 | 7. 空間タイプ語彙ハードコード | `loc_type` を**ルックアップテーブル化**（固定集合の CHECK enum は残す・[09章 9.2](./09-portability.md)） |
| 🟡 P2 | 5. v_equip_flow マテビュー鮮度 | 変更系 DML 後に明示的 REFRESH を呼ぶサービス層フローを定義；フレッシュネス UX 表示を推奨（[09章](./09-portability.md)） |
| 🟡 P2 | 6. redundancy_group 意図—実態乖離 | SPOF・容量充足検証をサービス層 / 監視ビューに置く（[04章](./04-validation-queries.md)・[09章 原則3](./09-portability.md)） |
| 🟡 P2 | 9. VA/W 誤称 | `available_power_va` 改名 ＋ `sqrt(3.0)` ＋ `power_factor` 列（overflow は bigint で問題なし） |
| 🟡 P2 | 10. 課金は numeric | 金額・kWh を `numeric` でキャスト・計算（コロ契約課金含む） |
| 🟡 P2 | 8. CHECK-IN 列挙の増えうる語彙 | 増えうる語彙のみ**ルックアップテーブル化**（固定集合は CHECK のまま） |
| 🟡 P2 | 1. 排他アーク残差 uniqueness | アーク別部分インデックスで重複行を防ぐ（`WHERE location_id IS NOT NULL` 等） |
| 🟡 P3 | 11. 三相・力率モデル | `power_feed_phase` 追加、`power_factor` 導入（item 9 と連動） |
| 🟡 P3 | 12. タイムゾーン | サイト相当 `location` に `time_zone` を追加 |
| 🟡 P3 | 13. 運用パラメータ外出し | chunk/retention/poll を config 化（[09章](./09-portability.md) 時系列アダプタ境界） |
| 🟡 P3 | 14. カスタムメトリック名前空間 | `metric.code` の builtin/custom 区分と命名規約を明文化 |
| 🟢 P4 | 15. 孤児時系列 | series 退役時クリーンアップ運用を明記（`drop_chunks` 併用） |
| ✅ 解決済 | EAV タグレイヤ（旧 A-1） | スコープ外として削除（EAV 面なし）。将来追加はリレーショナル派生スーパータイプで |
| ✅ 解決済 | 多態スコープキー（旧 A-3） | 排他アーク FK で解決（item 1 の部分インデックス残差を除く） |
| ✅ 解決済 | 3軸 point_type / quantity_unit（旧 A-4） | フラット `metric` カタログに一本化（単位は metric 行が保持） |
| ✅ 解決済 | equip_kind DAG（旧 A-2） | 単一親ツリーに確定。DAG caveat は将来有効化時のみ（item 4） |
| ✅ good | 16. U 重なり禁止 | 占有行＋複合 UNIQUE で解決（LCD・全エンジン移植可） |
| ✅ good | 17. マルチテナント/コロ | 分離モジュールとして [08章](./08-tenancy-colocation.md) で設計済み |
| 🟢 P4 | 18. スコープ（IPAM/0U/discovery 等） | 拡張ロードマップとして明示 |

> **総評**: [03章](./03-finalists.md) 確定案は旧案の最重要弱点（多態スコープキー・3軸 point_type・タグ EAV）を構造改善で解消しており、「ゆるい IoT スキーマにしない」という目標を量・単位・点意味の層で強く実現している（`metric` カタログが単位整合を構造的に保証・排他アーク FK が参照整合を保証・タグ EAV が不在）。残る最重要課題は **空間タイプ語彙のルックアップテーブル化（🔴 P1）** と、**`v_equip_flow` マテビュー鮮度管理・冗長グループ実態検証（🟡 P2 新規）** の 2 点。VA/W 誤称・float 課金・CHECK 列挙・三相力率も改善して多 DC パッケージとしての完成度を高める（🟡 P2-P3）。設計判断として許容する箇所（hypertable FK 無し・ツリー階層・U 占有行）は引き続き健全（🟢）。
