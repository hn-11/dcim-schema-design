# DCIM データスキーマ設計検討

データセンターインフラ管理（DCIM）パッケージソフトの **RDBMS テーブル構造**の設計検討書。
時系列データは **TimescaleDB**（PostgreSQL 拡張）に格納する前提で、構成・マスタデータの
リレーショナルモデルと時系列ストレージの接続点を設計する。

## 設計ゴール

> **Project Haystack のセマンティクス**（点の意味を直交タグの合成で表す）を **RDBMS の参照表＋FK＋CHECK に
> reify（実体化）**し、**Schneider EcoStruxure IT のドメインモデル**（power path / Genome / 容量管理 / 冷却チェーン）を
> 型付きで取り込む。「**ゆるい IoT スキーマにせず、データセンターのドメイン知識を DB 制約で担保する**」。

加えて、多数の DC へ横展開する**パッケージ**として **可変性**（機種・メトリック追加が DDL 不要）と、別スタックへ
載せ替え可能な **ポータビリティ（LCD・[09章](./09-portability.md)）**、**マルチテナント/コロケーション
（[08章](./08-tenancy-colocation.md)）**を両立する。

## ドキュメント構成

> **表記方針**: テーブルは原則 **Mermaid ER 図**で示す。CHECK 制約・生成列・複合 FK など ER で表せない
> ドメイン制約のみ SQL 抜粋を併記する。

| ファイル | 内容 |
|---------|------|
| [00-overview.md](./00-overview.md) | **チーム共有/相互レビュー用 概要**：ER 図のガイドツアー＋解説で全体像を 10 分で把握 |
| [01-research-and-domain.md](./01-research-and-domain.md) | ドメイン知識 + 参照モデル調査（**Project Haystack** / **EcoStruxure IT** / NetBox / Brick）+ オントロジを RDBMS に reify するパターン |
| [02-candidate-patterns.md](./02-candidate-patterns.md) | **設計が割れる4判断点**（オントロジ格納 / 点の意味論 / 電力冷却チェーン / 階層 DAG）の数パターン比較と採否 |
| [03-finalists.md](./03-finalists.md) | 確定設計 **Semantic-Typed DCIM** の全レイヤ（L1〜L10 の ER 図 + 重要制約の SQL 抜粋） |
| [04-validation-queries.md](./04-validation-queries.md) | ユースケース駆動のスキーマ検証 SQL（電気代/PUE・温度推移・U 空き・上流トレース・SPOF・容量） |
| [05-er-diagram.md](./05-er-diagram.md) | 全体 ER 図（Mermaid、ドメイン別 + 全体俯瞰 + 図に表れない制約） |
| [06-self-review.md](./06-self-review.md) | セルフレビュー（EAV/DAG 閉包/多態キー等のアンチパターン是正・優先度付き） |
| [08-tenancy-colocation.md](./08-tenancy-colocation.md) | テナント/コロケーション拡張（cage 境界・contracted power・運用プロファイルで切替可能な加算レイヤ） |
| [09-portability.md](./09-portability.md) | ポータビリティ設計（拡張は timescaledb のみ・別スタック載せ替えに耐える LCD アーキテクチャ） |
| [10-room-group-derived-metrics.md](./10-room-group-derived-metrics.md) | 部屋グループ & 派生メトリクス（pPUE 等を Point 化。分電盤の部屋またぎ → 電力境界の連結成分を集計単位に） |
| [11-standards-gap-analysis.md](./11-standards-gap-analysis.md) | **標準・法規ギャップ分析＋スコープ判定**：JDCC・日本法規・国際標準（TIA-942/EN 50600/ISO 30134/OCP）・省エネ法/温対法/EU EED と突合し欠落観点を 6 クラスタで抽出 → **「DC に必要 ≠ DCIM に必要」で ✅コア/🔶連携シーム/❌対象外に判定**（§0.5） |

---

## 結論（先出し）

### 🏆 確定設計：**Semantic-Typed DCIM**

データセンターの実体モデル（空間・資産・配電・冷却・容量・計測）を関係モデルで持ち、ドメイン制約を DB に効かせる 10 レイヤ構成。

