# 05. ER 図（推奨案 P05）

推奨案 **P05「パッケージ標準」**（[03-finalists.md](./03-finalists.md)）のリレーショナルモデルを
Mermaid ER 図で可視化する。可読性のため**ドメイン別に分割**し、最後に全体俯瞰を置く。

> **凡例・前提**
> - 時系列（`measurement` / `measurement_1m|1h|1d` / `current_value`）は **TimescaleDB** 側。
>   hypertable には **実 FK を張らない**（チャンク DROP/圧縮を阻害するため）。`series_id` による
>   **論理参照**は図中 `logical(no FK)` と注記する。
> - `cable_termination.term_id` は **多態 FK**（`term_kind` で参照先が変わる）で実 FK が張れない。
>   図では代表的な参照先への論理リンクとして示す。
> - 集約 CAGG（`measurement_1m/1h/1d`）は hypertable から派生する**マテリアライズドビュー**。

---

## 5.1 空間階層（Space）

`Rack` を固定アンカーとして専用テーブル化し、`Site〜Row` は汎用 `location` 木（隣接リスト＋**閉包テーブル**）。
U 位置は **`rack_unit_occupancy` の占有U行 + 複合UNIQUE**で物理重なり禁止（[09章](./09-portability.md)の LCD 化）。

```mermaid
erDiagram
    TENANT ||--o{ LOCATION : owns
    LOCATION ||--o{ LOCATION : "parent_id (self)"
    LOCATION ||--o{ LOCATION_CLOSURE : "ancestor/descendant"
    LOCATION ||--o{ RACK : contains
    RACK ||--o{ RACK_MOUNT : holds
    DEVICE ||--o| RACK_MOUNT : "mounted (UNIQUE device)"
    RACK_MOUNT ||--o{ RACK_UNIT_OCCUPANCY : "occupies U×face"

    TENANT {
        bigint id PK
        text name UK
    }
    LOCATION {
        bigint id PK
        bigint parent_id FK "self ON DELETE RESTRICT"
        text loc_type "region|campus|building|floor|room|cage|zone|row"
        text name
        text code "[A-Za-z0-9_]"
        bigint tenant_id FK
    }
    LOCATION_CLOSURE {
        bigint ancestor_id PK "FK"
        bigint descendant_id PK "FK"
        int depth "0=self (ltree廃止・LCD)"
    }
    LOC_TYPE_RULE {
        text child_type PK
        text parent_type PK "NULL=root許可"
    }
    RACK {
        bigint id PK
        bigint location_id FK
        text name
        int u_height "1..100"
        int starting_unit
        boolean desc_units
        int width_in "10|19|21|23"
        numeric max_weight_kg
        int rated_power_w
        text airflow
    }
    RACK_MOUNT {
        bigint id PK
        bigint rack_id FK
        bigint device_id FK "UNIQUE"
        int position
        int u_height
        text face "front|rear"
        boolean is_full_depth
        text rack_side "full|left|right"
    }
    RACK_UNIT_OCCUPANCY {
        bigint rack_id PK
        text rack_side PK
        text face PK
        int u PK
        bigint mount_id FK "占有U行+複合UNIQUE=重なり禁止(LCD)"
    }
```

> `LOC_TYPE_RULE` は親子関係のホワイトリスト（`location` の親子検証）。`LOCATION_CLOSURE` は配下/祖先を
> 標準 JOIN で引く（`ltree` 廃止）。**U 重なり禁止は `RACK_UNIT_OCCUPANCY` の複合主キー**で担保
> （`EXCLUDE`/`btree_gist`/range 型を使わない LCD 化。フルデプスは front/rear 両面に占有行）。

---

## 5.2 機器マスタ（Asset）

型番（`device_type`）に固有属性とポートテンプレート、実機（`device`）に個体属性（NetBox 流の定義/実体分離）。
コア数値は型付き列、ロングテール属性は `specs jsonb`。

