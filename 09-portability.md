# 09. ポータビリティ設計（エンジン非依存アーキテクチャ）

**要件**: 「PostgreSQL + TimescaleDB」を、例えば「MySQL + ClickHouse」のような**別スタックへ載せ替えても
大胆な設計変更（アーキテクチャの作り直し）が要らない**作りにする。
対象は [03章](./03-finalists.md) の確定設計 **Semantic-Typed DCIM**。

前提となる設計方針は次の通り。
オントロジは**有界な型付き小マスタ＋CHECK enum**で表し、汎用 `def` DAG は持たない。
点は `metric` フラットカタログに正規化し、metric 行が単位を直接保持する。
電力フローは `connection` / `power_connection` から `v_equip_flow` で導出する。

> **達成目標の線引き**：方言差は不可避なので「**ゼロ変更**」は狙わない。狙うのは
> 「**論理データモデル・テーブル構成・リレーショナル側のクエリは不変。差し替えは "方言DDL生成 /
> 時系列アダプタ実装 / 集約制約のサービス層" の3か所のみ**」という状態。

---

## 9.1 全体方針：「ANSI-SQL コア + エンジンアダプタ」

3 層に分け、エンジン固有性を**最外周のアダプタ**に閉じ込める。

```
┌─────────────────────────────────────────────┐
│ (c) エンジンアダプタ（差し替え対象・薄い）           │
│   ├ RDBMSアダプタ: 方言DDL生成 / UPSERT / JSON索引   │
│   └ 時系列アダプタ: hypertable⇄MergeTree, CAGG⇄MV   │
├─────────────────────────────────────────────┤
│ (b) 時系列DAL（5操作の抽象。論理ティア定義は共通）      │
├─────────────────────────────────────────────┤
│ (a) LCDリレーショナル・コア（不変）                  │
│   最小公倍数SQL: 標準の表/PK/FK/UNIQUE/CHECK/CTE/JSON │
└─────────────────────────────────────────────┘
```

### 防火壁の 5 原則（これがポータビリティの本体）

1. **拡張は `timescaledb` だけに限定**する。論理モデルは `ltree`/`btree_gist`/`pg_trgm` 等の PG 拡張に
   一切依存させない（TSDB アダプタを ClickHouse に差し替えれば拡張ゼロでも成立する形を保つ）。
2. **TSDB に FK を張らない**（hypertable も MergeTree も FK 非対応／非推奨。整合は `series` 台帳で）。
3. **TSDB とメタデータを JOIN しない**（ClickHouse は越境 JOIN が苦手）。**`series_id`（整数）だけ越境**させ、
   ラベル・階層・属性の解決は常に RDBMS 側で行う。`series` 台帳が「点 ↔ 時系列の同一性」を保持する唯一の境界。
4. **集約制約（予約≤定格・SPOF・容量超過・pPUE 算出・連結成分）はサービス層**で実施
   （deferrable/トリガの方言差を避ける）。`location_closure`・占有U行・`measurement_scope_member`、および採用するなら
   （任意の）`equip_kind_closure` の維持も同様にサービス層。
5. **PG 固有機能を論理モデルに使わない**（下記 LCD 代替）。

この 5 原則は確定設計 [03章](./03-finalists.md) が**既に満たすよう設計されている**（Narrow + `series` 台帳 +
メタ/TSDB 分離、階層は親参照＋（読み多なら）閉包、重なり禁止は占有行）。本章はその LCD 性を構文レベルで検証し、残差を潰す。

> **参照モデルの単純化がそのまま LCD の単純化**: 旧案の汎用 `def`/`def_super`/`def_closure`（Haystack `is` の多重継承 DAG を
> reify）を [03章](./03-finalists.md) は**廃止**し、オントロジを **`equip_kind` 単一親ツリー**（surrogate `smallint id` ＋
> `code` UNIQUE・self-FK・読み多なら任意で `equip_kind_closure`）と **`metric` フラットカタログ**（`code` UNIQUE・
> 単位を1行に保持）、固定集合の **CHECK enum** で表すよう改めた。
> 階層が要るのは実質 `equip_kind` だけで、これは**親参照ツリー（self-FK）**で持つ。後述するように、これは LCD ストーリーを
> **明確に簡単にする**（DAG の閉包維持＝`path_count`・サイクル検査というサービス層の塊が丸ごと消える・9.3）。

