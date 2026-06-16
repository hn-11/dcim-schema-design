# 04. ユースケース駆動のスキーマ検証

確定設計 **Semantic-Typed DCIM**（[03-finalists.md](./03-finalists.md)）に対し、実アプリで叩く代表クエリを
書き下し、パフォーマンスと扱いやすさを検証する。各クエリに「効くインデックス／設計上の勘所」を付す。
スキーマ全体は [05章 ER 図](./05-er-diagram.md)、移植性（LCD）方針は [09章](./09-portability.md)、
派生メトリック（pPUE・部屋グループ）は [10章](./10-room-group-derived-metrics.md)。

前提テーブル（[09章](./09-portability.md)の LCD 版）: `location`(+`location_closure`), `rack`(+`rack_unit_occupancy`),
`equipment`, `equipment_type`(+`equip_kind_id`), `manufacturer`, 電力チェーン `power_panel`→`breaker`→`power_feed`,
導出フロービュー `v_equip_flow`（physical path から equipment 間電力フローを導出），
メトリックカタログ `metric`（フラット・`code`/`category` で識別）,
`data_point`(+`role`: sensor/sp/cmd/derived), `series`(+`metric_id`,`equipment_id`,`rack_id`,`location_id`,`room_group_id`),
`measurement`(hypertable), `measurement_1m/1h/1d`(CAGG), `current_value`(+`cur_status`,`alert_state`),
`capacity_budget`（排他アーク FK: `location_id`/`rack_id`/`breaker_id` nullable＋CHECK 1つ）/`equipment_demand`,
`threshold`, `redundancy_group`/`redundancy_member`。

> **メトリックの意味づけは `metric` フラットカタログで識別**（旧 `point_type` ＋ 3軸小マスタ廃止・[03章 L4](./03-finalists.md)）。
> `metric` は `code`（`active_power`|`rack_inlet_temp`|`chw_supply_temp` 等）/ `category`（`power`|`temperature`|...）/
> `unit` / `datatype` / `value_kind` / `default_agg` / `is_derived` を1行に持つ。
> - **有効電力系列** = `m.code = 'active_power'`
> - **吸気温度系列** = `m.code = 'rack_inlet_temp'`（または `m.category = 'temperature'` で温度系全体）
>
> `series` は `metric_id` を非正規化（`series.metric_id` → `metric.id` FK）。
> 量・物質・haystack_symbol の小マスタ JOIN はすべて不要になった。

---

## UC-1. 建屋全体の今月の電気代計算（電力集約値 → kWh → 料金）

**狙い**: 空間階層（建屋配下の全機器）× 時系列（電力量積分）× 集約（日次ロールアップ）の総合。

有効電力(W) を時間積分して kWh を出す。日次ロールアップ `measurement_1d` の平均電力 ×24h で近似、
avg は `Σsum_v/Σn` × 期間で算出。建屋配下は `series.location_id` を `location_closure` で JOIN して抽出。

```sql
-- 当該建屋配下・有効電力 series を「閉包テーブル」で抽出（ltree 不使用・LCD）
WITH target_series AS (
    SELECT s.series_id
    FROM series s
    JOIN location_closure lc ON lc.descendant_id = s.location_id
    JOIN metric m            ON m.id = s.metric_id AND m.code = 'active_power'  -- ★ フラットカタログ: code 直指定
    WHERE lc.ancestor_id = :building_id
      AND s.retired_at IS NULL
),
-- ★ series ごとに当月の平均電力(W)と観測日数を出す（series 横断の sum/avg 取り違え回避）
per_series AS (
    SELECT t.series_id,
           sum(d.sum_v) / NULLIF(sum(d.n),0) AS avg_w,  -- avg = Σsum_v/Σn（toolkit 不使用で厳密）
           count(*)                           AS n_days  -- 観測のあった日次バケット数
    FROM measurement_1d d
    JOIN target_series t USING (series_id)
    WHERE d.bucket >= date_trunc('month', :as_of)
      AND d.bucket <  date_trunc('month', :as_of) + INTERVAL '1 month'
    GROUP BY t.series_id                                 -- ← series 別に集計してから後段で合計
)
SELECT
    -- 各 series の電力量(kWh) = 平均W × 観測時間h ÷ 1000、を建屋内の全 series で合計
    sum(avg_w * n_days * 24) / 1000.0                                    AS kwh_month,
    -- 金額は丸め誤差を避けるため numeric で計算（FLOAT 課金アンチパターン回避）
    round( (sum(avg_w * n_days * 24) / 1000.0 * :jpy_per_kwh)::numeric ) AS jpy_energy
FROM per_series;
```

