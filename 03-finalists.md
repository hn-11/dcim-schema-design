# 03. 有力 5 案の詳細設計

[02-candidate-patterns.md](./02-candidate-patterns.md) で絞り込んだ 5 案の DDL スケッチ・制約・トレードオフ。
推奨案 **P05** を最も厚く記述し、他 4 案は **P05 との差分**として示す。

共通の前提（拡張は **timescaledb のみ**）:

```sql
-- ★ ポータビリティ方針（[09 章](./09-portability.md)）: メタデータは「最小公倍数SQL(LCD)」で書き、
--    PG 固有機能（ltree/btree_gist/EXCLUDE/range型/配列）に依存しない。時系列だけ TSDB に置く。
CREATE EXTENSION IF NOT EXISTS timescaledb;  -- hypertable / 連続集約 / 圧縮 / retention
-- ltree / btree_gist / timescaledb_toolkit / pg_jsonschema は使わない（下記の LCD 代替を採用）
--   階層 → 閉包テーブル / U重なり → 占有U行+UNIQUE / avg → sum+count / JSON検証 → アプリ層
```


---

## P05 — パッケージ標準（★推奨）

**設計思想**: 「Rack を固定アンカーで特別扱い、それ以外の空間は汎用 Location 木」「型番(DeviceType)に固有属性、
実機(Device)に個体属性」「配線は種別別ポート + Cable 実体」「テレメトリは Narrow + series 台帳」
「集約は非正規化キー + 階層ロールアップ」。可変性と制約を両立する。

> **ポータビリティ版（本章の既定）**: 拡張は timescaledb のみ。PG 固有機能を避け、別スタック
> （例: MySQL + ClickHouse）へ**アーキテクチャ作り直しなし**で載せ替えられる LCD（最小公倍数SQL）で記述する。
> 設計原則・方言マップ・時系列アダプタ境界は [09-portability.md](./09-portability.md) を参照。

### S5: 空間階層（ハイブリッド / 閉包テーブルで LCD 化）

`ltree` を使わず **閉包テーブル**で配下集約を表現（標準 SQL の JOIN のみ・MySQL でも同形）。
`location`（隣接リスト＝真実源）＋ `location_closure`（祖先-子孫の全ペア）。

```sql
-- 汎用 Location 木（隣接リスト＝真実源）
CREATE TABLE location (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- MySQL: BIGINT AUTO_INCREMENT
    parent_id   bigint REFERENCES location(id) ON DELETE RESTRICT,
    loc_type    text NOT NULL CHECK (loc_type IN
                  ('region','campus','building','floor','room','cage','zone','row')),
    name        text NOT NULL,
    code        text NOT NULL CHECK (code ~ '^[A-Za-z0-9_]+$'),   -- MySQL: REGEXP_LIKE
    tenant_id   bigint REFERENCES tenant(id),                     -- コロのテナント所有
    -- ルート(region/campus)以外は親必須
    CHECK ( loc_type IN ('region','campus') OR parent_id IS NOT NULL ),
    UNIQUE (parent_id, code)
);

-- 閉包テーブル: 祖先-子孫の全ペア（depth 付き）。配下/祖先クエリが再帰不要の JOIN で最速。
-- トリガ（or サービス層）で維持。拡張・range・配列なしで全エンジン移植可。
CREATE TABLE location_closure (
    ancestor_id   bigint NOT NULL REFERENCES location(id) ON DELETE CASCADE,
    descendant_id bigint NOT NULL REFERENCES location(id) ON DELETE CASCADE,
    depth         int    NOT NULL CHECK (depth >= 0),     -- 0 = 自分自身
    PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX location_closure_desc ON location_closure (descendant_id);
-- 配下抽出: JOIN location_closure lc ON lc.descendant_id = x.id WHERE lc.ancestor_id = :root

-- 許可される親子関係をデータで持つ（パッケージなので可変）
CREATE TABLE loc_type_rule (
    child_type  text NOT NULL,
    parent_type text,                       -- NULL = ルート可
    PRIMARY KEY (child_type, parent_type)
);
-- 例: ('building','campus'),('floor','building'),('room','floor'),('room','building'),
--     ('cage','room'),('row','room'),('row','cage'),('row','zone') ...
-- → location BEFORE トリガで (loc_type, 親の loc_type) を loc_type_rule で検証

-- Rack は固定レベル（専用制約を強制）
CREATE TABLE rack (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    location_id   bigint NOT NULL REFERENCES location(id) ON DELETE RESTRICT,
    name          text   NOT NULL,
    u_height      int    NOT NULL DEFAULT 42 CHECK (u_height BETWEEN 1 AND 100),
    starting_unit int    NOT NULL DEFAULT 1,
    desc_units    boolean NOT NULL DEFAULT false,
    width_in      int    CHECK (width_in IN (10,19,21,23)),
    outer_depth_mm int   CHECK (outer_depth_mm > 0),
    max_weight_kg numeric(8,2) CHECK (max_weight_kg > 0),
    rated_power_w int     CHECK (rated_power_w > 0),
    airflow       text    CHECK (airflow IN ('front-to-rear','rear-to-front','passive','mixed')),
    UNIQUE (location_id, name)
);
```

