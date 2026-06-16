# 11. 標準・法規ギャップ分析 — JDCC ほか一次情報との突合

確定案（[03章](./03-finalists.md) / ER 図 [05章](./05-er-diagram.md)）を、**JDCC（日本データセンター協会）を起点に、
日本の法規制・国際標準（TIA-942 / EN 50600 = ISO/IEC 22237 / ISO/IEC 30134 / Uptime / OCP）・
サステナビリティ報告（省エネ法 / 温対法 / EU EED）** と突き合わせ、**現設計に欠けている、
スキーマに落とすべきデータ観点**を洗い出す。

[06章](./06-self-review.md) のセルフレビューが「SQL アンチパターン × 一般化」の**内省**だったのに対し、
本章は**外部基準との照合（外挿）**である。06章 B-8 が「初版スコープ外」と明示したもの（IPAM・0U/busway・discovery 来歴・
DC給電/OCP）に加え、**06章が認識すらしていなかった領域**（耐震・災害・法定点検・物理セキュリティ記録・運用ライフサイクル・
サブシステム別可用性クラス・カーボン/水報告）を本章で可視化する。

> **位置づけ**: 本章は是正の**指示書ではなく観点カタログ**。多くは [08章](./08-tenancy-colocation.md) と同じ
> **運用プロファイルで切替可能な加算モジュール**として設計するのが妥当（小規模自社 DC では大半が不要、
> 日本のコロ事業者・ハイパースケールでは必須、という可変性を保つ）。コア（[03章](./03-finalists.md)）は汚さない。

> ⚠️ **最重要の但し書き — 「DC に必要」≠「DCIM に必要」**：本章の調査は「あらゆる標準に対する欠落を探せ」という
> 網羅指示で行ったため、**DC 事業者がどこかで必要とするデータ**を広く拾っている。だがその多くは **DCIM が *own* すべきデータではなく、
> 隣接システム（ESG/サステナ基盤・CMMS/設備管理・PSIM/入退管理・ITSM・ITAM/ERP）が own し、DCIM は連携でデータを出し入れするだけ**の領域である。
> スキーマに抱え込むと製品が CMMS/ESG/ITSM の劣化コピーに膨らむ。**§0.5 のスコープ判定が本章の結論**であり、
> 以下の各 G の詳細はあくまで「その判定の根拠資料」として読むこと。

**スコープ判定の凡例**: ✅ **DCIM コア**（スキーマで own） / 🔶 **連携シーム**（参照キー・属性だけ持ち、本体は外部） / ❌ **対象外**（別システムが own。API 連携に留める）

重大度（コア内の優先度）: 🔴 早期に骨格を決める / 🟡 加算モジュール / 🟢 属性追加で足りる

---

## 0. サマリ — 6 つの欠落クラスタ

各クラスタは内部で**コア/シーム/対象外が混在**する（だから「クラスタ単位で採否」は誤り。項目単位の判定が §0.5）。

| # | クラスタ | クラスタ内の主たる判定 | 代表ギャップ |
|---|---------|---------------------|------------|
| G1 | **信頼性・保護の多次元等級** | 🔶 シーム（冗長モデルは既存コア／多次元等級は属性） | 単一 `redundancy` enum では「電気=Tier3／機械=Tier2」を表現不能 |
| G2 | **日本固有：耐震・災害・立地** | 🔶 シーム（building の属性として） | 建屋/ラックの耐震構造・PML・洪水/活断層ハザードが皆無 |
| G3 | **法定点検・コンプライアンス** | ✅一部（発電機・燃料容量）＋❌大半（法定点検は CMMS） | 無給油運転可能時間が無い／法定点検記録は別システム領域 |
| G4 | **環境・カーボン・水の報告** | ✅一部（計量）＋❌大半（炭素会計・法定申告は ESG 基盤） | 水/排熱メータが無い／係数レジストリ・Scope1/2/3 は対象外 |
| G5 | **運用ライフサイクル** | ✅一部（回線）＋🔶（MAC）＋❌（入退室/ITSM/財務） | キャリア回線が無い／入退室・インシデント・財務は別システム |
| G6 | **ラック規格・OCP/DC給電・構造化配線** | ✅ コア | EIA-310 暗黙前提・OCP busbar/48V・MDA/HDA が表現不能 |

---

## 0.5. ★スコープ判定 — DCIM が own すべきは何か（本章の結論）

### 境界線の原則
**DCIM のコア = 物理インフラのモデル ＋ テレメトリ ＋ キャパシティ ＋ 構内接続**。
隣接領域（ESG/サステナ・CMMS/設備保全・PSIM/入退管理・ITSM・ITAM/ERP）は **DCIM が*持つ*のでなく*連携する***
——キーや属性だけを保持し、本体は外部システムが own する。判断基準は一貫して **「DCIM が一次的に*測る・配置する・容量計算する*対象か？」**。
Yes ならコア、No（会計・申告・チケット・人事・財務）なら対象外。