- **勘所**:
  - **`metric.code` 直指定**: 「有効電力」は `m.code = 'active_power'` の1 JOIN で完結（旧 `quantity`/`phenomenon`
    への `haystack_symbol` 2軸 JOIN は不要）。`metric` 行が単位を1つ持つので、集計対象が物理的に W で
    あることを DB が保証する（[03章 L4](./03-finalists.md)）。
  - 配下抽出は `location_closure` の JOIN（`ltree` 不使用・MySQL でも同形）。空間 JOIN を
    テレメトリ前段で済ませるのが Narrow モデルの定石。
  - **集計の向き**: 「建屋総電力」は **series 別に平均 → 全 series で合計** の順（日付のみで GROUP BY すると
    series 横断の平均になり N 倍ぶん過小評価になる）。avg は `Σsum_v/Σn` で厳密（avg-of-avg を回避）。
  - 積算電力量メトリック（`metric.value_kind='counter'`, `code='active_energy'` 等）があれば差分計算がより正確。
    **金額・kWh は `numeric`** で計算する。
  - 日次 CAGG は長期 retention（10 年）なので過去月も即答。生データ retention（30 日）に依存しない。
- **PUE 拡張**: 同じ枠組みで「IT 負荷合計」（`equip_kind.power_class='it'` の機器配下）と「施設総入力」を集計し
  比を取る。派生 pPUE は `metric.is_derived=true`（`code='ppue'`）× 部屋グループ集約で持つ（[10章](./10-room-group-derived-metrics.md)）。

---

## UC-2. 特定ラックの過去 1 ヶ月の温度推移の高速取得

**狙い**: 時系列の単一系列高速読み出し（Narrow の最得意領域）＋ ロールアップ粒度の自動選択。

```sql
-- 当該ラックの吸気温度 series（metric.code で識別）
WITH temp_series AS (
    SELECT s.series_id
    FROM series s
    JOIN metric m ON m.id = s.metric_id AND m.code = 'rack_inlet_temp'  -- ★ code 直指定
    WHERE s.rack_id = :rack_id
      AND s.retired_at IS NULL
    -- 複数センサがあれば全取得。特定センサに絞る場合は AND s.data_point_id = :dp_id
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
  - `series.rack_id` は非正規化キー（空間集約用）。ラック直指定の読み出しは追加 JOIN なしで効く。
    意味（吸気温度）の絞り込みは `metric.code = 'rack_inlet_temp'` 1 JOIN だけ（小マスタはキャッシュ常駐）。
    温度系全体を取りたい場合は `m.category = 'temperature'`（横断グルーピング）。
  - 粒度は期間で自動選択（24h→生 or 1m、1ヶ月→1h、1年→1d）。アプリ側でティアを切り替える。
  - `max/min` は再集約安全、`avg` は `Σsum_v/Σn` で厳密。**p95 は任意列**（[09章](./09-portability.md)）—
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
  - 連番生成は PG=`generate_series`、MySQL=再帰CTE。占有判定は単純な範囲比較のみ（range 型不要・LCD）。
  - 「ラックに収まる」（`start_u + need_u - 1 ≤ u_height`）は連番上限で自然に担保。

---

## UC-4. ある機器の上流電源パスをトレース（A/B 系統可視化）

**狙い**: 電力有向グラフの `WITH RECURSIVE` トレース。dual-corded / ATS の分岐も全パス列挙。

物理 path（`power_panel→breaker→power_feed→port/cable`）を真実源とし、`v_equip_flow`（導出ビュー）が
equipment 間の電力エッジを展開する。汎用 `equip_link` グラフや `substance_id`→`phenomenon` の JOIN は持たない。
下流が上流を指す向きなので、`v_equip_flow.to_equipment_id = :equipment_id` を起点に遡る。

```sql
-- v_equip_flow の想定列: from_equipment_id, to_equipment_id, feed_id, breaker_id, panel_id, leg
WITH RECURSIVE upstream AS (
    -- 起点機器の直上流（v_equip_flow で to=自分 を辿る）
    SELECT vf.from_equipment_id          AS up_equip_id,
           vf.leg,                       -- A/B/none（power_feed 系統から導出）
           1                             AS depth,
           ARRAY[:equipment_id, vf.from_equipment_id] AS path
    FROM v_equip_flow vf
    WHERE vf.to_equipment_id = :equipment_id

    UNION ALL

    SELECT vf.from_equipment_id, u.leg, u.depth + 1,
           u.path || vf.from_equipment_id
    FROM upstream u
    JOIN v_equip_flow vf ON vf.to_equipment_id = u.up_equip_id
    WHERE vf.from_equipment_id <> ALL (u.path)  -- サイクル防止
      AND u.depth < 50
)
SELECT u.leg, u.depth, u.up_equip_id,
       ek.code AS equip_kind_code,
       u.path