### 補助軸: U 位置の物理重なり禁止（占有U行 + UNIQUE で LCD 化）

`EXCLUDE`/`btree_gist`/`range型` を使わず、**占有を「U×面」ごとに1行展開**して **複合主キー（UNIQUE）**で
重なりを禁止する。原子的・並行挿入安全・拡張ゼロで、MySQL でも同形（フルデプス問題も自然に解ける）。

```sql
-- 搭載（1機器1行・属性）
CREATE TABLE rack_mount (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    rack_id       bigint NOT NULL REFERENCES rack(id)   ON DELETE RESTRICT,
    device_id     bigint NOT NULL REFERENCES device(id) ON DELETE CASCADE,
    position      int    NOT NULL CHECK (position >= 1),     -- 占有最下端 U
    u_height      int    NOT NULL CHECK (u_height >= 1),     -- device_type 由来
    face          text   NOT NULL DEFAULT 'front' CHECK (face IN ('front','rear')),
    is_full_depth boolean NOT NULL DEFAULT true,
    rack_side     text   NOT NULL DEFAULT 'full' CHECK (rack_side IN ('full','left','right')),
    UNIQUE (device_id)                                       -- 1機器=1搭載
);

-- 占有U（搭載時にトリガ/サービス層で position..position+u_height-1 を1行ずつ展開）
CREATE TABLE rack_unit_occupancy (
    rack_id   bigint NOT NULL REFERENCES rack(id),
    rack_side text   NOT NULL,
    face      text   NOT NULL CHECK (face IN ('front','rear')),
    u         int    NOT NULL,
    mount_id  bigint NOT NULL REFERENCES rack_mount(id) ON DELETE CASCADE,
    -- ★ これが「物理重なり禁止」: 同一ラック・同一サイド・同一面・同一Uは1機器のみ
    PRIMARY KEY (rack_id, rack_side, face, u)
);
-- フルデプス機器は front と rear の両方に占有行を入れる → 片面機器と必ず衝突。
-- front/rear の片面機器同士は face が違うので共存可。空きU検索は generate_series − occupancy の差集合。
-- position + u_height - 1 <= rack.u_height は親値参照のため CONSTRAINT TRIGGER で検証
```

### A5: 機器マスタ（DeviceType/Device + JSONB ハイブリッド）

