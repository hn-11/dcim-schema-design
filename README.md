# DCIM データスキーマ設計検討

データセンターインフラ管理（DCIM）パッケージソフトの **RDBMS テーブル構造**の設計検討書。
時系列データは **TimescaleDB**（PostgreSQL 拡張）に格納する前提で、構成・マスタデータの
リレーショナルモデルと時系列ストレージの接続点を設計する。

> ⚠️ 本ディレクトリは P-TAG（スポーツ向けフィジカルデータ基盤）とは独立した
> DCIM 製品のスキーマ検討資料です。検討用ブランチ `claude/dcim-schema-design-o23aez` 上の
> 成果物であり、本リポジトリの既存プロダクトには影響しません。

## 設計ゴール

> いろんなデータセンターに適用できる**パッケージソフト**にしたい。ある程度融通が利き、
> それでいてドメインナレッジをふんだんに活かして RDBMS としての制約をしっかり付けた
> データモデル。

この「**可変性（多数の DC へ横展開）**」と「**ドメイン制約の厳格さ（誤データを DB が弾く）**」
の両立が全評価軸の根底にある。

## ドキュメント構成

| ファイル | 内容 |
|---------|------|
| [01-research-and-domain.md](./01-research-and-domain.md) | データセンターのドメイン知識、既存 DCIM 製品（NetBox / OpenDCIM 等）のデータモデル調査、設計軸（Axis）の定義 |
| [02-candidate-patterns.md](./02-candidate-patterns.md) | **20 パターン**のテーブル構造案、比較マトリクス、**有力 5 案への絞り込み** |
| [03-finalists.md](./03-finalists.md) | 有力 5 案の詳細設計（DDL スケッチ・制約・トレードオフ） |
| [04-validation-queries.md](./04-validation-queries.md) | Step 6：ユースケース駆動のスキーマ検証 SQL（電気代計算・温度推移・U 空き検索・SPOF 検出 等） |
| [05-er-diagram.md](./05-er-diagram.md) | 推奨案 P05 の ER 図（Mermaid、ドメイン別 + 全体俯瞰） |
| [06-self-review.md](./06-self-review.md) | セルフレビュー：SQL アンチパターン & データセンター一般化（優先度付き是正リスト） |
| [07-er-diagram-alternatives.md](./07-er-diagram-alternatives.md) | 他の有力 4 案（P04/P06/P08/P02）の ER 図（Mermaid、P05 との差分） |
| [08-tenancy-colocation.md](./08-tenancy-colocation.md) | テナント/コロケーション拡張モジュール（エリア貸し等・運用プロファイルで切替可能な加算レイヤ） |
| [09-portability.md](./09-portability.md) | ポータビリティ設計（拡張は timescaledb のみ・別スタック載せ替えに耐える LCD アーキテクチャ） |

## 検討プロセス（本書の作り方）

ユーザー提示の 6 ステップ計画に沿って、5 つの設計軸（空間階層 / 機器マスタ / 配線トポロジー /
テレメトリ・TimescaleDB / 集約・キャパシティ）を並列調査。各軸の選択肢を組み合わせて
20 の整合的なスキーマ案を構成し、7 つの評価基準で採点して 5 案に絞り込んだ。

```
Step1 空間階層 ─┐
Step2 機器マスタ ┤
Step3 配線      ┼─→ 軸ごとの選択肢 ─→ 20パターン構成 ─→ 比較採点 ─→ 有力5案 ─→ Step6検証
Step4 テレメトリ ┤
Step5 集約      ─┘
```

---

## 結論（先出し）

### 🏆 推奨：**P05「パッケージ標準（NetBox 流 + JSONB ハイブリッド）」**

多数の DC に横展開するパッケージとして、可変性とドメイン制約のバランスが最良。

> **ポータビリティ方針（[09 章](./09-portability.md)）**：拡張は **timescaledb のみ**。PG 固有機能を避けた
> 最小公倍数SQL(LCD)で記述し、別スタック（例: MySQL + ClickHouse）へ**アーキテクチャ作り直しなし**で
> 載せ替え可能にする（差し替えは方言DDL/時系列アダプタ/サービス層の3か所のみ）。

