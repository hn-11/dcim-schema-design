# 01. ドメイン知識 & 既存製品調査 & 設計軸

本書は 20 パターン検討の土台となる (1) データセンターのドメイン知識、(2) 既存 DCIM 製品の
データモデル、(3) スキーマ設計の選択肢（設計軸）を整理する。

---

## 1. データセンター物理ドメイン知識

### 1.1 空間階層と粒度

一般的な包含階層（上位が下位の親）:

```
Region/Metro → Campus → Building → Floor → Room(Data Hall) → [Cage] → Row → Rack → RackUnit(U)
```

- **Region/Campus/Building/Floor**: 地理・建屋。DC によって有無・段数が異なる（小規模 DC は建屋・フロア概念が薄い）。
- **Room / Data Hall**: IT 機器が載るホワイトスペース。ほかに MEP/プラント室、MMR（meet-me room）、バッテリ室など `room_type` で区別。
- **Cage**（コロケーション）: データホール内のテナント占有区画（メッシュ囲い）。テナント FK・割当面積・割当電力を持つ。
- **Row**: ラックの列。ホット/コールドアイルの整列単位。
- **Rack / Cabinet**: 機器を収める筐体。
- **RackUnit (U)**: ラック内の垂直マウント位置。**EIA-310: 1U = 44.45mm**。機器高さは U の整数倍。

ラック寸法の業界標準値（制約のデフォルト値に使える）:
- レール幅: **19 インチ**（標準）/ 23 インチ（通信キャリア）/ OCP は 21 インチ
- ラック高さ: **42U**（標準）/ 45U / 48U（高密度）、ハーフラック 24U 等
- 床荷重: データセンターは **≥1,000 kg/m²（1 t/m²）**、高密度で 1.5〜2.0 t/m²

### 1.2 電力チェーンと冗長

電力経路（各要素が上流／下流リンクを持つ有向グラフ）:

```
Utility → ATS/STS(⇄ Generator) → Switchgear → UPS(+battery) → PDU/RPP → [Busway] → Rack PDU(CDU) → Outlet → Device PSU
```

- **ATS**（自動切替・数百ms〜秒）/ **STS**（半導体・4〜8ms）: 2 入力 1 出力の切替器。単電源機器に冗長を与える。
- **冗長ティア**: **N**（余裕なし）/ **N+1**（予備1）/ **2N**（A系・B系の完全独立2系統）/ **2N+1**。
- **A系/B系フィード**: 2 つの独立配電経路。**dual-corded 機器**は PSU を A・B 双方に接続。
- **相**: 単相 / 三相（DC 標準、L1/L2/L3 バランス）。
- **電圧/電流**: ラック PDU 入力は 120/208/230/240V 単相、208/400-415V 三相。ブレーカは 15/20/30/50/60/100A。
- **NEC 80% ルール**: 連続負荷（≥3時間）はブレーカ定格の 125% で選定 → **使用可能は定格の 80%**。NetBox の `PowerFeed.max_utilization` デフォルトが 80。

### 1.3 冷却・環境

- **CRAC**（DX 直膨）/ **CRAH**（冷水・チラー連携、高効率）/ **Chiller**（冷水製造）。
- **ホット/コールドアイル**、**封じ込め**（CAC/HAC）、**RDHx**（リアドア熱交換器、高密度向け）。
- **ASHRAE TC9.9 サーマルガイドライン**（温湿度のバリデーション境界として encode）:
  - **推奨（全クラス）**: 18〜27℃、露点 −9〜15℃、≤60%RH
  - **許容**: A1 15〜32℃ / A2 10〜35℃ / A3 5〜40℃ / A4 5〜45℃
  - 測定点はラック前面（サーバ吸気口、コールドアイル）。
- **冷却換算**: 1 ton = 12,000 BTU/h = 3.52 kW。

### 1.4 キャパシティの「3 状態」

成熟した DCIM は各次元（電力 kW / U / 重量 / 冷却 / ポート）で次の 3 状態を区別する:

