# 01. ドメイン知識 & 参照モデル調査 & reify パターン

本書は確定設計（[03章](./03-finalists.md)）の土台となる (1) データセンターの物理ドメイン知識、
(2) 参照する既存モデル（**Project Haystack** / **EcoStruxure IT** / NetBox / Brick）、
(3) セマンティックなオントロジを **RDBMS の制約に reify（実体化）**するための設計パターンを整理する。

> 設計ゴール: **Project Haystack のセマンティクス**（点の意味を直交タグの合成で表す）を RDBMS の参照表＋FK＋CHECK に
> 落とし、**EcoStruxure IT のドメインモデル**（power path / Genome / 容量管理 / 冷却）を型付きで取り込み、
> 「**ゆるい IoT スキーマにせず、ドメイン知識を DB 制約で担保する**」。

---

## 1. データセンター物理ドメイン知識

### 1.1 空間階層

```
Region/Metro → Campus → Building → Floor → Room(Data Hall) → [Cage] → Row → Rack → RackUnit(U)
```

- **Room / Data Hall**: IT 機器が載るホワイトスペース。MEP/プラント室・MMR・バッテリ室は `room_type` で区別。
- **Cage**（コロケーション）: データホール内のテナント占有区画。テナント所有・割当面積・割当電力を持つ（[08章](./08-tenancy-colocation.md)）。
- **Row / Rack / RackUnit(U)**: 列・筐体・垂直マウント位置。**EIA-310: 1U=44.45mm**、19 インチ標準、42U 標準。
- 床荷重: **≥1,000 kg/m²**、高密度で 1.5〜2.0 t/m²。
- EcoStruxure IT の確認済み階層は **Location → Room（room design）→ Row / Cage → Rack → Device（→ blade enclosure → blade）**。
  上位（Company/Building）は製品上は明示されず Location がルート。

### 1.2 電力チェーンと冗長

```
Utility → ATS/STS(⇄ Generator) → Switchgear → UPS(+battery) → PDU/RPP → [Busway] → Rack PDU → Outlet → Device PSU
```

- **冗長ティア**: N / N+1 / 2N（A系・B系完全独立）/ 2N+1。**A/B フィード**、**dual-corded 機器**は PSU を A・B 双方に接続。
- EcoStruxure は電力経路を **power path**（"power supplier"上流 / "power consumer"下流）として検証
  （"An Invalid Power Path has been Configured"）。**breaker panel → breaker（位置・容量を厳密化し stranded を削減）**、
  rack PDU の bank/outlet、**paired power consumers**（A/B 二重給電の冗長ペア）、branch circuit monitoring が一級。
- **NEC 80% ルール**: 連続負荷は使用可能をブレーカ定格の 80% に。三相は √3 倍。

### 1.3 冷却・環境

- **CRAC**（DX 直膨）/ **CRAH**（冷水コイル + チラー連携）/ **Chiller** / **Cooling Tower** / **CDU** / in-row / RDHx。
- EcoStruxure は冷却容量を **rack / row / room の3レベル**で評価し、**room-based / row-based / rack-based cooling**、
  封じ込め（contained aisle）、気流（既定 160 CFM/kW・perforated tile ~700 CFM）をモデル化。冷却冗長は **N+x**。
- **ASHRAE TC9.9**（温湿度のバリデーション境界）: 推奨 18〜27℃、許容 A1 15〜32 / A2 10〜35 / A3 5〜40 / A4 5〜45℃。測定はラック前面吸気。

### 1.4 キャパシティ（WP-150 分類）

APC ホワイトペーパー150（Schneider）の**容量5要素**: **空間（床/ラック）/ 電力 / 電力分配 / 冷却 / 冷却分配**
（電力・冷却はそれぞれ source 容量と distribution 容量に分かれる）+ 重量 / ネットワークポート。

過剰容量分類（WP-150）: **Spare / Idle / Safety margin / Stranded / Active**。
**Stranded capacity（死蔵）= 設計/構成上 IT 負荷が使えない容量**（例: 盤に容量はあるがブレーカ位置が空かない）。

EcoStruxure の**推定負荷戦略**（働く電力値の決め方）:

| 戦略 | 意味 |
|---|---|
| **nameplate** | 銘板（最も悲観的） |
| **adjusted_nameplate** | 補正銘板（既定・より精密） |
| **predicted** | 実測値から導出 |
| **contracted** | コロ契約電力を rack PDU/receptacle に配分（[08章](./08-tenancy-colocation.md)） |

