# 06. セルフレビュー — SQL アンチパターン & データセンター一般化

推奨案 **P05**（[03-finalists.md](./03-finalists.md) / [05-er-diagram.md](./05-er-diagram.md)）を、
(A) **SQL アンチパターン**（主に Bill Karwin『SQL Antipatterns』の分類）と
(B) **データセンターへの一般化（多数 DC へ展開するパッケージとしての汎用性）** の 2 観点で批判的に点検する。

重大度: 🔴 要修正（設計の前提を崩す） / 🟡 要改善（運用で効く） / 🟢 設計判断（許容だが明示すべき）

---

## A. SQL アンチパターンのセルフレビュー

### A-1. 🔴 Polymorphic Associations（多態関連）— 最大の課題

該当箇所（複数）:
- `cable_termination(term_kind, term_id)` … ケーブル端点が `power_port`/`interface`/… のどれかを指す多態 FK
- `threshold(scope_type, scope_id)` … global/location/device のいずれか
- `capacity_budget(scope, location_or_rack_id)` … location か rack
- `series.series_id` ↔ `measurement`（hypertable 側）… 論理参照

**問題**: 多態 FK は **DB レベルの参照整合性を放棄**する（`term_id` が実在ポートを指す保証が無い・孤児が出る・JOIN が型分岐で複雑化）。これは本企画の宣言目標「**RDBMS としての制約をしっかり付ける**」と正面衝突する。現状の既定設計は整合をトリガ＋定期チェックに委ねており、**目標に対して弱い**。

**修正方針（厳格版を既定に格上げ）**:
1. **排他的アーク（exclusive arc）**: 参照先ごとに nullable な実 FK 列を並べ、`CHECK` で「ちょうど1つ非NULL」を強制。
   ```sql
   -- threshold を排他アーク化（実FKが張れる）
   CREATE TABLE threshold (
       id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       device_id   bigint REFERENCES device(id)   ON DELETE CASCADE,
       location_id bigint REFERENCES location(id) ON DELETE CASCADE,
       is_global   boolean NOT NULL DEFAULT false,
       metric_def_id smallint NOT NULL REFERENCES metric_definition(metric_def_id),
       ...,
       CHECK ( (device_id IS NOT NULL)::int + (location_id IS NOT NULL)::int
               + is_global::int = 1 )      -- ちょうど1スコープ
   );
   ```
2. **種別別終端テーブル + UNION ビュー**（`cable_termination` 向け、03章で「厳格版」として言及済み）:
   `ct_power_port(cable_id, side, power_port_id FK)` / `ct_interface(...)` … と分割し**実 FK + 個別 UNIQUE**、
   参照は `UNION ALL` ビューで束ねる。型安全 100%。テーブル数増がコスト。
3. NetBox 流（多態 + アプリ層整合 + path cache）は実績はあるが、本パッケージの「制約厳格」志向では
   **1 または 2 を既定**にすべき。`capacity_budget` も `capacity_budget_rack` / `capacity_budget_location` に分割推奨。

> **結論**: 現状の「多態 FK を既定、厳格版はオプション」は逆。**厳格版（排他アーク / 種別別 + UNION）を既定**にし、
> 多態 FK は「拡張点を増やしたい場合のオプション」に降格する。

### A-2. 🟡 EAV の JSONB 退避は是だが、検証が甘い

`device_type.specs jsonb` / `data_point.addr jsonb` は EAV アンチパターンの正しい回避（JSONB 上位互換）。
ただし現状の `addr_shape` CHECK は**キー存在しか見ていない**（`addr ? 'oid'`）。値の型・範囲・余剰キーは野放しで、
「silent bad data」を許す。

**改善**:
- `pg_jsonschema` の `jsonb_matches_schema()` を `CHECK` に組み込み、**カテゴリ/プロトコル別の JSON Schema** で
  型・必須・enum・追加禁止（`additionalProperties:false`）まで検証。
- 「**クエリ・制約・集計の対象になる属性は JSONB に入れず実列にする**」原則を明文化（現状はコア数値を実列化しており方向は正しい）。
- JSONB はあくまで「ロングテール・ベンダ固有・表示専用」属性に限定。

### A-3. 🟡 31 Flavors（CHECK-IN 列挙のハードコード）— 一般化と直結

