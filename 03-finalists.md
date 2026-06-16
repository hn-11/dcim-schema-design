# 03. 確定設計 — Semantic-Typed DCIM

DCIM パッケージの RDBMS スキーマ確定案。
**EcoStruxure IT のドメインモデル**（power path / Genome / 容量 / 冷却 / 冗長）を型付きで取り込み、
必要な意味づけだけを**参照表＋FK**で持つ。

基本方針は「**ゆるい IoT スキーマにせず、ドメイン知識を DB 制約で担保する**」こと
（[01章](./01-research-and-domain.md)）。
初見者向け概要は [00章](./00-overview.md)。
本章をスキーマと ER 図の正本とし、旧 05 章の全体 ER は本章へ統合した。

## 設計の骨子

中核は、**データセンターの実体モデル**（空間・資産・配電・冷却・容量・計測）を関係モデルで持ち、
ドメイン制約を DB に効かせること。
意味づけ（種類・量）は**参照表＋FK**で添える。
実装上の指針は次の通り:

- **接続の真実源は1つ** — `equipment` 間の物理接続を `connection` に保存し、電気固有の属性だけを
  `power_connection` に分ける。電力フロー / A系B系 / dual-cord はそこから**導出**する。
- **分類は必要な場所の参照表へ寄せる** — `equip_kind` は資産層、`medium` は接続層に置く。
  独立した「参照カタログ」レイヤは作らない。
- **収集接続は `data_point` に内包する** — `conn` 表は持たない。プロトコル・URI・アドレスは点の属性として保持する。
- **Haystack/Brick は発想の参考**で互換は必須にしない。必要になれば境界で後付けマッピングする。
  汎用 def / 多重継承 DAG /「何でも指せる」ref は持ち込まない。

> **表記方針**: テーブルは **Mermaid ER 図**で示す。CHECK 制約・生成列・複合 FK・排他アークなど ER で表せない
> ドメイン制約のみ SQL 抜粋を併記する。

> **横断制約**: **LCD（[09章](./09-portability.md)）**＝拡張は timescaledb のみ・PG 固有機能に依存しない。
> **マルチテナント/コロ（[08章](./08-tenancy-colocation.md)）**＝ `tenant` 所有・`cage` 境界・contracted power を加算。

```
L1 空間             location 木(+closure) / rack / equipment.location_id / 占有U行
L2 資産             equip_kind(tree) / equipment_type(Genome) → equipment（定義/実体分離）
L3 メトリック       metric（フラットカタログ：何を・単位・型・既定集約）
L4 点・収集         data_point（equipment×metric×role×phase + protocol/uri/addr。機器由来の点だけ）
L5 時系列           series 台帳 / measurement(raw) / current_value(cur) / rollup / derived(pPUE:10章)
L6 電力・冷却・冗長 equipment ノード ＋ connection/power_connection(CTI) ＋ redundancy_group / v_equip_flow(導出)
L7 容量             capacity_budget / equipment_demand（WP-150 × 推定負荷戦略・排他アーク）
L8 監視             threshold（severity / hysteresis・排他アーク）
```

---

## 全体俯瞰（主要結節点）

