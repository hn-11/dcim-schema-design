# 08. テナント / コロケーション拡張モジュール

「いろんな DC に提供するパッケージ」ゆえ、**テナント運用は顧客ごとに大きく異なる**
（自社専用のエンタープライズ DC / ホールセールのエリア貸し / リテールコロのラック・U 貸し / kW 販売）。
そこで賃貸・再課金・テナントポータルを **コア（[03章](./03-finalists.md) の Semantic-Typed DCIM）から切り離した
加算的（additive）オプションモジュール**として設計し、**運用プロファイルで有効化・粒度を切り替える**。
これが本章の「いい感じの区切り方」。

コア側は既に**最小のテナント原始要素**を型付きで持つ（[03章 L2/L3](./03-finalists.md)）:
`tenant(id, name, kind IN ('internal','colo_customer'))`・`location.tenant_id`（cage 等の区画所有）・
`equipment.tenant_id`（資産所有）。本章はこの上に**重い機構（賃貸・課金・分離）だけを加算する**。
EcoStruxure IT がコロケーションを **"Tenant Portal"** で提供し、容量を **"contracted power"（契約電力）負荷戦略**で
管理する設計（[03章 L8](./03-finalists.md) の `equipment_demand.load_strategy='contracted'` / `contracted` 列）を、
本章のテナント / cage / 契約電力の議論に接続する。

---

## 8.1 区切りの原則：コアは最小原始要素のみ、重い機構は加算レイヤ

| 層 | 役割 | テナント無効時 |
|----|------|---------------|
| **コア（[03章](./03-finalists.md)）** | 物理（location/rack/equipment）・配線・テレメトリ・集約 ＋ **テナント原始要素**（`tenant` / `location.tenant_id` / `equipment.tenant_id`） | そのまま完結。単一運用 DC は `kind='internal'` を部門タグに使うだけ |
| **テナントモジュール（本章）** | 賃貸（`space_lease`）・再課金・テナントポータル（RLS）・アクセス・データ分離 | テーブルを作らなければ存在しないのと同じ |

コアは既に `tenant` を持つので、**モジュールが足す接点（seam）は「nullable な denorm 列 2 つ」だけ**に絞る。
これによりコアの DDL/クエリはテナント有無で変わらず、モジュールを足す/外すが容易になる:

```sql
-- 03章が既にコアに持つテナント原始要素（本章では作らない・再掲）:
--   tenant(id, name, kind IN ('internal','colo_customer'))   -- L2
--   location.tenant_id   -- L2: cage 等、コロ顧客所有の区画
--   equipment.tenant_id  -- L3: 資産の所有者（コロ顧客）
-- 本章モジュールが足す非正規化シームは 2 列だけ（すべて NULL 許容・無効時は未使用）
ALTER TABLE series     ADD COLUMN tenant_id bigint;  -- 再課金/RLS 用に非正規化（L6 series 台帳）
ALTER TABLE power_feed ADD COLUMN tenant_id bigint;  -- 専用フィードの帰属（任意・L7）
```

> モジュール本体は **別スキーマ `tenancy`** に置くと物理的にも区切れる（`tenancy.space_lease` 等）。
> 本章では簡潔さのため修飾を省く。FK はモジュール有効時のみ張る（無効構成では seam 列は単なる未使用列）。

---

## 8.2 運用プロファイル（顧客差をここで吸収）

顧客の運用を 4 プロファイルに整理し、**設定テーブルで宣言**する。粒度・課金モデル・ポータル有無を
データで切り替えるので、同一コードベースで全パターンに対応できる。`tenant.kind` がこの分岐の起点で、
`internal` は A、`colo_customer` は B/C/D に対応する。