### 項目別の判定表（これが採否の決定版）
| 判定 | 項目 | 由来クラスタ | 根拠 |
|------|------|------------|------|
| ✅ **コア** | ラック実装規格（19/21/23"・U/OU・OCP busbar/48V・rectifier/power shelf） | G6 | 資産・搭載適合・U占有計算そのもの。EIA-310 暗黙前提の修正は純 DCIM |
| ✅ **コア** | TIA-942 構造化配線スペース（MDA/HDA/ZDA/EDA）と cross-connect | G6 | 構内配線トポロジー＝既存 cable/equip_link の正当な拡張 |
| ✅ **コア** | キャリア回線・cross-connect・MMR | G5-P6 | コロ DC の接続管理は DCIM 中核（Device42 等も own）。※IP/VLAN=IPAM は対象外で正しい |
| ✅ **コア** | 発電機 ＋ 燃料タンク容量 → **無給油運転可能時間**をキャパシティ次元に | G3 | 電力/冷却/U/重量と同じ容量軸。BCP 容量は DCIM が算定する |
| ✅ **コア** | 水・排熱の**計量**（補給水・排熱輸出メータ）と PUE 系メトリクスの測定 | G4 | DCIM は「測る」側。WUE/ERF の素データ取得はテレメトリの自然な拡張 |
| 🔶 **シーム** | 可用性/保護/粒度の**等級を属性として**（`room.tier` 等の少数列） | G1 | 冗長モデル(SPOF検出)は既にコア。等級は営業・容量計画用メタ。多スキーム `resilience_rating`＋認証イベント表は**作り込み過剰**、最小列で足りる |
| 🔶 **シーム** | 耐震構造・PML・立地ハザードを **building の属性として** | G2 | 日本市場なら数列追加は安い。但し本質は BCP/不動産/立地データで「機能」ではない。**ハザード"モジュール"は作らない** |
| 🔶 **シーム** | 物理 MAC（Move/Add/Change）の作業ワークフロー | G5-P2 | 物理移設の作業管理は DCIM 寄り。但しフルな変更管理プロセスは ITSM。**MAC を ITSM チケットに紐付ける ID** に留める |
| 🔶 **シーム** | 機器の保証/EOL/EOSL・保守契約の SLA 属性 | G5-P3 | 機器マスタの属性として持つのは妥当。点検"実施"管理は CMMS 寄り |
| ❌ **対象外** | 排出係数レジストリ・Scope1/2/3 炭素会計・省エネ法/温対法/EED の**申告** | G4 | DCIM はエネルギー/水データを*エクスポート*する。炭素会計と法定申告は別製品（ESG/サステナ基盤） |
| ❌ **対象外** | 消防法/電気事業法/高圧ガスの**法定点検記録・有資格者・報告提出** | G3 | 設備保全管理（CMMS/施設管理システム）の領域。DCIM は機器＋保守"予定"参照まで |
| ❌ **対象外** | 入退室イベント・バッジ・監視カメラ | G5-P1 | 物理セキュリティ（PSIM/入退管理システム）が own。DCIM はゾーンを参照するだけ |
| ❌ **対象外** | インシデント/問題/SLA/RCA チケット | G5-P4 | ITSM。DCIM はアラートを*発火*して ITSM に渡す。チケット管理は ITSM |
| ❌ **対象外** | 減価償却・調達(PO)・簿価・リース会計 | G5-P5 | ITAM/ERP。DCIM は物理資産（資産番号・シリアル・設置位置）までで、財務は持たない |
| ❌ **対象外** | 棚卸監査・汎用 `audit_log` | G5-P5/P7 | 監査ログは全システム共通インフラで DCIM 固有のドメイン機能ではない（フレームワーク層で担保） |

### 連携シームの設計（own しないものをどう扱うか）
❌ 項目は**捨てる**のではなく、**DCIM 側に「外部キー（参照ID）と最小限の状態」だけを置く**:
- ESG 基盤へ → DCIM は `measurement`/`series`（電力量・水・排熱）を**読み取り API/ビューで公開**。係数・申告は受け手が持つ。
- CMMS へ → `device`/`equip` に **`external_cmms_id`** を持たせ、点検実施は CMMS。DCIM は保守*予定*（次回点検日）の表示参照のみ。
- PSIM へ → `location` のゾーンに **保護クラス属性**（🔶 G1）は持つが、入退室イベントは PSIM の参照キーで連携。
- ITSM へ → アラート発火時に **ticket_ref** を記録。インシデント本体は ITSM。
- ERP/ITAM へ → `device` に **`asset_tag`/`external_asset_id`**（既存）。財務属性は ERP 側。

> **結論**：6 クラスタのうち DCIM が own すべきは **G6 全部・G5-P6・発電機キャパシティ(G3一部)・水/排熱計量(G4一部)・冗長/Tier 属性(G1)** に限る。
> 残り（炭素会計・法定点検・入退室・ITSM・財務）は**連携シーム（参照キー）に格下げ**する。これにより製品は「DCIM の本分」に収まり、
> 設計ゴール「可変性 × ドメイン制約の厳格さ」を保ったまま肥大化を防げる。

> **横断の発見（コア採用分のみ）**：own する範囲に絞っても、「**人・組織マスタ（`vendor`/`carrier`）**」と
> 「**汎用台帳（係数 `factor_value` 等）**」の共通基盤は有効。個別テーブルを乱立させず共通受け皿を先に決める方針は維持する。

---

## G1. 🔶シーム 信頼性・保護の「多次元等級」— 単一 redundancy enum の限界

> **スコープ**：冗長モデル（SPOF 検出）は**既にコア**。本節の多次元等級（resilience_rating／certification）は
> **属性・営業/容量計画用メタ**であり、`room.tier` 等の少数列で足りる。多スキーム表＋認証イベント表まで作るのは**過剰**。

### 何が足りないか
現設計の可用性表現は `equip_link.redundancy`（`A|B|N|N+1|2N` の単一 enum・**電力/冷却エッジ単位**）と、
暗黙の施設 Tier のみ。だが国際標準はいずれも**「対象 × 軸 × 等級」の多次元格付け**を要求する:

- **TIA-942**：Rated-1〜4 を **Telecom / Electrical / Architectural / Mechanical の 4 サブシステム別**に評価（高位は下位を内包）。
- **EN 50600 / ISO/IEC 22237**：3 つの独立軸 —
  - **Availability Class（VK）1〜4**：電力供給・冷却供給・配線に**別々に**適用
  - **Protection Class（SK）1〜4**：物理セキュリティ等級（可用性とは**直交**。「高可用だが低保護」がありうる）
  - **Granularity Level（GN）1〜3**：エネルギー計測の範囲/精度クラス（PUE 値の比較可能性を担保）
- **JDCC ファシリティスタンダード（FS）**：稼働信頼性をティア 1〜4 で区分し、**マルチティア DC**（同一センター内で
  サーバ室ごとに異なるティア）を明示的に許容。Uptime と違い**商用電源をメイン・自家発電をバックアップ**と位置づける。