`loc_type` / `status` / `protocol` / `phase` / `airflow` / `face` / `value_kind` / `default_agg` など多数が
`CHECK (col IN ('a','b',...))`。**パッケージを多数 DB に配布する場合、列挙値の追加に全 DB への ALTER（マイグレーション）が必要**になり、汎用性を阻害する（B-1 と同根）。

**判断軸**:
- **不変の語彙**（`face`=front/rear、`side`=A/B、`supply`=ac/dc、`phase`=single/three）は CHECK or PostgreSQL `ENUM` で良い（変わらない）。
- **増えうる語彙**（`status` ライフサイクル、`loc_type`、`protocol`、`airflow`、`device_category`〔既にテーブル化済〕）は
  **ルックアップテーブル化**し、データで拡張可能にする。
- 既に `device_category` / `metric_definition` / `loc_type_rule` はテーブル化しているのに `loc_type` 本体や `status` が
  CHECK 列挙のまま、という**不整合**を解消する。

### A-4. 🟡 Rounding Errors（金額・電力量に FLOAT）

[04 章](./04-validation-queries.md) UC-1 の電気代計算は `double precision`（`avg_w`）から kWh・円を算出している。
**通貨・課金額は `numeric` を使う**のが鉄則（浮動小数の丸め誤差は課金で許されない）。
テレメトリ実測値の `double` は妥当だが、**派生する金額・kWh は `numeric` にキャスト**して計算・保存する。

### A-5. 🟢 Keyless Entry（hypertable に FK 無し）— 設計判断だが副作用を明示

TimescaleDB の `measurement` に FK を張らないのは正しい（チャンク DROP/圧縮の阻害回避）。ただし副作用:
**`device` 削除で `data_point`/`series` は CASCADE 削除されるが、`measurement` 行は FK が無いので残る**（孤児時系列）。
→ retention で自然消滅するが、即時整合が要るなら **series 退役時にバックグラウンドで該当 series_id を物理削除**する
ジョブ（または `drop_chunks` と併用）を運用に組み込む、と明記すべき。

### A-6. 🟡 単位の誤称（`available_power_w` は実は VA）

`power_feed.available_power_w = round(V×A×util/100×√3)` は **皮相電力(VA)** であって有効電力(W)ではない（力率が未考慮）。
W と VA の混同はドメイン上の誤り。**`available_power_va` に改名**し、必要なら `power_factor` 列を足して
`available_power_w = available_power_va × power_factor` を別途用意する。`√3` も `sqrt(3.0)` を使い `1.732` の
マジックナンバー誤差を避ける。

### A-7. 🟢 その他（軽微）

- **ID Required**: 自然複合キー（`capacity_budget`/`device_demand`/`loc_type_rule`）は surrogate を付けず複合 PK にしており、過剰な代理キーは無い。良。
- **秘密情報**: `protocol_profile.secret_ref`（Secrets Manager 参照）で平文を DB に置かない設計は良（Readable Password 回避）。
- **Multicolumn Attributes 回避**: A/B 冗長を `PanelID/PanelID2`（OpenDCIM 流の多列）でなく `redundancy` enum + 行で表現しており良。
- **Metadata Tribbles 回避**: `measurement_1m/1h/1d` は CAGG ロールアップであり「期間ごとのテーブル乱立」ではない。良。

---

## A-8. 🔴→✅ 発見した正当性バグ：U 位置 EXCLUDE のフルデプス畳み込み（修正済）

> **更新①（PR レビュー）**: いったん `face_range int4range` + EXCLUDE で修正。
> **更新②（ポータビリティ化・[09 章](./09-portability.md)）**: 最終的に下記 (a)「占有を面ごと行展開」を採用。
> `rack_unit_occupancy(rack_id, rack_side, face, u)` の**複合主キー**で重なりを禁止し、
> `EXCLUDE`/`btree_gist`/range 型を排除（MySQL でも同形）。full-depth は front/rear 両面に占有行を入れて
> 片面機器と確実に衝突させる。原子的・拡張ゼロ・全エンジン移植可。


[03 章](./03-finalists.md) の `rack_mount` の排他制約:
```sql
EXCLUDE USING gist (
    rack_id WITH =, rack_side WITH =,
    (CASE WHEN is_full_depth THEN 'both' ELSE face END) WITH =,   -- ← 問題
    u_range WITH &&)
```
**バグ**: フルデプス行は face キーが `'both'`、ハーフデプス前面行は `'front'`。EXCLUDE は**キーが等しい行同士**でしか
衝突判定しないため、`'both'` と `'front'` は**衝突しない**。結果、フルデプス機器と同一 U の前面/背面ハーフデプス機器が
**重なって搭載できてしまう**（意図と逆）。