```sql
CREATE TABLE manufacturer (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL UNIQUE
);
CREATE TABLE device_category (
    code  text PRIMARY KEY CHECK (code ~ '^[a-z_]+$'),  -- 'server','ups','pdu','switch','crac','sensor'...
    label text NOT NULL
);

-- 型番マスタ: 固有属性のテンプレート（コア数値は型付き列、ロングテールは JSONB）
CREATE TABLE device_type (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    manufacturer_id bigint NOT NULL REFERENCES manufacturer(id) ON DELETE RESTRICT,
    category        text   NOT NULL REFERENCES device_category(code),
    model           text   NOT NULL,
    u_height        int    NOT NULL DEFAULT 1 CHECK (u_height BETWEEN 1 AND 60),
    is_full_depth   boolean NOT NULL DEFAULT true,
    weight_kg       numeric(8,2) CHECK (weight_kg > 0),
    rated_power_w   int          CHECK (rated_power_w   > 0),  -- コア数値=型付き
    rated_current_a numeric(6,2) CHECK (rated_current_a > 0),
    ashrae_class    text CHECK (ashrae_class IN ('A1','A2','A3','A4')),  -- 温湿度検証用
    specs           jsonb NOT NULL DEFAULT '{}'::jsonb,        -- ロングテール固有属性
    UNIQUE (manufacturer_id, model)
    -- pg_jsonschema 利用時: CHECK (jsonb_matches_schema(category_schema(category), specs))
);
CREATE INDEX device_type_specs_gin ON device_type USING gin (specs);

-- ポート構成テンプレート（型番側 → 実機作成時に展開: NetBox 流 instantiate）
CREATE TABLE interface_template (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    device_type_id bigint NOT NULL REFERENCES device_type(id) ON DELETE CASCADE,
    name text NOT NULL, if_type text NOT NULL,
    UNIQUE (device_type_id, name)
);
-- power_port_template / power_outlet_template / console_port_template も同型

-- 実機（共通属性のみ）
CREATE TABLE device (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    device_type_id  bigint NOT NULL REFERENCES device_type(id) ON DELETE RESTRICT,
    asset_tag       text UNIQUE,
    serial          text UNIQUE,
    status          text NOT NULL DEFAULT 'active'
        CHECK (status IN ('planned','staged','active','offline','decommissioned')),
    location_id     bigint REFERENCES location(id) ON DELETE RESTRICT,
    tenant_id       bigint REFERENCES tenant(id),
    contract_id     bigint REFERENCES maintenance_contract(id) ON DELETE SET NULL,
    installed_on    date
);
```

### C3: 配線（種別別ポート + Cable/CableTermination）

```sql
CREATE TYPE redundancy_t AS ENUM ('A','B');
CREATE TYPE termination_kind_t AS ENUM
    ('power_port','power_outlet','power_feed','interface','front_port','rear_port','console');

-- 電力入力（PSU 側）。dual-corded は複数行
CREATE TABLE power_port (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    device_id bigint NOT NULL REFERENCES device(id) ON DELETE CASCADE,
    name text NOT NULL,
    redundancy redundancy_t,                          -- A系/B系想定
    rated_current_a numeric(8,2) CHECK (rated_current_a > 0),
    voltage_v numeric(7,1),
    UNIQUE (device_id, name)
);
-- power_outlet (upstream_power_port_id で機内分配/ATS アクティブ入力), interface,
-- front_port/rear_port（パッチパネル透過）も 01/配線調査の DDL に準拠

CREATE TABLE power_panel (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    location_id bigint NOT NULL REFERENCES location(id), name text NOT NULL,
    UNIQUE (location_id, name)
);
CREATE TABLE power_feed (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    power_panel_id bigint NOT NULL REFERENCES power_panel(id) ON DELETE CASCADE,
    rack_id    bigint REFERENCES rack(id),
    name       text NOT NULL,
    type       text NOT NULL DEFAULT 'primary' CHECK (type IN ('primary','redundant')),
    redundancy redundancy_t,
    supply     text NOT NULL DEFAULT 'ac' CHECK (supply IN ('ac','dc')),
    phase      text NOT NULL DEFAULT 'single' CHECK (phase IN ('single','three')),
    voltage_v  numeric(7,1) NOT NULL,
    amperage_a numeric(8,2) NOT NULL CHECK (amperage_a > 0),
    max_utilization int NOT NULL DEFAULT 80 CHECK (max_utilization BETWEEN 1 AND 100),
    -- NEC 80% + 三相 √3 を反映した供給可能電力(W)。int4 では V×A overflow しうるので bigint 派生
    available_power_w bigint GENERATED ALWAYS AS (
        round(voltage_v * amperage_a * max_utilization / 100.0
              * CASE WHEN phase='three' THEN 1.732 ELSE 1 END)::bigint) STORED,
    UNIQUE (power_panel_id, name)
);

-- ケーブル実体 + 両端 termination（NetBox 流）
CREATE TABLE cable (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    label text, cable_type text, color text, length_m numeric(6,2),
    status text NOT NULL DEFAULT 'connected'
        CHECK (status IN ('connected','planned','decommissioning'))
);
CREATE TABLE cable_termination (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cable_id bigint NOT NULL REFERENCES cable(id) ON DELETE CASCADE,
    side char(1) NOT NULL CHECK (side IN ('A','B')),
    term_kind termination_kind_t NOT NULL,
    term_id   bigint NOT NULL,
    -- ★ ポートは高々1ケーブルに終端（片側1接続）
    UNIQUE (term_kind, term_id)
);
-- 多態 FK(term_kind/term_id) は実 FK が張れない → 整合は検証トリガ + 定期チェック、
-- または kind ごとにテーブル分割 + UNION ビューで実 FK 化（厳格版オプション）
-- 許可ペア(power_outlet→power_port 等)・自己ループ禁止もトリガで担保
```