```mermaid
erDiagram
    TENANT ||--o{ LOCATION : owns
    LOCATION ||--o{ LOCATION : parent
    LOCATION ||--o{ LOCATION_CLOSURE : closure
    LOCATION ||--o{ RACK : contains
    LOCATION ||--o{ EQUIPMENT : installed
    RACK ||--o{ RACK_MOUNT : holds
    EQUIPMENT ||--o| RACK_MOUNT : mounted
    RACK_MOUNT ||--o{ RACK_UNIT_OCCUPANCY : occupies

    EQUIP_KIND ||--o{ EQUIP_KIND : parent
    EQUIP_KIND ||--o{ EQUIPMENT_TYPE : classifies
    MANUFACTURER ||--o{ EQUIPMENT_TYPE : makes
    EQUIPMENT_TYPE ||--o{ EQUIPMENT : instantiates
    EQUIPMENT_TYPE ||--o{ POINT_TEMPLATE : templates
    METRIC ||--o{ POINT_TEMPLATE : metric

    EQUIPMENT ||--o{ DATA_POINT : points
    METRIC ||--o{ DATA_POINT : measured
    DATA_POINT ||--o| SERIES : raw
    METRIC ||--o{ SERIES : denorm
    SERIES ||--o{ MEASUREMENT : "raw logical(no FK)"
    SERIES ||--o{ DERIVED_MEASUREMENT : "derived logical(no FK)"
    SERIES ||--o| CURRENT_VALUE : "current logical(no FK)"

    MEDIUM ||--o{ CONNECTION : medium
    EQUIPMENT ||--o{ CONNECTION : from
    EQUIPMENT ||--o{ CONNECTION : to
    CONNECTION ||--o| POWER_CONNECTION : electric
    REDUNDANCY_GROUP ||--o{ REDUNDANCY_MEMBER : members
    EQUIPMENT ||--o{ REDUNDANCY_MEMBER : belongs

    LOCATION ||--o{ CAPACITY_BUDGET : arc
    RACK ||--o{ CAPACITY_BUDGET : arc
    EQUIPMENT ||--o{ EQUIPMENT_DEMAND : demand
    METRIC ||--o{ THRESHOLD : arc
    EQUIPMENT ||--o{ THRESHOLD : arc
    LOCATION ||--o{ THRESHOLD : arc
```

> `equipment.location_id` が設置先の空間を直接指す。
> ラック搭載機器は追加で `rack_mount` にリンクし、ラック ID・U 位置・面を持つ。
> つまり `location_id` は空間集約用、`rack_mount` はラック内の物理搭載詳細用。

---

## L1. 空間層

`location` 隣接リスト＝真実源 ＋ `location_closure`（`ltree` 不使用 LCD）。
`equipment` は設置先 `location` を直接参照する。
`rack` は固定アンカーで、U 物理重なり禁止は**占有U行＋複合UNIQUE**（拡張ゼロ LCD）で担保する。

```mermaid
erDiagram
    TENANT ||--o{ LOCATION : owns
    LOCATION ||--o{ LOCATION : "parent_id (self)"
    LOCATION ||--o{ LOCATION_CLOSURE : "ancestor/descendant"
    LOCATION ||--o{ RACK : contains
    LOCATION ||--o{ EQUIPMENT : "installed (spaceRef)"
    RACK ||--o{ RACK_MOUNT : holds
    EQUIPMENT ||--o| RACK_MOUNT : "mounted (UNIQUE equipment)"
    RACK_MOUNT ||--o{ RACK_UNIT_OCCUPANCY : "occupies U×face"

    TENANT {
        bigint id PK
        text name UK
        text kind "internal|colo_customer"
    }
    LOCATION {
        bigint id PK
        bigint parent_id FK "self, RESTRICT"
        text loc_type "region|campus|building|floor|room|dataCenter|cage|zone|row"
        text name
        text code "現場の区画/機器番号"
        bigint tenant_id FK "cage 等の所有(08章)"
    }
    LOCATION_CLOSURE {
        bigint ancestor_id PK,FK
        bigint descendant_id PK,FK
        int depth "0=self"
    }
    RACK {
        bigint id PK
        bigint location_id FK
        int u_height "1..100"
        numeric max_weight_kg
        int rated_power_w
        text airflow
    }
    EQUIPMENT {
        bigint id PK
        bigint location_id FK
    }
    RACK_MOUNT {
        bigint id PK
        bigint rack_id FK
        bigint equipment_id FK "UNIQUE"
        int position
        int u_height
        text face "front|rear"
    }
    RACK_UNIT_OCCUPANCY {
        bigint rack_id PK
        text rack_side PK
        text face PK
        int u PK
        bigint mount_id FK
    }
```

```sql
CHECK ( loc_type IN ('region','campus') OR parent_id IS NOT NULL )      -- ルート以外は親必須
-- U: position + u_height - 1 <= rack.u_height は CONSTRAINT TRIGGER（親値参照）
-- フルデプス機器は front/rear 両面に占有U行 → 片面機器と必ず衝突（複合PKで原子的拒否）
```

---

