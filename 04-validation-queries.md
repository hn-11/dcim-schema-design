# 04. Step 6 — ユースケース駆動のスキーマ検証

推奨案 **P05** のスキーマ（[03-finalists.md](./03-finalists.md)）に対し、実アプリで叩く代表クエリを書き下し、
パフォーマンスと扱いやすさを検証する。各クエリに「効くインデックス／設計上の勘所」を付す。

前提テーブル（P05・[09 章](./09-portability.md)の LCD 版）: `location`(+`location_closure`), `rack`, `rack_mount`(+`rack_unit_occupancy`), `device`, `device_type`,
`power_port`/`power_outlet`/`power_feed`/`power_panel`, `cable`/`cable_termination`,
`metric_definition`, `series`(+`rack_id`,`location_id`), `measurement`(hypertable),
`measurement_1m/1h/1d`(CAGG), `current_value`, `capacity_budget`, `threshold`。

メトリックコード例: `active_power`(W), `inlet_temp`(degC), `humidity`(%RH)。

---

## UC-1. 建屋全体の今月の電気代計算（電力集約値 → kWh → 料金）

**狙い**: 空間階層（建屋配下の全機器）× 時系列（電力量積分）× 集約（日次ロールアップ）の総合。

`active_power`(W) を時間積分して kWh を出す。日次ロールアップ `measurement_1d` の平均電力 ×24h で近似、
avg は `Σsum_v/Σn` × 期間で算出。建屋配下は `series.location_id` を `location_closure` で JOIN して抽出。

```sql
-- 当該建屋配下・電力メトリックの series を「閉包テーブル」で抽出（ltree 不使用・LCD）
WITH target_series AS (
    SELECT s.series_id
    FROM series s
    JOIN location_closure lc ON lc.descendant_id = s.location_id
    WHERE lc.ancestor_id = :building_id
      AND s.metric_def_id = (SELECT metric_def_id FROM metric_definition WHERE code='active_power')
      AND s.retired_at IS NULL
),
-- ★ series ごとに当月の平均電力(W)と観測日数を出す（series 横断の sum/avg 取り違え回避）
per_series AS (
    SELECT t.series_id,
           sum(d.sum_v) / NULLIF(sum(d.n),0) AS avg_w,  -- avg = Σsum_v/Σn（toolkit 不使用で厳密）
           count(*)                          AS n_days  -- 観測のあった日次バケット数
    FROM measurement_1d d
    JOIN target_series t USING (series_id)
    WHERE d.bucket >= date_trunc('month', :as_of)
      AND d.bucket <  date_trunc('month', :as_of) + INTERVAL '1 month'
    GROUP BY t.series_id                           -- ← series 別に集計してから後段で合計
)
SELECT
    -- 各 series の電力量(kWh) = 平均W × 観測時間h ÷ 1000、を建屋内の全 series で合計
    sum(avg_w * n_days * 24) / 1000.0                                   AS kwh_month,
    -- 金額は丸め誤差を避けるため numeric で計算（FLOAT 課金アンチパターン回避）
    round( (sum(avg_w * n_days * 24) / 1000.0 * :jpy_per_kwh)::numeric ) AS jpy_energy
FROM per_series;
```

- **勘所**:
  - 配下抽出は `location_closure` の JOIN（`ltree` 不使用・MySQL でも同形）。空間 JOIN を
    テレメトリ前段で済ませるのが Narrow モデルの定石。
  - **集計の向き**: 「建屋総電力」は **series 別に平均 → 全 series で合計** の順（日付のみで GROUP BY すると
    series 横断の平均になり N 倍ぶん過小評価になる）。avg は `Σsum_v/Σn` で厳密（avg-of-avg を回避）。
  - 積算電力量メトリック（kWh counter）があれば差分計算がより正確（時間刻みのムラに強い）。
    **金額・kWh は `numeric`** で計算する。
  - 日次 CAGG は長期 retention（10 年）なので過去月も即答。生データ retention（30 日）に依存しない。
- **PUE 拡張**: 同じ枠組みで「IT 負荷合計」と「施設総入力」を別 `metric_def` で集計し比を取れば PUE。

---

## UC-2. 特定ラックの過去 1 ヶ月の温度推移の高速取得

**狙い**: 時系列の単一系列高速読み出し（Narrow の最得意領域）＋ ロールアップ粒度の自動選択。