1. **Rated / Budget / Nameplate**（定格・設計上限）— 銘板値は実消費の 2〜3 倍が普通で、銘板基準だと容量を死蔵する。
2. **Reserved / Provisioned**（予約・計画確保）— 設置計画時にブロックされる空間/電力/ポート（NetBox `RackReservation`、Sunbird の auto-reserve）。
3. **Actual / Measured**（実測）— メータ付き PDU/CDU からの SNMP 実測。
4. **Stranded capacity（死蔵容量）= Reserved − Actual**（または別次元がボトルネックで使えない容量）— DCIM の主要 KPI。

`PUE = Total Facility Energy / IT Equipment Energy`（理想 1.0）。総入力電力と IT 負荷合計のメータリングが必要。

### 1.5 収集プロトコル

| プロトコル | トランスポート | モデル | 自己記述 | 点の識別子 | 主な機器 |
|---|---|---|---|---|---|
| SNMP v1/2c/3 | UDP 161/162 | ポーリング+trap | あり(MIB) | OID `1.3.6.1…` | PDU, UPS, network, sensor |
| Modbus RTU/TCP | RS-485 / TCP 502 | ポーリング | なし(ベンダmap) | レジスタアドレス+FC | 電力計, PDU, CRAC, BMS |
| IPMI | LAN UDP 623 | ポーリング+SEL | 一部(SDR) | sensor番号/SDR | server BMC |
| Redfish | HTTPS/JSON | ポーリング+Telemetry | あり(OData/JSON Schema) | リソースURI+property | server BMC, chassis, thermal |
| BACnet | IP 47808 / MS/TP | ポーリング+COV | あり(object) | object型+instance+property | HVAC, CRAC, chiller |
| MQTT / REST | TCP 1883/8883, HTTPS | **プッシュ** | アプリ定義 | topic / JSON path | IoT sensor, gateway |

代表メトリクスと単位: 電圧(V)、電流(A)、有効電力(W/kW)、皮相電力(VA/kVA)、力率(0-1)、電力量(kWh)、周波数(Hz)、温度(℃)、湿度(%RH)、露点(℃)、ファン回転(RPM)、気流(CFM)、接点(discrete)、漏水(discrete)、PUE(比)。

**ポーリングは欠測・タイムアウトが前提**（サンプル品質を記録）。trap/COV/Redfish Trigger 等の**イベントは別ストリーム**として扱う。Modbus のスケール係数・単位は取込時に正規化（canonical SI）。

---

## 2. 既存 DCIM 製品のデータモデル

### 2.1 NetBox（OSS, Django/PostgreSQL）— 本検討の主要参照

- **組織グルーピングと物理包含を分離**: `Region`/`SiteGroup`（再帰ツリー）→ `Site`（物理アンカー）→ `Location`（無限ネスト可、`parent` FK + `site` FK）→ `Rack` → `Device`。
- **`Location` は隣接リスト + ネストツリー（django-mptt, MPTT=入れ子集合系）** で部分木クエリを高速化。
- **DeviceType / Device 分離**: 型番（`DeviceType`）が U 高さ・フルデプス・重量・ポート構成のテンプレート（`InterfaceTemplate`/`PowerPortTemplate` 等）を持ち、実機 `Device` 作成時に展開。**新機種追加 = データ 1 行追加（DDL 不要）**。`devicetype-library` に数千機種が YAML 定義済み。
- **配線は種別別ポート + Cable/CableTermination**: `PowerPort`(inlet)/`PowerOutlet`/`Interface`/`FrontPort`/`RearPort` と `Cable`+`CableTermination`（A/B 両端 M2M、多態 GenericFK）。end-to-end トレースは `CablePath` 非正規化キャッシュ。
- **電力**: `PowerPanel` → `PowerFeed`(type=primary/redundant, phase, voltage, amperage, max_utilization=80, **available_power** 計算列: 単相 `V×A×util/100`、三相は `×1.732`)。`PowerOutlet.upstream_power_port` で機内分配。
- **キャパシティ**: `Rack.get_utilization()`（reserved を消費扱い）、`get_power_utilization()`（PowerPort 合計 / feed available_power）。
- **設計教訓**: 葉コンポーネントに site/location/rack を**意図的に非正規化**してフィルタ高速化。`available_power` は V×A が 32767 を超え overflow した（列型サイズの教訓）。