```mermaid
erDiagram
    MANUFACTURER ||--o{ DEVICE_TYPE : makes
    DEVICE_CATEGORY ||--o{ DEVICE_TYPE : classifies
    DEVICE_TYPE ||--o{ INTERFACE_TEMPLATE : "defines ports"
    DEVICE_TYPE ||--o{ POWER_PORT_TEMPLATE : "defines ports"
    DEVICE_TYPE ||--o{ DEVICE : instantiates
    MAINTENANCE_CONTRACT ||--o{ DEVICE : covers
    TENANT ||--o{ DEVICE : owns
    LOCATION ||--o{ DEVICE : "installed at"

    MANUFACTURER {
        bigint id PK
        text name UK
    }
    DEVICE_CATEGORY {
        text code PK "server|ups|pdu|switch|crac|sensor"
        text label
    }
    DEVICE_TYPE {
        bigint id PK
        bigint manufacturer_id FK
        text category FK
        text model
        int u_height "1..60"
        boolean is_full_depth
        numeric weight_kg
        int rated_power_w
        numeric rated_current_a
        text ashrae_class "A1..A4"
        jsonb specs "longtail attrs"
    }
    INTERFACE_TEMPLATE {
        bigint id PK
        bigint device_type_id FK
        text name
        text if_type
    }
    POWER_PORT_TEMPLATE {
        bigint id PK
        bigint device_type_id FK
        text name
        numeric rated_current_a
    }
    MAINTENANCE_CONTRACT {
        bigint id PK
        text vendor
        text contract_no UK
        date starts_on
        date ends_on
    }
    DEVICE {
        bigint id PK
        bigint device_type_id FK
        text asset_tag UK
        text serial UK
        text status "planned|staged|active|offline|decommissioned"
        bigint location_id FK
        bigint tenant_id FK
        bigint contract_id FK
        date installed_on
    }
```

> `(manufacturer_id, model)` 一意 / `serial`,`asset_tag` 一意。`UNIQUE(manufacturer_id, model)` は
> ER 図の属性 UK では複合表現できないため本文補足。`*_TEMPLATE` は実機作成時に実コンポーネント
> （`interface`/`power_port` 等）へ展開される。

---

## 5.3 配線トポロジー（Power & Network）

種別別ポート（`power_port`/`power_outlet`/`interface`/`front_port`/`rear_port`）と
`cable`/`cable_termination`（実体モデル）。電力上流は `power_panel → power_feed`。

```mermaid
erDiagram
    DEVICE ||--o{ POWER_PORT : has
    DEVICE ||--o{ POWER_OUTLET : has
    POWER_PORT ||--o{ POWER_OUTLET : "upstream (機内分配/ATS)"
    DEVICE ||--o{ INTERFACE : has
    DEVICE ||--o{ REAR_PORT : has
    DEVICE ||--o{ FRONT_PORT : has
    REAR_PORT ||--o{ FRONT_PORT : "front-rear map"
    LOCATION ||--o{ POWER_PANEL : houses
    POWER_PANEL ||--o{ POWER_FEED : supplies
    RACK ||--o{ POWER_FEED : "feeds (A/B)"
    CABLE ||--o{ CABLE_TERMINATION : "A/B ends"
    CABLE_TERMINATION }o..o| POWER_PORT : "term (poly,no FK)"
    CABLE_TERMINATION }o..o| POWER_OUTLET : "term (poly,no FK)"
    CABLE_TERMINATION }o..o| POWER_FEED : "term (poly,no FK)"
    CABLE_TERMINATION }o..o| INTERFACE : "term (poly,no FK)"

    POWER_PORT {
        bigint id PK
        bigint device_id FK
        text name
        enum redundancy "A|B"
        numeric rated_current_a
        numeric voltage_v
    }
    POWER_OUTLET {
        bigint id PK
        bigint device_id FK
        bigint upstream_power_port_id FK
        text name
        text feed_leg "A|B|C"
        numeric rated_current_a
    }
    INTERFACE {
        bigint id PK
        bigint device_id FK
        text name
        text if_type
        bigint lag_id FK "self"
    }
    REAR_PORT {
        bigint id PK
        bigint device_id FK
        text name
        int positions
    }
    FRONT_PORT {
        bigint id PK
        bigint device_id FK
        bigint rear_port_id FK
        int rear_position
        text name
    }
    POWER_PANEL {
        bigint id PK
        bigint location_id FK
        text name
    }
    POWER_FEED {
        bigint id PK
        bigint power_panel_id FK
        bigint rack_id FK
        text name
        text type "primary|redundant"
        enum redundancy "A|B"
        text supply "ac|dc"
        text phase "single|three"
        numeric voltage_v
        numeric amperage_a
        int max_utilization "default 80 (NEC)"
        bigint available_power_w "GENERATED VxAxutilx√3"
    }
    CABLE {
        bigint id PK
        text label
        text cable_type
        text color
        numeric length_m
        text status "connected|planned|decommissioning"
    }
    CABLE_TERMINATION {
        bigint id PK
        bigint cable_id FK
        char side "A|B"
        enum term_kind "power_port|power_outlet|power_feed|interface|front_port|rear_port|console"
        bigint term_id "poly FK (UNIQUE term_kind,term_id)"
    }
```

