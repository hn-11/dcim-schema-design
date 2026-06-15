# 09. ポータビリティ設計（エンジン非依存アーキテクチャ）

**要件**: 「PostgreSQL + TimescaleDB」を、例えば「MySQL + ClickHouse」のような**別スタックへ載せ替えても
大胆な設計変更（アーキテクチャの作り直し）が要らない**作りにする。

> **達成目標の線引き**：方言差は不可避なので「**ゼロ変更**」は狙わない。狙うのは
> 「**論理データモデル・テーブル構成・リレーショナル側のクエリは不変。差し替えは “方言DDL生成 /
> 時系列アダプタ実装 / 集約制約のサービス層” の3か所のみ**」という状態。

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

### 防火壁の 4 原則（これがポータビリティの本体）

1. **TSDB に FK を張らない**（hypertable も MergeTree も FK 非対応／非推奨。整合は series 台帳で）。
2. **TSDB とメタデータを JOIN しない**（ClickHouse は越境 JOIN が苦手）。**series_id（整数）だけ越境**させ、
   ラベル・階層・属性の解決は常に RDBMS 側で行う。
3. **集約制約（予約≤定格・SPOF・容量超過）はサービス層**で実施（deferrable/トリガの方言差を避ける）。
4. **PG 固有機能を論理モデルに使わない**（下記 LCD 代替）。

この 4 原則は推奨案 P05 が**既に満たしている**（Narrow + series 台帳 + メタ/TSDB分離）。本章は残差を潰す。

---

## 9.2 メタデータ RDBMS：PostgreSQL → MySQL 移植性

| 使いたい機能 | PG | MySQL 8.0+ | 採用する LCD 代替 |
|---|---|---|---|
| 表 / PK / FK / UNIQUE / NOT NULL / 基本 CHECK | ○ | ○(8.0.16+) | そのまま |
| 再帰 CTE（電源トレース等） | ○ | ○(8.0+) | そのまま |
| JSON 列 | jsonb | JSON | `jsonb`→`JSON`（索引は生成列+functional index） |
| **階層 path** | ltree | ✕ | **閉包テーブル `location_closure`**（標準 JOIN） |
| **重なり禁止** | EXCLUDE+btree_gist | ✕ | **占有行 + 複合 UNIQUE**（`rack_unit_occupancy`） |
| **範囲型** | int4range/tstzrange | ✕ | **lower/upper の2列** + 比較。重なりは占有行で |
| **配列** | text[] | ✕ | **明細テーブル** or JSON 配列 |
| 部分索引 / `EXCLUDE WHERE` | ○ | ✕ | 構造で回避（占有行・別表） |
| 列挙 | CHECK IN / ENUM型 | CHECK IN / ENUM | **CHECK IN は両対応**。増えうる語彙は**ルックアップ表**（[06章B-1](./06-self-review.md)） |
| 生成列 | ○ | ○(STORED) | 範囲型の生成列だけ避ければ可 |
| GIN(JSON) 索引 | ○ | ✕ | **生成列 + functional index**（両対応の形） |
| 自動採番 | IDENTITY | AUTO_INCREMENT | 方言マップ(9.5) |
| inet / macaddr 型 | ○ | ✕ | **VARCHAR + CHECK** |
| 部分一意（NULL扱い差） | ○ | 概ね○ | 占有行/別表で回避 |

**結論**: PG 固有の「ltree / EXCLUDE+btree_gist / range型 / 配列 / 部分索引」の 5 つを使わなければ、
リレーショナル・コアは MySQL とほぼ同形になる。本リポジトリの [03章](./03-finalists.md) はこの LCD 形に更新済み。

---

## 9.3 時系列ストア：TimescaleDB → ClickHouse 移植性

| 概念 | TimescaleDB | ClickHouse | 移植性 |
|---|---|---|---|
| 生テーブル | hypertable(`create_hypertable`) | `MergeTree ORDER BY (series_id, ts)` + `PARTITION BY toYYYYMMDD(ts)` | **概念一致**（Narrow が効く） |
| 時間バケット | `time_bucket` | `toStartOfInterval` / `toStartOfHour` | 概念一致（関数名差） |
| ロールアップ | continuous aggregate | Materialized View + `AggregatingMergeTree`/`SummingMergeTree` | **概念一致・DDLは別** |
| avg の段階集計 | sum_v+n（or stats_agg） | sum+count（or `avgState`/`avgMerge`） | **sum+count なら完全共通** |
| p95（任意） | `percentile_agg`+`rollup`(toolkit) | `quantileTDigestState`+`quantileTDigestMerge` | **両者ネイティブ**（素のPGのみ非対応→任意列） |
| 最新値 | 専用 `current_value` 表 UPSERT | `ReplacingMergeTree` or 外部KV / RDBMS側 current 表 | 概念一致 |
| 圧縮 | compression(segmentby/orderby) | コーデック `CODEC(Delta, Gorilla, ZSTD)` | 概念一致・設定差 |
| 保持 | `add_retention_policy` | `TTL ts + INTERVAL n DELETE` | 概念一致・設定差 |
| last/first | `last()`/`first()` | `argMax`/`anyLast` | 概念一致 |

**結論**: 時系列は「**論理ティア定義（raw/1m/1h/1d、各ティアの集計関数 {sum,count,min,max,(p95任意)}、保持期間）を
共通仕様として宣言**」し、**エンジン別 DDL はアダプタが生成**する。生テーブルが Narrow なので生成は機械的。

---

## 9.4 時系列 DAL（5 操作の最小インターフェース）

アプリ／サービス層はこの 5 操作だけを呼ぶ。実装をアダプタで差し替えればエンジンが替わる。