### 2.2 OpenDCIM（OSS, PHP/MySQL）

- **固定階層**: `fac_Container`(自己ネスト) → `fac_DataCenter`(MaxkW) → `fac_Zone` → `fac_CabRow` → `fac_Cabinet`(CabinetHeight, MaxKW, MaxWeight) → `fac_Device`(Position, Height, BackSide, HalfDepth)。子は親 ID を**非正規化保持**。
- **テンプレート駆動**: `fac_DeviceTemplate` / `fac_CDUTemplate`(SNMP OID 定義、ATS フラグ) / `fac_SensorTemplate`(TempOID 等)。
- **電力チェーン**: `fac_PowerPanel`(ParentPanelID で木) → `fac_PowerDistribution`(rack PDU) → `fac_PowerConnection`(outlet↔device PSU)。**A/B 冗長給電が一級**: PDU が `PanelID`+`PanelID2`、`PanelPole`+`PanelPole2`、`FailSafe`。
- **重大な弱点**:
  - **宣言的 FK がほぼ無い**（参照整合性は PHP 層頼み、孤児行が出やすい）。未設定が `0`/空文字で NULL と曖昧。
  - **時系列が無い**: `fac_PDUStats`・`fac_SensorReadings` は最新値 1 行のみ。履歴トレンドは外部ストレージが必要 → **本検討の TimescaleDB 前提の正当化**。
  - sensor/PDU を `fac_Device` に相乗り（テーブルが横に広い）。
  - 接続がポート行に内包（`ConnectedDeviceID`）でケーブル実体が無い。
  - `varchar` の擬似 enum 多用。

### 2.3 商用製品

- **Sunbird dcTrack**: 実測ベースの **Auto Power Budget**（機種銘板でなく実 outlet 実測で予算化 → 死蔵容量回収）。計画品目で空間/電力/データ接続を auto-reserve。
- **Device42**: CMDB+DCIM+IPAM 統合。**Building → Room → Rack**（Row は情報のみ・真の親オブジェクトでない）。**Device は IP を持つ / Asset は持たない**で二分。Hardware Model テンプレート駆動。電力・ケーブルを end-to-end グラフ化（Cable の端点は patch panel port / switch port / device / circuit / 別 cable）。**discovery-first**（SNMP/CDP/LLDP で自動投入）。DOQL（読み取り専用 PostgreSQL）。
- **Nlyte**: 履歴ドローに ML 需要予測。
- **Redfish TelemetryService**: `MetricDefinition`（メトリクス定義+単位）/ `MetricReportDefinition`（何をいつ）/ `MetricReport`（実データ）/ `Triggers`（閾値）。**定義とサンプルの分離**は本検討の収集定義設計の手本。

---

## 3. 設計軸（Axis）の定義

20 パターンは以下 5 軸（+ 補助軸）の選択肢の組み合わせとして構成する。

### Axis S — 空間階層

| 記号 | 方式 | 任意深度 | 部分木クエリ | 移動 | 制約 | 備考 |
|---|---|---|---|---|---|---|
| **S1** | 隣接リスト（parent_id） | ◎ | △再帰 | ◎ | ◎ FK自己参照 | 真実源として堅牢 |
| **S2** | 経路列挙 / `ltree`（隣接+派生path） | ◎ | ◎ GiST | △ path一括更新 | △ 二重管理 | 読み多に好相性 |
| **S3** | 閉包テーブル（Closure） | ◎ | ◎ 再帰不要 JOIN | △ エッジ再生成 | ◎ | クエリ最速・行膨張 |
| **S4** | 固定レベル（Site/Bldg/…/Rack 別表） | ✕ 段数固定 | ◎ 明示JOIN | ◎ | ◎ 各層に専用CHECK | 制約最強・可変性弱 |
| **S5** | ハイブリッド（Rack 固定 + 汎用 Location 木） | ◎ | ◎ | ○ | ◎ | **本命** |