---

## 9.2 メタデータ RDBMS：PostgreSQL → MySQL 移植性

| 使いたい機能 | PG | MySQL 8.0+ | 採用する LCD 代替 |
|---|---|---|---|
| 表 / PK / FK / UNIQUE / NOT NULL / 基本 CHECK | ○ | ○(8.0.16+) | そのまま |
| 再帰 CTE（`connection` 上のフロートレース・SPOF / `equip_kind` ツリー集約） | ○ | ○(8.0+) | そのまま |
| JSON 列（`equipment_type.specs` / `data_point.addr`） | jsonb | JSON | `jsonb`→`JSON`（索引は生成列+functional index） |
| **階層 path / 木**（`location_closure` / 任意の `equip_kind_closure`） | ltree でも可 | ✕(ltree なし) | **親参照＋（読み多なら）閉包テーブル**（標準 JOIN・9.3） |
| **重なり禁止**（U 物理重なり） | EXCLUDE+btree_gist | ✕ | **占有U行 + 複合 UNIQUE**（拡張ゼロ） |
| **範囲型** | int4range/tstzrange | ✕ | **lower/upper の2列** + 比較。重なりは占有行で |
| **配列** | text[] | ✕ | **明細テーブル** or JSON 配列 |
| **部分索引 / 部分 UNIQUE**（`WHERE series_kind='derived'`） | ○ | **✕** | **生成列の NULL 一意 or 別表**（9.4・重要） |
| 列挙 | CHECK IN / ENUM型 | CHECK IN / ENUM | **CHECK IN は両対応**。増えうる語彙は**型付き小マスタ**（`equip_kind`/`metric`・[03章](./03-finalists.md)） |
| **正規表現 CHECK**（`code ~ '^[A-Za-z...]'`） | `~` | `REGEXP_LIKE` | **方言マップで吸収**（9.8） |
| 生成列（`power_connection.available_power_w` 等） | ○ | ○(STORED・5.7+) | **そのまま**（範囲型の生成列だけ避ければ可・9.5） |
| GIN(JSON) 索引 | ○ | ✕ | **生成列 + functional index**（両対応の形） |
| 自動採番（`GENERATED ALWAYS AS IDENTITY`） | IDENTITY | AUTO_INCREMENT | 方言マップ(9.8) |
| inet / macaddr 型 | ○ | ✕ | **VARCHAR + CHECK** |
| 部分一意（NULL扱い差） | ○ | 概ね○ | 占有行/別表で回避 |

**結論**: PG 固有の「ltree / EXCLUDE+btree_gist / range型 / 配列 / **部分索引**」の 5 つを使わなければ、
リレーショナル・コアは MySQL とほぼ同形になる。確定設計 [03章](./03-finalists.md) はこの LCD 形で書かれている。
ただし派生系列の部分 UNIQUE（[10章](./10-measurement-scope-derived-metrics.md)）だけは MySQL 非対応のため代替が要る（9.4）。

---

## 9.3 階層を親参照＋閉包テーブルで（`location_closure` / 任意の `equip_kind_closure`）

確定設計の階層は**隣接リスト（親参照 self-FK）が真実源**で、配下集約を高速化したい木だけ**任意で閉包テーブル**を
重ねる。閉包は PG 専用機能を使わず**標準 SQL の JOIN だけ**で配下集約でき、MySQL でも同形で動く。これが
「階層を `ltree` や再帰 CTE に依存させない」LCD 設計の核。

| 階層 | 表現する関係 | 形 | 閉包 |
|---|---|---|---|
| `location`（→`location_closure`） | 空間木（region→…→rack）の祖先-子孫 | tree（単一親・self-FK） | `location_closure`（`depth`・読み多用に既定で常設） |
| `equip_kind`（→任意の `equip_kind_closure`） | 機器種別の集約（「全 UPS / 全 hvac」） | **tree（単一親・self-FK）** | **任意**。読み多なら `equip_kind_closure`（`depth`のみ）。`location_closure` と完全同型 |