→ 単一 enum では「この施設は電気 Rated-3 だが機械 Rated-2」「冷却 AC3／配線 AC2／保護 SK2」を構造化できず、
**律速サブシステムをクエリできない**。施設 Tier 単一値と equip_link 冗長が二重化する。

### スキーマ案
```text
resilience_rating(
  scope_type {facility|room|power_panel|cooling_unit|cage|...}, scope_id,
  scheme     {uptime|tia942|en50600|iso22237|jdcc_fs},
  subsystem  {facility|power|cooling|telecom|architectural|security|metering},
  class_value text,              -- '1'..'4' / 'N'..'2N+1' / 'SK1'..'SK4' / 'GN1'..'GN3'
  concurrently_maintainable bool, fault_tolerant bool, distribution_paths int,
  assessed_at, basis_version,
  PRIMARY KEY(scope_type, scope_id, scheme, subsystem))
```
- `equip_link.redundancy` は**実装事実**、`resilience_rating` は**設計等級**として分離する。
- JDCC のマルチティアは `scope_type='room'` 行で受ける（`room.fs_tier` を直接列に持つ簡易版でも可）。
- GN（粒度レベル）は [10章](./10-room-group-derived-metrics.md) の pPUE 算出境界（`room_group`）のメタとして持たせ、PUE の準拠性を記録。

### 認証は「等級」でなく「イベント」
Tier 値を 1 つ持つだけでは、**設計等級と実構築/運用認証の差**（「設計 Tier III・構築は未認証」は頻出）を表せない。
Uptime は **TCDD（設計）/ TCCF（構築）/ TCOS（運用持続性）/ M&O Stamp**、JDCC FS は **JQA 等による稼働信頼性個別検査**、
TIA-942 は Rated ラベルと、いずれも**発行体・発行日・失効を持つ取得イベント**。

```text
certification(scope_type, scope_id, scheme, award_type {TCDD|TCCF|TCOS|MO_stamp|tia942_rated|en50600_ac|jdcc_fs},
              level, status {certified|in_review|expired}, body, issued_on, expires_on, evidence_ref)
```
`scheme`/`award_type` は**ルックアップ表化**（06章 A-8 方針）。基準は版を重ねる（FS Ver.2.3 → 3.0 等）ため CHECK 列挙は不可。

出典: TIA-942 認証 https://tiaonline.org/products-and-services/tia942certification/tia-942-certifications-ratings/ ／
EN 50600 / ISO 22237 の VK・SK・GN https://www.hknow.de/en/certifications/iso-22237/ , https://www.hknow.de/en/planning/en50600/ ／
Uptime M&O・Tier 認証区分 https://uptimeinstitute.com/professional-services/management-operations , https://journal.uptimeinstitute.com/explaining-uptime-institutes-tier-classification-system/ ／
JDCC FS 概要 https://www.jdcc.or.jp/pdf/facility.pdf

---

## G2. 🔶シーム 日本固有 — 耐震・自然災害・立地ハザード

> **スコープ**：日本市場向けなら `building` への**数列の属性追加**は安価で妥当。但し本質は BCP/不動産/立地データで
> DCIM の「機能」ではない。**ハザード判定の"モジュール"は作らず**、外部評価結果を属性として保持するに留める。

JDCC が Uptime Tier と最も差別化した点が、**地震大国の事情を反映した災害・耐震軸**。FS は日本独自に
「敷地の地震危険度」「施設の耐震性」「地盤の安定性」「経済損失（**PML**）」を基準項目に持つ
（PML 帯 <10% / 10–20% / 20–25% / 25–30%、1981 年改正建築基準法準拠、官庁施設耐震性能 I 類/II 類、予測震度 6 弱/6 強）。
現設計の Building/Site は空間アンカーのみで**これらを一切持たない**。06章 B-8 のスコープ除外リストにも挙がっておらず、未認識のギャップ。

### スキーマ案（location への部分 1:1 補助表 — コアを汚さない）
```text
building_attr(location_id PK→location,         -- loc_type='building' の行のみ
  seismic_structure {taishin|seishin|menshin}, -- 耐震/制振/免震
  seismic_grade, seismic_code_year,            -- 1981 改正前/後
  design_seismic_intensity {6弱|6強|...}, pml_percent numeric,
  raised_floor_height_mm,                       -- FS 推奨項目（床下高さ）
  rated_power_capacity_kw, completion_year)

site_hazard(location_id PK→location,            -- site/building
  flood_depth_class, tsunami_zone bool, storm_surge_zone bool,
  active_fault_dist_km, liquefaction_risk {low|mid|high},
  j_shis_pga numeric, elevation_m, assessed_on, source)
```
- ラック側も `rack.seismic_mount {anchor|ubolt|damped|isolated|isolation_floor|none}` を追加（免震床/制振ラックは搭載可否・床仕様に影響）。
- 建築設備の**耐震クラス S/A/B**・設計用標準震度 Ks は `device` 補助列（重要機器=クラス S 要求）として持てる。
- 法区分の閾値（PML 帯・Ks 値）は**データ駆動**にし、法改正にマイグレーション無しで追従。

出典: JDCC FS https://www.jdcc.or.jp/pdf/facility.pdf ／ DC 認証解説 https://www.scsk.jp/sp/netxdc/column/datacenter_certification.html ／
建築設備耐震設計・施工指針 https://www.mlit.go.jp/common/001211554.pdf ／ DC ラック耐震 https://tech.synapse.jp/entry/2024/11/05/143000 ／
立地選定 https://www.wanbishi.co.jp/datasolution/blog/importance-of-datacenter-location.html

---

## G3. ✅一部 / ❌大半 法定点検・コンプライアンス（消防 / 電気事業法 / 高圧ガス / 発電機・燃料）