```sql
-- 当該ラックの inlet 温度 series
WITH temp_series AS (
    SELECT series_id FROM series
    WHERE rack_id = :rack_id
      AND metric_def_id = (SELECT metric_def_id FROM metric_definition WHERE code='inlet_temp')
      AND retired_at IS NULL
)
-- 1ヶ月なら 1時間粒度で十分。max/min/avg を一度に（avg=Σsum_v/Σn）
SELECT h.bucket,
       max(h.max_v)                       AS max_temp,
       min(h.min_v)                       AS min_temp,
       sum(h.sum_v) / NULLIF(sum(h.n),0)  AS avg_temp
FROM measurement_1h h
JOIN temp_series USING (series_id)
WHERE h.bucket >= :as_of - INTERVAL '1 month'
GROUP BY h.bucket
ORDER BY h.bucket;
-- p95 が必要なら（任意・engine依存）: 生 or 1m に対し PG=percentile_cont / CH=quantile を短期窓で都度実行
```

- **勘所**:
  - `measurement (series_id, time DESC)` インデックス（およびロールアップの自動グループインデックス）で
    単一系列・期間スキャンが最速。`compress_segmentby=series_id` で同一系列が連続し圧縮も効く。
  - 粒度は期間で自動選択（24h→生 or 1m、1ヶ月→1h、1年→1d）。アプリ側でティアを切り替える。
  - `max/min` は再集約安全、`avg` は `Σsum_v/Σn` で厳密。**p95 は任意列**（[09 章](./09-portability.md)）—
    既定では持たず必要時に都度計算。TSDB 対応構成では sketch 列を追加可。

---

## UC-3. ラックの空き U 検索（新規機器の設置可能位置）

**狙い**: U 位置モデル（占有U行）の活用。指定 U 高さが連続で空いている位置を、占有行との差集合で探す。

```sql
-- 当該面の占有済み U（rack_unit_occupancy。フルデプスは front/rear 両面に行があるので対象面で拾える）
WITH occupied AS (
    SELECT u FROM rack_unit_occupancy
    WHERE rack_id = :rack_id AND rack_side = 'full' AND face = :face
),
-- ラック全 U を candidate 下端として走査し、need_u 連続が全て空いているものを選ぶ
candidates AS (
    SELECT gs AS start_u
    FROM rack r, generate_series(1, r.u_height - :need_u + 1) gs   -- MySQL は再帰CTEで連番生成
    WHERE r.id = :rack_id
)
SELECT c.start_u
FROM candidates c
WHERE NOT EXISTS (
    SELECT 1 FROM occupied o
    WHERE o.u >= c.start_u AND o.u < c.start_u + :need_u           -- 占有Uが範囲内に1つも無い
)
ORDER BY c.start_u
LIMIT 20;
```

- **勘所**:
  - 実際の搭載時は `rack_unit_occupancy` への INSERT が**複合主キー (`rack_id, rack_side, face, u`)**で
    **二重予約・重なりを原子的に拒否**。アプリの探索結果が古くても安全（並行挿入耐性）・拡張ゼロ・全エンジン移植可。
  - 連番生成は PG=`generate_series`、MySQL=再帰CTE。占有判定は単純な範囲比較のみ（range 型不要）。
  - 「ラックに収まる」（`start_u + need_u - 1 ≤ u_height`）は連番上限で自然に担保。

---

## UC-4. ある機器の上流電源パスを分電盤までトレース（A/B 系統可視化）

**狙い**: 配線（電力有向グラフ）の `WITH RECURSIVE` トレース。dual-corded / ATS の分岐も全パス列挙。

```sql
-- 電力エッジ（下流→上流）を正規化したビュー（03章 power_arc 相当）
WITH RECURSIVE upstream AS (
    SELECT 'power_port'::termination_kind_t AS kind, pp.id,
           pp.redundancy AS leg, 1 AS depth,
           ARRAY['power_port:'||pp.id] AS path
    FROM power_port pp
    JOIN device d ON d.id = pp.device_id
    WHERE d.id = :device_id                              -- 各 PSU が起点

    UNION ALL

    SELECT a.up_kind, a.up_id, u.leg, u.depth + 1,
           u.path || (a.up_kind||':'||a.up_id)
    FROM upstream u
    JOIN power_arc a ON a.down_kind = u.kind AND a.down_id = u.id
    WHERE (a.up_kind||':'||a.up_id) <> ALL (u.path)      -- サイクル防止
      AND u.depth < 50
)
SELECT leg, depth, kind, id, path
FROM upstream
ORDER BY leg, depth;
-- kind='power_panel' に到達した行がルート。leg(A/B) ごとに独立系統が見える
```

- **勘所**:
  - `power_arc` は (1)ケーブル outlet→power_port、(2)機内分配 outlet.upstream_power_port、
    (3)feed→panel を UNION したビュー。トレースは正規化グラフ上の標準的再帰 CTE。
  - 本番は end-to-end を `CablePath` 相当のキャッシュ表に固めて参照（NetBox 流）。再帰は構成変更時に再計算。
  - `path` 配列でサイクル検出、`depth < 50` で暴走防止。