- **親参照（self-FK）は全エンジン自明に動く**: `equip_kind.parent_id → equip_kind.id`、`location.parent_id →
  location.id` は標準の自己参照外部キーで、PG/MySQL とも無条件にポータブル。木が浅く件数も小さい `equip_kind` は、
  「全 hvac 配下」を**素の再帰 CTE**（MySQL 8.0+ 可）で都度たどるだけでも十分実用になる。
- **`location` 隣接リスト＝真実源、`location_closure`＝配下集約キャッシュ**。「あるフロア配下の全ラック」は
  `JOIN location_closure ON ancestor_id = :floor`。`ltree` の `<@` 演算子も再帰 CTE も不要。
- **`equip_kind_closure` は同じ閉包パターンの"任意"適用**。読み負荷が高ければ `location_closure` と**完全に同型**の
  `(ancestor_id, descendant_id, depth)` を載せ、「全 UPS / 全 hvac を再帰なしで集約」を `JOIN equip_kind_closure
  ON ancestor_id = :hvac` の一発にできる。読み取りクエリは両エンジンでそのまま通る（PG 専用の `ltree` や
  `WITH RECURSIVE` に頼らない）。単一親の木なので閉包行は経路1本＝`path_count` は不要。
- **DAG（`path_count`）はもう既定構造ではない**: 旧案は Haystack `is` の多重継承（同一ノード対に複数経路があり得る
  DAG）を辺テーブル＋`path_count` 付き閉包で reify していたが、確定設計はこれを**廃止**した。多重継承
  （例: `rtu is [ahu, equip]`）の実需が**もし将来出た場合に限り**、当該木を DAG に拡張し、閉包行に経路数
  `path_count` を持たせる（辺追加で加算・削除で減算・0 で行削除）という、`location_closure` と同パターンの拡張で
  対応できる——という**逃げ道だけ**を残す。既定では使わない。
- 配下集約に `descendant`/`ancestor` 索引（`location_closure_desc` 等）を張る点も両エンジン共通。

> **対比**: PG なら `ltree` や `WITH RECURSIVE` で書けるが、前者は PG 専用拡張、後者も MySQL 8.0 未満では不可。
> 閉包テーブルは**標準の表 + JOIN だけ**なので、エンジンを問わず同じ読み取りクエリが通る。維持コスト
> （書き込み時の閉包更新）をサービス層に寄せる代わりに、読み取り側の方言ゼロを買っている。

> **これは LCD ストーリーの単純化である（重要）**: 汎用 `def` DAG エンジンを捨てたことで、サービス層から
> **DAG 閉包の差分維持（`path_count` の加減算）とサイクル禁止検査の一式が丸ごと消えた**。残るのは単一親の木の
> 親参照（self-FK・全エンジン自明）と、読み多なら同型の閉包（任意）だけ。**維持すべき不変条件が減る＝
> 移植時に検証すべきサービス層ロジックが減る**——ポータビリティと実装単純性の両取りである。

---

## 9.4 部分 UNIQUE 索引の LCD 代替（**重要**）

PG は**部分索引**（`CREATE UNIQUE INDEX ... WHERE 条件`）を持つが、**MySQL は持たない**。確定設計では
派生系列の重複防止に部分 UNIQUE が現れる（[10章](./10-measurement-scope-derived-metrics.md)）:

```sql
-- PG なら書ける（MySQL では不可）
CREATE UNIQUE INDEX ON series (measurement_scope_id, metric_id)
    WHERE series_kind = 'derived' AND scope_type = 'measurement_scope';
```

これは「**derived 系列は (計測スコープ × metric) で一意。raw 系列はこの一意制約の対象外**」を表す。
MySQL でも成立させる LCD 代替は次の 2 通り。

