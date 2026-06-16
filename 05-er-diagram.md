# 05. ER 図（Semantic-Typed DCIM 全体）

確定設計（[03章](./03-finalists.md)）のリレーショナルモデルを Mermaid ER 図で俯瞰する。03章はレイヤ別の詳細、
本章は**ドメインをまたぐ結節点**を中心に、最後に全体俯瞰を置く。DDL/制約の根拠は [03章](./03-finalists.md)。

> **凡例・前提**
> - 時系列（`measurement` / `measurement_1h` / `derived_measurement` / `current_value`）は **TimescaleDB** 側。
>   hypertable には **実 FK を張らない**（`series_id` の論理参照のみ＝図中 `logical(no FK)`・[09章](./09-portability.md)）。
> - `capacity_budget` / `threshold` のスコープは**排他アーク FK**（nullable FK 3列 + `CHECK num_nonnull=1`）。多態キー `(scope_type, scope_id)` は使わない。
> - 閉包（`location_closure` / 任意の `equip_kind_closure`）は再帰なし JOIN で部分木集約（`ltree` 不使用 LCD・DAG はオプション [02章D](./02-candidate-patterns.md)）。
> - **物理 path = 真実源、フローは導出**：`power_panel→breaker→power_feed` が connectivity の真実源。電力フロー / A系B系 / dual-cord は `v_equip_flow`（導出ビュー）で取得する。並行する別グラフを手で維持しない。

---

## 5.1 参照カタログ（L1）

分類用の小さな参照表は2つのみ。機器の種類（`equip_kind`）と媒体（`medium`）。汎用 def / DAG / Quantity / Phenomenon テーブルは持たない。

```mermaid
erDiagram
    EQUIP_KIND ||--o{ EQUIP_KIND : "parent (tree)"

    EQUIP_KIND {
        smallint id PK
        smallint parent_id FK "self (tree・「全HVAC」集約)"
        text code UK "ups|pdu|crah|crac|chiller|generator|ats|server|switch"
        text label
        text power_class "it|cooling|facility|other (CHECK・pPUE判別)"
    }
    MEDIUM {
        smallint id PK
        text code UK "elec|air|chilled_water|condenser_water|refrigerant|fuel|water"
        text label
    }
```

> `equip_kind` は機器分類ツリー（多重継承の実需が出たら閉包へ拡張・[02章D](./02-candidate-patterns.md)）。`medium` は L7 の冷却/電力フロー媒体に使う。
> role / phase / dimension / severity 等の固定小集合は各テーブルで **CHECK enum** とし、参照テーブルを作らない。

---

## 5.2 空間層（L2）

`location` 隣接リスト＝真実源 ＋ `location_closure`（`ltree` 不使用 LCD）。`rack` は固定アンカー。U 物理重なり禁止は
**占有U行＋複合UNIQUE**（拡張ゼロ LCD）。