> `CABLE_TERMINATION (term_kind, term_id)` の UNIQUE が「ポート片側1接続」を保証。多態のため実 FK は
> 張れず、整合は検証トリガ＋定期チェック（厳格版は kind 別テーブル分割 + UNION ビュー）。
> 電力トレースは `power_arc` ビュー上の `WITH RECURSIVE`、end-to-end は `CablePath` 相当キャッシュ。

---

## 5.4 テレメトリ & 集約（Telemetry / Aggregation）

収集定義（`metric_definition`/`data_point`/`series`）と TimescaleDB 側（Narrow `measurement` + 階層 CAGG +
`current_value`）。キャパシティ・閾値も含む。**点線（logical/derived）は実 FK なし**。

```mermaid
erDiagram
    METRIC_DEFINITION ||--o{ DATA_POINT : defines
    DEVICE ||--o{ DATA_POINT : "monitored points"
    DATA_POINT ||--|| SERIES : "1:1 registry"
    DEVICE ||--o{ SERIES : "denorm"
    METRIC_DEFINITION ||--o{ SERIES : "denorm"
    SERIES ||--o{ MEASUREMENT : "logical (no FK)"
    SERIES ||--o| CURRENT_VALUE : "logical (no FK)"
    MEASUREMENT ||..o{ MEASUREMENT_1M : "CAGG derive"
    MEASUREMENT_1M ||..o{ MEASUREMENT_1H : "CAGG derive"
    MEASUREMENT_1H ||..o{ MEASUREMENT_1D : "CAGG derive"
    METRIC_DEFINITION ||--o{ THRESHOLD : scopes
    DEVICE ||--o{ DEVICE_DEMAND : "nameplate/reserved"

    METRIC_DEFINITION {
        smallint metric_def_id PK
        text code UK "active_power|inlet_temp|humidity"
        text unit "W|degC|%RH"
        text value_kind "gauge|counter|status|enum"
        text default_agg "avg|min|max|sum|last|count"
    }
    DATA_POINT {
        bigint data_point_id PK
        bigint device_id FK
        smallint metric_def_id FK
        text protocol "snmp|modbus|ipmi|redfish|bacnet|api"
        jsonb addr "oid/reg+fc/path"
        double scale
        double offset_
        text phase
        boolean enabled
    }
    SERIES {
        bigint series_id PK
        bigint data_point_id FK "UNIQUE"
        bigint device_id FK "denorm"
        smallint metric_def_id FK "denorm"
        bigint rack_id "denorm (空間集約)"
        bigint location_id "denorm (閉包JOIN・ltree廃止)"
        timestamptz retired_at
    }
    MEASUREMENT {
        timestamptz time "hypertable"
        bigint series_id "logical ref"
        double value
        smallint quality "0good|1unc|2bad"
    }
    MEASUREMENT_1M {
        timestamptz bucket
        bigint series_id
        double sum_v "avg = sum_v/n"
        bigint n
        double max_v
        double min_v
        any p95 "任意列(engine依存・LCDでは省略可)"
    }
    MEASUREMENT_1H {
        timestamptz bucket
        bigint series_id
        double sum_v
        bigint n
        double max_v
        double min_v
    }
    MEASUREMENT_1D {
        timestamptz bucket
        bigint series_id
        double sum_v
        bigint n
        double max_v
        double min_v
    }
    CURRENT_VALUE {
        bigint series_id PK
        timestamptz time
        double value
        text alert_state "normal|warning|critical|stale"
    }
    THRESHOLD {
        bigint id PK
        text scope_type "global|location|device"
        bigint scope_id
        smallint metric_def_id FK
        numeric warn_high
        numeric crit_high
        numeric warn_low
        numeric crit_low
        numeric hysteresis
    }
    CAPACITY_BUDGET {
        bigint location_or_rack_id PK
        text scope PK "location|rack"
        text dimension PK "power_w|space_u|cooling_w|floor_load_kg|port"
        numeric rated
        numeric soft_limit "<= rated"
    }
    DEVICE_DEMAND {
        bigint device_id PK
        text dimension PK
        numeric nameplate
        numeric reserved
    }
```

> `CAPACITY_BUDGET` は `scope`（location/rack）で対象が変わる多態キー、`THRESHOLD` は
> `scope_type`（global/location/device）優先で評価。いずれも集約制約（予約合計 ≤ 定格 等）は
> 行間集約のため CONSTRAINT TRIGGER／監視ビューで担保（[04 章](./04-validation-queries.md)）。