入れ子集合（Nested Set）単独は機器の出入りが頻繁な DCIM では更新コスト過大で却下。

### Axis A — 機器マスタ（Asset）

| 記号 | 方式 | 型安全 | 種別追加 | 制約 | クエリ | 備考 |
|---|---|---|---|---|---|---|
| **A1** | Single Table + JSONB | △ JSON Schema | ◎ DDL不要 | ○ | ○ GIN | 可変性最大 |
| **A2** | Class Table Inheritance（共通+種別子表） | ◎ | △ 新表+デプロイ | ◎ | △ JOIN | 整合最強 |
| **A3** | DeviceType/Device 分離（NetBox 流） | ◎ | ◎ データ追加 | ◎ | ◎ | **核心** |
| **A4** | A3 + カテゴリ別 CTI（固有属性を型付け） | ◎ | △ | ◎ | ○ | 制約最強の A3 |
| **A5** | A3 + JSONB（コア列+ロングテール JSONB） | ◎/△ | ◎ | ○ | ○ | **バランス本命** |

EAV と PostgreSQL テーブル継承（`INHERITS`、UNIQUE/FK が継承されない）はアンチパターンとして却下。

### Axis C — 配線トポロジー

| 記号 | 方式 | 型安全 | 制約 | 再帰トレース | 備考 |
|---|---|---|---|---|---|
| **C1** | 汎用接続 1 本（a_port,b_port） | ✕ | △ | ◎ 最簡 | 誤接続を DB で防げない |
| **C2** | 電力用/ネット用テーブル分離 | ◎ | ◎ | ○ | UNION 必要 |
| **C3** | 種別別ポート + Cable/CableTermination（NetBox 流） | ◎ | ◎ | △→path cache | **実績本命** |
| **C4** | 種別別ポート + 軽量有向エッジ（両端1行） | ◎ | ◎ CHECK | ○ | ループ/向きを CHECK で |
| **C5** | エッジテーブル + WITH RECURSIVE + path cache | ○ | ○ | ◎ | グラフ第一級 |

### Axis T — テレメトリ / TimescaleDB

| 記号 | 方式 | 書込 | 新メトリック | 圧縮 | 備考 |
|---|---|---|---|---|---|
| **T1** | Narrow `(time, series_id, value)` + series 台帳 | ◎ | ◎ 1行INSERT | ◎ segmentby=series_id | **本命** |
| **T2** | Wide `(time, device_id, 各列)` | ○ | ✕ ALTER TABLE | ○ | 列固定向き |
| **T3** | メトリック種別ごと hypertable | ◎ | △ 新表 | ◎ | 物理量少なく固定なら |
| **T4** | JSONB payload | ○ | ◎ | ✕ | クエリ性能出ず却下 |

### Axis G — 集約 / キャパシティ

| 記号 | 方式 | 鮮度 | 履歴正確性 | 備考 |
|---|---|---|---|---|
| **G1** | オンザフライ階層 JOIN + time_bucket | 常に最新 | 現在マッピング | 高負荷で詰まる |
| **G2** | CAGG with JOIN（2.10+/2.16+ で緩和） | refresh ラグ | **次元変更を追わない** | 罠あり |
| **G3** | 非正規化キー + JOIN なし CAGG + 階層 CAGG | 準リアルタイム | ingest 時点固定（実質 Type-2） | **本命** |
| **G4** | マテリアライズ job（analytics-batch 等） | 書込ラグ | 任意制御 | 自前実装多い |
| **G5** | Type-2 時系列配置（valid_from/to）+ job | 任意 | **真の point-in-time** | 課金正確性向け |