| プロファイル | 例 | 貸出粒度 | 課金 | ポータル(RLS) |
|------------|----|---------|------|--------------|
| **A. 単一運用者** | 自社・官公庁 DC | なし（`tenant`=部門タグのみ） | なし | 無効 |
| **B. ホールセール / エリア貸し** | ハイパースケール向け | suite / cage | 面積・kW | 任意 |
| **C. リテールコロ** | 一般コロ | cabinet / partial(U) | ラック・U・kWh 再課金 | 有効 |
| **D. パワー販売** | 高密度コロ | cage / cabinet | **kW コミット**中心 | 有効 |

```sql
CREATE TABLE tenancy_config (                 -- デプロイ1つにつき1行
    singleton boolean PRIMARY KEY DEFAULT true CHECK (singleton),
    enabled       boolean NOT NULL DEFAULT false,           -- モジュール自体の ON/OFF
    billing_model text NOT NULL DEFAULT 'space'
                  CHECK (billing_model IN ('none','space','rack','power','hybrid')),
    portal_rls    boolean NOT NULL DEFAULT false            -- テナントポータル提供有無
);
-- 許可する貸出粒度は配列(text[])を使わず明細テーブルで（LCD・MySQL移植可・[09章](./09-portability.md)）
CREATE TABLE tenancy_lease_granularity (
    granularity text PRIMARY KEY
        CHECK (granularity IN ('suite','cage','cabinet','partial'))
);  -- 例: INSERT 'cage','cabinet'  → 許可粒度。space_lease 作成時にサービス層で照合
```

> `portal_rls` は EcoStruxure IT の **Tenant Portal**（コロ顧客に自区画の電力・環境・容量を見せる）に相当する。
> 提供する顧客（C/D）だけ 8.6 の RLS を有効化し、提供しない顧客はコア機能のまま運用する。

`space_lease` 作成時に「その粒度が `tenancy_lease_granularity`（許可粒度の明細表）に含まれるか」を
サービス層で検証すれば、**B の顧客にうっかり partial(U貸し) を作らせない**といった運用ガードもデータ駆動でかけられる。

---

## 8.3 テナント階層

コアの `tenant` は最小（`id`/`name`/`kind`）。本章はその上に**リセラー/親会社/部門の階層**と
**契約ステータス**を加算的に足す（無効構成では不要）。

```sql
-- コア（03章 L2）のテナント（再掲・最小原始要素・本章では作らない）
-- CREATE TABLE tenant (
--     id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
--     name text NOT NULL UNIQUE,
--     kind text NOT NULL DEFAULT 'internal' CHECK (kind IN ('internal','colo_customer'))
-- );

-- 本章が足す拡張（任意）: リセラー・親会社・部門などの階層
CREATE TABLE tenant_group (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    parent_id bigint REFERENCES tenant_group(id) ON DELETE RESTRICT,
    name text NOT NULL,
    UNIQUE (parent_id, name)
);
ALTER TABLE tenant ADD COLUMN group_id bigint REFERENCES tenant_group(id) ON DELETE SET NULL;
ALTER TABLE tenant ADD COLUMN status   text NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','closed'));

-- 本章が足す denorm シームに FK（モジュール有効時）
ALTER TABLE series     ADD CONSTRAINT series_tenant_fk
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE SET NULL;
ALTER TABLE power_feed ADD CONSTRAINT feed_tenant_fk
    FOREIGN KEY (tenant_id) REFERENCES tenant(id) ON DELETE SET NULL;
-- equipment.tenant_id / location.tenant_id の FK はコア（03章 L2/L3）が既に張る
```

- **A プロファイル**では `kind='internal'` の `tenant` を「部門」として使うだけ（lease なし）。`tenant_group` で組織階層を表現。

---

## 8.4 賃貸（space_lease）— 粒度横断・期間付き

物理階層から分離した第一級エンティティ。粒度は **排他アーク**（[06章](./06-self-review.md) で推奨した
多態回避＝実 FK 化）で表す。**cage は既存 `location`（`loc_type='cage'`）**に乗るので、配下の集約クエリが
そのまま効く。