---

## UC-5. 冗長性違反（SPOF）検出 — A/B が同一上流に収束する機器

**狙い**: ドメイン制約「A 給電と B 給電は異なる上流」を**監視ビュー**で検出（集約制約のため DB 制約化不可）。

```sql
WITH RECURSIVE up_roots AS (
    -- 全機器の全 PSU を起点に上流展開（UC-4 を全機器版に）
    SELECT d.id AS device_id, 'power_port'::termination_kind_t AS kind, pp.id,
           ARRAY['power_port:'||pp.id] AS path
    FROM device d JOIN power_port pp ON pp.device_id = d.id
    UNION ALL
    SELECT r.device_id, a.up_kind, a.up_id, r.path||(a.up_kind||':'||a.up_id)
    FROM up_roots r JOIN power_arc a ON a.down_kind=r.kind AND a.down_id=r.id
    WHERE (a.up_kind||':'||a.up_id) <> ALL (r.path) AND array_length(r.path,1) < 50
)
SELECT device_id,
       count(DISTINCT id) FILTER (WHERE kind='power_panel') AS independent_feeds
FROM up_roots
GROUP BY device_id
HAVING count(DISTINCT id) FILTER (WHERE kind='power_panel') < 2;   -- 単一系統 = SPOF
```

- **勘所**: 「全 PSU が同一 root panel に収束 = 冗長性不足」を 1 クエリで抽出。dual-corded 機器が
  誤って同一系統に挿さっていれば検出。`device_placement` を Type-2（P08）にすれば過去時点の冗長性も検証可。
- 同種の監視ビュー: **回路負荷超過**（feed の接続機器 draw 合計 > `available_power_w`）、
  **温湿度 ASHRAE 逸脱**（`current_value` vs `device_type.ashrae_class` の許容域）。

---

## UC-6. 部屋単位のリアルタイム総消費電力 & ホットスポット検出

**狙い**: 最新値（`current_value`）× 空間集約（`location_closure` JOIN）× キャパシティ対比。

```sql
-- 部屋配下の現在総電力 & 使用率（配下は location_closure で抽出・ltree不使用）
SELECT
    sum(cv.value)                                   AS total_w_now,
    (SELECT rated FROM capacity_budget
      WHERE scope='location' AND location_or_rack_id=:room_id AND dimension='power_w') AS rated_w,
    round(100.0 * sum(cv.value) /
      (SELECT rated FROM capacity_budget
        WHERE scope='location' AND location_or_rack_id=:room_id AND dimension='power_w'), 1)
                                                    AS utilization_pct
FROM current_value cv
JOIN series s ON s.series_id = cv.series_id
JOIN location_closure lc ON lc.descendant_id = s.location_id AND lc.ancestor_id = :room_id
WHERE s.metric_def_id = (SELECT metric_def_id FROM metric_definition WHERE code='active_power');

-- ホットスポット: 部屋配下で critical 状態 or 高温の機器
SELECT s.rack_id,
       max(cv.value) FILTER (WHERE md.code='inlet_temp')  AS max_inlet_temp,
       count(*) FILTER (WHERE cv.alert_state='critical')  AS critical_count
FROM current_value cv
JOIN series s ON s.series_id = cv.series_id
JOIN metric_definition md ON md.metric_def_id = s.metric_def_id
JOIN location_closure lc ON lc.descendant_id = s.location_id AND lc.ancestor_id = :room_id
GROUP BY s.rack_id
HAVING count(*) FILTER (WHERE cv.alert_state='critical') > 0
    OR max(cv.value) FILTER (WHERE md.code='inlet_temp') > 27   -- ASHRAE 推奨上限
ORDER BY max_inlet_temp DESC NULLS LAST;
```
> `FILTER` は PG 構文。MySQL では `MAX(CASE WHEN ... THEN value END)` / `COUNT(CASE WHEN ... THEN 1 END)` に置換（方言差・[09 章 9.5](./09-portability.md)）。

- **勘所**:
  - `current_value` は PK 1 行参照で最速。ingest 時に UPSERT + `alert_state` 評価（`threshold` を
    device>location>global 優先で引き、hysteresis でフラッピング抑制）。
  - 空間集約は `series.location_id` を `location_closure` で JOIN（`ltree` 不使用・全エンジン同形）。
  - 使用率は `rated` 対比。`stranded = reserved - actual` を出せば過剰プロビジョニング検出。

---

## UC-7. キャパシティ：ラックの 定格 vs 予約 vs 実測（死蔵容量）