**補助軸**:
- **U 位置**: `int4range` + `EXCLUDE USING gist`（`btree_gist`）で物理重なり禁止（フルデプス両面畳み込み・左右サイド独立）。
- **時間ロールアップの正確性**: `sum/min/max/count` は再集約安全。`avg` は `sum+count` か `stats_agg`+`rollup`。**p95 は `percentile_agg`（sketch）保存 + `rollup` + `approx_percentile`**（スカラ値の再集約は誤り）。`last/first` は `candlestick_agg` か timestamp 同梱で。
- **最新値**: 専用 `current_value` 表 UPSERT（最速・書込2倍）/ `DISTINCT ON`+SkipScan（メンテ不要）/ CAGG `last()`（履歴付き）。

---

## 4. RDBMS 制約に落とすべき業務ルール（ドメインナレッジの結晶）

パッケージとして「ドメインナレッジを活かした制約」を効かせる候補。**★=DB制約で表現可、▲=トリガ/監視ビュー（集約制約のため単一行 CHECK 不可）**:

**空間・搭載**
- ★ 機器の `position + u_height - 1 ≤ rack.u_height`（ラックに収まる）→ 親値参照のため CHECK 関数/トリガ
- ★ 同一ラック・同一面・同一サイドで U 範囲が重ならない（`EXCLUDE USING gist`）
- ★ ラックは 1 つの row/room に、cage は 1 テナントに属す（FK + NOT NULL）
- ▲ 機器の気流方向が row のホット/コールドアイル整列と整合（逆向き警告）

**電力**
- ▲ 回路の接続機器 draw 合計 ≤ ブレーカ定格（連続負荷は ×0.8）
- ▲ ラックの A 給電と B 給電が異なる上流（PDU/UPS/panel）に由来（同一なら SPOF 違反）
- ★ dual-corded 機器の 2 PSU は別サイド（A/B）の outlet に接続
- ★ outlet/コネクタ型と機器電源コードのプラグ型が一致（C14→C13 等）
- ★ ポートは高々 1 ケーブルに終端（`UNIQUE(term_kind, term_id)`）、自己ループ禁止
- ★ available_power は `V×A×util/100`（三相 ×√3）、列型は V×A の overflow を考慮（int4 以上）

**冷却・環境**
- ▲ センサ温湿度が機器の ASHRAE クラス許容域内（推奨域外→warn、許容域外→alert）
- ▲ ラック発熱(kW) ≤ そのラック/row/room を賄う冷却容量

**重量・構造**
- ▲ ラック内機器重量合計 ≤ ラック最大荷重
- ▲ ラック点荷重（重量÷footprint）≤ 床/タイル荷重定格

**キャパシティ全般**
- ▲ 各レベルで Allocated ≤ Design（電力/冷却/U/ポート/床荷重）— 超過は reject か flag
- ★ cage 割当 ≤ 親データホールがその cage にプロビジョニングした量

---

## 5. 出典（主要）

**ドメイン**: EIA-310 (RackSolutions), Equinix/Dgtl Infra (cage/cabinet), CoreSite/MEP Academy (N+1/2N), SemiAnalysis (電気系統), ASHRAE TC9.9 (airatwork PDF, CKY), Green Grid/LBNL WP49 (PUE), Uptime Institute (Tier), NEC 80%ルール (TRG/Schneider)。
**プロトコル**: RFC 1628 (UPS-MIB), DMTF DSP0266/DSP2051 (Redfish), ANSI/ISO 16484-5 (BACnet), mqtt.org/OASIS, OCP Open Rack V3。
**製品**: NetBox source (`racks.py`/`power.py`/`sites.py`), NetBox docs (DeviceType/PowerFeed/Location/Cable), OpenDCIM `create.sql`, Device42 docs, Sunbird dcTrack。
**PostgreSQL/TimescaleDB**: 各設計ドキュメント末尾に詳細 URL を記載。