---

## 5.5 全体俯瞰（主要エンティティと結節点）

ドメイン間の結節点のみを示した俯瞰図。詳細属性は 5.1〜5.4 を参照。

```mermaid
erDiagram
    TENANT ||--o{ LOCATION : owns
    TENANT ||--o{ DEVICE : owns
    LOCATION ||--o{ LOCATION : parent
    LOCATION ||--o{ RACK : contains
    LOCATION ||--o{ DEVICE : "installed at"
    LOCATION ||--o{ POWER_PANEL : houses
    RACK ||--o{ RACK_MOUNT : holds
    RACK ||--o{ POWER_FEED : feeds
    DEVICE ||--o| RACK_MOUNT : mounted
    MANUFACTURER ||--o{ DEVICE_TYPE : makes
    DEVICE_CATEGORY ||--o{ DEVICE_TYPE : classifies
    DEVICE_TYPE ||--o{ DEVICE : instantiates
    DEVICE ||--o{ POWER_PORT : has
    DEVICE ||--o{ INTERFACE : has
    DEVICE ||--o{ DATA_POINT : "monitored"
    POWER_PANEL ||--o{ POWER_FEED : supplies
    CABLE ||--o{ CABLE_TERMINATION : "A/B ends"
    CABLE_TERMINATION }o..o| POWER_PORT : "term (poly)"
    CABLE_TERMINATION }o..o| INTERFACE : "term (poly)"
    METRIC_DEFINITION ||--o{ DATA_POINT : defines
    DATA_POINT ||--|| SERIES : registers
    SERIES ||--o{ MEASUREMENT : "logical (no FK)"
    SERIES ||--o| CURRENT_VALUE : "logical (no FK)"
    MEASUREMENT ||..o{ MEASUREMENT_1H : "CAGG (1m→1h→1d)"

    TENANT { bigint id PK }
    LOCATION { bigint id PK }
    RACK { bigint id PK }
    RACK_MOUNT { bigint id PK }
    MANUFACTURER { bigint id PK }
    DEVICE_CATEGORY { text code PK }
    DEVICE_TYPE { bigint id PK }
    DEVICE { bigint id PK }
    POWER_PORT { bigint id PK }
    INTERFACE { bigint id PK }
    POWER_PANEL { bigint id PK }
    POWER_FEED { bigint id PK }
    CABLE { bigint id PK }
    CABLE_TERMINATION { bigint id PK }
    METRIC_DEFINITION { smallint metric_def_id PK }
    DATA_POINT { bigint data_point_id PK }
    SERIES { bigint series_id PK }
    MEASUREMENT { timestamptz time }
    MEASUREMENT_1H { timestamptz bucket }
    CURRENT_VALUE { bigint series_id PK }
```

---

## 5.6 図に表れない主要制約（再掲）

ER 図のカーディナリティ／FK では表現しきれない、本設計の肝となる制約:

| 種別 | 制約 | 実装 |
|------|------|------|
| 一意 | ラック U の物理重なり禁止 | `RACK_UNIT_OCCUPANCY` 複合主キー `(rack_id, rack_side, face, u)`（占有U行・拡張ゼロ・全エンジン移植可。full-depth は front/rear 両面に占有行） |
| 複合一意 | 型番一意 | `DEVICE_TYPE UNIQUE(manufacturer_id, model)` |
| 複合一意 | ポート片側1接続 | `CABLE_TERMINATION UNIQUE(term_kind, term_id)` |
| 生成列 | 供給可能電力 | `POWER_FEED.available_power_w = round(V×A×util/100×√3)` bigint |
| トリガ | ラックに収まる | `position + u_height - 1 ≤ rack.u_height`（親値参照） |
| トリガ | 親子関係の妥当性 | `LOC_TYPE_RULE` 突合 |
| トリガ/ビュー | 予約 ≤ 定格 / SPOF / ASHRAE 逸脱 | 集約制約のため監視（[04 章](./04-validation-queries.md)） |

> 他案（P04/P06/P08/P02）の ER は P05 との差分（[03 章](./03-finalists.md)）で読み替え可能。
> 例: P06 は `LOCATION` に `LOCATION_CLOSURE(ancestor_id, descendant_id, depth)` を追加、
> P08 は `RACK_MOUNT` を有効期間付き `DEVICE_PLACEMENT`（PG版=`valid tstzrange`+EXCLUDE / LCD版=`valid_from`/`valid_to` 2列 + 占有行 + サービス層）に置換。