```sql
SELECT b.dimension, b.rated, b.soft_limit,
       r.reserved_sum,
       a.actual_now,
       round(100.0 * r.reserved_sum / b.rated, 1) AS reserved_pct,
       round(100.0 * a.actual_now   / b.rated, 1) AS actual_pct,
       (r.reserved_sum - COALESCE(a.actual_now,0)) AS stranded     -- 死蔵容量
FROM capacity_budget b
LEFT JOIN LATERAL (   -- 予約合計（device_demand から）
    SELECT sum(dd.reserved) AS reserved_sum
    FROM device d JOIN rack_mount rm ON rm.device_id=d.id
    JOIN device_demand dd ON dd.device_id=d.id AND dd.dimension=b.dimension
    WHERE rm.rack_id = b.location_or_rack_id
) r ON true
LEFT JOIN LATERAL (   -- 実測（current_value の電力合計）
    SELECT sum(cv.value) AS actual_now
    FROM rack_mount rm
    JOIN device d ON d.id=rm.device_id
    JOIN series s ON s.device_id=d.id
    JOIN current_value cv ON cv.series_id=s.series_id
    JOIN metric_definition md ON md.metric_def_id=s.metric_def_id
    WHERE rm.rack_id = b.location_or_rack_id AND md.code='active_power'
      AND b.dimension='power_w'
) a ON true
WHERE b.scope='rack' AND b.location_or_rack_id = :rack_id;
```

- **勘所**: DCIM の中核 KPI「**stranded = reserved − actual**」。銘板基準でなく実測ベースで予算化すると
  死蔵容量を回収できる（Sunbird Auto Power Budget の思想）。「予約合計 ≤ 定格」は
  `device_demand` INSERT 時の CONSTRAINT TRIGGER で担保。

---

## 検証から得た設計上の結論

| 観点 | 検証結果 |
|------|---------|
| **空間集約の高速性** | `series.location_id` を `location_closure` で JOIN し建屋/部屋配下を一発抽出（`ltree` 不要・全エンジン同形）。テレメトリ前段で空間 JOIN を解決する Narrow + 非正規化キーが効く |
| **時系列読み出し** | 単一系列は `(series_id, time DESC)` で最速。粒度を期間でティア切替。p95 は任意列（既定は都度計算） |
| **U 位置の安全性** | `rack_unit_occupancy` の複合主キーが並行挿入下でも重なりを原子的に拒否。空き検索は占有行との差集合で簡潔（拡張不要） |
| **配線トレース** | 正規化エッジビュー + `WITH RECURSIVE`（PG/MySQL 両対応）。本番は path キャッシュ。A/B・ATS・SPOF も同一枠組み |
| **集約制約** | SPOF・容量超過・温湿度逸脱は行間集約のため CHECK 不可 → サービス層（移植性のため DB トリガに依存させない） |
| **電気代/PUE** | 日次ロールアップ（長期 retention）で過去月も即答。avg は `Σsum_v/Σn`、kWh counter があれば差分でより厳密 |

→ 推奨案 **P05**（[09 章](./09-portability.md)の LCD 版）は 6 ステップ計画の全ユースケースを、
可変性（多数 DC 対応）と**エンジン非依存**を保ったままドメイン制約付きで満たせることを確認した。

## 参考（時系列アダプタ＝TimescaleDB 実装時の注意）

> 下記は **TimescaleDB アダプタ**固有の注意。ClickHouse アダプタでは MV+AggregatingMergeTree/TTL に読み替え（[09 章](./09-portability.md)）。

- **連続集約(CAGG)の制約**: `time_bucket` を GROUP BY 必須・全式 IMMUTABLE。`percentile_cont`/`FILTER`/`DISTINCT`/
  window 関数は定義内 NG。**avg は `sum`+`count` 列で段階ロールアップ**（toolkit 非依存）。
- **階層 CAGG**: 下段は finalized（2.7+ 既定）必須。上段 bucket は下段の整数倍。avg=`sum(sum_v)/sum(n)`。
  p95 を持つ構成のみ TSDB ネイティブの sketch（Timescale=`percentile_agg`+`rollup`）を任意で追加。
- **real-time aggregation**: 2.13 以降は既定 OFF（`materialized_only=true`）。ダッシュボードは
  `ALTER MATERIALIZED VIEW ... SET (timescaledb.materialized_only=false)` で明示 ON。
- **retention の罠**: 生データ retention は CAGG の refresh `start_offset` より**長く**する
  （生が消える前に集約確定）。`drop_after > compress_after`。
- **圧縮(columnstore)**: 2.18+ は `enable_columnstore`/`add_columnstore_policy`（旧 `compress`/
  `add_compression_policy` も有効）。CAGG にも適用可。columnstore の `after` は refresh `start_offset` より大きく。
- **バージョン差**: 上記 API はバージョンで名称/既定が変わる。導入バージョンで `\df` 確認を推奨。