1. **生成列の NULL 一意キー（推奨）**: 「条件を満たす時だけ値を持ち、それ以外は NULL」になる STORED 生成列を作り、
   その列に通常の UNIQUE を張る。UNIQUE は **NULL を重複扱いしない**ので raw 行は素通りし、derived 行だけが
   一意化される。PG / MySQL 5.7+ の両方で動く。

   ```sql
   ALTER TABLE series ADD COLUMN derived_key text
       GENERATED ALWAYS AS (
           CASE WHEN series_kind = 'derived' AND scope_type = 'measurement_scope'
                THEN measurement_scope_id || ':' || metric_id END) STORED;
   CREATE UNIQUE INDEX series_derived_uk ON series (derived_key);   -- raw は NULL=対象外
   ```

2. **派生系列を別表に分離**: `derived_series` を切り出し、そこに素の `UNIQUE (measurement_scope_id, metric_id)` を張る。
   構造で部分性を表現（重なり禁止を占有行に落とすのと同じ発想）。`series_id` 空間は共有する。

どちらも**部分索引という PG 固有機能を論理モデルから排除**できる。`threshold` や `capacity_budget` の排他アーク
（9.5）で「特定 scope だけ一意」が要るときも同パターンで対応する。

---

## 9.5 LCD 確認（metric・排他アーク・v_equip_flow・生成列・その他）

確定設計が採用した構成は、いずれも**標準 SQL に収まり方言差を生まない**ことを個別に確認する。

- **metric が単位を直接保持（移植しやすい）**: `metric` 行が `unit` を1列で持ち、`data_point` は
  `metric_id` の単一 FK だけを参照する。旧来の「`point_type (quantity_id, unit_id) → quantity_unit` 複合 FK」は
  廃止済み（[03章 L3](./03-finalists.md)）。単位整合は **metric 行が単位を1つ保持**することで成立し、
  複合 FK の方言差・composite UNIQUE の維持も不要になる——標準の単一 FK で完結する。
- **排他アーク FK（`CHECK num_nonnull(...)=1`）**: `capacity_budget`（location/rack）と `threshold`
  （metric/equipment/location）はスコープを**排他アーク**で持つ（[03章 L7/L8](./03-finalists.md)）。
  `num_nonnull` は標準 SQL 関数で PG/MySQL 両対応——参照整合は標準 CHECK で完全に担保できる。
  ただし「排他アーク内の（scope_id × dimension）一意」など arc をまたぐ部分 UNIQUE が必要な場合は
  **PG の部分索引**が必要になり、MySQL では生成列 NULL 一意（9.4 同パターン）で代替する。
- **`v_equip_flow`（導出フロー）**: 電力フローは `connection` / `power_connection` からの**導出ビュー**
  であり並行グラフを持たない（[03章 L6](./03-finalists.md)）。PG では**マテリアライズドビュー**（`REFRESH MATERIALIZED VIEW`）、
  MySQL では通常の**ビュー**か**テーブル＋バッチ更新ジョブ**で代替する——論理モデルの変更は不要。
- **生成列（GENERATED ... STORED）**: `power_connection.available_power_w`（= `V × A × util × √3` を **bigint** にキャストして
  V×A のオーバーフローを回避）をはじめ、生成列は **PG と MySQL 5.7+ の双方がサポート**。STORED 指定で両対応。
  唯一の注意は「**範囲型の生成列**」だけ避けること（範囲型自体が MySQL に無い）。本設計は数値・真偽の生成列のみ
  なので問題なし。
- **`connection` / breaker 相当の `equipment` 行 / 電力フロートレース**: いずれも**普通の表 + FK + UNIQUE + CHECK** で構成され、
  PG 固有機能は使っていない。`connection` 上の上流トレースは**標準の再帰 CTE**（MySQL 8.0+ 可）。
  breaker の盤内位置は `equipment(parent_equipment_id, panel_position)` の標準複合 UNIQUE で担保する。
- **正規表現 CHECK**（`code ~ '^[A-Za-z][A-Za-z0-9_-]*$'` 等、`location.code`/`measurement_scope.code` に使用）: 演算子が
  **PG `~` ↔ MySQL `REGEXP_LIKE(...)`** と異なるだけで、制約の意味は同一。**方言マップ（9.8）で機械的に吸収**する。