電力グラフ／ネットグラフのトレースは正規化エッジビュー `power_arc` 上の `WITH RECURSIVE`
（[04 章](./04-validation-queries.md)に SQL）。end-to-end は `CablePath` 相当のキャッシュ表に固める。

### T1: テレメトリ（Narrow + series 台帳）

```sql
-- 物理量カタログ（パッケージ同梱・全 DC 共通の固定コード表）
CREATE TABLE metric_definition (
    metric_def_id smallint PRIMARY KEY,
    code text NOT NULL UNIQUE,           -- 'active_power','current','voltage','inlet_temp','humidity'
    unit text NOT NULL,                  -- 'W','A','V','degC','%RH'
    value_kind text NOT NULL CHECK (value_kind IN ('gauge','counter','status','enum')),
    default_agg text NOT NULL DEFAULT 'avg'
        CHECK (default_agg IN ('avg','min','max','sum','last','count'))
);

-- データポイント（機器のどの点をどのプロトコルで取るか。プロトコル固有アドレスは jsonb）
CREATE TABLE data_point (
    data_point_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    device_id     bigint NOT NULL REFERENCES device(id) ON DELETE CASCADE,
    metric_def_id smallint NOT NULL REFERENCES metric_definition(metric_def_id),
    protocol      text NOT NULL CHECK (protocol IN ('snmp','modbus','ipmi','redfish','bacnet','api')),
    addr          jsonb NOT NULL,        -- {"oid":...} / {"reg":...,"fc":3} / {"path":...}
    scale         double precision NOT NULL DEFAULT 1.0,
    offset_       double precision NOT NULL DEFAULT 0.0,
    phase         text,
    enabled       boolean NOT NULL DEFAULT true,
    UNIQUE (device_id, metric_def_id, phase),
    CONSTRAINT addr_shape CHECK (
        (protocol='snmp'   AND addr ? 'oid') OR
        (protocol='modbus' AND addr ? 'reg' AND addr ? 'fc') OR
        (protocol='redfish' AND addr ? 'path') OR
        (protocol IN ('ipmi','bacnet','api')) )
);

-- series 台帳（data_point と 1:1。series_id が TSDB 側の整数キー）
CREATE TABLE series (
    series_id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    data_point_id bigint NOT NULL REFERENCES data_point(data_point_id) ON DELETE CASCADE,
    device_id     bigint NOT NULL REFERENCES device(id),     -- 非正規化（クエリ高速化）
    metric_def_id smallint NOT NULL REFERENCES metric_definition(metric_def_id),
    rack_id       bigint,                                    -- ★ 非正規化（空間集約・ingest時点固定）
    location_id   bigint,                                    -- ★ 非正規化（閉包JOINで階層集約。ltree廃止）
    retired_at    timestamptz,
    UNIQUE (data_point_id)
);
CREATE INDEX series_rack ON series (rack_id, metric_def_id);
CREATE INDEX series_loc  ON series (location_id, metric_def_id);
-- 配下集約: JOIN location_closure lc ON lc.descendant_id = series.location_id WHERE lc.ancestor_id = :root

-- 時系列本体（Narrow hypertable, FK は張らない）
CREATE TABLE measurement (
    time      timestamptz NOT NULL,
    series_id bigint NOT NULL,
    value     double precision,
    quality   smallint NOT NULL DEFAULT 0   -- 0=good,1=uncertain,2=bad
);
SELECT create_hypertable('measurement', by_range('time', INTERVAL '1 day'));
CREATE INDEX ON measurement (series_id, time DESC);

ALTER TABLE measurement SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'series_id',
    timescaledb.compress_orderby   = 'time DESC');
SELECT add_compression_policy('measurement', compress_after => INTERVAL '7 days');
SELECT add_retention_policy('measurement', drop_after => INTERVAL '30 days');  -- 生は短期
```