> **コロのテナント境界＝cage（EcoStruxure）**: EcoStruxure IT はコロケーションのテナント境界を
> **room 内の cage** としてモデル化する。コア（[03章 L2](./03-finalists.md)）の `loc_type` は `dataCenter`/`cage` を
> 含み、`Room → cage → Row → Rack` の木で区画を表せる。cage の所有は `location.tenant_id` に直接乗る。

> **コミット容量＝契約電力（EcoStruxure）**: `committed_kw` は EcoStruxure の **contracted power 負荷戦略**の
> 契約面（区画単位）にあたる。機器面の契約電力は [03章 L8](./03-finalists.md) の
> `equipment_demand.load_strategy='contracted'` / `equipment_demand.contracted` が持つ。両者は同じ「契約電力」概念の
> 区画粒度 / 機器粒度の2面で、8.7 のキャパシティ（コミット vs 実測）で突き合わせる。

> **LCD 化（[09章](./09-portability.md)）**: `tstzrange`/`int4range`/`EXCLUDE` を使わず、
> 期間は `valid_from`/`valid_to` の2列、U は `lo_u`/`hi_u` の2列、二重貸し防止は **占有U行 + 複合UNIQUE**で表現
> （MySQL でも同形）。期間重なりの厳密判定（valid 込み）はサービス層で担保する。

```sql
CREATE TABLE space_lease (
    id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- MySQL: AUTO_INCREMENT
    tenant_id bigint NOT NULL REFERENCES tenant(id) ON DELETE RESTRICT,
    granularity text NOT NULL CHECK (granularity IN ('suite','cage','cabinet','partial')),
    -- 排他アーク: suite/cage→location、cabinet/partial→rack
    location_id bigint REFERENCES location(id) ON DELETE RESTRICT,
    rack_id     bigint REFERENCES rack(id)     ON DELETE RESTRICT,
    lo_u int, hi_u int,                                    -- partial のときだけ（range型を使わず2列）
    valid_from timestamptz NOT NULL DEFAULT now(),         -- 契約期間（range型を使わず2列）
    valid_to   timestamptz,                                -- NULL=継続中、解約時に設定
    -- コミット（販売した容量）= EcoStruxure contracted power の区画面
    committed_kw      numeric(8,2) CHECK (committed_kw  >= 0),
    committed_u       int          CHECK (committed_u   >= 0),
    committed_area_m2 numeric(8,2) CHECK (committed_area_m2 >= 0),
    -- 課金
    mrc numeric(12,2), nrc numeric(12,2), billing_ref text,        -- 月額/初期/外部課金ID
    CHECK ( valid_to IS NULL OR valid_to > valid_from ),
    -- 粒度と参照先の整合（排他アーク）
    CHECK ( (granularity IN ('suite','cage')) = (location_id IS NOT NULL) ),
    CHECK ( (granularity IN ('cabinet','partial')) = (rack_id IS NOT NULL) ),
    CHECK ( (granularity = 'partial') = (lo_u IS NOT NULL AND hi_u IS NOT NULL) )
);
-- granularity が tenancy_lease_granularity（許可粒度の明細表）に含まれるかはサービス層で検証

-- シェアドラックの二重貸し防止（現に有効なリースのU占有を1行ずつ展開して複合UNIQUE）
-- ※「期間重なり」の厳密判定はサービス層で行い、ここは「現行有効な占有」の一意性を担保
CREATE TABLE lease_unit_occupancy (
    rack_id  bigint NOT NULL REFERENCES rack(id),
    u        int    NOT NULL,
    lease_id bigint NOT NULL REFERENCES space_lease(id) ON DELETE CASCADE,
    PRIMARY KEY (rack_id, u)          -- 同一ラックの同一Uは現行1リースのみ（partial/cabinet 混在も防止）
);
```