## L2. 資産層 — equip_kind / Genome（equipment_type）→ 実機（equipment）

EcoStruxure の **Genome = 型番テンプレート** → 実機インスタンス化（NetBox 流 定義/実体分離）。
`equip_kind` はここで機器を分類する小マスタとして使う。独立レイヤにはしない。

```mermaid
erDiagram
    EQUIP_KIND ||--o{ EQUIP_KIND : "parent (tree)"
    MANUFACTURER ||--o{ EQUIPMENT_TYPE : makes
    EQUIP_KIND ||--o{ EQUIPMENT_TYPE : classifies
    EQUIPMENT_TYPE ||--o{ POINT_TEMPLATE : "機種が持つ点"
    EQUIPMENT_TYPE ||--o{ EQUIPMENT : instantiates
    METRIC ||--o{ POINT_TEMPLATE : ""
    LOCATION ||--o{ EQUIPMENT : "installed (spaceRef)"
    EQUIPMENT ||--o{ EQUIPMENT : "parent_equip (panel→breaker等)"
    TENANT ||--o{ EQUIPMENT : owns

    EQUIP_KIND {
        smallint id PK
        smallint parent_id FK "self (tree・「全HVAC」集約)"
        text code UK "ups|pdu|panel|breaker|crah|crac|chiller|generator|server|switch"
        text label
        text power_class "it|cooling|facility|other (CHECK・pPUE判別)"
    }
    MANUFACTURER {
        bigint id PK
        text name UK
    }
    EQUIPMENT_TYPE {
        bigint id PK
        bigint manufacturer_id FK
        smallint equip_kind_id FK "分類"
        text model
        int u_height "1..60"
        boolean is_full_depth
        numeric weight_kg
        int nameplate_power_w
        int derated_power_w "~50-75%"
        text ashrae_class "A1..A4"
    }
    POINT_TEMPLATE {
        bigint id PK
        bigint equipment_type_id FK
        bigint metric_id FK
        text role
        text name
    }
    EQUIPMENT {
        bigint id PK
        bigint equipment_type_id FK
        text asset_tag UK
        text serial UK
        text status "planned|staged|active|offline|decommissioned"
        bigint location_id FK
        bigint parent_equipment_id FK "self"
        int panel_position "breaker等の盤内位置(任意)"
        int poles "breaker等"
        numeric rated_a "breaker等"
        bigint tenant_id FK
    }
```

```sql
EQUIPMENT_TYPE: UNIQUE (manufacturer_id, model)
EQUIPMENT:      UNIQUE (parent_equipment_id, panel_position)  -- panel_position NULL は対象外扱い
```

> `point_template` は機種が持つ点の雛形で、実機作成時に `data_point`（L4）へ展開する。
> `equipment.location_id` は設置先の空間を直接指す。
> ラック搭載機器は、これに加えて `rack_mount` でラック ID・U 位置・面を持つ。
> `location_id` は空間集約用、`rack_mount` はラック内の物理搭載詳細用と役割を分ける。
> breaker は専用表にしない。
> `equipment(equip_kind='breaker')` と `parent_equipment_id` / `panel_position` / `rated_a` で表す。
> **「意味ある点の組合せ」は機種側に置く**。
> グローバルな意味カタログは作らない。

---

## L3. メトリック — フラットカタログ

「何を測るか」は**1つのフラットなカタログ**。
量・単位・データ型・既定集約を1行に持つ。
medium/position（吸気/排気・冷水往/還）は**コードに織り込む**（`rack_inlet_temp` / `chw_supply_temp`）。
Redfish/Prometheus/EcoStruxure と同じ作法で、DCIM の点は数えられる量（〜100）。

```mermaid
erDiagram
    METRIC ||--o{ DATA_POINT : "何を測るか"
    METRIC {
        smallint id PK
        text code UK "active_power|rack_inlet_temp|chw_supply_temp|fuel_level|ppue"
        text name
        text category "power|energy|voltage|current|pf|frequency|temperature|humidity|airflow|pressure|level|status|ratio (横断グルーピング)"
        text unit "kW|degC|%|cfm|Hz"
        text datatype "Number|Bool|Str"
        text value_kind "gauge|counter|status"
        text default_agg "avg|last|sum|min|max"
        boolean is_derived "pPUE等(10章)"
    }
```