- **自動採番**: `GENERATED ALWAYS AS IDENTITY`（`bigint` PK のエンティティ）は MySQL では `BIGINT AUTO_INCREMENT`、
  小マスタの `smallint id` は `SMALLINT AUTO_INCREMENT`。いずれも既に方言マップ（9.8）に含まれる定番置換であることを
  **確認**した。

---

## 9.6 時系列ストア：TimescaleDB → ClickHouse 移植性

時系列は **raw（`measurement`）/ rollup（`measurement_1m/1h/1d`）/
派生（`derived_measurement`・[10章](./10-measurement-scope-derived-metrics.md)）/ cur（`current_value`）** に分ける。
いずれも Narrow（long format）で、`series_id` だけが越境する（FK なし）。
機器別・メトリック別に物理テーブルを切る方式は採らない。
テーブル数、CAGG/MV、retention policy、DDL 生成が点数に比例して膨らみ、移植時のアダプタ差分も増えるため。

| 概念 | TimescaleDB | ClickHouse | 移植性 |
|---|---|---|---|
| 生テーブル `measurement`（raw） | hypertable(`create_hypertable`) + `(series_id, time DESC)` | `MergeTree ORDER BY (series_id, time)` + `PARTITION BY toYYYYMMDD(time)` | **概念一致**（Narrow が効く） |
| 派生テーブル `derived_measurement`（pPUE 等） | 別 hypertable（**retention 長期=5y**） | 別 `MergeTree`（独自 TTL） | **概念一致**。生と分離する理由は下記 |
| 最新値 `current_value`（raw/derived 共用） | 専用表 UPSERT | `ReplacingMergeTree` or 外部KV / RDBMS側 current 表 | 概念一致 |
| 時間バケット | `time_bucket` | `toStartOfInterval` / `toStartOfHour` | 概念一致（関数名差） |
| ロールアップ（1m→1h→1d） | continuous aggregate | Materialized View + `AggregatingMergeTree`/`SummingMergeTree` | **概念一致・DDLは別** |
| avg の段階集計 | sum_v+n（or stats_agg） | sum+count（or `avgState`/`avgMerge`） | **sum+count なら完全共通** |
| p95（任意） | `percentile_agg`+`rollup`(toolkit) | `quantileTDigestState`+`quantileTDigestMerge` | **両者ネイティブ**（素のPGのみ非対応→任意列） |
| 圧縮 | compression(`compress_segmentby=series_id`) | コーデック `CODEC(Delta, Gorilla, ZSTD)` | 概念一致・設定差 |
| 保持 | `add_retention_policy` | `TTL time + INTERVAL n DELETE` | 概念一致・設定差 |
| last/first | `last()`/`first()` | `argMax`/`anyLast` | 概念一致 |

> **生 `measurement` と `derived_measurement` を分ける理由**（[10章](./10-measurement-scope-derived-metrics.md)）: retention は
> テーブル単位なので「pPUE は 5 年・生テレメトリは 30 日」を同一表で両立できない。生の firehose（短期・高圧縮）と
> 計算 KPI（低頻度・長期）を別 hypertable / 別 MergeTree に分離する。`series_id` 空間は共有なので
> `current_value` は raw/derived を主キー 1 行参照で同じ枠組みで扱える。**どちらの hypertable にも FK は張らない**
> （防火壁原則 2）。

> **テーブルを切る単位**: 「機器ごと」「メトリックごと」ではなく、**raw / rollup / derived / current という用途・保持期間ごと**。
> raw は時間チャンクで append、`series_id` で並べる。高負荷環境だけ `series_id` hash partition を足す。
> 対象機器・メトリック・空間は RDBMS の `series` 台帳で先に `series_id` 集合へ解決し、TSDB では範囲スキャンに徹する。

**結論**: 時系列は「**論理ティア定義（raw/1m/1h/1d、各ティアの集計関数 {sum,count,min,max,(p95任意)}、保持期間、
生 vs 派生の別**）を共通仕様として宣言」し、**エンジン別 DDL はアダプタが生成**する。生テーブルが Narrow なので
生成は機械的。

---