設計のキモ:
- **期間付き（`valid_from`/`valid_to`）**：解約・移転の履歴が残る。過去月の再課金（8.7）が正確（時間軸を持つ履歴設計と親和）。
- **cage は location に同居**：`loc_type='cage'`（必要なら `suite` も loc_type に追加）。テナント区画の階層化も `location` の木でそのまま表現し、`location.tenant_id` が cage の所有を直接持つ。
- **committed_* と実測の差＝販売余地**（8.7 のキャパシティ）。これは EcoStruxure の契約電力 vs 実測の stranded に対応。

---

## 8.5 所有 vs 設置の整合

- `equipment.tenant_id`（コア・[03章 L3](./03-finalists.md)） = 資産所有者（コロ顧客）。**床・ラックは DC 所有、中身はテナント所有**を表せる。
- `location.tenant_id`（コア・[03章 L2](./03-finalists.md)） = cage 等の区画所有者。lease を導入しない軽量構成では、この2列の直接突合だけで区画整合を取れる。
- **整合チェック（監視ビュー/サービス層）**：機器の設置位置（rack_mount → rack/location）を**覆う有効 lease のテナント**（または cage の `location.tenant_id`）と
  `equipment.tenant_id` が一致するか。不一致＝「他人の区画に置かれた機器」を検出。

```sql
-- 区画違反の検出（監視ビュー）: 機器の所有者が、その設置場所の有効リース先と異なる
-- LCD: 期間は valid_from/valid_to の2列比較、配下判定は location_closure JOIN（ltree不使用・[09章](./09-portability.md)）
CREATE VIEW v_lease_placement_violation AS
SELECT e.id AS equipment_id, e.tenant_id, rm.rack_id
FROM equipment e
JOIN rack_mount rm ON rm.equipment_id = e.id
JOIN rack rk ON rk.id = rm.rack_id
LEFT JOIN space_lease l
  ON l.valid_from <= now() AND (l.valid_to IS NULL OR l.valid_to > now())   -- 有効期間
 AND l.tenant_id = e.tenant_id
 AND ( (l.granularity IN ('cabinet','partial') AND l.rack_id = rm.rack_id)  -- U範囲詳細はサービス層で判定
    OR (l.granularity IN ('suite','cage')
        AND EXISTS (SELECT 1 FROM location_closure lc
                    WHERE lc.ancestor_id = l.location_id
                      AND lc.descendant_id = rk.location_id)) )
WHERE e.tenant_id IS NOT NULL AND l.id IS NULL;   -- 覆う自テナントのリースが無い
```

---

## 8.6 データ分離（テナントポータル提供時のみ・RLS）

`tenancy_config.portal_rls = true` の顧客だけ Row-Level Security を有効化する
（EcoStruxure IT の **Tenant Portal** 相当をコロ顧客に開放する場合）。

```sql
ALTER TABLE equipment ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON equipment
  USING (tenant_id = current_setting('app.tenant_id', true)::bigint);
-- series も同様に tenant_id で。アプリは接続時に SET app.tenant_id = '<id>'
```

⚠️ **重要な落とし穴**：`measurement`（hypertable）に**直接 RLS を張ると continuous aggregate が壊れる**
（CAGG の定義元にポリシーがあると不可）。時系列のテナント絞り込みは **RLS でなく `series.tenant_id` を
JOIN/WHERE で**行う。RLS は `equipment`/`series`/構成系テーブルに限定する。

---

## 8.7 再課金・キャパシティ（テナント軸）

### テナント別 月次 kWh（[04章 UC-1](./04-validation-queries.md) のテナント版）

`series.tenant_id`（非正規化）で割るだけ。共有環境は機器所有で按分、専用フィードは `power_feed.tenant_id`。
メトリックの絞り込みは [03章 L4](./03-finalists.md) の**フラット `metric` カタログ**の `code` 列で行う（旧 3軸 `point_type` は廃止・`series` が `metric_id` を非正規化保持）。