> 単位整合は **metric 行が単位を1つ持つ**ことで成立する。
> 読み値は単位を持たないので、「power に degC」は起こり得ない。
> 横断は `category` 1列で行う（「全温度点」= `category='temperature'`）。
> 旧 3軸（quantity×phenomenon×func×duct のカタログ）は**廃止**。

---

## L4. 点・収集 — data_point（equipment×metric×role×phase）

点（`data_point`）= ある機器のある metric を、ある**役割**で、ある**相**で取得する設定。
`conn` 表は作らず、プロトコル・エンドポイント・取得アドレスを `data_point` に直接持たせる。
ゲートウェイ単位の集約が必要な場合も、`equipment`（ゲートウェイ機器）または `uri` で束ねられる。
`role` で sensor / sp（設定値）/ cmd（指令）を区別する。
システムが計算する pPUE 等の派生値は `data_point` ではなく、L5 の derived `series` として扱う。

```mermaid
erDiagram
    EQUIPMENT ||--o{ DATA_POINT : "点(実体)"
    METRIC ||--o{ DATA_POINT : "何を"

    DATA_POINT {
        bigint id PK
        bigint equipment_id FK
        smallint metric_id FK
        text role "sensor|sp|cmd (CHECK, default sensor)"
        text phase "L1|L2|L3|none (CHECK, default none)"
        boolean is_writable
        boolean is_cur
        boolean is_his
        text protocol "bacnet|modbus|snmp|redfish|ipmi|mqtt|opcua|haystack|api"
        text uri "gateway/endpoint"
        jsonb addr "{protocol}Cur/Write: oid/reg+fc/path"
        int poll_time_ms
        int stale_time_ms
        double scale
        double offset_
        boolean enabled
    }
```

```sql
-- 1機器内で同義の点は1つ。キー列は全て NOT NULL（phase は 'none' センチネル）→ UNIQUE が確実に効く
UNIQUE (equipment_id, metric_id, role, phase)
-- 例(CRAH): (温度,supply,role=sensor) 実測 / 同 (role=sp, is_writable) 設定 / (percent,role=cmd) ファン指令 が共存
-- writable の 16+1 段優先配列は制御本実装時に別表 write_level。今は is_writable フラグで拡張点だけを用意
```

---

## L5. 時系列 — raw / rollup / cur / derived

時系列は、**機器ごと・メトリックごとに物理テーブルを切らない**。
`series_id` を軸にした Narrow テーブルへ集約し、分割単位は性能と retention が変わる**ライフサイクル別**にする。
RDBMS 側の `series` 台帳だけが `data_point` / `metric` / `equipment` / `rack` / `location` / `measurement_scope` を知り、
TSDB 側へは整数の `series_id` だけを渡す（FK 越境なし・[09章](./09-portability.md)）。

```mermaid
erDiagram
    DATA_POINT ||--o| SERIES : "raw 1:1"
    METRIC ||--o{ SERIES : "denorm"
    MEASUREMENT_SCOPE ||--o{ SERIES : "derived scope (10章)"
    SERIES ||--o{ MEASUREMENT : "raw his・logical(no FK)"
    SERIES ||--o| CURRENT_VALUE : "cur・logical(no FK)"
    SERIES ||--o{ DERIVED_MEASUREMENT : "derived・logical(no FK)"
    MEASUREMENT ||..o{ MEASUREMENT_1H : "rollup (1m→1h→1d)"

    MEASUREMENT_SCOPE {
        bigint id PK
        text code UK
        text scope_kind "energy_boundary|cooling_boundary|tenant_boundary|custom"
    }
    SERIES {
        bigint series_id PK
        text series_kind "raw|derived"
        text scope_type "equipment|rack|location|measurement_scope"
        bigint data_point_id FK "raw のみ"
        smallint metric_id FK "denorm"
        bigint equipment_id "denorm(raw)"
        bigint rack_id "denorm(空間集約)"
        bigint location_id "denorm(閉包JOIN)"
        bigint measurement_scope_id FK "derived scope・10章"
        timestamptz retired_at
    }
    MEASUREMENT {
        timestamptz time "hypertable"
        bigint series_id
        double value
        smallint quality
    }
    CURRENT_VALUE {
        bigint series_id PK
        timestamptz time
        double value
        text cur_status "ok|down|fault|disabled|stale"
        text alert_state "normal|warning|critical"
    }
    MEASUREMENT_1H {
        timestamptz bucket
        bigint series_id
        double sum_v "avg=Σsum_v/Σn"
        bigint n
        double max_v
        double min_v
    }
```