`PUE = 総施設エネルギー / IT 機器エネルギー`（Green Grid、理想 1.0）。**pPUE = 境界内の総 ÷ 同境界内の IT**。
EcoStruxure は **site/room 単位**で PUE を算出（EN 50600-4-2）。境界＝ゾーンが集計単位（[10章](./10-room-group-derived-metrics.md) の部屋グループに直結）。

### 1.5 収集プロトコル

| プロトコル | モデル | 点の識別子 | 主な機器 |
|---|---|---|---|
| SNMP v1/2c/3 | ポーリング+trap | OID `1.3.6.1…` | PDU, UPS, network, sensor |
| Modbus RTU/TCP | ポーリング | レジスタ+FC | 電力計, PDU, CRAC, BMS |
| IPMI | ポーリング+SEL | sensor番号/SDR | server BMC |
| Redfish | ポーリング+Telemetry | リソースURI+property | server BMC, chassis, thermal |
| BACnet | ポーリング+COV | object型+instance+property | HVAC, CRAC, chiller |
| MQTT / REST | **プッシュ** | topic / JSON path | IoT sensor, gateway |

**ポーリングは欠測前提**（サンプル品質を記録）。スケール係数・単位は取込時に正規化。Redfish の `MetricDefinition`/
`MetricReport`/`Triggers`（定義とサンプルの分離）は収集定義設計の手本。

---

## 2. 参照モデル — Haystack / EcoStruxure / Brick / NetBox

### 2.1 Project Haystack（セマンティックモデルの主参照）

ビル/設備/IoT 運用データの**タグベース・セマンティック標準**。本設計は Haystack の意味体系を RDBMS に reify する。

- **エンティティ階層**: `site` → `space`（`floor`/`dataCenter` はそのサブタイプ）→ `equip` → `point`。
  参照は子が親を指す `{parent}Ref`（`siteRef`/`spaceRef`/`equipRef`）。`equip` は `equipRef` でネスト可。`device`/`network` は phIct。
- **点の意味＝3つの直交 choice**:
  - `pointFunction` — **sensor / cmd / sp / synthetic を正確に1つ**（"All points must define exactly one"）。
  - `pointQuantity` — 量（`temp`/`active-power`/…）。
  - `pointSubject` — 物質 phenomenon（`air`/`water`/`elec`…）＋ ダクト区間（`discharge`/`return`… は **`ductSection` マーカー**、air のサブタイプではない）。
  - 例: discharge air temp sensor = `discharge` + `air` + `temp` + `sensor`。
- **能力マーカー（直交・併存可）**: `cur`（現在値 + curVal/curStatus/curErr）/ `his`（履歴 + hisMode/hisStatus）/
  `writable`（**BACnet 16+1 段の優先配列**、writeVal/writeLevel 1..17）。`kind` = Number/Bool/Str、`unit`、`tz`、`enum`、`minVal`/`maxVal`。
- **def システム**: 各 def は `is` で**スーパータイプを1つ以上**指定（**list 可＝多重継承＝実質 DAG**。spec は "strict tree"
  とも書くが list を許容し、サイクルのみ禁止）。`conjunct`（`chilled-water` 等の合成）、`tagOn`/`tags`（タグの適用先）、
  `quantity`/`quantityOf`/`prefUnit`。RDF へは `is`→`rdfs:subClassOf`。
- **量・単位**: 正規 `units.txt`（~60 次元グループ）が単位データベース。`prefUnit` が量→推奨単位を結ぶ。
  ❗`volt`/`current` は marker、量形は `elec-volt`/`elec-current`。力率は **`pf`**、電力は **`active-power`/`apparent-power`/`reactive-power`**（ハイフン）。
- **関係**: ❗**`feeds`/`fedBy` は存在しない**（フォーラムで Brick 由来案が却下）。流れは **`inputs`/`outputs` ＋ 物質参照
  `{substance}Ref`**（`elecRef`/`chilledWaterRef`/`airRef`）で、**下流側が上流を指す**（"flows from the referent to this entity"）。
- **コネクタ**: `{protocol}Conn` 機器 + 点側の `connRef`（汎用）/ `{protocol}ConnRef` + アドレス `{protocol}Cur|His|Write`。
  tuning は `pollTime`/`staleTime`/`writeMinTime` 等。BACnet/Modbus/Haystack で確立、SNMP/OPC/MQTT も同パターン（接頭辞は実装依存）。