```sql
WITH per_series AS (
    SELECT s.tenant_id, s.series_id,
           sum(d.sum_v) / NULLIF(sum(d.n), 0) AS avg_w,   -- 1d ロールアップ avg=Σsum_v/Σn（[10章](./10-room-group-derived-metrics.md)）
           count(*) AS n_days
    FROM measurement_1d d
    JOIN series s  USING (series_id)
    JOIN metric m  ON m.id = s.metric_id AND m.code = 'active_power'   -- 03章 L4: フラットカタログ
    WHERE d.bucket >= date_trunc('month', :as_of)
      AND d.bucket <  date_trunc('month', :as_of) + INTERVAL '1 month'
      AND s.tenant_id IS NOT NULL
    GROUP BY s.tenant_id, s.series_id
)
SELECT tenant_id,
       sum(avg_w * n_days * 24) / 1000.0                              AS kwh_month,
       round((sum(avg_w * n_days * 24)/1000.0 * :jpy_per_kwh)::numeric) AS jpy_energy
FROM per_series
GROUP BY tenant_id;
```

> **計測境界は room_group と一致しうる（[10章](./10-room-group-derived-metrics.md)）**: 個別計量された cage は
> 電力境界としての `room_group` に切り出せる。`series.room_group_id` と `series.tenant_id` は同じ測定流を
> **空間境界 / 契約帰属**の2軸で分割しており、per-cage 計測と per-tenant 再課金は同一の集約パイプで導出できる。

### テナント別キャパシティ（コミット vs 実測 → 販売余地）

```sql
-- 契約電力(committed_kw) と 実測ピーク/平均 の対比。stranded = 売ったが使われていない容量
-- EcoStruxure の contracted power 戦略 ＋ WP-150 の stranded = reserved − actual（[03章 L8](./03-finalists.md)）
SELECT l.tenant_id,
       sum(l.committed_kw)                       AS committed_kw,
       (/* 上の per_series を kW 平均にした実測 */) AS used_kw_avg,
       sum(l.committed_kw) - (/* used */)        AS stranded_kw   -- 追加販売 or 是正の余地
FROM space_lease l
WHERE l.valid_from <= now() AND (l.valid_to IS NULL OR l.valid_to > now())
GROUP BY l.tenant_id;
```

機器粒度では同じ突合を [03章 L8](./03-finalists.md) の `equipment_demand`（`load_strategy='contracted'` の
`contracted` vs `estimated`）で行える。区画粒度（lease）と機器粒度（equipment_demand）の双方で契約電力 vs 実測を見る。

### 販売可能在庫（vacant）

`rack.u_height` − 有効 partial lease の占有 U、`location` 内の未リース cage（`location.tenant_id IS NULL`）等で
「売れる区画」を一覧化。

---

## 8.8 付随モジュール（任意・さらに区切る）

| 機能 | 表 | いつ入れる |
|------|----|-----------|
| クロスコネクト | `cable` 両端に tenant、MMR=`loc_type='mmr'`（local 拡張・suite 同様に追加） | キャリア接続を売る場合 |
| アクセス制御 | `access_grant(tenant_id, location_id\|rack_id, person_id, valid)` | ケージ入退室を管理する場合 |
| SLA/契約メタ | `lease_term`/`sla` を `space_lease` に紐付け | 契約管理を DCIM に寄せる場合 |

---

## 8.9 ER 図（テナントモジュール）