FROM upstream u
JOIN equipment e       ON e.id = u.up_equip_id
JOIN equipment_type et ON et.id = e.equipment_type_id
JOIN equip_kind ek     ON ek.id = et.equip_kind_id
ORDER BY u.leg, u.depth;
-- equip_kind.code が ups/switchgear/utility 等（v_equip_flow に上流を持たない）行が系統の頂点
```

- **勘所**:
  - **`v_equip_flow` ビュー**: `power_panel→breaker→power_feed` ＋ ポート/ケーブルを辿る再帰ビュー。
    物理 path が真実源なので、並行する汎用グラフを手で維持する必要がない（[03章 L7](./03-finalists.md)）。
    `leg`（A/B）は `power_feed` が属する系統から導出（`equipment.power_side` 非正規化も可・構成変更時に再計算）。
  - **電力・冷却の分離**: `v_equip_flow` は電力フロー専用。冷却フローが必要な場合は `equip_logical_link`
    （`medium_id` で冷水/空気を区別）を別途辿る（[03章 L7](./03-finalists.md)）。
  - インデックス `v_equip_flow (to_equipment_id)` で下流→上流の各ホップが定数時間。
  - `path` 配列でサイクル検出、`depth < 50` で暴走防止。本番は end-to-end を path キャッシュ表に固める。
  - **物理詳細**: ブレーカ位置・回路容量は `power_panel→breaker→power_feed` に保持。
    `v_equip_flow.feed_id` / `breaker_id` / `panel_id` を拾えば物理台帳と突合できる。

---

## UC-5. 冗長性違反（SPOF）検出 — A/B が同一上流に収束する機器

**狙い**: ドメイン制約「A 給電と B 給電は異なる上流に収束する」を**サービス層の監視ビュー**で検出
（行間集約のため単一行 CHECK では表せない・移植性のため DB トリガに依存させない・[09章](./09-portability.md)）。

SPOF = ある機器の全給電脚が、**相異なる上流ソース**に 2 系統以上収束していない状態（独立ソース数 < 2）。
`v_equip_flow` 上で全機器を上流展開し、根（さらに上流を持たない機器）の distinct 数を数える。
`redundancy_group`（intent 側）と突合することで、冗長設計の意図と現物構成のズレを可視化できる。

```sql
WITH RECURSIVE up_walk AS (
    -- 全機器 × v_equip_flow を起点に上流展開（UC-4 を全機器版に）
    SELECT vf.to_equipment_id   AS equipment_id,
           vf.from_equipment_id AS up_equip_id,
           ARRAY[vf.to_equipment_id, vf.from_equipment_id] AS path
    FROM v_equip_flow vf

    UNION ALL

    SELECT w.equipment_id, vf.from_equipment_id,
           w.path || vf.from_equipment_id
    FROM up_walk w
    JOIN v_equip_flow vf ON vf.to_equipment_id = w.up_equip_id
    WHERE vf.from_equipment_id <> ALL (w.path)
      AND array_length(w.path, 1) < 50
),
-- 上流ソースの「根」: v_equip_flow にさらに上流フローを持たない機器（UPS/switchgear/utility 等）
roots AS (
    SELECT w.equipment_id, w.up_equip_id
    FROM up_walk w
    WHERE NOT EXISTS (
        SELECT 1 FROM v_equip_flow vf2
        WHERE vf2.to_equipment_id = w.up_equip_id
    )
),
spof_equip AS (
    SELECT equipment_id,
           count(DISTINCT up_equip_id) AS independent_sources
    FROM roots
    GROUP BY equipment_id
    HAVING count(DISTINCT up_equip_id) < 2   -- 収束先が単一 = SPOF
)
-- redundancy_group の intent（2N/N+1 等）と照合：冗長グループに属しているのに SPOF なら設計違反
SELECT se.equipment_id,
       se.independent_sources,
       rg.name     AS redundancy_group,
       rg.topology AS intended_topology     -- 2N なら独立ソース=2 が必要