**修正案**:
- (a) **占有を面ごとに 2 行で表現**: フルデプス機器は `front` と `rear` の 2 占有行を持たせ（生成 or トリガ）、
  排他キーは純粋に `(rack_id, rack_side, face, u_range)`。シンプルで正しい。
- (b) **CONSTRAINT TRIGGER で検証**: 挿入時に「同 U・(同面 or 相手がフルデプス)」を明示チェック。
- (c) 面を範囲に織り込む高度な手（front=range[0,1), rear=[1,2), full=[0,2)）も可能だが (a) が明快。
→ **(a) を採用**し、05 章 ER と 03 章 DDL を訂正すべき（本レビューで判明）。

---

## B. データセンターへの一般化（パッケージ汎用性）のセルフレビュー

「いろんなデータセンターに適用するパッケージ」という目標に対し、**特定 DC を暗黙に仮定して硬直化している箇所**を洗い出す。

### B-1. 🔴 空間タイプ語彙のハードコード

`loc_type IN ('region','campus','building','floor','room','cage','zone','row')` は語彙を固定している。
実 DC は **"data hall" / "suite" / "pod" / "white space" / "MMR" / "MDF/IDF" / "電気室"** など呼称・段数が多様。
**`loc_type` をルックアップテーブル化**（`location_type(code, label, rank)`）し、親子規則 `loc_type_rule` と整合させる。
これで「建屋なし DC」「ケージ階層が深いコロ」「ゾーン主体のハイパースケール」すべてをデータで吸収できる
（A-3 と同じ修正で一石二鳥）。

### B-2. 🔴 単位系・計測系の正規化が無い

`metric_definition.unit text` は単位を文字列で持つだけで、**正規化・換算が無い**。多リージョン展開では
**℃/℉・kg/lb・kW/BTU・kWh** が混在する。
**改善**: 物理量ごとに **canonical SI 単位で保存**（°C, W, Wh, Pa…）し、表示単位は別管理（`display_unit`）。
取込時に換算（研究調査でも「ingest 時に SI 正規化」を推奨済だが、スキーマに落ちていない）。閾値・ASHRAE 域も SI で持つ。

### B-3. 🟡 三相電力・力率モデルの浅さ

`power_feed` は単一の `voltage_v`/`amperage_a` のみ。実 3 相は **L1/L2/L3 の相別電流**を持ち、
**相不平衡**が容量計算に効く。`feed_leg(A/B/C)` は outlet にあるが feed 側の相別メータリングが無い。
キャパシティ算定（`available_power`）も相不平衡・力率（VA↔W、A-6）を無視している。
**改善**: `power_feed_phase(feed_id, leg, voltage_v, amperage_a, current_a)` を追加 or 相別列、`power_factor` を導入。

### B-4. 🟡 タイムゾーン非対応

`location` に時間帯が無い。多リージョン DC で **ローカル日次の電気代/PUE 集計**や CAGG の `time_bucket` は
タイムゾーンを要する。**サイト相当の location に `time_zone text`** を持たせ、課金・日次ロールアップで使用する
（`time_bucket('1 day', ts, 'Asia/Tokyo')`）。

### B-5. 🟡 運用パラメータが DDL にベタ書き

`chunk_time_interval => '1 day'`、retention `30 days/2 years/10 years`、ポーリング周期などが固定。
**DC 規模で最適値が大きく変わる**（高頻度・大量 series の DC と小規模 DC で別）。パッケージとしては
**デプロイ別の設定（config テーブル or デプロイ変数）から流し込み**、DDL に焼き込まない方が良い。

### B-6. 🟡 カスタムメトリック / カスタム属性の余地

`metric_definition.metric_def_id smallint` はパッケージ同梱の固定コード。これは**全 DC で ID をブレさせない**利点が
ある一方、**顧客独自メトリックを追加する余地が無い**（ID 衝突）。
**改善**: 顧客カスタム用に **ID レンジを予約**（例: 1–9999 はパッケージ、10000+ は顧客）または `source('builtin'|'custom')` +
`code` 名前空間。資産側も NetBox の `custom_field_data` 相当（顧客定義フィールド）を用意すると汎用性が増す。

