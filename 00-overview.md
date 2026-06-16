# 00. Overview — チーム共有 & 相互レビュー用

> 初見のメンバーが **ER 図をたどって 10 分で全体像を掴み**、レビューに参加するための概要。
> 各図に短い解説を添える。
> 完全な属性・制約・網羅 ER は [03章（確定設計）](./03-finalists.md)。

## これは何か

データセンターを統合管理する DCIM パッケージの **RDBMS スキーマ**。
構成・資産・配電・冷却・容量はリレーショナルに持ち、時系列（計測値）は **TimescaleDB** に置く。
EMS/BMS/IT を横断し、将来の制御（設定値・指令）も視野に入れる。

狙いは、**ドメイン知識を DB 制約で担保すること**と、**多 DC へ横展開できるパッケージにすること**。
つまり「ゆるい IoT スキーマ」にしない。

---

## 1. 全体スパイン（まずこれ）

中核の流れは2本 ──「**空間に機器が載り、機器に点があり、点が時系列を生む**」と「**機器は配電・冷却され、冗長を組む**」。

```mermaid
erDiagram
    LOCATION ||--o{ RACK : contains
    RACK ||--o{ RACK_MOUNT : holds
    EQUIPMENT ||--o| RACK_MOUNT : mounted
    EQUIPMENT_TYPE ||--o{ EQUIPMENT : instantiates
    EQUIPMENT ||--o{ DATA_POINT : "点(計測/制御)"
    METRIC ||--o{ DATA_POINT : "何を測るか"
    DATA_POINT ||--o| SERIES : registers
    SERIES ||--o{ MEASUREMENT : "履歴(TimescaleDB)"
    SERIES ||--o| CURRENT_VALUE : "最新値"
    EQUIPMENT ||--o{ CONNECTION : "from/to"
    REDUNDANCY_GROUP ||--o{ EQUIPMENT : "冗長(N+1/2N)"

    LOCATION {
        bigint id PK
        text loc_type
    }
    RACK {
        bigint id PK
    }
    EQUIPMENT_TYPE {
        bigint id PK
        text model
    }
    EQUIPMENT {
        bigint id PK
        text serial
    }
    METRIC {
        smallint id PK
        text code
    }
    DATA_POINT {
        bigint id PK
        text role
    }
    SERIES {
        bigint series_id PK
    }
    MEASUREMENT {
        timestamptz time
    }
    CURRENT_VALUE {
        bigint series_id PK
    }
    CONNECTION {
        bigint id PK
        text medium
    }
    REDUNDANCY_GROUP {
        bigint id PK
    }
```

**読み方**：左から右へ「場所→ラック→機器」、機器から下に「点→時系列」。
`EQUIPMENT_TYPE` は型番（Genome）で、実機 `EQUIPMENT` を生む。
`METRIC` は「何を測るか」のカタログ。
配電・冷却は `CONNECTION` から導出し、冗長の意図は `REDUNDANCY_GROUP` が持つ。
時系列だけ TimescaleDB 側に置き、`series_id` だけを越境させる（ここに FK は張らない）。

---

## 2. 空間と資産（どこに・何が在るか）

```mermaid
erDiagram
    TENANT ||--o{ LOCATION : owns
    LOCATION ||--o{ LOCATION : "parent (木)"
    LOCATION ||--o{ RACK : contains
    RACK ||--o{ RACK_MOUNT : holds
    EQUIPMENT ||--o| RACK_MOUNT : "mounted (1機器=1搭載)"
    EQUIP_KIND ||--o{ EQUIPMENT_TYPE : classifies
    MANUFACTURER ||--o{ EQUIPMENT_TYPE : makes
    EQUIPMENT_TYPE ||--o{ EQUIPMENT : instantiates
    LOCATION ||--o{ EQUIPMENT : "installed"

    LOCATION {
        bigint id PK
        bigint parent_id FK
        text loc_type "room|cage|row|..."
        bigint tenant_id FK
    }
    RACK {
        bigint id PK
        int u_height
    }
    RACK_MOUNT {
        bigint id PK
        int position
        text face "front|rear"
    }
    EQUIP_KIND {
        smallint id PK
        text code "ups|pdu|crah|server"
        text power_class "it|cooling|facility"
    }
    EQUIPMENT_TYPE {
        bigint id PK
        smallint equip_kind_id FK
        text model
        int nameplate_power_w
    }
    EQUIPMENT {
        bigint id PK
        text serial UK
        bigint location_id FK
        bigint tenant_id FK
    }
```