FROM spof_equip se
LEFT JOIN redundancy_member rm ON rm.equipment_id = se.equipment_id
LEFT JOIN redundancy_group rg  ON rg.id = rm.group_id AND rg.domain = 'power'
ORDER BY se.equipment_id;
```

- **勘所**:
  - **`v_equip_flow` ベース**: 物理 path 導出ビューのみで SPOF 検出が完結。ソースの一意性は
    `count(DISTINCT up_equip_id)` で判定（`FILTER` 不要なのでそのまま MySQL でも動く）。
  - **`redundancy_group` との照合**: 冗長グループ（`2N`/`N+1`）に所属しながら独立ソース < 2 の機器が
    設計違反。`rg.topology`（意図）vs `independent_sources`（現実）を並べることで intent×reality を可視化。
    dual-corded 機器が誤って同一 UPS/PDU 系統に挿さっていれば独立ソース数=1 で検出。
  - 「根」判定を `NOT EXISTS`（さらなる上流フローエッジ無し）で行うため、トポロジが UPS 止まりでも
    上位 switchgear まで描かれていても同一クエリで成立。
- 同種の監視ビュー（いずれもサービス層）:
  - **回路負荷超過**: `power_feed.available_power_w`（V×A×util×√3 の生成列）に対し、当該 feed 配下機器の
    `equipment_demand` / 実測 draw 合計が超過していないか。
  - **温湿度 ASHRAE 逸脱**: `current_value` vs `equipment_type.ashrae_class` の許容域。

---

## UC-6. 部屋単位のリアルタイム総消費電力 & ホットスポット検出

**狙い**: 最新値（`current_value`）× 空間集約（`location_closure` JOIN）× キャパシティ対比。

```sql
-- 部屋配下の現在総電力 & 使用率（配下は location_closure で抽出・ltree不使用）
SELECT
    sum(cv.value)                              AS total_w_now,
    (SELECT rated FROM capacity_budget
      WHERE location_id = :room_id
        AND dimension   = 'power_w')           AS rated_w,
    -- 排他アーク CHECK により rack_id/breaker_id は自動的に NULL が保証される
    round(100.0 * sum(cv.value) /
      (SELECT rated FROM capacity_budget
        WHERE location_id = :room_id
          AND dimension   = 'power_w'), 1)     AS utilization_pct
FROM current_value cv
JOIN series s ON s.series_id = cv.series_id
JOIN metric m ON m.id = s.metric_id AND m.code = 'active_power'  -- ★ code 直指定
JOIN location_closure lc ON lc.descendant_id = s.location_id AND lc.ancestor_id = :room_id
WHERE cv.cur_status = 'ok';        -- ★ down/fault/stale を総和から除外

-- ホットスポット: 部屋配下で critical 状態 or 高温の機器
SELECT s.rack_id,
       max(cv.value) FILTER (WHERE m.code = 'rack_inlet_temp') AS max_inlet_temp,
       count(*)      FILTER (WHERE cv.alert_state = 'critical') AS critical_count
FROM current_value cv
JOIN series s ON s.series_id = cv.series_id
JOIN metric m ON m.id = s.metric_id
JOIN location_closure lc ON lc.descendant_id = s.location_id AND lc.ancestor_id = :room_id
GROUP BY s.rack_id
HAVING count(*) FILTER (WHERE cv.alert_state = 'critical') > 0
    OR max(cv.value) FILTER (WHERE m.code = 'rack_inlet_temp') > 27  -- ASHRAE 推奨上限