```mermaid
erDiagram
    TENANT ||--o{ LOCATION : owns
    LOCATION ||--o{ LOCATION : "parent_id (self)"
    LOCATION ||--o{ LOCATION_CLOSURE : "ancestor/descendant"
    LOCATION ||--o{ RACK : contains
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
        bigint tenant_id FK "cage等の所有(08章)"
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

## 5.3 資産層（L3）— Genome（equipment_type）→ 実機（equipment）

EcoStruxure の **Genome = 型番テンプレート** → 実機インスタンス化（NetBox 流 定義/実体分離）。`equip_kind` FK で意味づけ。
`point_template` は機種が持つ点の雛形で、実機作成時に `data_point`（L5）へ展開。

```mermaid
erDiagram
    MANUFACTURER ||--o{ EQUIPMENT_TYPE : makes
    EQUIP_KIND ||--o{ EQUIPMENT_TYPE : classifies
    EQUIPMENT_TYPE ||--o{ POINT_TEMPLATE : "機種が持つ点"
    EQUIPMENT_TYPE ||--o{ EQUIPMENT : instantiates
    METRIC ||--o{ POINT_TEMPLATE : ""
    LOCATION ||--o{ EQUIPMENT : "installed (spaceRef)"
    EQUIPMENT ||--o{ EQUIPMENT : "parent_equipment (enclosure等)"
    TENANT ||--o{ EQUIPMENT : owns

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
        bigint tenant_id FK
    }
```

> `UNIQUE (manufacturer_id, model)` で型番一意、`serial`/`asset_tag` で個体一意。**「意味ある点の組合せ」はここ（機種）に在る** ── グローバルな意味カタログを作らない。

---

## 5.4 メトリック & 収集層（L4 + L5）

**L4** — 「何を測るか」は1つのフラットなカタログ。medium / position（吸気/排気・冷水往/還）はコードに織り込む
（`rack_inlet_temp` / `chw_supply_temp`）。旧3軸（quantity×phenomenon×func×duct）は廃止（[02章B](./02-candidate-patterns.md)）。

**L5** — 点（`data_point`）= ある機器のある metric を、ある**役割**（`role`）で、ある**相**（`phase`）で取得する設定。
`role` で sensor / sp（設定値）/ cmd（指令）/ derived を区別 ── 制御を見据えた一級の軸。

```mermaid
erDiagram
    EQUIPMENT ||--o{ CONN : "gateway (任意)"
    EQUIPMENT ||--o{ DATA_POINT : "点(実体)"
    METRIC ||--o{ DATA_POINT : "何を"
    CONN ||--o{ DATA_POINT : connRef

    METRIC {
        smallint id PK
        text code UK "active_power|rack_inlet_temp|chw_supply_temp|fuel_level|ppue"
        text name
        text category "power|energy|voltage|current|pf|frequency|temperature|humidity|airflow|pressure|level|status|ratio"
        text unit "kW|°C|%|cfm|Hz"
        text datatype "Number|Bool|Str"
        text value_kind "gauge|counter|status"
        text default_agg "avg|last|sum|min|max"
        boolean is_derived "pPUE等(10章)"
    }
    CONN {
        bigint id PK
        bigint equipment_id FK
        text protocol "bacnet|modbus|snmp|redfish|ipmi|mqtt|opcua|haystack|api"
        text uri
        int poll_time_ms
        int stale_time_ms
        text conn_status "ok|down|fault|disabled|unknown"
    }
    DATA_POINT {
        bigint id PK
        bigint equipment_id FK
        smallint metric_id FK
        text role "sensor|sp|cmd|derived (CHECK, default sensor)"
        text phase "L1|L2|L3|none (CHECK, default none)"
        boolean is_writable
        boolean is_cur
        boolean is_his
        bigint conn_id FK "connRef"
        jsonb addr "{protocol}Cur/Write: oid/reg+fc/path"
        double scale
        double offset_
        boolean enabled
    }
```

```sql
-- 1機器内で同義の点は1つ。キー列は全て NOT NULL（phase は 'none' センチネル）→ UNIQUE が確実に効く
UNIQUE (equipment_id, metric_id, role, phase)
-- 単位整合: metric 行が単位を1つ保持（読み値は単位を持たない）→「power に °C」は起こり得ない
```

---

## 5.5 時系列層（L6）— his / cur / 派生

his＝`measurement` hypertable、cur＝`current_value`。Narrow＋`series` 台帳（`series_id` だけ TSDB へ越境・FK 越境なし・[09章](./09-portability.md)）。

```mermaid
erDiagram
    DATA_POINT ||--o| SERIES : "raw 1:1"
    METRIC ||--o{ SERIES : "denorm"
    ROOM_GROUP ||--o{ SERIES : "derived scope (10章)"
    SERIES ||--o{ MEASUREMENT : "his・logical(no FK)"
    SERIES ||--o| CURRENT_VALUE : "cur・logical(no FK)"
    SERIES ||--o{ DERIVED_MEASUREMENT : "logical(no FK)"
    MEASUREMENT ||..o{ MEASUREMENT_1H : "CAGG (1m→1h→1d)"

    SERIES {
        bigint series_id PK
        text series_kind "raw|derived"
        text scope_type "equipment|rack|location|room_group"
        bigint data_point_id FK "raw のみ"
        smallint metric_id FK "denorm"
        bigint equipment_id "denorm(raw)"
        bigint rack_id "denorm(空間集約)"
        bigint location_id "denorm(閉包JOIN)"
        bigint room_group_id FK "denorm(電力境界・10章)"
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
    DERIVED_MEASUREMENT {
        timestamptz time
        bigint series_id
        double value
    }
```

```sql
CHECK ( (series_kind='raw' AND data_point_id IS NOT NULL AND scope_type='equipment')
     OR (series_kind='derived' AND data_point_id IS NULL) )
```

> ロールアップ・派生 hypertable・部屋グループは [10章](./10-room-group-derived-metrics.md)。圧縮 `compress_segmentby=series_id`・retention は [09章](./09-portability.md)。

---

## 5.6 電力・冷却・冗長層（L7）

**物理 path が真実源**：`power_panel→breaker→power_feed` が connectivity の真実源。
電力フロー / A系B系 / dual-cord は `v_equip_flow`（**導出ビュー**）で取得する。並行する別グラフを手で維持しない。
物理エッジを持たない純論理（冷却空気の到達等）だけ `equip_logical_link` に明示。
冗長の**意図**は `redundancy_group` で第一級に持つ。

```mermaid
erDiagram
    LOCATION ||--o{ POWER_PANEL : houses
    EQUIPMENT ||--o| POWER_PANEL : "on (PDU/RPP)"
    POWER_PANEL ||--o{ BREAKER : "位置・容量"
    BREAKER ||--o{ POWER_FEED : "分岐回路"
    RACK ||--o{ POWER_FEED : feeds
    EQUIPMENT ||--o{ EQUIP_LOGICAL_LINK : "from/to (物理エッジ無しの論理のみ)"
    MEDIUM ||--o{ EQUIP_LOGICAL_LINK : substance
    REDUNDANCY_GROUP ||--o{ REDUNDANCY_MEMBER : members
    EQUIPMENT ||--o{ REDUNDANCY_MEMBER : belongs
    LOCATION ||--o| REDUNDANCY_GROUP : "serves (任意)"

    POWER_PANEL {
        bigint id PK
        bigint location_id FK
        bigint on_equipment_id FK "PDU/RPP"
        text name
    }
    BREAKER {
        bigint id PK
        bigint power_panel_id FK
        int position
        numeric rated_a
        int poles "1|2|3"
        text phase
    }
    POWER_FEED {
        bigint id PK
        bigint breaker_id FK
        bigint rack_id FK
        numeric voltage_v
        numeric amperage_a
        text phase "single|three"
        int max_utilization "NEC 80%"
        bigint available_power_w "GENERATED VxAxutilx√3"
    }
    EQUIP_LOGICAL_LINK {
        bigint id PK
        bigint from_equipment_id FK
        bigint to_equipment_id FK
        smallint medium_id FK
        text note
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
POWER_FEED: available_power_w = round(voltage_v*amperage_a*max_utilization/100.0
                                      * CASE WHEN phase='three' THEN 1.732 ELSE 1 END)::bigint  -- STORED
BREAKER:    UNIQUE (power_panel_id, position),  rated_a > 0
-- 電力フローは構造から導出（手で並行グラフを持たない）:
--   v_equip_flow = power_port/outlet/cable と feed→breaker→panel を辿る再帰ビュー（[04章]）
-- A/B 系: v_equip_flow を root まで辿り独立系統を判定。多用するなら equipment.power_side を非正規化(構成変更時に再計算)
-- dual-cord: equipment の各 PSU が繋がる outlet の side（A/B feed）から導出
-- redundancy 検証(intent×reality): 2N は leg=A/B が独立 root か(SPOF・[04章UC-5])、N+1 は 1台落ちても容量充足か(L8)
```

---

## 5.7 容量 & 監視層（L8 + L9）

**L8** — 容量5要素（空間/電力/電力分配/冷却/冷却分配）＋重量/ポート。スコープ（location/rack/breaker）は
**排他アーク FK**（多態キー廃止）で参照整合を担保。

**L9** — severity は informational/warning/critical、しきい値 high/low、hysteresis でチャタリング抑制。スコープは排他アーク FK。

```mermaid
erDiagram
    LOCATION ||--o{ CAPACITY_BUDGET : "arc"
    RACK ||--o{ CAPACITY_BUDGET : "arc"
    BREAKER ||--o{ CAPACITY_BUDGET : "arc"
    EQUIPMENT ||--o{ EQUIPMENT_DEMAND : 需要
    METRIC ||--o{ THRESHOLD : "arc (metric単位の既定)"
    EQUIPMENT ||--o{ THRESHOLD : "arc (equipment個別)"
    LOCATION ||--o{ THRESHOLD : "arc (location既定)"

    CAPACITY_BUDGET {
        bigint id PK
        bigint location_id FK "arc (nullable)"
        bigint rack_id FK "arc (nullable)"
        bigint breaker_id FK "arc (nullable)"
        text dimension "space_u|power_w|power_dist_w|cooling_w|cooling_dist|weight_kg|network_port"
        numeric rated
        numeric soft_limit "<= rated"
        numeric redundancy_reserve "failover/冗長確保(L7と連動)"
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
-- 排他アーク: スコープ対象はちょうど1つ（多態キー(scope_type,scope_id)廃止・実FKで整合）
CAPACITY_BUDGET: CHECK ( num_nonnull(location_id, rack_id, breaker_id) = 1 )
THRESHOLD:       CHECK ( num_nonnull(metric_id, equipment_id, location_id) = 1 )
CHECK (crit_high IS NULL OR warn_high IS NULL OR crit_high >= warn_high)
-- 評価優先: equipment > location > metric既定。alert_state は current_value に同居(L6)
-- 過剰容量分類(spare/idle/safety/stranded/active・WP-150)・「予約 ≤ rated」はサービス層([04章][09章])
```

---

## 5.8 全体俯瞰（結節点のみ）

ドメイン間の結節点だけを示す俯瞰図。詳細属性は 5.1〜5.7 を参照。

```mermaid
erDiagram
    EQUIP_KIND ||--o{ EQUIP_KIND : "parent (tree)"
    EQUIP_KIND ||--o{ EQUIPMENT_TYPE : classifies
    MEDIUM ||--o{ EQUIP_LOGICAL_LINK : substance
    TENANT ||--o{ LOCATION : owns
    TENANT ||--o{ EQUIPMENT : owns
    LOCATION ||--o{ LOCATION : parent
    LOCATION ||--o{ RACK : contains
    LOCATION ||--o{ EQUIPMENT : installed
    LOCATION ||--o{ POWER_PANEL : houses
    LOCATION ||--o{ CAPACITY_BUDGET : "arc"
    LOCATION ||--o{ THRESHOLD : "arc"
    RACK ||--o{ RACK_MOUNT : holds
    RACK ||--o{ POWER_FEED : feeds
    RACK ||--o{ CAPACITY_BUDGET : "arc"
    EQUIPMENT ||--o| RACK_MOUNT : mounted
    EQUIPMENT_TYPE ||--o{ EQUIPMENT : instantiates
    EQUIPMENT ||--o{ DATA_POINT : monitored
    EQUIPMENT ||--o{ EQUIP_LOGICAL_LINK : "from/to"
    EQUIPMENT ||--o{ EQUIPMENT_DEMAND : capacity
    EQUIPMENT ||--o{ REDUNDANCY_MEMBER : belongs
    EQUIPMENT ||--o{ THRESHOLD : "arc"
    POWER_PANEL ||--o{ BREAKER : breakers
    BREAKER ||--o{ POWER_FEED : circuits
    BREAKER ||--o{ CAPACITY_BUDGET : "arc"
    METRIC ||--o{ DATA_POINT : types
    METRIC ||--o{ SERIES : denorm
    METRIC ||--o{ THRESHOLD : "arc"
    DATA_POINT ||--o| SERIES : registers
    SERIES ||--o{ MEASUREMENT : "his (no FK)"
    SERIES ||--o| CURRENT_VALUE : "cur (no FK)"
    ROOM_GROUP ||--o{ SERIES : "derived (10章)"
    REDUNDANCY_GROUP ||--o{ REDUNDANCY_MEMBER : members

    EQUIP_KIND { smallint id PK }
    MEDIUM { smallint id PK }
    TENANT { bigint id PK }
    LOCATION { bigint id PK }
    RACK { bigint id PK }
    RACK_MOUNT { bigint id PK }
    EQUIPMENT_TYPE { bigint id PK }
    EQUIPMENT { bigint id PK }
    POWER_PANEL { bigint id PK }
    BREAKER { bigint id PK }
    POWER_FEED { bigint id PK }
    EQUIP_LOGICAL_LINK { bigint id PK }
    METRIC { smallint id PK }
    DATA_POINT { bigint id PK }
    SERIES { bigint series_id PK }
    MEASUREMENT { timestamptz time }
    CURRENT_VALUE { bigint series_id PK }
    EQUIPMENT_DEMAND { bigint equipment_id PK }
    CAPACITY_BUDGET { bigint id PK }
    THRESHOLD { bigint id PK }
    REDUNDANCY_GROUP { bigint id PK }
    REDUNDANCY_MEMBER { bigint group_id PK }
    ROOM_GROUP { bigint id PK }
```

---

## 5.9 図に表れない主要制約（再掲）

ER のカーディナリティ／FK では表せない、本設計の肝（詳細は [03章](./03-finalists.md)・[06章](./06-self-review.md)）:

| 種別 | 制約 | 実装 |
|------|------|------|
| 単位整合 | power に °C 不可 | `metric` 行が単位を1つ保持（読み値は単位を持たない） |
| 複合一意 | 機器内の点は同義で1つ | `DATA_POINT UNIQUE(equipment_id, metric_id, role, phase)`（全列 NOT NULL） |
| 役割区別 | 制御を見据えた一級の軸 | `data_point.role`（sensor/sp/cmd/derived） |
| ツリー | 「全 HVAC / 全 UPS」集約 | `EQUIP_KIND.parent_id` 親参照（読み多なら closure・DAG はオプション [02章D](./02-candidate-patterns.md)） |
| 複合一意 | U 物理重なり禁止 | `RACK_UNIT_OCCUPANCY (rack_id, rack_side, face, u)`（占有U行・拡張ゼロ） |
| 生成列 | 供給可能電力 = V×A×util×√3 | `POWER_FEED.available_power_w` bigint STORED |
| 複合一意 | ブレーカ位置・容量 | `BREAKER UNIQUE(power_panel_id, position)` + `rated_a > 0` |
| 排他アーク | 容量スコープの参照整合 | `CAPACITY_BUDGET CHECK(num_nonnull(location_id, rack_id, breaker_id)=1)` |
| 排他アーク | 監視スコープの参照整合 | `THRESHOLD CHECK(num_nonnull(metric_id, equipment_id, location_id)=1)` |
| 冗長意図と検証 | intent × reality | `redundancy_group` + member（leg/role）→ SPOF/容量で reality 検証 |
| 論理参照 | TSDB 越境 FK 不可 | `measurement`/`current_value` に実 FK を張らない（`series_id` 論理参照のみ） |
| 導出ビュー | 電力フロー・A/B系・dual-cord | `v_equip_flow`（物理 path からの再帰ビュー）。並行グラフを手で維持しない |
| トリガ/ビュー | ラック収まり / SPOF / ASHRAE / 予約≤定格 / stranded | サービス層（[04章](./04-validation-queries.md)・[09章](./09-portability.md)） |

> 検証クエリは [04章](./04-validation-queries.md)、部屋グループ/pPUE は [10章](./10-room-group-derived-metrics.md)、
> テナント/コロ拡張は [08章](./08-tenancy-colocation.md)。