### G3: 集約（非正規化キー + 階層 CAGG + 最新値表）

```sql
-- 時間ロールアップ: 1m → 1h → 1d（cagg-on-cagg）。
-- ★ toolkit 非依存: avg は sum_v/n で厳密ロールアップ。min/max/sum/count は再集約安全。
--   p95 等の分位点は「論理ティアの任意列」扱い（[09 章](./09-portability.md)）。
--   素の PG では非対応のため既定では持たず、必要時に生/1m へ percentile_cont を都度実行。
--   TSDB が対応する場合のみ sketch 列を追加（Timescale=percentile_agg、ClickHouse=quantileTDigestState）。
CREATE MATERIALIZED VIEW measurement_1m
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 minute', time) AS bucket, series_id,
       sum(value) AS sum_v, count(*) AS n,         -- avg = sum_v/n（段階ロールアップで厳密）
       max(value) AS max_v, min(value) AS min_v
FROM measurement WHERE value IS NOT NULL
GROUP BY 1, 2
WITH NO DATA;

CREATE MATERIALIZED VIEW measurement_1h
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', bucket) AS bucket, series_id,
       sum(sum_v) AS sum_v, sum(n) AS n,           -- avg = sum(sum_v)/sum(n)
       max(max_v) AS max_v, min(min_v) AS min_v
FROM measurement_1m GROUP BY 1, 2 WITH NO DATA;
-- measurement_1d も同型（time_bucket('1 day', bucket)）。読み出しは avg = sum_v::float8 / n。

SELECT add_continuous_aggregate_policy('measurement_1m',
  start_offset => INTERVAL '3 hours', end_offset => INTERVAL '1 minute',
  schedule_interval => INTERVAL '1 minute');
-- 1h/1d も同様（下段確定後に上段を回す）。集約は長期 retention:
SELECT add_retention_policy('measurement_1h', drop_after => INTERVAL '2 years');
SELECT add_retention_policy('measurement_1d', drop_after => INTERVAL '10 years');

-- 最新値（専用 current 表 UPSERT + alert_state 同居）
CREATE TABLE current_value (
    series_id bigint PRIMARY KEY,
    time timestamptz NOT NULL,
    value double precision NOT NULL,
    alert_state text NOT NULL DEFAULT 'normal'
        CHECK (alert_state IN ('normal','warning','critical','stale'))
);
-- ingest 時: INSERT ... ON CONFLICT (series_id) DO UPDATE ... WHERE EXCLUDED.time > current_value.time

-- キャパシティ（次元別 定格 vs 予約 vs 実測。stranded = reserved - actual）
CREATE TABLE capacity_budget (
    location_or_rack_id bigint NOT NULL,
    scope text NOT NULL CHECK (scope IN ('location','rack')),
    dimension text NOT NULL CHECK (dimension IN
        ('power_w','space_u','cooling_w','floor_load_kg','port')),
    rated numeric NOT NULL CHECK (rated > 0),
    soft_limit numeric CHECK (soft_limit IS NULL OR soft_limit <= rated),  -- 例 rated×0.8
    PRIMARY KEY (location_or_rack_id, scope, dimension)
);
-- 「予約合計 ≤ rated」は集約制約 → CONSTRAINT TRIGGER か監視ビューで担保

-- 機器ごとの需要（銘板 nameplate と 予約 reserved）。UC-7 で実測 actual と対比
CREATE TABLE device_demand (
    device_id bigint NOT NULL REFERENCES device(id) ON DELETE CASCADE,
    dimension text NOT NULL CHECK (dimension IN
        ('power_w','space_u','cooling_w','floor_load_kg','port')),
    nameplate numeric NOT NULL CHECK (nameplate >= 0),   -- 銘板値
    reserved  numeric NOT NULL CHECK (reserved  >= 0),   -- 設計確保値（=予約）
    PRIMARY KEY (device_id, dimension)
);

-- 閾値（global < location < device の優先で評価。ホットスポット検出）
CREATE TABLE threshold (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    scope_type text NOT NULL CHECK (scope_type IN ('global','location','device')),
    scope_id bigint,
    metric_def_id smallint NOT NULL REFERENCES metric_definition(metric_def_id),
    warn_high numeric, crit_high numeric, warn_low numeric, crit_low numeric,
    hysteresis numeric DEFAULT 0,
    CHECK (crit_high IS NULL OR warn_high IS NULL OR crit_high >= warn_high)
);
```