```mermaid
erDiagram
    TENANT_GROUP ||--o{ TENANT_GROUP : "parent (self)"
    TENANT_GROUP ||--o{ TENANT : groups
    TENANT ||--o{ SPACE_LEASE : leases
    LOCATION ||--o{ SPACE_LEASE : "suite/cage (arc)"
    RACK ||--o{ SPACE_LEASE : "cabinet/partial (arc)"
    TENANT ||--o{ EQUIPMENT : "owns (equipment.tenant_id, core L3)"
    TENANT ||--o{ LOCATION : "owns cage (location.tenant_id, core L2)"
    TENANT ||--o{ SERIES : "denorm (tenant_id, nullable seam)"
    TENANT ||--o{ POWER_FEED : "dedicated feed (nullable seam)"
    SPACE_LEASE ||--o{ LEASE_UNIT_OCCUPANCY : "partial U occupancy"

    TENANCY_CONFIG {
        boolean singleton PK
        boolean enabled
        text billing_model "none|space|rack|power|hybrid"
        boolean portal_rls "EcoStruxure Tenant Portal 相当"
    }
    TENANCY_LEASE_GRANULARITY {
        text granularity PK "許可粒度(配列の代わりに明細表・LCD)"
    }
    TENANT_GROUP {
        bigint id PK
        bigint parent_id FK "self"
        text name
    }
    TENANT {
        bigint id PK
        text name UK
        text kind "internal|colo_customer (core L2)"
        bigint group_id FK "module ext"
        text status "active|suspended|closed (module ext)"
    }
    SPACE_LEASE {
        bigint id PK
        bigint tenant_id FK
        text granularity "suite|cage|cabinet|partial"
        bigint location_id FK "arc: suite/cage"
        bigint rack_id FK "arc: cabinet/partial"
        int lo_u "partial only (range型を使わず2列)"
        int hi_u "partial only"
        timestamptz valid_from "契約期間(2列・LCD)"
        timestamptz valid_to "NULL=継続"
        numeric committed_kw "contracted power(区画面)"
        numeric committed_u
        numeric mrc
        numeric nrc
    }
    LEASE_UNIT_OCCUPANCY {
        bigint rack_id PK
        int u PK
        bigint lease_id FK "複合UNIQUEで二重貸し防止(LCD)"
    }
    EQUIPMENT {
        bigint id PK
        bigint tenant_id FK "core seam (L3)"
    }
    LOCATION {
        bigint id PK
        bigint tenant_id FK "core seam: cage 所有 (L2)"
    }
    SERIES {
        bigint series_id PK
        bigint tenant_id FK "nullable seam (RLS/billing)"
    }
    POWER_FEED {
        bigint id PK
        bigint tenant_id FK "nullable seam"
    }
```

---

## 8.10 まとめ：どう「区切った」か

1. **コアは最小原始要素のみ**：コア（[03章](./03-finalists.md)）が型付きで持つのは `tenant(id/name/kind)` ＋
   `location.tenant_id` ＋ `equipment.tenant_id`。本章が足す接点は denorm 列 2 つ（`series.tenant_id` / `power_feed.tenant_id`）のみで、
   重い機構（賃貸・課金・分離）は別スキーマ `tenancy` に隔離可。テナント無効構成では存在しないのと同じ。
2. **運用差は `tenancy_config` で宣言**：A 単一運用 / B エリア貸し / C リテールコロ / D パワー販売を、
   **許可粒度（`tenancy_lease_granularity`）・課金モデル・ポータル(RLS)** の組合せで表現。`tenant.kind` が分岐の起点。コードは共通。
3. **賃貸は粒度横断の 1 エンティティ**：`space_lease` の排他アーク + 期間(2列) + 占有行UNIQUE で
   suite/cage/cabinet/partial と二重貸し防止を一括。cage は既存 `location`（`loc_type='cage'`、EcoStruxure のコロ境界）に同居し集約が効く。
4. **再課金・キャパシティはテナント軸の派生**：既存 UC を `metric.code` ＋ `series.tenant_id` で割るだけ。
   コミット（契約電力）vs 実測で販売余地（stranded）を出し、機器粒度は [03章 L8](./03-finalists.md) の
   `equipment_demand.contracted`（EcoStruxure contracted power 戦略）と対応。per-cage 計測は [10章](./10-room-group-derived-metrics.md) の `room_group` 境界と一致。

> これにより [06章](./06-self-review.md)（マルチテナント/コロの浅さ）は **「分離可能モジュールとして解決方針あり」** に更新。
> 顧客ごとに違う運用は、**スキーマ分岐ではなく設定（プロファイル）で吸収**するのが本設計の肝。