## 9.7 時系列 DAL（5 操作の最小インターフェース）

アプリ／サービス層はこの 5 操作だけを呼ぶ。実装をアダプタで差し替えればエンジンが替わる。

```text
append(rows: [(time, series_id, value, quality)])          -- バルク投入（measurement / derived_measurement）
range_query(series_ids, from, to, grain)                   -- grain ∈ {raw,1m,1h,1d} を返す
upsert_latest(rows: [(series_id, time, value)])            -- current_value 更新
define_tier(name, source, bucket, aggs[], retention)       -- ロールアップ定義（DDL生成）
define_retention(table, keep)                              -- 保持（policy/TTL生成）
```

- `range_query` の戻りは常に `(bucket, series_id, sum_v, n, min_v, max_v[, p95])` の**正規化タプル**。
  呼び出し側は avg=`sum_v/n` を一様に扱う（エンジン非依存）。
- ラベル/階層解決は RDBMS 側で `series_id` 集合 → 属性 JOIN（`series` 台帳経由・越境しない）。

---

## 9.8 方言マップ（アダプタが吸収する差分）

| 項目 | PostgreSQL | MySQL | ClickHouse(時系列) |
|---|---|---|---|
| 自動採番 | `GENERATED ALWAYS AS IDENTITY` | `BIGINT/SMALLINT AUTO_INCREMENT` | （採番はRDBMS側のみ） |
| UPSERT | `INSERT ... ON CONFLICT DO UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` | `ReplacingMergeTree`(マージ時) |
| JSON 型 / 取得 | `jsonb` / `->>` | `JSON` / `->>`,`JSON_EXTRACT` | `String`+`JSONExtract` |
| JSON 索引 | GIN | 生成列+INDEX | （分析側は不要） |
| **正規表現 CHECK** | `col ~ '...'` | `REGEXP_LIKE(col, '...')` | — |
| **部分 UNIQUE** | `UNIQUE INDEX ... WHERE` | 生成列の NULL 一意（9.4） | — |
| 生成列 | `GENERATED ALWAYS AS (...) STORED` | `GENERATED ALWAYS AS (...) STORED` | — |
| boolean | `boolean` | `TINYINT(1)` | `UInt8` |
| 文字列前方一致索引 | `text_pattern_ops` | 接頭辞索引/`COLLATE` | — |
| 時刻関数 | `time_bucket` | `FROM_UNIXTIME`系 | `toStartOfInterval` |

> これらは**列定義・関数名の機械的置換**であり、テーブル構成やクエリの論理は変わらない＝「作り直し」ではない。

---

## 9.9 p95（分位点）の扱い

- 分位点はエンジン依存度が高い（素の PG は段階ロールアップ不可、Timescale/ClickHouse は可）ため、
  **「論理ティアの任意（optional）列」**とする。
- 既定（最小公倍数）：CAGG/MV には **持たない**。必要時に**生 or 1m へ `percentile_cont`（PG）/`quantile`（CH）を
  都度実行**（短期窓限定）。max/avg/min が主 KPI の DCIM では実用上十分。
- TSDB が対応する構成では、アダプタが sketch 列（`percentile_agg` / `quantileTDigestState`）を**追加**する。
  → 論理仕様（「p95 が欲しいティア」）は共通、実体はアダプタ任せ。

---

## 9.10 エンジンプロファイル（設定で宣言）

```sql
CREATE TABLE engine_profile (         -- デプロイ1つにつき1行
    singleton boolean PRIMARY KEY DEFAULT true CHECK (singleton),
    rdbms text NOT NULL CHECK (rdbms IN ('postgres','mysql')),
    tsdb  text NOT NULL CHECK (tsdb  IN ('timescaledb','clickhouse')),
    enable_percentiles boolean NOT NULL DEFAULT false   -- p95 ティア列の有無
);
```

アダプタは起動時にこの値を見て、DDL 生成・UPSERT 構文・正規表現/部分 UNIQUE/ロールアップ実装を切り替える。

---

## 9.11 何が「差し替え」で、何は「不変」か（線引き）