| 軸 | 採用方式 |
|----|---------|
| 空間階層 | **ハイブリッド**：`Rack` は専用テーブル（固定アンカー）＋ `Site〜Row` は汎用 `location` 木（隣接リスト＝真実源 ＋ **閉包テーブル** `location_closure`。`ltree` 不使用） |
| 機器マスタ | **DeviceType／Device 分離**（NetBox 流）：型番マスタに固有属性とポートテンプレート、実機は共通属性のみ。コア数値属性は型付き列、ロングテール属性は `JSON`（検証はアプリ層） |
| 配線 | **種別別ポート ＋ Cable／CableTermination**（実体モデル）。電力／ネットの 2 有向グラフを `WITH RECURSIVE` でトレース＋ path キャッシュ |
| U 位置 | **占有U行 `rack_unit_occupancy` ＋ 複合UNIQUE**で物理重なりを DB が保証（拡張ゼロ・全エンジン移植可。フルデプスは front/rear 両面に占有行） |
| テレメトリ | **Narrow（long format）** `(time, series_id, value)` ＋ `series` 台帳。`compress_segmentby=series_id`、retention、プロトコル固有マッピングは型付き列＋`JSON` ハイブリッド |
| 時間集約 | **ロールアップの階層化**（1m→1h→1d）。avg は `sum+count` で厳密、min/max/sum/count は再集約安全。**p95 は任意列**（engine依存・既定は都度計算） |
| 空間集約 | `location_closure` の**オンザフライ JOIN**を主、最頻アクセスの粗粒度のみ `current_value` 表へマテリアライズ |
| 最新値 | 専用 `current_value` 表を UPSERT（`alert_state` 同居） |

### 他の有力 4 案（適合シナリオ別）

| 案 | 性格 | 向くケース |
|----|------|-----------|
| **P04 NetBox 忠実（強整合）** | 制約最優先・実績重視。固有属性も CTI で型付け | 機器種別が安定し、整合性を最大化したい大規模単一運用者 |
| **P06 閉包テーブル読み最適化** | 階層クエリ最速（再帰不要の JOIN） | 階層が深く読みが支配的・ダッシュボード多数同時 |
| **P08 時間軸正確（Temporal）** | 機器移設の履歴を `valid_from/to` で正確に保持 | 電力課金・エネルギー会計の**履歴正確性**が最重要 |
| **P02 JSONB 軽量（MVP）** | 可変性最大・実装最小。`ltree`＋全部 JSONB | 小規模 DC・短期立ち上げ・PoC／単一テナント |

詳細な根拠は [02-candidate-patterns.md](./02-candidate-patterns.md) と [03-finalists.md](./03-finalists.md) を参照。

## 主要な設計判断（全案共通の原則）

1. **時系列とマスタの分離 + 越境しない** — TimescaleDB hypertable には FK を張らず、TSDB とメタを JOIN しない。`series_id`（整数）だけ越境し、整合は `series` 台帳で担保（[09 章](./09-portability.md)の防火壁）。
2. **「定義」と「実体」の分離** — 型番（DeviceType）に固有属性・ポート構成、実機（Device）に個体属性。DDL 変更なしで機種追加。
3. **集約制約はサービス層で** — 「合計負荷 ≤ ブレーカ容量」のような行間集約制約は単一行 CHECK では表現不能。移植性のため DB トリガに依存させず、単一書き込み口のサービス層で担保。
4. **物理重なりは占有U行 + UNIQUE で根絶** — ラック U の衝突はアプリに任せず `rack_unit_occupancy` の複合主キーで DB が原子的に拒否（`EXCLUDE`/拡張に依存しない LCD 化）。
5. **p95 等の分位点は任意列** — 段階ロールアップで正確な分位点はエンジン依存（Timescale/ClickHouse は sketch 対応、素の PG は非対応）。既定では持たず、必要時に `percentile_cont`（PG）/`quantile`（CH）を都度実行。

## 参考にした主な一次情報源

- NetBox DCIM models（DeviceType/Cable/CableTermination/PowerFeed/PowerOutlet）
- OpenDCIM `create.sql`（`fac_*` テーブル群、A/B 冗長給電 `PanelID/PanelID2`）
- PostgreSQL 公式：`ltree`、Exclusion Constraints、`WITH RECURSIVE`
- TimescaleDB 公式：hypertable / compression(columnstore) / continuous aggregate / retention
- TimescaleDB Toolkit：`percentile_agg` / `stats_agg` / `rollup`

各ドキュメント末尾に出典 URL を記載。