ORDER BY max_inlet_temp DESC NULLS LAST;
```
> `FILTER` は PG 構文。MySQL では `MAX(CASE WHEN m.code = 'rack_inlet_temp' THEN cv.value END)` /
> `COUNT(CASE WHEN cv.alert_state = 'critical' THEN 1 END)` に置換（方言差・[09章 9.5](./09-portability.md)）。

- **勘所**:
  - **`cur_status` と `alert_state` の使い分け**: `current_value` は**収集状態**（`cur_status`:
    ok/down/fault/disabled/stale）と**アラート状態**（`alert_state`: normal/warning/critical）を別列で持つ。
    総電力は「健全に取れている値だけ」を足すため `cur_status='ok'` で絞り、ホットスポットは
    `alert_state='critical'` で拾う。stale を total に混ぜない（過大/過小評価防止）のが要点。
  - **`metric.code` で意味を絞る**: 「有効電力」は `m.code = 'active_power'`、「吸気温度」は
    `m.code = 'rack_inlet_temp'`（または `m.category = 'temperature'`）。旧 `quantity`/`phenomenon` 2軸 JOIN 不要。
    ホットスポット側は全 series を1つの JOIN で取り込み、`FILTER`（MySQL では `CASE WHEN`）で metric 種類を切り分ける。
  - **排他アーク FK**: `capacity_budget` の検索は `location_id = :room_id`（CHECK 制約により
    `rack_id` と `breaker_id` は DB が NULL を保証・旧 `scope`/`scope_id` 多態キーを廃止・[03章 L8](./03-finalists.md)）。
  - `current_value` は PK 1 行参照で最速。ingest 時に UPSERT + `alert_state` 評価（`threshold` を
    equipment>location>metric 優先で引き、hysteresis でフラッピング抑制）。
  - 空間集約は `series.location_id` を `location_closure` で JOIN（`ltree` 不使用・全エンジン同形）。
  - 使用率は `capacity_budget.rated` 対比。`stranded = reserved - actual` を出せば過剰プロビジョニング検出（UC-7）。

---

## UC-7. キャパシティ：ラックの 定格 vs 予約 vs 実測（死蔵容量）

**狙い**: `capacity_budget`（定格・冗長予備・排他アーク FK）× `equipment_demand`（推定負荷戦略の予約）× `current_value`（実測）。

```sql
SELECT b.dimension, b.rated, b.soft_limit, b.redundancy_reserve,
       r.reserved_sum,
       a.actual_now,
       round(100.0 * r.reserved_sum / b.rated, 1) AS reserved_pct,
       round(100.0 * a.actual_now   / b.rated, 1) AS actual_pct,
       (r.reserved_sum - COALESCE(a.actual_now,0)) AS stranded     -- 死蔵容量
FROM capacity_budget b
LEFT JOIN LATERAL (   -- 予約合計（equipment_demand から。推定負荷戦略適用後の reserved）
    SELECT sum(ed.reserved) AS reserved_sum
    FROM rack_mount rm
    JOIN equipment_demand ed ON ed.equipment_id = rm.equipment_id AND ed.dimension = b.dimension
    WHERE rm.rack_id = b.rack_id
) r ON true
LEFT JOIN LATERAL (   -- 実測（current_value の電力合計・健全値のみ）
    SELECT sum(cv.value) AS actual_now
    FROM rack_mount rm
    JOIN series s        ON s.equipment_id = rm.equipment_id
    JOIN current_value cv ON cv.series_id = s.series_id AND cv.cur_status = 'ok'
    JOIN metric m        ON m.id = s.metric_id AND m.code = 'active_power'  -- ★ code 直指定
    WHERE rm.rack_id = b.rack_id
      AND b.dimension  = 'power_w'
) a ON true
WHERE b.rack_id     = :rack_id
  AND b.location_id IS NULL
  AND b.breaker_id  IS NULL;