### P05 の効くドメイン制約（まとめ）

| 制約 | 実装 |
|------|------|
| U 範囲の物理重なり禁止 | ★ `rack_unit_occupancy` の複合主キー（占有U行・拡張ゼロ・全エンジン移植可） |
| ラックに収まる | ▲ CONSTRAINT TRIGGER / サービス層（`rack.u_height` 突合） |
| ポート片側1接続 | ★ `cable_termination` の UNIQUE（厳格版は種別別+UNION、[06 章](./06-self-review.md)） |
| 型番一意 / 個体一意 | ★ `(manufacturer_id,model)` / `serial`,`asset_tag` UNIQUE |
| 供給可能電力 = V×A×util×√3 | ★ 生成列（bigint で overflow 回避。VA/W は [06 章 A-6](./06-self-review.md)） |
| 階層の親子関係 | ▲ `loc_type_rule` + トリガ／サービス層 |
| 予約 ≤ 定格 / SPOF / 温湿度 ASHRAE | ▲ 集約制約はサービス層（移植性のため DB トリガに依存させない・[09 章](./09-portability.md)） |

### P05 のチューニングオプション（高得点変種を吸収）

- **P16 化**: `measurement` に `rack_id`/`location_id` を**直接埋め込み**、`series` JOIN なしで空間集約ロールアップを作る（読み最速・ingest 時点固定で履歴正確）。高頻度ダッシュボードで採用。
- **P15 化**: `cable`/`cable_termination` を持たず**軽量有向エッジ（両端1行）**にすると、自己ループ・向きを `CHECK` で表現でき制約が単純化。ケーブル物理属性（長さ・色・ラベル）が不要な小規模構成向け。
- **P17 化**: 空間階層を `ltree` でなく**閉包テーブル**にすると階層 JOIN が再帰不要で最速（P06 参照）。

---

## P08 — 時間軸正確 Temporal（P05 との差分）

**差分**: 機器の搭載位置を**履歴付き（Type-2）**にする。`rack_mount` を時系列化し、
テレメトリ集約を「サンプル時刻に有効だったマッピング」で行う → 機器移設をまたいでも過去のラック別電力が正確。

P08 は範囲×期間の排他が本質なので、`tstzrange`+`EXCLUDE`(+`btree_gist`) が最も簡潔。
**PG 専用で良い構成**ならこのまま、**LCD（MySQL 移植）したい場合**は `valid_from/valid_to` の2列 +
「現行有効な占有U行（複合UNIQUE）」+ 期間重なり判定をサービス層、に置換する（[09 章](./09-portability.md)）。