**解説**：`LOCATION` は Region→…→Row までを1本の**隣接リスト木**で表す（配下集約は閉包テーブルで高速化・[03章 L1](./03-finalists.md)）。
`RACK` だけは固定アンカーで専用表にし、U の物理重なりは占有U行＋複合 UNIQUE で DB が禁止する。
`EQUIPMENT.location_id` は設置先の空間を直に指す。
ラック搭載機器はこれに加えて `RACK_MOUNT` を持ち、ラック ID・U 位置・面を表す。
つまり `location_id` は空間集約用、`rack_mount` はラック内の物理搭載詳細用。
**型番 `EQUIPMENT_TYPE`（EcoStruxure の Genome 相当）と実機 `EQUIPMENT` を分離**し、新機種はデータ追加だけで増やせる（DDL 不要）。
`EQUIP_KIND` は機器の種類（UPS/PDU/CRAH…）を分類し、`power_class` で pPUE の IT/施設を判別する。

---

## 3. 計測（何を・どう取り・どう貯めるか）

```mermaid
erDiagram
    METRIC ||--o{ DATA_POINT : "何を測るか"
    EQUIPMENT ||--o{ DATA_POINT : "どの機器の点か"
    DATA_POINT ||--o| SERIES : "raw 1:1"
    MEASUREMENT_SCOPE ||--o{ SERIES : "derived scope"
    SERIES ||--o{ MEASUREMENT : "履歴(his)"
    SERIES ||--o| CURRENT_VALUE : "最新値(cur)"
    SERIES ||--o{ DERIVED_MEASUREMENT : "派生(pPUE等)"

    METRIC {
        smallint id PK
        text code "active_power|rack_inlet_temp"
        text category "power|temperature|..."
        text unit "kW|°C"
        boolean is_derived
    }
    DATA_POINT {
        bigint id PK
        smallint metric_id FK
        text role "sensor|sp|cmd"
        text phase "L1|L2|L3|none"
        boolean is_writable
        text protocol "bacnet|modbus|snmp|..."
        text uri "gateway/endpoint"
        jsonb addr "OID/register/path"
    }
    SERIES {
        bigint series_id PK
        text series_kind "raw|derived"
        smallint metric_id "denorm"
    }
    MEASUREMENT_SCOPE {
        bigint id PK
        text scope_kind "energy_boundary|..."
    }
    MEASUREMENT {
        timestamptz time
        bigint series_id
        double value
    }
    CURRENT_VALUE {
        bigint series_id PK
        double value
        text alert_state
    }
```

**解説**：「何を測るか」は **`METRIC` という1つのフラットなカタログ**で表す。
量・単位・型を1行に置き、`rack_inlet_temp` のように位置もコードへ織り込む。
`DATA_POINT` は「その機器のその metric を、どの**役割**(sensor/設定値 sp/指令 cmd)で、どの相で取るか」を表す。
取得アドレスは `DATA_POINT.protocol` / `uri` / `addr` で OID や register へマップする。

役割を点の属性にしたことで、同じ量の「実測センサ／設定値／指令」が自然に共存できる。
時系列は履歴 `MEASUREMENT`(hypertable) と最新値 `CURRENT_VALUE` に分け、`SERIES` 台帳が両者と点を結ぶ。
pPUE 等の派生値も `is_derived` な metric として同じ枠で流す（[10章](./10-measurement-scope-derived-metrics.md)）。
ただし派生値は `DATA_POINT` ではなく、計測スコープに紐づく derived `SERIES` として持つ。

---

## 4. 配電・冷却・冗長（どう供給され、どう冗長か）

```mermaid
erDiagram
    EQUIPMENT ||--o{ CONNECTION : "from (上流)"
    EQUIPMENT ||--o{ CONNECTION : "to (下流)"
    EQUIPMENT ||--o{ EQUIPMENT : "panel→breaker"
    MEDIUM ||--o{ CONNECTION : medium
    CONNECTION ||--o| POWER_CONNECTION : "subtype (medium=elec)"
    REDUNDANCY_GROUP ||--o{ REDUNDANCY_MEMBER : members
    EQUIPMENT ||--o{ REDUNDANCY_MEMBER : belongs

    CONNECTION {
        bigint id PK
        bigint from_equipment_id FK
        bigint to_equipment_id FK
        smallint medium_id FK "elec|chilled_water|air"
        text redundancy "A|B|N+1|2N|none"
    }
    POWER_CONNECTION {
        bigint connection_id PK "FK(共有PK)"
        numeric voltage_v
        text phase
        numeric rated_a
        bigint available_power_w "生成列 VxAxutilx√3"
    }
    REDUNDANCY_GROUP {
        bigint id PK
        text domain "power|cooling|network"
        text topology "N+1|2N|..."
        int n_required
    }
    REDUNDANCY_MEMBER {
        bigint group_id PK
        bigint equipment_id PK
        text leg "A|B|C|none"
    }
```

**解説**：受電設備・変圧器・UPS・盤・breaker・PDU は**すべて `EQUIPMENT` ノード**として持つ。
breaker は専用表を作らず、`equip_kind='breaker'` の `EQUIPMENT` 行として、親盤・盤内位置・定格を持つ。
その間を **汎用 `CONNECTION`（`medium` 付き）＋ 電気サブタイプ `POWER_CONNECTION`（PK 共有）** で結ぶ。
これにより、受電→変圧器→UPS→分電盤→PDU→ラックのチェーン全体がグラフになる。