| レイヤ | 採用方式 | 由来 |
|----|---------|------|
| L1 参照カタログ | `equip_kind`（機器種別ツリー・power_class）/ `medium`（媒体）。固定集合は CHECK enum | — |
| L2 空間 | `location` 木＋閉包（`ltree` 不使用 LCD）/ `rack` 固定アンカー / U 重なりは占有U行 | EcoStruxure |
| L3 資産 | **Genome=`equipment_type` → 実機 `equipment`**（定義/実体分離・`equip_kind` で意味づけ） | EcoStruxure Genome / NetBox |
| L4 メトリック | **`metric` フラットカタログ**（code/category/unit/datatype/既定集約・位置はコードに織込） | Redfish/Prometheus 流 |
| L5 収集 | `conn`＋`data_point`（equipment×metric×**role**×phase + 取得アドレス）/ cur・his・writable | EcoStruxure / BMS |
| L6 時系列 | Narrow `measurement`(his) ＋ `current_value`(cur) ＋ `series` 台帳（FK 越境なし） | TimescaleDB |
| L7 電力・冷却・冗長 | `equipment` ノード ＋ **`connection`/`power_connection`(CTI)** で受電→変圧器→UPS→盤→PDU→rack を表現 / `v_equip_flow`(導出) / `redundancy_group`(N+1/2N) | EcoStruxure power path |
| L8 容量 | **WP-150 容量5要素 × 推定負荷戦略**（nameplate/adjusted/predicted/contracted）/ stranded・排他アーク | EcoStruxure / APC WP-150 |
| L9 監視 | `threshold`（severity informational/warning/critical・hysteresis・排他アーク） | EcoStruxure |

### 採用した主な設計判断（[02章](./02-candidate-patterns.md)）

| 判断 | 採用 | 一言 |
|------|------|------|
| 点の定義 | **`metric` フラットカタログ** | 量・単位・型を1行。位置はコードに織り込む（〜100 行で爆発しない） |
| 役割 | **`data_point.role`**（sensor/sp/cmd/derived） | 制御を見据え、役割は型でなく点の属性 |
| connectivity | **物理 path＝真実源、フローは導出** | `v_equip_flow`。並行する別グラフを持たない |
| 冗長 | **`redundancy_group` + member** | 意図(N+1/2N)を持ち、SPOF/容量で物理を検証 |
| スコープ参照 | **排他アーク FK** | 多態キー（type+id）を廃し参照整合を DB に残す（固定3対象） |

## 主要な設計判断（全レイヤ共通の原則）

1. **ドメインの実体を関係モデルで、分類は参照表 FK で** — 種類・量は `equip_kind`/`metric` への FK で貼る。EAV や汎用オントロジ（万能 def / 多重継承 DAG）にしない（[02章](./02-candidate-patterns.md)）。
2. **connectivity の真実源は1つ** — equipment 間の接続を `connection`/`power_connection`(CTI) に保存し、電力フロー/A系B系/dual-cord は `v_equip_flow` で**導出**（並行する別グラフを手で持たない・[03章L7](./03-finalists.md)）。
3. **時系列とマスタの分離 + 越境しない** — TimescaleDB hypertable に FK を張らず、`series_id`（整数）だけ越境（[09章](./09-portability.md)）。
4. **「定義」と「実体」の分離** — Genome（`equipment_type`）に型情報、実機（`equipment`）に個体情報。DDL 変更なしで機種追加。
5. **集約制約はサービス層で** — 「合計負荷 ≤ ブレーカ容量」等の行間集約は単一行 CHECK 不能。移植性のため DB トリガに依存させず単一書き込み口で担保。
6. **誤データは DB が弾く** — U 衝突は占有U行＋複合UNIQUE、単位整合は metric が単位を1つ持つこと、スコープ参照は排他アーク FK で。
7. **制御を見据える** — `data_point.role`(sensor/sp/cmd)＋`is_writable` でシームを切る（優先配列・write 監査は本実装時）。
8. **Haystack/Brick は発想の参考**で互換は必須にしない（必要なら境界で後付けマッピング）。

## 参考にした主な一次情報源

- **Project Haystack 4 defs**（site/space/equip/point、pointFunction/Quantity/Subject、is(DAG)、units.txt/prefUnit、inputs/outputs + substance-ref、connector、dataCenter/rack/ups/crac）
- **EcoStruxure IT / APC**（Genome、power path、breaker panel、paired power consumers、WP-150 容量管理、cooling rack/row/room、PUE/pPUE、Tenant Portal）
- **Brick Schema / ASHRAE 223P**（RDF/OWL 形式オントロジとの対比・収束）
- NetBox DCIM models / OpenDCIM `create.sql`（DeviceType/Cable/PowerFeed、A/B 冗長給電）
- **SQL Antipatterns**（Karwin: EAV / Naive Trees / 閉包テーブル）/ PoEAA（CTI/STI）
- PostgreSQL 公式 / TimescaleDB 公式（hypertable / continuous aggregate / 圧縮 / retention）

各ドキュメント末尾に出典 URL を記載。