```sql
-- 搭載位置を有効期間付きで保持（Type-2 SCD・PG版。LCD版は2列+占有行+サービス層）
CREATE TABLE device_placement (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    device_id bigint NOT NULL REFERENCES device(id) ON DELETE CASCADE,
    rack_id   bigint NOT NULL REFERENCES rack(id),
    position  int NOT NULL, u_height int NOT NULL, face text NOT NULL,
    valid     tstzrange NOT NULL,
    -- ★ 同一機器の有効期間が重ならない
    EXCLUDE USING gist (device_id WITH =, valid WITH &&),
    -- ★ 同一ラック・同一面で U 範囲×期間が重ならない（時間込み排他）
    u_range int4range GENERATED ALWAYS AS (int4range(position, position+u_height,'[)')) STORED,
    EXCLUDE USING gist (rack_id WITH =, face WITH =, u_range WITH &&, valid WITH &&)
);
```

集約は CAGG にできない（範囲 JOIN が必要）ため、空間集約は**オンザフライ範囲 JOIN（G5）**または
**analytics-batch でのマテリアライズ（G4）**で実装:

```sql
SELECT time_bucket('1 hour', m.time) AS bucket, p.rack_id, sum(m.value)
FROM measurement m
JOIN series s        ON s.series_id = m.series_id
JOIN device_placement p
     ON p.device_id = s.device_id
    AND m.time >= lower(p.valid) AND m.time < upper(p.valid)   -- サンプル時点のマッピング
WHERE s.metric_def_id = (SELECT metric_def_id FROM metric_definition WHERE code='active_power')
GROUP BY bucket, p.rack_id;
```

- **強み**: 真の point-in-time。機器移設時にその時刻で寄与が旧→新ラックへ正しく分割。監査可能。
- **弱み**: CAGG が使えず JOIN が重い。電力課金・エネルギー会計を主目的とする DC で採用。
- それ以外（空間 S5・機器 A5・配線 C3・テレメトリ T1）は P05 と同一。

---

## P06 — 閉包テーブル読み最適化（P05 との差分）

**差分**: 空間階層を `ltree`（S2）でなく**閉包テーブル（S3）**に。階層集約 JOIN が再帰不要で最速。

```sql
CREATE TABLE location (   -- 隣接リスト（真実源）は P05 と同じ
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    parent_id bigint REFERENCES location(id) ON DELETE RESTRICT,
    loc_type text NOT NULL, name text NOT NULL, ...
);
-- 祖先-子孫の全ペア（depth 付き）。トリガで維持
CREATE TABLE location_closure (
    ancestor_id   bigint NOT NULL REFERENCES location(id),
    descendant_id bigint NOT NULL REFERENCES location(id),
    depth         int NOT NULL CHECK (depth >= 0),
    PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX ON location_closure (descendant_id);
```

- **強み**: 「ある building 配下の全ラック電力」が `JOIN location_closure ON ancestor_id = :building` の
  単純 JOIN で済む（再帰 CTE 不要・最速）。読み支配・多数同時ダッシュボードに最適。機器 A3、配線 C3、
  テレメトリ T1、集約 G3 は P05 と同じ。
- **弱み**: サブツリー移動で子孫×祖先のエッジ行を再生成（書込増幅）。行数膨張。
- **P17 オプション**: 機器マスタを A5（JSONB ハイブリッド）にすれば可変性も両立（合計 27 の変種）。

---

## P04 — NetBox 忠実・強整合（P05 との差分）

**差分**: 機器固有属性を JSONB でなく**カテゴリ別 CTI（A4）**で型付け。制約最強・OSS 実績・
`devicetype-library` 流用可。

```sql
-- 型番マスタ（共通テンプレート）は P05 と同じ device_type（ただし specs jsonb は使わない）
-- カテゴリごとに子テーブルで固有属性を型付け
CREATE TABLE device_type_server (
    device_type_id bigint PRIMARY KEY REFERENCES device_type(id) ON DELETE CASCADE,
    cpu_sockets int CHECK (cpu_sockets > 0),
    max_ram_gb  int CHECK (max_ram_gb > 0),
    psu_count   int NOT NULL CHECK (psu_count >= 1),
    airflow     text CHECK (airflow IN ('front-to-rear','rear-to-front'))
);
CREATE TABLE device_type_ups (
    device_type_id bigint PRIMARY KEY REFERENCES device_type(id) ON DELETE CASCADE,
    capacity_kva numeric(8,2) NOT NULL CHECK (capacity_kva > 0),
    capacity_kw  numeric(8,2) NOT NULL CHECK (capacity_kw  > 0),
    topology text CHECK (topology IN ('online','line-interactive','standby')),
    runtime_min int CHECK (runtime_min > 0),
    CHECK (capacity_kw <= capacity_kva)                  -- 力率 ≤ 1
);
-- device_type_pdu / device_type_crac ... カテゴリ追加 = 新テーブル + デプロイ
-- category と子テーブルの一致は CONSTRAINT TRIGGER で担保
```