### B-7. 🟡 マルチテナント / コロケーションの深さ

`tenant_id` 単一 FK のみ。コロ事業者は **テナント階層（TenantGroup）・ケージ/ラック/U 単位の割当・再課金境界**が要る。
`cage` loc_type + `tenant_id` で最低限は表せるが、**割当（allocation）と所有（ownership）の分離**、
テナント別キャパシティ予算は未モデル。コロ向けに展開するなら拡張が必要。

### B-8. 🟢 スコープの明示（意図的な割り切り）

以下は「DCIM コア」に絞った結果で、汎用化の観点では**スコープ判断として明示すべき**:
- **IPAM 不在**: `interface` はあるが IP/VLAN/VRF が無い（NetBox/Device42 は IPAM 統合）。純 DCIM なら割り切りで可。
- **冷却トポロジーの薄さ**: `capacity_budget` に cooling 次元はあるが、**CRAC/CRAH→ラック/部屋の冷却供給割当・
  気流・封じ込め・RDHx ループ**が無い（Schneider EcoStruxure の強み領域）。サーマル本格運用には不足。
- **0U / バスウェイ / 架空配線**: `rack_mount` は U 位置前提で、**0U 縦型 PDU・busway tap-off**の表現が弱い
  （Hyperview の RackPosition=Left/Right/Above/Below 相当）。`rack_side` で部分対応だが 0U 専用表現を足すべき。
- **ディスカバリ来歴**: Device42/Hyperview は discovery-first。**source（discovered/manual/import）・last_seen・
  merge** 来歴が無く、自動投入運用には弱い。
- **DC/直流給電・OCP**: `supply='dc'` はあるが 48V 通信局給電・OCP busbar(51/54V) のモデルは薄い。

これらは「初版スコープ」として README に**意図的非対象**と明記し、拡張ロードマップに置くのが誠実。

---

## C. 優先度付き是正リスト

| 優先 | 項目 | 対応 |
|------|------|------|
| ✅ done | A-8 U位置 フルデプスバグ | **占有U行 + 複合UNIQUE** で解決（[09 章](./09-portability.md)・03/05 章反映。EXCLUDE 廃止で移植性も両立） |
| 🔴 P0 | A-1 多態関連 | 厳格版（排他アーク / 種別別+UNION）を既定に格上げ（[08 章](./08-tenancy-colocation.md)の lease で適用済） |
| 🔴 P1 | B-1 / A-3 空間タイプ・status 等の語彙 | ルックアップテーブル化（増えうる語彙のみ。[09 章 9.2](./09-portability.md) でも推奨） |
| 🔴 P1 | B-2 単位正規化 | canonical SI 保存 + display_unit + 取込換算 |
| 🟡 P2 | A-2 JSONB 検証 | アプリ層で JSON Schema 検証（pg_jsonschema 非依存・移植性のため） |
| 🟡 P2 | A-6 / B-3 VA/W・三相・力率 | `_va` 改名 + power_factor + 相別モデル |
| 🟡 P2 | A-4 金額は numeric | 課金・kWh は numeric で計算 |
| 🟡 P3 | B-4 タイムゾーン | site location に time_zone |
| 🟡 P3 | B-5 運用パラメータ外出し | chunk/retention/poll を config 化 |
| 🟡 P3 | B-6 カスタム拡張 | metric ID レンジ予約 + custom fields |
| 🟢 P4 | A-5 孤児時系列 | series 退役時クリーンアップ運用を明記 |
| ✅ done | B-7 マルチテナント/コロ | 分離可能モジュールとして設計（[08 章](./08-tenancy-colocation.md)）。運用プロファイルで切替 |
| 🟢 P4 | B-8 スコープ（IPAM/冷却/0U 等） | 拡張ロードマップとして明示 |

> 総評: **空間階層・機器マスタ・U 位置（バグ除く）・テレメトリ/CAGG の骨格は健全**。一方で
> **(1) 多態関連を既定で実 FK 化、(2) 増えうる語彙と単位をデータ駆動化、(3) フルデプス排他バグの修正**の 3 点が、
> 「制約厳格 × 多 DC 汎用」という当初目標を本当に満たすための最重要改修である。次版（P05.1）でこれらを反映する。