- **DC 専用 def**: `dataCenter`（space）/ `rack` / `server` / `ups` / `crac` は標準。
  ❗`pdu` / `crah` / `busway` / `ats` / `generator` / `pue` は**標準に無い** → 自前 lib（`is`/`tagOn` 付き）で定義。
- **格納**: Haystack は**モデル＋プロトコル標準で、格納は実装任せ**。参照実装 Folio はタグ/ドキュメント型（非リレーショナル）。
  作者 Brian Frank 自身が「Haystack は SQL より RDF 寄り」と述べ、リレーショナル化は標準と矛盾しない（既製の Haystack→SQL マッパは無い）。

### 2.2 EcoStruxure IT（ドメインモデルの主参照）

Schneider の DCIM（旧 StruxureWare DCO → IT Advisor、監視は Data Center Expert / IT Expert）。

- **Genome / Genome Library**: 製品（メーカ+型番）の**テンプレート定義**。weight/depth/height/width/design limit、equipment kind、
  **測定済み電力プロファイル**（センサ無しでも消費推定）、rack PDU の bank/outlet を持つ。**Genome（読取専用ライブラリ）→
  ドラッグで配置インスタンス**化（→ 本設計の DeviceType/Device 分離に対応）。
- **power path / capacity**: §1.2・§1.4 の通り。容量は **best-fit 配置 / what-if / 予約 / 変更管理（ITIL ワークフロー）**。
- **sensors / alarms**: NetBotz 等の環境センサ。severity = **informational / warning / critical**、しきい値 high/low、
  **hysteresis**（チャタリング抑制）。numeric/state sensor を区別。
- **sustainability**: PUE を site/room 単位で算出、water/renewable 等を自動レポート（EN 50600-4-2）。
- **colocation**: **Tenant Portal**、cage がテナント境界、contracted power（§1.4）（[08章](./08-tenancy-colocation.md)）。

### 2.3 NetBox / OpenDCIM（リレーショナル実装の手本）

- **NetBox**（Django/PostgreSQL）: **DeviceType/Device 分離**（テンプレート展開・新機種=データ追加）、**種別別ポート + Cable/
  CableTermination**（多態、`CablePath` キャッシュ）、`PowerPanel→PowerFeed`（`available_power` 計算列、`max_utilization=80`）、
  葉に site/location/rack を**非正規化**してフィルタ高速化。`available_power` の int4 overflow は列型の教訓。
- **OpenDCIM**（PHP/MySQL）: 固定階層 + テンプレート。**A/B 冗長給電が一級**（`PanelID`+`PanelID2`）。だが**宣言的 FK がほぼ無く**
  孤児行が出やすい / **時系列が最新値1行のみ**（→ 本設計の TimescaleDB 前提の正当化）。

### 2.4 Brick Schema & ASHRAE 223P（収束先）

- **Brick** は RDF/OWL の**形式的**ビルオントロジ（クラスを明示）。Haystack はタグの適用で型が決まる**非形式的**モデル。
  Brick は Haystack の上位互換的（Haystack→Brick 変換可）。格納は RDF トリプルストア + SPARQL。
- **ASHRAE 223P** が両者を収束（2018 連携）。Haystack 4 は RDF エクスポート（`defc -output turtle`）を持つ。
  → 本設計は **Haystack の意味体系を発想に採りつつ、「クラスの明示」を `equip_kind`/`metric` の型付き表で実現**する折衷。

---

## 3. セマンティックを RDBMS 制約に reify するパターン

「タグの辞書（schemaless）」を**強い制約のある RDBMS**へ落とすときの定石（[02章](./02-candidate-patterns.md) で採否）。

- **EAV（Entity-Attribute-Value）はアンチパターン**（Karwin『SQL Antipatterns』）: 汎用 value 列は**型・NOT NULL・UNIQUE・FK が
  効かず**、再構成に多重 JOIN。→ 生 EAV は採らない。
- **クラス継承**: CTI（種別ごとに子表・FK 自在だが JOIN 増）/ STI（1表+判別列・NULL 多）。Fowler PoEAA。
- **階層**: 隣接リスト単独は "Naive Trees" アンチパターン。**閉包テーブル**は部分木クエリ高速・**FK 整合・複数親可**で、
  **DAG の推移閉包**の標準表現（Karwin）。`ltree`/Nested Set は移植性・整合で劣る。