「どの機器がどの機器に給電/冷却するか」の多段トレースは、この `CONNECTION` グラフを辿る
**導出ビュー `v_equip_flow`**で得る。手で別グラフを持たない。
冷却も同じ `CONNECTION`（medium=chilled_water/air）で表す（将来 `cooling_connection` サブタイプ）。
**冗長は `REDUNDANCY_GROUP` ＋メンバーで「意図」を一級に持つ**：N+1/2N とメンバー（A系/B系の `leg`）を宣言し、
「2N の A/B が本当に独立電源か（SPOF 検出）」「N+1 で1台落ちても容量が足りるか」を、
**物理（v_equip_flow/容量）と突き合わせて検証**する。

---

## 5. 容量・監視・テナント（運用の制約）

```mermaid
erDiagram
    EQUIPMENT ||--o{ EQUIPMENT_DEMAND : "需要(銘板/契約/予約)"
    CAPACITY_BUDGET }o--|| RACK : "定格(排他アーク)"
    METRIC ||--o{ THRESHOLD : "閾値"
    TENANT ||--o{ SPACE_LEASE : "賃貸(コロ)"

    CAPACITY_BUDGET {
        bigint id PK
        text dimension "power_w|space_u|cooling_w"
        numeric rated
    }
    EQUIPMENT_DEMAND {
        bigint equipment_id PK
        text load_strategy "nameplate|predicted|contracted"
        numeric estimated
    }
    THRESHOLD {
        bigint id PK
        numeric warn_high
        numeric crit_high
        numeric hysteresis
    }
    SPACE_LEASE {
        bigint id PK
        text granularity "cage|cabinet|partial"
    }
```

**解説**：容量は WP-150 の5要素（空間/電力/電力分配/冷却/冷却分配）＋重量/ポートを、
`CAPACITY_BUDGET`(定格) と `EQUIPMENT_DEMAND`(需要) で持つ。
需要は推定負荷戦略（銘板/予測/コロ契約）で評価し、**stranded = 予約 − 実測**（死蔵容量）を出す。

監視は `THRESHOLD`（severity と hysteresis）。
テナント/コロは加算モジュールとして扱う（cage 境界・契約電力・賃貸 `SPACE_LEASE`・[08章](./08-tenancy-colocation.md)）。
`CAPACITY_BUDGET`/`THRESHOLD` のスコープは多態キーでなく**排他アーク FK**（対象ごとに NOT NULL FK＋「ちょうど1つ」）で参照整合を保つ。

---

## 主要な設計判断（なぜこの形か）

| 判断 | 採用 | 一言 |
|---|---|---|
| 点の定義 | **`metric` フラットカタログ** | 量・単位・型を1行。位置はコードに織り込む（〜100 行で爆発しない） |
| 役割 | **`data_point.role`** | 制御を見据え、sensor/sp/cmd を点の属性に |
| 接続 | **物理 connection＝真実源、フローは導出** | 並行する別グラフを持たない |
| 冗長 | **`redundancy_group` + member** | 意図(N+1/2N)を持ち、SPOF/容量で物理を検証 |
| スコープ参照 | **排他アーク FK** | 多態キー（type+id）をやめ参照整合を DB に残す（固定3対象） |
| 階層 | **閉包テーブル / 親参照ツリー** | `ltree`/拡張に依存しない（移植可・[09章](./09-portability.md)） |

## ドキュメントの歩き方

| 章 | 内容 |
|---|---|
| [01](./01-research-and-domain.md) | ドメイン知識＋参照モデル調査（EcoStruxure/Haystack/NetBox）＋ reify パターン |
| [03](./03-finalists.md) | **確定設計（L1〜L8 の ER＋全体俯瞰＋制約）**（必読） |
| [04](./04-validation-queries.md) | 代表ユースケースの検証 SQL |
| [08](./08-tenancy-colocation.md) | テナント/コロ拡張 |
| [09](./09-portability.md) | ポータビリティ（LCD・別エンジン移植） |
| [10](./10-measurement-scope-derived-metrics.md) | 計測スコープ & 派生メトリクス（pPUE） |
| [11](./11-standards-gap-analysis.md) | 標準・法規ギャップ分析 |

## レビューで特に見てほしい点

1. **`metric` カタログの粒度** — 位置/相をコードに織り込む方針で、横断クエリが `category` ＋命名で足りるか。
2. **A/B 系の持ち方** — 導出を真実源とし、多用時に `equipment.power_side` を非正規化する案で良いか。
3. **`redundancy_group` の検証** — 2N 独立性・N+1 容量充足をどこまで自動検証するか。
4. **排他アーク化の範囲** — threshold/capacity_budget 以外に多態キー（type+id）が残っていないか。カスタム属性層（旧 entity_tag）は不採用。
5. **制御（writable）の作り込み度** — 今は `is_writable` フラグで拡張点だけを用意している。優先配列・write 監査は本実装段階で決める。