- **強み**: 固有属性が型付き列で `NOT NULL`/`CHECK`/`FK` フル。制約満点（D=5）。集計・検索が高速。
- **弱み**: 新カテゴリ追加に DDL + デプロイが要る（可変性は P05 に劣る F=4）。
- 空間 S5・配線 C3・テレメトリ T1・集約 G3 は P05 と同じ。**機器種別が安定した大規模単一運用者**に最適。

---

## P02 — JSONB 軽量 MVP（P05 との差分）

**差分**: 機器を**単一テーブル + JSONB（A1）**、配線を**電力/ネット分離の最小形（C2）**、
集約を**オンザフライ（G1）**に簡素化。テーブル数最小・実装最速・移行容易。

```sql
-- 空間（Rack も location に統合、rack 固有属性は JSONB or 専用列最小）
-- LCD 版は path ltree を使わず closure or path text+前方一致（[09 章](./09-portability.md)）
CREATE TABLE location (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    parent_id bigint REFERENCES location(id) ON DELETE RESTRICT,
    loc_type text NOT NULL, name text NOT NULL, path text,   -- 'a.b.c' 前方一致 or 閉包表
    attrs jsonb NOT NULL DEFAULT '{}'        -- u_height 等はここ or 専用列
);

-- 機器は単一テーブル + JSONB（型番概念も任意）
CREATE TABLE device (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category text NOT NULL,
    manufacturer text, model text, serial text UNIQUE,
    location_id bigint REFERENCES location(id),
    rack_position int, u_height int,
    specs jsonb NOT NULL DEFAULT '{}'         -- 固有属性すべて
        -- CHECK (jsonb_matches_schema(category_schema(category), specs))  -- pg_jsonschema
);
CREATE INDEX device_specs_gin ON device USING gin (specs);

-- テレメトリは P05 と同じ Narrow（T1）— ここは妥協しない（TS 整合の要）
-- 配線は電力/ネットの最小接続テーブル（C2）
CREATE TABLE power_connection (
    outlet_device_id bigint NOT NULL REFERENCES device(id),
    outlet_name text NOT NULL,
    psu_device_id bigint NOT NULL REFERENCES device(id),
    psu_name text NOT NULL,
    UNIQUE (psu_device_id, psu_name),         -- PSU は 1 接続
    CHECK (outlet_device_id <> psu_device_id) -- 自己ループ禁止
);
```

- **強み**: 可変性満点（F=5）。テーブル・トリガ最小で実装/運用が軽い（C=5）。後から P05 へ段階移行しやすい。
- **弱み**: U 重なり・誤配線・冗長を DB で弾けない（D=2、アプリ層頼み）。階層集約はオンザフライで規模が出ると重い。
- **小規模 DC / PoC / 単一テナント / 短期立ち上げ**向け。成長したら P05 へ。

---

## 5 案の使い分け早見表

| 要件 | 推奨案 |
|------|--------|
| 標準・多数 DC へ横展開（デフォルト） | **P05** |
| 電力課金・エネルギー会計の履歴正確性が最重要 | **P08**（または P05 + G5 オプション） |
| 深い階層・読み支配・多数同時ダッシュボード | **P06**（または P05 + P17 オプション） |
| 機器種別が安定・整合性最優先・大規模単一運用 | **P04** |
| 小規模 / PoC / 短期立ち上げ / 単一テナント | **P02**（成長後 P05 へ移行） |

検証クエリは [04-validation-queries.md](./04-validation-queries.md)。