```sql
CHECK ( (series_kind='raw' AND data_point_id IS NOT NULL AND measurement_scope_id IS NULL)
     OR (series_kind='derived' AND data_point_id IS NULL) )
CHECK ( scope_type <> 'measurement_scope' OR measurement_scope_id IS NOT NULL )
```

### TSDB テーブル分割方針（性能）

| テーブル | 役割 | 物理分割・索引 | 理由 |
|---|---|---|---|
| `measurement` | raw の firehose | 時間チャンク + `(series_id, time DESC)`。圧縮は `segmentby=series_id`、必要時だけ `series_id` hash partition | 書き込みが append になり、単一系列・期間読みが最速。機器別/メトリック別テーブルの爆発を避ける |
| `measurement_1m/1h/1d` | ダッシュボード・長期集計 | bucket + `series_id`。avg は `sum_v+n` で段階集計 | 期間に応じて raw/1m/1h/1d を切替。長期 retention でも即答しやすい |
| `derived_measurement` | pPUE 等の低頻度 KPI | raw と別 hypertable / 別 TTL | 派生 KPI は保持期間が長い。raw と混ぜると retention と圧縮設定が衝突する |
| `current_value` | 最新値 | RDBMS か KV 的な UPSERT 表。PK は `series_id` | ダッシュボード・監視は最新値を 1 行参照できる |

> 機器ごと / メトリックごとのテーブル分割は採らない。
> テーブル数・continuous aggregate・retention policy が機器数や点数に比例して増え、運用とプランニングが破綻しやすい。
> `series` に `metric_id` / `equipment_id` / `rack_id` / `location_id` を非正規化しておけば、
> RDBMS 側で対象 `series_id` を先に解決し、TSDB では Narrow な範囲スキャンだけで済む。
> pPUE のように機器へ直接結びつかない値は `data_point` にせず、`measurement_scope` に紐づく
> derived `series` として持つ。
> ロールアップ・派生 hypertable・計測スコープは [10章](./10-measurement-scope-derived-metrics.md)。
> 圧縮・retention のエンジン差分は [09章](./09-portability.md) の時系列アダプタが吸収する。

---

## L6. 電力・冷却・冗長 — equipment グラフ ＋ connection（CTI）＋ 冗長グループ

受電設備・変圧器・UPS・発電機・盤(panel)・breaker・PDU はすべて **`equipment` ノード**。
その間の接続を **汎用 `connection`（`medium` 付きエッジ）＋ 電気サブタイプ `power_connection`（PK 共有 CTI）** で表す。
これで **受電→変圧器→UPS→変圧器→分電盤→PDU→rack/機器**のチェーン全体がグラフになる
（`power_feed` は廃止＝その役割を connection＋power_connection が吸収）。
`v_equip_flow` はこのグラフを辿る**導出ビュー**。
冗長の**意図**は `redundancy_group` が持つ。
breaker は専用表にせず、`equipment(equip_kind='breaker')` と `parent_equipment_id` / `panel_position` / `rated_a` で表す。