```text
append(rows: [(ts, series_id, value, quality)])           -- バルク投入
range_query(series_ids, from, to, grain)                  -- grain ∈ {raw,1m,1h,1d} を返す
upsert_latest(rows: [(series_id, ts, value)])             -- current_value 更新
define_tier(name, source, bucket, aggs[], retention)      -- ロールアップ定義（DDL生成）
define_retention(table, keep)                             -- 保持（policy/TTL生成）
```

- `range_query` の戻りは常に `(bucket, series_id, sum_v, n, min_v, max_v[, p95])` の**正規化タプル**。
  呼び出し側は avg=`sum_v/n` を一様に扱う（エンジン非依存）。
- ラベル/階層解決は RDBMS 側で `series_id` 集合 → 属性 JOIN（越境しない）。

---

## 9.5 方言マップ（アダプタが吸収する差分）

| 項目 | PostgreSQL | MySQL | ClickHouse(時系列) |
|---|---|---|---|
| 自動採番 | `GENERATED ALWAYS AS IDENTITY` | `BIGINT AUTO_INCREMENT` | （採番はRDBMS側のみ） |
| UPSERT | `INSERT ... ON CONFLICT DO UPDATE` | `INSERT ... ON DUPLICATE KEY UPDATE` | `ReplacingMergeTree`(マージ時) |
| JSON 型 / 取得 | `jsonb` / `->>` | `JSON` / `->>`,`JSON_EXTRACT` | `String`+`JSONExtract` |
| JSON 索引 | GIN | 生成列+INDEX | （分析側は不要） |
| 正規表現 CHECK | `~` | `REGEXP_LIKE` | — |
| boolean | `boolean` | `TINYINT(1)` | `UInt8` |
| 文字列前方一致索引 | `text_pattern_ops` | 接頭辞索引/`COLLATE` | — |
| 時刻関数 | `time_bucket` | `FROM_UNIXTIME`系 | `toStartOfInterval` |

> これらは**列定義・関数名の機械的置換**であり、テーブル構成やクエリの論理は変わらない＝「作り直し」ではない。

---

## 9.6 p95（分位点）の扱い

- 分位点はエンジン依存度が高い（素の PG は段階ロールアップ不可、Timescale/ClickHouse は可）ため、
  **「論理ティアの任意（optional）列」**とする。
- 既定（最小公倍数）：CAGG/MV には **持たない**。必要時に**生 or 1m へ `percentile_cont`（PG）/`quantile`（CH）を
  都度実行**（短期窓限定）。max/avg/min が主 KPI の DCIM では実用上十分。
- TSDB が対応する構成では、アダプタが sketch 列（`percentile_agg` / `quantileTDigestState`）を**追加**する。
  → 論理仕様（「p95 が欲しいティア」）は共通、実体はアダプタ任せ。

---

## 9.7 エンジンプロファイル（設定で宣言）

```sql
CREATE TABLE engine_profile (         -- デプロイ1つにつき1行
    singleton boolean PRIMARY KEY DEFAULT true CHECK (singleton),
    rdbms text NOT NULL CHECK (rdbms IN ('postgres','mysql')),
    tsdb  text NOT NULL CHECK (tsdb  IN ('timescaledb','clickhouse')),
    enable_percentiles boolean NOT NULL DEFAULT false   -- p95 ティア列の有無
);
```

アダプタは起動時にこの値を見て、DDL 生成・UPSERT 構文・ロールアップ実装を切り替える。

---

## 9.8 何が「差し替え」で、何は「不変」か（線引き）

| 区分 | 例 | 載せ替え時 |
|---|---|---|
| **不変（作り直し不要）** | 30+ のリレーショナル表・ER・FK・UNIQUE・CHECK、配下集約の閉包 JOIN、電源トレースの再帰 CTE、Narrow の論理形、series 台帳、5操作の呼び出し側 | そのまま |
| **アダプタ差し替え** | 方言DDL生成、UPSERT 構文、JSON 索引方式、ロールアップ DDL（CAGG⇄MV）、圧縮/保持（policy⇄TTL）、p95 実装 | アダプタ実装の置換 |
| **サービス層** | 集約制約（予約≤定格・SPOF・容量超過・ASHRAE 逸脱）、閉包/占有行の維持 | DB 方言に依存しない実装 |

---

## 9.9 正直な限界（トレードオフ）

- **PG の旨味は捨てる**：`EXCLUDE` の宣言的・原子的保証、`ltree` の表現力など。占有行 UNIQUE で原子性は
  保てるが**行数は増える**（ラックは小さいので実害小）。閉包テーブルは**サブツリー移動でエッジ再生成**。
- **集約制約が DB から出る**：サービス層に移すと、DB 直叩きの経路では強制されない。重要制約は
  ストアドではなく**単一の書き込み口（サービス）に集約**して担保する設計運用が前提。
- **分位点の一級扱いを諦める**：p95 は任意列。素の PG 構成では長期 p95 は近似/都度計算に劣化。
- **ClickHouse の更新セマンティクス**：`ReplacingMergeTree` の重複解決は非同期。最新値は **RDBMS 側 current 表**で
  持つ方が即時整合に強い（本設計の既定）。

> まとめ: P05 は「Narrow + series 台帳 + メタ/TSDB 分離 + 越境JOINなし」という**ポータブルな骨格**を持つ。
> 本章の LCD 化（閉包・占有行・sum+count・サービス層制約）で、**PG+Timescale ⇄ MySQL+ClickHouse を
> アダプタ/方言/サービス層の差し替えだけ**で行き来できる。