| 区分 | 例 | 載せ替え時 |
|---|---|---|
| **不変（作り直し不要）** | リレーショナル表・ER・FK・UNIQUE・CHECK、排他アーク FK、metric 単位内包、親参照ツリーと閉包 JOIN、`connection` の再帰 CTE トレース、生成列（`available_power_w`）、Narrow の論理形、`series` 台帳、5操作の呼び出し側 | そのまま |
| **アダプタ差し替え** | 方言DDL生成、UPSERT 構文、正規表現 CHECK 構文、部分 UNIQUE の表現（9.4）、`v_equip_flow` の matview⇄view/table+job、JSON 索引方式、ロールアップ DDL（CAGG⇄MV）、圧縮/保持（policy⇄TTL）、p95 実装 | アダプタ実装の置換 |
| **サービス層** | 集約制約（予約≤定格・SPOF・容量超過・ASHRAE 逸脱・pPUE 算出）、`location_closure`（および採用するなら任意の `equip_kind_closure`）・占有U行・`measurement_scope` 候補生成/維持 | DB 方言に依存しない実装 |

> 旧案にあった「`def_closure` の path-count 維持・サイクル禁止検証」はサービス層から**消滅**した（DAG を廃止したため）。
> 移植時に検証すべきサービス層ロジックがそのぶん減っている。

---

## 9.12 正直な限界（トレードオフ）

- **PG の旨味は捨てる**：`EXCLUDE` の宣言的・原子的保証、`ltree` の表現力、**部分索引**など。占有U行 UNIQUE で
  原子性は保てるが**行数は増える**（ラックは小さいので実害小）。閉包テーブル（`location_closure`/任意の
  `equip_kind_closure`）は**サブツリー移動・辺変更でエッジ再生成**が要る。
- **部分 UNIQUE を生成列で代替**：9.4 の NULL 一意キーは動くが、derived 判定ロジックが生成列定義に焼き込まれ、
  条件変更時に列定義の移行が要る（別表分離なら回避可・トレードオフ）。排他アーク上の arc またぎ部分 UNIQUE も
  同パターンの制約を受ける。
- **集約制約が DB から出る**：サービス層に移すと、DB 直叩きの経路では強制されない。重要制約は
  ストアドではなく**単一の書き込み口（サービス）に集約**して担保する設計運用が前提。
- **多重継承を将来入れるなら閉包の DAG 化が要る**：現状は `equip_kind` 単一親ツリーで `path_count` 不要だが、
  もし `rtu is [ahu, equip]` のような多重継承を有効化する日が来たら、当該閉包を DAG（`path_count`）に拡張し、
  その差分維持・サイクル検査をサービス層に戻すことになる（9.3）。それまでは**払わない**コスト。
- **分位点の一級扱いを諦める**：p95 は任意列。素の PG 構成では長期 p95 は近似/都度計算に劣化。
- **ClickHouse の更新セマンティクス**：`ReplacingMergeTree` の重複解決は非同期。最新値は **RDBMS 側 `current_value`**で
  持つ方が即時整合に強い（本設計の既定）。

> まとめ: 確定設計 [03章](./03-finalists.md) は「Narrow + `series` 台帳 + メタ/TSDB 分離 + 越境JOINなし + 拡張は
> timescaledb のみ」という**ポータブルな骨格**を持つ。本章の LCD 化（階層＝親参照＋任意の閉包
> `location_closure`/`equip_kind_closure`、占有U行、metric 単位内包（単一 FK で完結）・排他アーク FK
> （`CHECK num_nonnull(...)=1`・標準 SQL）・生成列の標準 SQL 化、
> 部分 UNIQUE の生成列代替、sum+count、サービス層集約制約）で、
> **PG+Timescale ⇄ MySQL+ClickHouse を アダプタ/方言/サービス層の差し替えだけ**で行き来できる。
> なお汎用 `def` DAG を外したことで DAG 閉包維持の複雑さがサービス層から消え、移植性の骨格はむしろ
> **より単純で堅牢**になった。派生メトリクス（`derived_measurement`）の追加（[10章](./10-measurement-scope-derived-metrics.md)）も
> この骨格を崩さない。