- **DAG の閉包**は tree より重い: **path_count**（複数経路で到達）、挿入は O(祖先×子孫)、辺削除は減算→0 で行削除、サイクル禁止チェック。
  → **本設計の判断**（[02章 D](./02-candidate-patterns.md)）: Haystack の `is` は DAG だが、DCIM の機器分類は実用上ほぼツリー。
  内部は `equip_kind` の**親参照ツリー**（読み多なら閉包）に留め、DAG はメタモデル直輸入せず interop 境界でマッピング。多重継承の実需が出たら拡張。
- **制約付き語彙タグ（`tag_def`+`entity_tag` を FK）vs 生 JSONB**: 前者は**語彙・一意・整合を DB が強制**、後者は方言差・FK 不可。
- **コンセンサス = ハイブリッド**: 安定・検索・整合が要る中核は**型付き列＋FK/CHECK**、疎で進化する末尾のみ JSONB/タグ。
  PG JSONB は FK 不可・CHECK は「キー不在で NULL→通過」の罠あり。**これが本設計の中核方針**。

---

## 4. RDBMS 制約に落とすべき業務ルール

**★=DB 制約で表現可 / ▲=サービス層・監視ビュー（行間集約のため単一行 CHECK 不可、移植性のため DB トリガに依存させない・[09章](./09-portability.md)）**

**空間・搭載**
- ★ U の物理重なり禁止（**占有U行＋複合UNIQUE**。`EXCLUDE`/range を使わない LCD 化。フルデプスは front/rear 両面）
- ★ `position + u_height - 1 ≤ rack.u_height`（親値参照のため CONSTRAINT TRIGGER）/ ★ cage は 1 テナント（FK）
- ▲ 機器気流方向が row のアイル整列と整合

**電力**
- ★ ブレーカ位置一意・回路定格（`breaker UNIQUE(panel,position)` + `rated_a`）/ ★ available_power = V×A×util×√3（生成列・bigint）
- ★ A/B 二重給電の冗長ペア（`paired_power`）/ ▲ A 給電と B 給電が異なる上流（同一なら SPOF）
- ▲ 回路の接続機器 draw 合計 ≤ ブレーカ定格（連続負荷 ×0.8）

**意味論（Haystack）**
- ★ 量と単位の整合（`metric` 行が単位を1つ保持＝読み値は単位を持たない。power に °C 不可）
- ★ 点の機能は正確に1つ（`point_function` FK）/ ★ 1機器に同一意味の点は1つ（`data_point` UNIQUE）
- ★ 「全 hvac / 全 power」を再帰なし集約（`def_closure` DAG）

**冷却・容量・コロ**
- ▲ センサ温湿度が機器 ASHRAE クラス内 / ▲ ラック発熱 ≤ 冷却容量（rack/row/room）
- ▲ Allocated ≤ Design（5要素）/ ▲ stranded = reserved − actual / ★ cage 割当 ≤ 親データホール配分

---

## 5. 出典（主要）

**Project Haystack**: project-haystack.org `/doc/docHaystack/{Structure,Points,Choices,Subtyping,Ontology,Relationships,Units,DataCenters,Rdf}`、
`/doc/lib-{ph,phScience,phIoT,phIct}/*`（site/space/equip/point/pointFunction/pointQuantity/pointSubject/cur/his/writable/is/tagOn/
quantity/prefUnit/inputs/outputs/airRef/chilled-water/active-power/pf/dataCenter/rack/ups/crac 等）、`/download/units.txt`、haxall.io `Conns`/`ConnTuning`、forum topic 808/859。
**EcoStruxure IT**: Schneider help/sxwhelpcenter（Genome、power path、breaker panel、paired power consumers、capacity、cooling、alarms、PUE/sustainability）、
APC WP-150「Power and Cooling Capacity Management」、WP「Room/Row/Rack Cooling」、Green Grid/LBNL WP49（PUE/pPUE）。
**Brick / 223P**: brickschema.org、BuildSys 2016、ASHRAE 223P 連携プレスリリース。
**RDBMS パターン**: Karwin『SQL Antipatterns』（EAV / Naive Trees / 閉包テーブル）、Fowler PoEAA（CTI/STI）、PostgreSQL 公式（JSONB）。
**DCIM 実装**: NetBox source（racks/power/sites）・docs、OpenDCIM `create.sql`、Redfish DSP0266/DSP2051、ASHRAE TC9.9、NEC 80%。

各章末にも個別 URL を記載。