```mermaid
erDiagram
    EQUIPMENT ||--o{ CONNECTION : "from (上流)"
    EQUIPMENT ||--o{ CONNECTION : "to (下流)"
    MEDIUM ||--o{ CONNECTION : medium
    CONNECTION ||--o| POWER_CONNECTION : "subtype (PK共有・medium=elec)"
    CABLE ||--o| CONNECTION : "物理裏付け(任意)"
    EQUIPMENT ||--o{ EQUIPMENT : "parent_equipment (盤→breaker)"
    REDUNDANCY_GROUP ||--o{ REDUNDANCY_MEMBER : members
    EQUIPMENT ||--o{ REDUNDANCY_MEMBER : belongs
    LOCATION ||--o| REDUNDANCY_GROUP : "serves (任意)"

    MEDIUM {
        smallint id PK
        text code UK "elec|air|chilled_water|condenser_water|refrigerant|fuel|water|data|fiber"
        text label
    }
    CONNECTION {
        bigint id PK
        bigint from_equipment_id FK "上流(源側)"
        bigint to_equipment_id FK "下流(負荷側)"
        smallint medium_id FK "elec|chilled_water|condenser_water|air"
        text redundancy "A|B|N|N+1|2N|none (CHECK)"
        bigint cable_id FK "物理裏付け(任意)"
    }
    POWER_CONNECTION {
        bigint connection_id PK "FK→connection(id) 共有PK(1:1)"
        numeric voltage_v
        text phase "single|three"
        numeric rated_a "回路ampacity"
        int max_utilization "NEC 80%"
        bigint available_power_w "GENERATED VxAxutilx√3"
    }
    REDUNDANCY_GROUP {
        bigint id PK
        text name
        text domain "power|cooling|network (CHECK)"
        text topology "N|N+1|N+2|2N|2N+1 (CHECK)"
        int n_required "負荷に要る台数 N"
        bigint serves_location_id FK "対象(任意)"
    }
    REDUNDANCY_MEMBER {
        bigint group_id PK,FK
        bigint equipment_id PK,FK
        text leg "A|B|C|none (CHECK・2N系で使用)"
        text role "active|standby|none (CHECK)"
    }
```

```sql
CONNECTION:        CHECK (from_equipment_id <> to_equipment_id)
POWER_CONNECTION:  available_power_w = round(voltage_v*rated_a*max_utilization/100.0
                                     * CASE WHEN phase='three' THEN 1.732 ELSE 1 END)::bigint  -- STORED
EQUIPMENT:         UNIQUE (parent_equipment_id, panel_position)  -- breaker等の盤内位置
-- 例: 受電設備 -[elec]-> 変圧器 -[elec]-> UPS -[elec]-> 変圧器 -[elec]-> 分電盤 -[elec]-> PDU -[elec]-> rack/機器
-- v_equip_flow = connection を辿る再帰ビュー（medium で絞る）。上流トレース/A・B系/SPOF はこの上で（[04章]）
-- CTI 整合: connection.medium='elec' ⟺ power_connection 行あり → サービス層検証
-- breaker は equipment の分類・親子・盤内位置として扱い、専用 breaker 表は作らない
-- 将来: 冷却サブタイプ `cooling_connection`（flow・往/還温度）を同型で追加（基底 connection は無改造）
-- A/B 系は connection.redundancy ＋ v_equip_flow 由来。多用時 equipment.power_side を非正規化(構成変更時に再計算)
-- 冗長検証: 2N は leg=A/B が独立 root か(SPOF・[04章UC-5])、N+1 は 1台落ちても容量充足か(L7)
```

---

## L7. 容量 — WP-150 × 推定負荷戦略（排他アーク）

容量5要素（空間/電力/電力分配/冷却/冷却分配）＋重量/ポート。
需要は推定負荷戦略で評価し **stranded = reserved − actual**。
スコープ（location/rack）は**多態キーをやめ排他アーク FK**で参照整合を担保する。
回路容量は `capacity_budget` ではなく、`power_connection.available_power_w` と breaker 機器の `rated_a` が持つ。

```mermaid
erDiagram
    LOCATION ||--o{ CAPACITY_BUDGET : "arc"
    RACK ||--o{ CAPACITY_BUDGET : "arc"
    EQUIPMENT ||--o{ EQUIPMENT_DEMAND : 需要

    CAPACITY_BUDGET {
        bigint id PK
        bigint location_id FK "arc (nullable)"
        bigint rack_id FK "arc (nullable)"
        text dimension "space_u|power_w|power_dist_w|cooling_w|cooling_dist|weight_kg|network_port"
        numeric rated
        numeric soft_limit "<= rated"
        numeric redundancy_reserve "failover/冗長確保(L6と連動)"
    }
    EQUIPMENT_DEMAND {
        bigint equipment_id PK,FK
        text dimension PK
        text load_strategy "nameplate|adjusted_nameplate|predicted|contracted"
        numeric nameplate
        numeric estimated
        numeric reserved
        numeric contracted "コロ契約電力(08章)"
    }
```