> **スコープ**：✅ **発電機＋燃料タンク容量 →「無給油運転可能時間」をキャパシティ次元に**する部分のみ DCIM コア。
> ❌ 消防法/電気事業法/高圧ガスの**法定点検記録・有資格者・報告提出**は CMMS/施設管理システムの領域で、DCIM は
> 機器マスタ＋保守*予定*（次回点検日）の参照と `external_cmms_id` 連携に留める。以下の `inspection_record` 等は
> **CMMS を別途持たない小規模事業者向けの任意モジュール**として位置づけ、コアには入れない。

日本の DC は多数の法定点検義務を負うが、現設計は `maintenance_contract`（契約のみ）しか持たず、
**実施記録・周期・有資格者・報告提出**を保持できない。**単一の汎用台帳に `regime` で多態化**するのが要点。

### 共通受け皿
```text
compliance_obligation(building_id, regime, interval_months, authority)   -- 義務マスタ
inspection_record(
  target_kind, target_id,                       -- 多態（建屋/設備/発電機/タンク）
  regime {fire|elec|hpgas|generator|...},
  inspection_type {機器点検|総合点検|monthly|annual|load_test|internal|...},
  performed_on, due_on, next_due_on,
  inspector, qualification, result, report_filed_on)
```

### 個別の不足エンティティ
| 領域 | 法令 | 持つべきデータ | スキーマ案 |
|------|------|--------------|-----------|
| **消火・感知** | 消防法（サーバ室は不活性ガス消火が適合・水損回避） | 消火方式/薬剤量/区画耐火時間・感知方式（煙/熱/**VESDA**） | `fire_suppression(room_id, agent_type, agent_qty_kg, zone)` / `fire_detector(room_id, detect_type)` / `room.fire_rating_hours` |
| **法定点検報告** | 消防法 17条の3の3（機器点検6ヶ月・総合点検1年・報告 特定1年/非特定3年） | 周期・実施者資格・報告提出日 | 上記 `inspection_record`(regime='fire') |
| **受変電** | 電気事業法 42/43条（保安規程・電気主任技術者・月次/年次点検） | 受電電圧区分・契約kW・保安規程・主任技術者選任・点検 | `electrical_facility(building_id, voltage_class, contract_kw, safety_reg_filed_on)` / `chief_engineer(facility_id, person, license_class, appointed_on)` |
| **冷凍機** | 高圧ガス保安法（法定冷凍能力トンで第一種/第二種区分） | 冷凍能力(トン)・冷媒・製造者区分・許可番号 | `refrigeration_unit(device_id, legal_refrig_capacity_ton, refrigerant, hpgas_class, permit_no)` |
| **非常用発電機** | — BCP 中核 | 定格 kVA・燃料種別・**無給油連続運転可能時間（72h 等）** | `generator(device_id, rated_kva, fuel_type, runtime_target_h)` |
| **燃料貯蔵** | 消防法・危険物（軽油 1000L 以上=指定数量、防油堤 110%） | タンク容量・現在量・指定数量区分・貯蔵所許可・防油堤 | `fuel_tank(location_id, capacity_l, current_level_l, fuel_type, hazmat_class, storage_permit_no, bund_ratio CHECK ≥1.1)` |
| **発電機試験** | 30% 以上負荷試験 or 代替点検 | 試験種別・負荷率・運転分数 | `inspection_record`(regime='generator', inspection_type='load_test', load_ratio, ran_minutes) |

- **キャパシティ次元の拡張**：`capacity_budget` に `dimension='fuel_runtime_h'` を足し、燃料/消費率から**無給油運転可能時間**を
  電力・冷却・U・重量と同じキャパシティ軸として扱える（BCP 訴求の中核指標）。
- 非常用発電機・燃料は G4 のカーボン（Scope1・燃料燃焼）とも接続する（同じ `generator`/`fuel_tank` を参照）。

出典: JDCC FS https://www.jdcc.or.jp/pdf/facility.pdf ／ 不活性ガス消火 https://cloud.watch.impress.co.jp/docs/wdc/1416780.html ／
消防用設備点検報告（東京消防庁） https://www.tfd.metro.tokyo.lg.jp/lfe/office_adv/tenken_houkoku.html ／
電気事業法 保安規制 https://www.mlit.go.jp/common/001318576.pdf ／ 高圧ガス 第一種冷凍 https://www.pref.fukui.lg.jp/doc/012643/kouatu_d/fil/4002.pdf ／
非常用電源 燃料/72h/負荷試験 https://fukashiken-techno.co.jp/column/emergency-generator/running-hours/

---

## G4. ✅一部 / ❌大半 環境・カーボン・水の報告 — 「係数レイヤ」が丸ごと無い

> **スコープ**：✅ **水・排熱の計量**（補給水・排熱輸出メータ）と **PUE 系メトリクスの測定**のみ DCIM コア（テレメトリの自然な拡張）。
> ❌ **排出係数レジストリ・Scope1/2/3 炭素会計・省エネ法/温対法/EED の申告**は ESG/サステナ基盤が own する別製品領域。
> DCIM は `measurement`/`series`（電力量・水・排熱）を**読み取り API/ビューで公開**し、係数適用・会計・申告は受け手に委ねる。
> 以下の `factor_series`/`emission_source`/`reporting_period` は **ESG 基盤を持たない場合の任意アドオン**であり、コアの前提にはしない。

現設計は PUE/pPUE・電力量・時系列までは持つ（[10章](./10-room-group-derived-metrics.md)）が、
**排出/水/再エネ/原油換算の「係数」を持つ場所が無く**、ISO/IEC 30134 系・省エネ法・温対法・EU EED の KPI が算出できない。

### ★中核ギャップ：時間変化する係数レジストリ
排出係数は**年度ごとに改定**され、**グリッド地域別／小売電気事業者別**に異なり、さらに
**location 基準（系統平均）と market 基準（契約由来）の二系統**を併存させる必要がある
（GHG Protocol Scope 2 の dual reporting）。これは「測定値」でなく**メタ参照データ**なので時系列 hypertable と分け、
**区間バイテンポラル**で持つ。`derived_measurement` に値を焼き込むと改定・遡及再計算・二重基準が破綻する。

```text
factor_series(id, factor_kind {grid_co2|water_scarcity|renewable_share|primary_energy|crude_oil_equiv},
              region_code, basis {location|market}, source {env_meti_FYxx|IEA|supplier|REC|PPA}, unit)
factor_value(factor_series_id, valid_from, valid_to, resolution {annual|monthly|hourly}, value,
             PRIMARY KEY(factor_series_id, valid_from))
```
KPI 算出は「測定エネルギー × **その時刻に有効な** `factor_value`」を JOIN（係数は焼き込まない）。
カーボンアウェア運用が要れば hourly 解像度 factor_value を別 hypertable 化可。

### ISO/IEC 30134 / EN 50600-4 系 KPI と前提データ
| KPI | 規格 | 不足している前提データ | スキーマ案 |
|-----|------|----------------------|-----------|
| **CUE**（炭素使用効率） | 30134-**8** / EN50600-4-8 | CUE = grid_CEF × PUE ＋ **Scope1（発電機燃料・冷媒漏えい）** の発生源 | `metric_definition` に `cue` ＋ `emission_source` から Scope1 |
| **WUE**（水使用効率） | 30134-**9** / 4-9 | 冷却**補給水(make-up)・蒸発・ブローダウン**の水メータ、WUE_source は水係数 | `device_category.resource_class` に water 区別 ＋ `makeup_water`(L) point |
| **REF**（再エネ係数） | 30134-**3** / 4-3 | オンサイト発電計量 ＋ **オフサイト契約証書（REC/非化石証書/PPA）の MWh** | `renewable_instrument(kind, mwh, valid_from/to, certificate_no, tenant_id)` |
| **ERF**（排熱再利用係数） | 30134-**6** / 4-6 | 外部供給した**排熱の熱量メータ（地域熱供給等）** | `equip_link` に `substance='heat'` の **outbound** を許可＋輸出メータ raw series |
| **ITEEsv / ITEUsv** | 30134-**4/5** | サーバの**定格性能・効率**と「server」フラグ（分子が計算不能） | `device_type.rated_peak_perf`, `rated_efficiency` ＋ `equip_kind` の server サブツリー |

> ⚠️ **規格番号の訂正**：ISO/IEC **30134-6 = ERF（排熱再利用）**、**30134-8 = CUE（炭素）**。
> 巷の資料（および本検討の初稿ブリーフ）で逆転している例が多いので注意。EN 50600-4 は -7=CER（冷却効率比）も持つ。

### 日本・EU の報告義務
- **省エネ法 定期報告**：エネルギー種別ごとの使用量を**原油換算 kL** に合算（電力/都市ガス/軽油で換算係数が異なる）。
  DC 業はベンチマーク指標が **PUE（≤1.4 で S ランク・年 1% 改善）**。→ `energy_carrier(code, crude_oil_equiv_factor, unit)`（factor_series の一種）。
- **温対法 GHG 報告**：排出量 = 活動量 × 排出係数。**エネルギー起源 CO2 と非エネ起源（冷媒 HFC・自家発燃料）を区別**し、
  電力は事業者別「実排出係数／調整後排出係数」（年度更新）。→ `emission_source(scope{1|2|3}, gas, source_type, activity_qty, factor_series_id, tenant_id)`。
- **EU EED 委任規則 2024/1364**：IT 電力 **500kW 以上**で年次報告。必須項目に PUE/WUE/ERF/REF に加え
  **設置面積・設置電力・キャパシティ利用率・廃熱・水消費・サーバ/ストレージ容量・データトラフィック量**。
  → `location.floor_area_m2` ＋ 報告ビューに `it_installed_power_w`/`data_traffic_volume`/`storage_capacity_tb`/`waste_heat_kwh`。

### 報告期間の凍結（監査対応）
係数は事後改定されるため、提出済み KPI は**その時点の係数・データで固定**する単位が要る。
```text
reporting_period(scope_type, scope_id, fy, period_start, period_end, regime {省エネ法|温対法|EED|CSRD|voluntary})
reported_kpi(period_id, metric {pue|cue|wue|ref|erf|crude_oil_kl|...}, value,
             factor_snapshot jsonb, submitted_at, status {draft|submitted|restated})
```
採用した `factor_value` を**スナップショット凍結**し、係数改定後も提出値を再現可能にする。
通貨・kWh・排出量の派生は [06章 A-6](./06-self-review.md) の通り `numeric`。

出典: ISO/IEC 30134-6(ERF) https://www.iso.org/standard/71717.html , -4(ITEEsv) https://www.iso.org/standard/66191.html ／
EN 50600-4 KPI https://www.stulz.com/newsroom/detail/making-efficiency-comparable-kpis-for-en-50600-compliant-data-centers/ ／
GHG Protocol Scope 2 dual https://www.persefoni.com/blog/scope-2-dual-reporting-market-and-location-based-carbon-accounting ／
EU EED 2024/1364 https://www.gov.ie/en/department-of-climate-energy-and-the-environment/publications/data-centre-energy-and-sustainability-performance-reporting-obligations/ ／
省エネ法 DC ベンチマーク https://www.enecho.meti.go.jp/category/saving_and_new/saving/enterprise/factory/support-tools/data/2023_01benchmark.pdf ／
温対法 算定・排出係数 https://policies.env.go.jp/earth/ghg-santeikohyo/calc.html ／
JDCC 地域共生ガイドライン（騒音・取水・排煙等の対外メトリクス） https://www.jdcc.or.jp/お知らせ/7389/

---

## G5. ✅一部 / 🔶 / ❌混在 運用ライフサイクル

> **スコープ**：項目ごとに判定が割れる。✅ **P6 キャリア回線/cross-connect/MMR** のみ DCIM コア。
> 🔶 **P2 物理 MAC ワークフロー**（ITSM チケットへの参照に留める）・**P3 保証/EOL/EOSL 属性**（機器マスタの列）はシーム。
> ❌ **P1 入退室=PSIM / P4 インシデント・SLA=ITSM / P5 減価償却・調達=ITAM/ERP / P7 監査ログ=フレームワーク層** は対象外。
> 以下の DDL 案は「外部システムを持たない場合の参考形」であり、**既定はいずれも参照キー（`ticket_ref`/`external_*_id`）での連携**とする。

現設計の運用情報は `maintenance_contract`（vendor/番号/期間）と `device`（status/installed_on/asset_tag/serial）のみ。
**保守・物理セキュリティ・インシデント・資産経理・回線・監査証跡が不在**。優先度順:

### P1. 物理セキュリティ / 入退室記録 ★最優先
SOC2 / ISO 27001 Annex A 7.2-7.4 / PCI-DSS Req.9、JDCC FS の**ゾーン別セキュリティ**（T4 は敷地・建物・サーバ室・ラックの 4 層）が
**入退室ログの取得・保持・定期レビュー、来訪者本人確認・常時エスコート**を明示要求。テナント分界（[08章](./08-tenancy-colocation.md)）がある DCIM ほど物理分界の証跡が必須。
```text
person(id, kind {employee|vendor|visitor}, company, tenant_id)
access_credential(id, person_id, tenant_id, badge_id, valid_from, valid_to, status)
access_grant(credential_id, scope_type {site|room|cage|rack}, scope_id, granted_by, expires_at)   -- 失効=termination control
access_event(credential_id, scope_type, scope_id, ts, direction {in|out}, result {granted|denied}, reader_id, escorted_by)  -- append-only
```
（08章の `access_grant` は「権限」のみ。**入退室イベント記録**と**ゾーン別セキュリティレベル宣言**が欠落 — 後者は G1 の `resilience_rating(subsystem='security')` で受ける。）

### P2. 変更管理 / 作業オーダー / MOP ★最優先
停止の最大要因は人的エラー（Uptime M&O）。配線/搭載/電力変更の「申請→承認→保守ウィンドウ→MOP→実施→検証」が記録できず、
`device.status` 遷移の来歴すら残らない。
```text
change_request(id, type {install|move|decommission|maintenance}, risk_level, requested_by, approved_by,
               state {draft|approved|scheduled|in_progress|closed}, maintenance_window_id)
maintenance_window(id, location_id, starts_at, ends_at, freeze bool)
mop(id, change_request_id, steps_doc, rollback_doc, approved_by)
work_order(id, change_request_id, assignee, vendor_id, scheduled_at, completed_at, outcome)
work_order_target(work_order_id, entity_type {device|cable|rack|feed}, entity_id)   -- 影響資産（多態キー方式・03章と統一）
```

### P3. 予防保全 + 保証/EOL/EOSL（現 contract が薄い）
発電機/UPS/CRAC の定期点検スケジュールと実施履歴、保証期限、**EOL/EOSL（保守終了）**、SLA（オンサイト応答時間）が持てない。
```text
pm_schedule(device_id|equip_kind, task, interval_months, next_due, vendor_id, mop_id)
pm_record(schedule_id, performed_at, work_order_id, result, findings)
asset_warranty(device_id, warranty_end, eol_date, eosl_date, support_level)
-- maintenance_contract に response_sla, coverage {24x7|NBD}, contract_type を追加
```
（G3 の `inspection_record` が法定点検、`pm_schedule/pm_record` がベンダ保全。両者は `regime` で統合してもよい。）

### P4. インシデント / ダウンタイム / SLA / RCA
テレメトリ閾値で「アラート発火」はできるが、**インシデントという事象オブジェクト**（影響範囲・停止区間・根本原因・復旧時刻）が無く、
SLA 可用性%・MTBF/MTTR が算出不能。コロ事業者は契約 SLA の順守証跡が必須。
```text
incident(id, severity, opened_at, detected_at, resolved_at, root_cause, rca_doc, tenant_id, change_request_id)
incident_impact(incident_id, entity_type, entity_id, downtime_from, downtime_to)   -- 影響資産＋停止区間→可用性集計
sla_definition(tenant_id|service, metric {availability|response}, target_pct)
```
MTTA=detected−opened, MTTR=resolved−detected を導出。閾値超過（既存 ASHRAE/ブレーカ）を `incident` に紐付け。

### P5. 資産ライフサイクル / 経理
調達(PO/受入)・所有形態(owned/leased)・取得原価/減価償却・撤去/廃棄証跡・棚卸しがモデル化されていない（財務監査・リース・除却の証跡）。
```text
procurement(id, po_number, vendor_id, ordered_at, received_at, cost, ownership {owned|leased}, lease_end)  -- device.procurement_id
asset_financial(device_id, acquisition_cost, depreciation_method, useful_life_months, in_service_date, book_value)
asset_disposal(device_id, disposed_at, method {resale|recycle|destroy}, certificate_ref, proceeds)
inventory_audit(id, location_id, date) / inventory_audit_line(audit_id, device_id, expected, found, discrepancy)
```

### P6. キャリア回線 / クロスコネクト / MMR（IPAM は対象外で正しい）
現 `cable`/`cable_termination` は施設内物理ケーブルのみ。**キャリア・回線ID・cross-connect・MMR・回線SLA** の論理回線層が無く、
コロ顧客の WAN 回線・キャリア中立接続を追跡できない。
```text
carrier(id, name, type {telco|isp|cloud})
circuit(id, circuit_id_external, carrier_id, tenant_id, type {wan|cross_connect}, bandwidth, install_date, sla_ref)
cross_connect(id, circuit_id, a_termination, z_termination, mmr_location_id, media {fiber|copper}, status)
-- meet-me room は loc_type='mmr' で吸収
```

### P7. 横断: 監査証跡（append-only change log）
P1〜P6 が個別証跡を持っても、スキーマ変更全体の「誰が・何を・いつ」を残す不変ログが要る（ISO/SOC2/PCI 共通）。
discovery 来歴（source/last_seen）は機器発見の話で、人手変更の証跡を兼ねられない。
```text
audit_log(ts, actor, entity_type, entity_id, action {create|update|delete}, before jsonb, after jsonb, change_request_id)  -- append-only・保持期間付き
```

> **設計整合**：入退室/監査ログは `measurement` 同様の **append-only ＋ 論理参照**、影響資産は既存の**多態キー方式**で統一可能。
> `person`/`vendor`/`carrier` は現状テナント以外の**人・組織マスタが皆無**なので、P1/P3/P6 の前提として早期に共通化すべき。
> いずれも range/EXCLUDE 不要で [09章](./09-portability.md) の LCD 方針と両立（停止区間/保守窓の重なり検査はサービス層）。

出典: 物理セキュリティ compliance https://www.dronestrategicpartners.com/post/data-center-physical-security-compliance-requirements-and-technology-architecture ／
ISO 27001 A.7.2 https://www.hicomply.com/en-us/iso-27001/access-control-to-premises-annex-a-7-2 ／
Uptime M&O https://uptimeinstitute.com/professional-services/management-operations ／ MOP/保守 https://eworkorders.com/cmms-industry-articles-eworkorders/data-center-maintenance-strategies/ ／
可用性メトリクス https://www.atlassian.com/incident-management/kpis/common-metrics ／ 資産ライフサイクル https://www.lansweeper.com/solutions/use-cases/asset-lifecycle-management/ ／
cross-connect/MMR https://dgtlinfra.com/cross-connects-interconnection-services/

---

## G6. ✅コア ラック規格・OCP/DC 給電・構造化配線

> **スコープ**：全項目 DCIM コア。資産モデル・搭載適合・U占有計算・構内配線トポロジーそのもので、
> 隣接システムに委ねる余地はない。**本章で最も優先的に取り込むべきクラスタ**。

### ラック実装規格（EIA-310 暗黙前提）
`rack` は `u_height` のみで **19"/EIA-310/44.45mm を暗黙前提**。だが実機は多様 —
**ETSI EN 300 119（21"/600mm 通信ラック）**、**OCP Open Rack V3（21"≈537mm, OpenU/OU=48mm, 44OU/47RU）**。
幅・ユニット高さ・レール規格が無いと OCP/ETSI 機器の**搭載互換性・U 重なり計算（[03章 L2](./03-finalists.md) 占有U行）が破綻**する。
```text
rack: mount_standard {eia310|iec60297|etsi300119|ocp_orv3}, mount_width_in {19|21|23}, unit_height_mm {44.45|48}
rack_unit_occupancy / rack_mount: unit_type {U|OU}   -- 占有計算を規格別に
device_type: mount_standard, width_in   -- 搭載適合を CHECK
```

### OCP DC 給電（48V busbar / power shelf）
現 [03章 L7](./03-finalists.md) は AC 前提（power_panel→breaker→power_feed→outlet）。OCP は
**垂直 DC busbar 48V（400/700/1400A, 18kW級）+ power shelf** が一級要素で **outlet 概念が無い**。
06章 B-8 が「DC給電/OCP は薄い」と認識済みの点を、ここで具体化:
```text
equip_kind に power_shelf|busbar|rectifier|bbu_shelf を追加
power_feed: power_type {ac|dc}, dc_voltage {48|...}, busbar_rating_a {400|700|1400}
-- busbar→OU位置 給電は equip_link substance='elec'、A/B 系は paired_power の代替に busbar 系統で
```
48V 通信局給電・OCP busbar(48/54V) は 06章 B-8 が薄いと認めた `phenomenon='elec'` を**媒体だけでなく給電方式**まで拡張する。

### TIA-942 構造化配線スペース
`loc_type` 語彙（room/zone/row…）に**配線分配エリアが無い**（NetBox 同様の穴）。TIA-942 の機能空間
**ER（Entrance Room）/ MDA（Main Dist Area）/ HDA / ZDA / EDA** と MC/HC クロスコネクトを表現できず、
バックボーン/水平配線の経路を構造化できない。
```text
loc_type 語彙に entrance_room|mda|hda|zda|eda を追加（room へのオーバーレイ role）
equip_kind に main_cross_connect|horizontal_cross_connect を追加
-- cabling は equip_link substance='data'/'fiber' 区間として載せる
```
（これは G5-P6 のキャリア回線/cross-connect と地続き。構内構造化配線＝物理層、回線＝論理層として両方持つ。）

出典: TIA-942 配線スペース https://www.fs.com/blog/comprehensive-guide-for-data-center-distribution-area-connectivity-9579.html ／
OCP Open Rack V3 https://www.opencompute.org/documents/open-rack-base-specification-version-3-pdf , https://en.wikipedia.org/wiki/Open_Rack ／
ETSI EN 300 119 https://www.etsi.org/deliver/etsi_en/300100_300199/30011907/01.01.01_60/en_30011907v010101p.pdf

---

## 7. スコープ判定に沿った採用リスト（06章 C の続番として）

§0.5 の判定を実装順に並べ替えたもの。**まず ✅ コアを取り込み、🔶 は最小列で、❌ は連携キーのみ**。

### ✅ コア — スキーマで own（実装する）
| 優先 | 項目 | 対応 |
|------|------|------|
| P0 | G6 ラック規格・OCP/DC給電 | `rack.mount_standard`/`mount_width_in`/`unit_height_mm`・`rack_unit_occupancy.unit_type{U\|OU}`・`power_feed.power_type{ac\|dc}`/`busbar_rating_a`・`equip_kind` に power_shelf/busbar |
| P0 | G6 構造化配線スペース | `loc_type` 語彙に entrance_room/mda/hda/zda/eda・`equip_kind` に cross-connect |
| P1 | G5-P6 キャリア回線・cross-connect・MMR | `carrier`/`circuit`/`cross_connect`・`loc_type='mmr'`。※IPAM は対象外 |
| P1 | G3 発電機・燃料 → 容量次元 | `generator`/`fuel_tank`・`capacity_budget` に `dimension='fuel_runtime_h'`（無給油運転可能時間） |
| P2 | G4 水・排熱の計量 | `device_category.resource_class` に water・`equip_link substance='heat'` outbound・補給水/排熱輸出メータの series |

### 🔶 シーム — 属性・参照キーだけ持つ（最小実装）
| 優先 | 項目 | 対応 |
|------|------|------|
| P1 | G1 信頼性/保護/粒度の等級 | `room.tier`・`location.protection_class`・`room_group.granularity_level` 程度の**少数列**。多スキーム表は作らない |
| P1 | G2 耐震・立地ハザード（日本市場） | `building_attr`（耐震構造/PML/床下高）・`site_hazard`（洪水/活断層）・`rack.seismic_mount`。外部評価結果の保持に限定 |
| P2 | G5-P2 物理 MAC ワークフロー | 最小の作業オーダー＋ `ticket_ref`（ITSM 連携）。フルな変更管理は持たない |
| P2 | G5-P3 保証/EOL/EOSL・SLA 属性 | `device`/`maintenance_contract` への属性追加（`eol_date`/`eosl_date`/`response_sla`） |

### ❌ 対象外 — 連携キーのみ（DCIM では実装しない）
| 項目 | 連携先 | DCIM 側に置くもの |
|------|--------|-----------------|
| G4 炭素会計・省エネ法/温対法/EED 申告 | ESG/サステナ基盤 | `measurement`/`series` の**読み取り公開ビュー**のみ |
| G3 法定点検記録・有資格者・報告 | CMMS/施設管理 | `device.external_cmms_id`・次回点検日の表示参照 |
| G5-P1 入退室・バッジ・カメラ | PSIM/入退管理 | ゾーンの保護クラス属性＋ PSIM 参照キー |
| G5-P4 インシデント/SLA/RCA | ITSM | アラート発火時の `ticket_ref` |
| G5-P5 減価償却・調達・簿価 | ITAM/ERP | `device.asset_tag`/`external_asset_id`（既存） |
| G5-P7 監査ログ | フレームワーク層 | （DCIM 固有のドメイン機能ではない） |

> **総括**：[06章](./06-self-review.md) が「設計の内的健全性」を保証したのに対し、本章は**外部基準との照合から DCIM の境界線を引く**。
> 当初の網羅調査は「DC に必要なデータ」を 6 クラスタで広く拾ったが、**その多くは DCIM が own すべきデータではなかった**。
> 結論は **(1) 実装すべき ✅ コアは G6（ラック規格/OCP/構造化配線）・G5-P6（キャリア回線）・発電機キャパシティ・水/排熱計量に限る**、
> **(2) G1 等級・G2 耐震は 🔶 最小列の属性に留める**、**(3) 炭素会計・法定点検・入退室・ITSM・財務は ❌ 連携シーム（参照キー）に格下げ**。
> こうして「DCIM の本分」に収めることが、肥大化を防ぎつつ「可変性 × ドメイン制約の厳格さ」を保つ唯一の道。**抱え込む範囲を絞ることが、本章の最大の成果**。

---

## 8. 出典（クラスタ別・主要）

- **JDCC**: ファシリティスタンダード概要 https://www.jdcc.or.jp/pdf/facility.pdf ／ 運用ガイドブック解説 https://cafe-dc.com/japan/jdcc-op-guidebook/ ／ 地域共生ガイドライン https://www.jdcc.or.jp/お知らせ/7389/ ／ FS 変遷 https://cloud.watch.impress.co.jp/docs/cdc/jdcc/1284723.html
- **日本法規**: 建築設備耐震指針 https://www.mlit.go.jp/common/001211554.pdf ／ 消防設備点検報告 https://www.tfd.metro.tokyo.lg.jp/lfe/office_adv/tenken_houkoku.html ／ 電気事業法 https://www.mlit.go.jp/common/001318576.pdf ／ 高圧ガス（冷凍） https://www.pref.fukui.lg.jp/doc/012643/kouatu_d/fil/4002.pdf ／ 非常用発電/燃料 https://fukashiken-techno.co.jp/column/emergency-generator/running-hours/
- **国際標準**: TIA-942 https://tiaonline.org/products-and-services/tia942certification/tia-942-certifications-ratings/ ／ EN 50600/ISO 22237（VK/SK/GN）https://www.hknow.de/en/certifications/iso-22237/ , https://www.hknow.de/en/planning/en50600/ ／ Uptime M&O/Tier https://uptimeinstitute.com/professional-services/management-operations ／ OCP Open Rack V3 https://www.opencompute.org/documents/open-rack-base-specification-version-3-pdf ／ ETSI EN 300 119 https://www.etsi.org/deliver/etsi_en/300100_300199/30011907/01.01.01_60/en_30011907v010101p.pdf
- **サステナビリティ**: ISO/IEC 30134 系 https://www.iso.org/standard/71717.html ／ EN 50600-4 KPI https://www.stulz.com/newsroom/detail/making-efficiency-comparable-kpis-for-en-50600-compliant-data-centers/ ／ GHG Protocol Scope 2 https://www.persefoni.com/blog/scope-2-dual-reporting-market-and-location-based-carbon-accounting ／ EU EED 2024/1364 https://www.gov.ie/en/department-of-climate-energy-and-the-environment/publications/data-centre-energy-and-sustainability-performance-reporting-obligations/ ／ 省エネ法 DC ベンチマーク https://www.enecho.meti.go.jp/category/saving_and_new/saving/enterprise/factory/support-tools/data/2023_01benchmark.pdf ／ 温対法 https://policies.env.go.jp/earth/ghg-santeikohyo/calc.html
- **運用**: 物理セキュリティ https://www.dronestrategicpartners.com/post/data-center-physical-security-compliance-requirements-and-technology-architecture ／ ISO 27001 A.7.2 https://www.hicomply.com/en-us/iso-27001/access-control-to-premises-annex-a-7-2 ／ 保守/MOP https://eworkorders.com/cmms-industry-articles-eworkorders/data-center-maintenance-strategies/ ／ 可用性メトリクス https://www.atlassian.com/incident-management/kpis/common-metrics ／ cross-connect/MMR https://dgtlinfra.com/cross-connects-interconnection-services/