-- 排他アーク FK: rack_id 列で直参照（旧 scope='rack' AND scope_id=:rack_id の多態キーを廃止）
```

- **勘所**:
  - **排他アーク FK**: `capacity_budget` のスコープ識別は `rack_id = :rack_id`（`location_id IS NULL AND
    breaker_id IS NULL` で排他を明示。CHECK 制約が DB でも保証）。`breaker_id = :breaker_id` に替えれば
    分電盤ブレーカ単位の容量予算も同じ表で扱える（[03章 L8](./03-finalists.md)）。
  - `equipment_demand.reserved` は **`load_strategy`**（nameplate / adjusted_nameplate / predicted / contracted）
    適用後の働く値。銘板ベタ張りでなく **adjusted_nameplate**（派生定格 ~50-75%）や **predicted** を使うと
    死蔵容量を圧縮できる（EcoStruxure 推定負荷戦略）。コロ顧客は `contracted`（契約電力）で予約する（[08章](./08-tenancy-colocation.md)）。
  - 実測の絞り込みも `m.code = 'active_power'` の1 JOIN（単位整合は `metric` 行が保証）。
  - **`rack_mount` 経由で機器を取得**: ラック搭載機器の一覧は `rack_mount`（`equipment_id UNIQUE`）経由が
    簡潔（`rack_unit_occupancy` は空き U 検索 UC-3 向け・U 数分の重複行がある）。
  - DCIM の中核 KPI「**stranded = reserved − actual**」。実測ベースで予算化すると死蔵容量を回収できる。
  - 「予約合計 ≤ rated」「rack 内合計 ≤ breaker 定格」等の**集約制約はサービス層**で担保
    （行間集約のため単一行 CHECK 不可・移植性のため DB トリガに依存させない・[09章](./09-portability.md)）。

---

## 検証から得た設計上の結論

| 観点 | 検証結果 |
|------|---------|
| **点の意味づけ** | `metric.code`（`active_power`/`rack_inlet_temp` 等）の1 JOIN で型安全に絞れる。`metric` 行が単位を1つ持つので集計対象の物理次元を DB が保証。旧 `point_type`+3軸 JOIN は不要 |
| **空間集約の高速性** | `series.location_id` を `location_closure` で JOIN し建屋/部屋配下を一発抽出（`ltree` 不要・全エンジン同形）。テレメトリ前段で空間 JOIN を解決する Narrow + 非正規化キーが効く |
| **時系列読み出し** | 単一系列は `(series_id, time DESC)` で最速。粒度を期間でティア切替。p95 は任意列（既定は都度計算） |
| **U 位置の安全性** | `rack_unit_occupancy` の複合主キーが並行挿入下でも重なりを原子的に拒否。空き検索は占有行との差集合で簡潔（拡張不要） |
| **電力/冷却トレース** | `v_equip_flow`（物理 path 導出ビュー）＋ `WITH RECURSIVE`（PG/MySQL 両対応）。物理 path が真実源なので並行グラフ不要。A/B・ATS・SPOF も同じビューで。本番は path キャッシュ |
| **収集状態とアラートの分離** | `current_value.cur_status`（健全性）と `alert_state`（しきい値）を別列に。総和は ok のみ、ホットスポットは critical で拾う。stale 混入による誤集計を回避 |
| **集約制約** | SPOF・容量超過・温湿度逸脱は行間集約のため CHECK 不可 → サービス層（移植性のため DB トリガに依存させない）。SPOF は `redundancy_group` の intent と照合 |
| **容量 KPI** | `capacity_budget`（排他アーク FK）× `equipment_demand(load_strategy)` × 実測で `stranded = reserved − actual`。adjusted_nameplate/predicted で死蔵を圧縮、breaker スコープまで同一表 |
| **電気代/PUE** | 日次ロールアップ（長期 retention）で過去月も即答。avg は `Σsum_v/Σn`、kWh counter があれば差分でより厳密。pPUE は `metric.is_derived=true`（`code='ppue'`）× 部屋グループ（[10章](./10-room-group-derived-metrics.md)） |

→ **Semantic-Typed DCIM**（[03章](./03-finalists.md)／[09章](./09-portability.md)の LCD 版）は全ユースケースを、
可変性（多数 DC 対応）と**エンジン非依存**を保ったまま、ドメイン知識を DB 制約で担保しつつ満たせることを確認した。

## 参考（時系列アダプタ＝TimescaleDB 実装時の注意）

> 下記は **TimescaleDB アダプタ**固有の注意。ClickHouse アダプタでは MV+AggregatingMergeTree/TTL に読み替え（[09章](./09-portability.md)）。

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