```sql
-- 排他アーク: スコープ対象はちょうど1つ（多態キー(scope_type,scope_id)を廃止し実FKで整合）
CAPACITY_BUDGET: CHECK ( num_nonnull(location_id, rack_id) = 1 )
-- 過剰容量分類(spare/idle/safety/stranded/active・WP-150)・「予約 ≤ rated」「負荷 ≤ 回路(power_connection.available_power_w)」は監視ビュー/サービス層([09章])
```

---

## L8. 監視 — threshold（severity / hysteresis・排他アーク）

severity は informational/warning/critical、しきい値 high/low、hysteresis でチャタリング抑制。
スコープは排他アーク FK。

```mermaid
erDiagram
    METRIC ||--o{ THRESHOLD : "arc (metric単位の既定)"
    EQUIPMENT ||--o{ THRESHOLD : "arc (equipment個別)"
    LOCATION ||--o{ THRESHOLD : "arc (location既定)"
    THRESHOLD {
        bigint id PK
        bigint metric_id FK "arc"
        bigint equipment_id FK "arc"
        bigint location_id FK "arc"
        numeric warn_high
        numeric crit_high
        numeric warn_low
        numeric crit_low
        numeric hysteresis
    }
```

```sql
CHECK ( num_nonnull(metric_id, equipment_id, location_id) = 1 )    -- 排他アーク
CHECK (crit_high IS NULL OR warn_high IS NULL OR crit_high >= warn_high)
-- 評価優先: equipment > location > metric既定。alert_state は current_value に同居(L5)
```

---

## 効くドメイン制約（まとめ）

| 制約 | 実装 | 由来 |
|------|------|------|
| 単位整合（power に degC 不可） | `metric` 行が単位を1つ保持（読み値は単位を持たない） | フラットカタログ |
| 1機器に同義の点は1つ | `data_point UNIQUE(equipment,metric,role,phase)`（全列 NOT NULL） | — |
| 制御の役割区別 | `data_point.role`（sensor/sp/cmd） | EcoStruxure/BMS |
| 「全 HVAC / 全 UPS」集約 | `equip_kind.parent_id` ツリー（DAG はオプション） | — |
| ブレーカ位置・容量 | `equipment(equip_kind='breaker')` + `UNIQUE(parent_equipment_id,panel_position)` + `rated_a` | EcoStruxure power path |
| 供給可能電力 = V×A×util×√3 | `power_connection.available_power_w`（生成列・bigint） | NEC + 三相 |
| 電力チェーンの表現 | `connection`(汎用エッジ・medium) ＋ `power_connection`(電気サブタイプ・PK共有) | 受電→変圧器→UPS→盤→PDU→rack |
| U 物理重なり禁止 | 占有U行 + 複合UNIQUE（拡張ゼロ） | LCD（09章） |
| 冗長の意図と検証 | `redundancy_group` + member（leg/role）→ SPOF/容量で現実構成を検証 | EcoStruxure 冗長 |
| スコープ参照の整合 | `capacity_budget`/`threshold` の**排他アーク FK**（固定対象・多態キー廃止） | 関係設計 |
| 電力/冷却フロー・A/B系 | `connection` グラフからの**導出**（v_equip_flow）。並行グラフを持たない | 真実源を1つに |
| TSDB 分割 | `measurement` raw / rollup / `derived_measurement` / `current_value`。機器別・メトリック別テーブルは作らない | 性能・運用性 |

> 集約制約（予約 ≤ 定格・SPOF・温湿度逸脱・冗長充足）は行間集約のため**サービス層/監視ビュー**で担保
> （移植性のため DB トリガに依存させない・[09章](./09-portability.md)）。検証クエリは [04章](./04-validation-queries.md)。
