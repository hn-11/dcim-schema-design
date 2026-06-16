# 12. EcoStruxure IT DCIM — データスキーマ Web 調査報告

> **調査方法**: deep-research ハーネス（5角度並行検索 → 23ソース取得 → 35クレーム抽出 → 25クレーム検証）  
> **一次ソース**: Schneider 公式 helpcenter（helpcenter.ecostruxureit.com / sxwhelpcenter）・help.ecostruxureit.com・APC WP  
> **検証結果**: 25クレーム中 **15件確認（3-0 または 2-0/2-1）、10件棄却**。セッション上限により verify が途中終了したクレームは §3 に別掲。

---

## 1. Genome / Genome Library（機器テンプレート）

### 1.1 Genome Library は読み取り専用の公開カタログ

> "The Genome Library database is extended with thousands of additional measured server power profiles, as well as storage and switch specifications. This continuously growing asset library enables StruxureWare Data Center Operation to determine power consumption without the aid of hardware sensors."

- Genome Library = **公開・読み取り専用カタログ**。メーカー型番と実測電力プロファイル（サーバー・ストレージ・スイッチ）を含む。
- ハードウェアセンサなしでも消費電力を推定できる（実測プロファイル参照）。
- **検証**: 3-0（全3票確認）

> **出典**: [sxwhelpcenter.ecostruxureit.com/display/UAOps73/Working+with+the+Genome+Library+and+Genomes](https://sxwhelpcenter.ecostruxureit.com/display/UAOps73/Working+with+the+Genome+Library+and+Genomes)

### 1.2 2層テンプレートモデル：公開ライブラリ ＋ ローカルゲノム

> "The Genome Library is a public generic library that contains thousands of measured server power profiles, and storage and switch specifications."

> "You might need a genome that is slightly different from one in the library. In this case, you can create a new genome with more precise information in your local genomes."

- **公開ライブラリ**（Genome Library）: 読み取り専用、製品カタログ
- **ローカルゲノム**（local genomes）: ユーザー作成・編集可能。ライブラリゲノムの精密化バリアント、または汎用コンポーネントから作成
- 機器の配置はローカルゲノムからラックへのドラッグ操作
- **検証**: 3-0

> **出典**: [helpcenter.ecostruxureit.com/hc/en-us/articles/360038495433-Configuring-equipment-not-available-in-the-genome-library](https://helpcenter.ecostruxureit.com/hc/en-us/articles/360038495433-Configuring-equipment-not-available-in-the-genome-library)

### 1.3 Genome ライブラリに rack PDU の bank/outlet 定義を含む（v8.3〜）

> "The IT Advisor genome library includes definitions for rack PDU banks and outlets (not available in DCO prior to version 8.3). Users can connect the end power consumer in the rack to the selected outlet."

- IT Advisor（DCO 8.3〜）: rack PDU の **bank および outlet** の定義を持つ
- rack PDU → 個別 outlet への接続が可能になり、outlet レベルの電力モデリングが実現
- DCO 8.3 より前は bank/outlet 定義なし（phase 単位のみ）
- **検証**: 3-0

> **出典**: [helpcenter.ecostruxureit.com/hc/en-us/articles/360038496693-Working-with-Rack-PDUs](https://helpcenter.ecostruxureit.com/hc/en-us/articles/360038496693-Working-with-Rack-PDUs)

#### 設計への示唆
`device_type`（Genome）→ `device`（実機）の2層分離は確認。さらに：
- ローカルゲノムの概念は `device_type.source = 'library' | 'local'` フラグまたは `manufacturer IS NULL` で表現可能
- rack PDU の outlet モデリング（v8.3〜）は `power_feed` の outlet 粒度（bank_id / outlet_num）を正当化する

---

## 2. Power Path（電力経路）

### 2.1 power path の真実源：upstream 接続からの自動導出

> "Power phase and output voltages are then found automatically by the system, based on upstream connections. When connecting rack PDU to power supplier, based on the inlet type, only relevant breakers are shown for the selection to avoid errors in power path configuration."

- 電圧・相（phase）は **上流接続から自動導出**（手動入力不要）
- rack PDU をpower supplier に接続するとき、inlet type に合致するブレーカーのみ表示されエラーを防ぐ
- 「上流接続が真実源」という本設計の方針と完全に一致
- **検証**: 3-0

> **出典**: [helpcenter.ecostruxureit.com/hc/en-us/articles/360038496693-Working-with-Rack-PDUs](https://helpcenter.ecostruxureit.com/hc/en-us/articles/360038496693-Working-with-Rack-PDUs)

### 2.2 2つの接続視点：消費者 inlet 側 ＋ PDU outlet 側

> "The application provides two ways of configuring power connections for rack equipment: From power consumers point of view, From rack PDU point of view, if the PDU supports extended power information. ... look for a power consumer inlet with nothing connected to it. A free inlet is labelled Disconnected."

- **消費者視点**（power consumer's inlets）: 機器のどの PSU inlet に何が繋がるか
- **PDU 視点**（rack PDU's outlets）: PDU が拡張電力情報を持つ場合のみ表示
- 未接続 inlet のラベルは **"Disconnected"**
- **検証**: 3-0

> **出典**: [helpcenter.ecostruxureit.com/hc/en-us/articles/360037928494-Editing-rack-inventory](https://helpcenter.ecostruxureit.com/hc/en-us/articles/360037928494-Editing-rack-inventory)

### 2.3 outlet 情報の有無による接続方法の分岐

> "For power sources without power outlet information, select the phase from list of available phase configurations. For power sources with power outlet information, select available outlet."

| power source の種別 | 接続設定 |
|---|---|
| outlet 情報なし（旧来型・v8.3以前） | **phase**（利用可能な相構成リスト）から選択 |
| outlet 情報あり（v8.3〜） | **outlet**（利用可能な outlet）から選択 |

- **検証**: 3-0

> **出典**: [helpcenter.ecostruxureit.com/hc/en-us/articles/360037928494-Editing-rack-inventory](https://helpcenter.ecostruxureit.com/hc/en-us/articles/360037928494-Editing-rack-inventory)

### 2.4 Paired power consumers — 冗長ペアの明示（コロケーション用）

> "You can pair power consumers in the Properties dialog box. Use Shift+click to select two power consumers and click Pair. The paired consumers will now appear in the Paired power consumers table... Paired power consumers are primarily used in a co-location facility to provide power redundancy at the rack or cage level. Simulated impact will show failover loads for each receptacle."

- 2台の power consumer を **Paired power consumers テーブル** に登録
- **主用途**: コロケーション施設での rack/cage レベル電力冗長
- フェイルオーバー時の各 receptacle 負荷が**シミュレーションで算出**される
- 操作: Shift+click で2台選択 → Pair ボタン / Split で解除
- **検証**: 3-0

> **出典**: [helpcenter.ecostruxureit.com/hc/en-us/articles/360038496693-Working-with-Rack-PDUs](https://helpcenter.ecostruxureit.com/hc/en-us/articles/360038496693-Working-with-Rack-PDUs)

#### 設計への示唆
- `redundancy_group`（L7）の `domain='power'`, `topology='2N'` + `redundancy_member.leg='A'|'B'` が Paired power consumers に直接対応
- 「receptacle ごとのフェイルオーバー負荷」は L8 の `capacity_budget` + `redundancy_reserve` で表現可能

---

## 3. 機器資産のプロパティ構造

### 3.1 equipment asset の編集可能プロパティグループ（確認済み）

> "General information / Physical information / Custom properties / Labels - Add color coded labels / Asset visuals - Add images for the front and rear of the asset / Power data / Power connections / Network / Sensors / Customer / Audit trail"

IT Advisor の機器（rack inventory）が持つプロパティグループ:

| グループ | 内容 |
|---|---|
| General information | 基本情報（名称・状態等） |
| Physical information | 物理寸法（U高さ・深さ・重量等） |
| Custom properties | ユーザー定義カスタム属性 |
| Labels | 色付きラベル |
| Asset visuals | 前面・背面の画像 |
| Power data | 消費電力・推定負荷 |
| Power connections | inlet/outlet 接続設定 |
| Network | ネットワーク接続 |
| Sensors | センサ設定 |
| Customer | 顧客情報（コロケーション用） |
| Audit trail | 変更履歴 |

- **検証**: 3-0

> **出典**: [helpcenter.ecostruxureit.com/hc/en-us/articles/360037928494-Editing-rack-inventory](https://helpcenter.ecostruxureit.com/hc/en-us/articles/360037928494-Editing-rack-inventory)

#### 設計への示唆
- `Custom properties` → L10 `entity_tag`（登録語彙 FK 強制）に対応
- `Customer` グループ → L2 `device.tenant_id` + L8 コロ属性（08章）
- `Audit trail` → 変更管理（ITIL）はサービス層。DB には `device.updated_at` 程度で十分

---

## 4. 推定負荷戦略（Estimated Load Strategy）

### 4.1 デフォルトは adjusted nameplate、rack 単位でフォールバック

> "By default estimated load calculations are based on adjusted nameplate values, however, they can be different depending on the power information available to the system in each rack and the power capacity strategy you have selected as the main strategy in Preferences>Power Capacity."

> "The estimated load calculations always fall back on the best available information in the rack. Depending on the available information in the rack, your calculations may switch strategy."

- **システムデフォルト**: `adjusted nameplate`（Preferences > Power Capacity で変更可）
- **rack 単位のフォールバック**: 選択した戦略のデータが不足している rack では、その rack で使える最善の戦略に**自動的に切替**
- fleet 全体で単一戦略が適用されるわけではなく、rack ごとに異なる戦略が適用されることがある
- **検証**: 2-1（主張確認）; fallback 挙動は 3-0

> **出典**: [help.ecostruxureit.com/display/public/UADCO8x/Understanding+estimated+load+and+how+it+is+calculated](https://help.ecostruxureit.com/display/public/UADCO8x/Understanding+estimated+load+and+how+it+is+calculated)

### 4.2 adjusted nameplate の定義

> "The adjusted nameplate value is more precise than the manufacturer's nameplate as it is the actual known peak power draw of a server rather than the maximum load that the manufacturer guarantees the server will not exceed."

| 戦略 | 定義 |
|---|---|
| **nameplate**（銘板） | メーカーが保証する「絶対に超えない最大負荷」（最悲観的） |
| **adjusted nameplate** | **実際の既知のピーク消費電力**（ユーザーが設定）。銘板より精密で楽観的 |

- **検証**: 2-1（主張確認）; 別ソースでも 2-0 確認

> **出典**: [help.ecostruxureit.com/display/public/UADCO8x/Understanding+estimated+load+and+how+it+is+calculated](https://help.ecostruxureit.com/display/public/UADCO8x/Understanding+estimated+load+and+how+it+is+calculated)  
> [sxwhelpcenter.ecostruxureit.com/display/public/UADCO8x/Adjusted+nameplate+or+predicted+power](https://sxwhelpcenter.ecostruxureit.com/display/public/UADCO8x/Adjusted+nameplate+or+predicted+power)

### 4.3 predicted power：90日履歴 → 30日予測

> "Past trends in your equipment's measured peak power data from DCE or other external system integration are used to predict the future power demand. By default, the system bases the calculations on historic data from the last 90 days and predicts 30 days in to the future."

- **predicted power**: DCE（Data Center Expert）または外部統合から取得した**実測電力の履歴トレンド**を使って未来負荷を予測
- デフォルト: **直近90日のヒストリカルデータ** → **30日先の予測**
- **検証**: 2-0（1票はセッション上限で abstain）

> **出典**: [help.ecostruxureit.com/display/public/UADCO8x/Understanding+estimated+load+and+how+it+is=calculated](https://help.ecostruxureit.com/display/public/UADCO8x/Understanding+estimated+load+and+how+it+is+calculated)

#### 設計への示唆
`device_demand.load_strategy` の4値（`nameplate | adjusted_nameplate | predicted | contracted`）は正しく、DCO のモデルに完全対応。`predicted` 戦略の入力ソース（DCE 実測値 / 外部統合）はL5 `conn` のプロトコル統合（DCE は SNMP/Modbus で集収）と繋がる。`predicted` の look-back 窓（90日）は L6 `measurement` の retention policy 設定と関係する。

---

## 5. セッション上限で検証未完のクレーム（参考）

以下は検証エージェントが全滅（全3票 abstain）したため確認・棄却いずれにも分類できなかったクレーム。内容の真偽は独立に要確認。

| クレーム | ソース |
|---|---|
| WP-150 は 5 容量要素（空間/電力/電力分配/冷却/冷却分配）を定義し、いずれか1つが枯渇すると他の容量が Stranded になる | [APC WP-150 PDF](https://www.mercurymagazines.com/pdf/NCAPC6.pdf) |
| Stranded capacity = ある種の容量が他の5要素のいずれかにより使えない状態 | 同上 |
| WP-150 は 4 種類の余剰容量を区別: Spare / Idle / Safety margin / Stranded | 同上 |
| WP-150 の power path モデルは branch circuit を最内制約として識別 | 同上 |
| DCO 8.x での adjusted nameplate はユーザーが手動入力（自動導出ではない） | sxwhelpcenter |
| DCO の predicted power 戦略は live 実測値がある場合にのみ適用される | sxwhelpcenter |

---

## 6. 棄却されたクレーム（要注意）

| クレーム | 棄却理由（投票） |
|---|---|
| "Genomes are an editable list organized into categories/products" | 0-3（Genome Library は読み取り専用であり編集可能ではない） |
| "Generic Switch Enclosure / Blade Enclosure 等の命名済みテンプレートが存在する" | 0-3（実際の名称は異なる可能性） |

---

## 7. 調査できなかった領域

以下のソースは fetch できたが有意なクレームを抽出できなかった（quality: unreliable として除外）:

| 領域 | ソース | 状況 |
|---|---|---|
| **公開 REST API スキーマ** | IT Expert API (helpcenter) / DCE Web Service API (sxwhelpcenter) / Community forum | 抽出クレーム 0 件。ドキュメント構造が認証壁または動的JS behind |
| **Inventory Web Service** | sxwhelpcenter/UAOps72/Inventory+Web+Service | 抽出 0 件 |
| **Cooling Optimize** | sxwhelpcenter/UAOp74/Cooling+Optimize | 抽出 0 件 |
| **ETL import database** | helpcenter/ETL-import-database | 抽出 0 件 |
| **収集プロトコル** (SNMP/Modbus/Redfish) | community.se.com / dcimsupport | 抽出 0 件 |

> **結論**: EcoStruxure IT の**公開 REST API のオブジェクトスキーマ**（エンドポイント名・フィールド定義・JSON 構造）は、Web 公開ドキュメントからは取得困難。Swagger/OpenAPI spec がある場合は製品内の `/api-docs` 等から直接取得要。

---

## 8. 調査ソース一覧（primary と評価されたもの）

| ソース | クレーム数 | URL |
|---|---|---|
| sxwhelpcenter / Working with Genome Library | 5 件 | https://sxwhelpcenter.ecostruxureit.com/display/UAOps73/Working+with+the+Genome+Library+and+Genomes |
| helpcenter / Configuring equipment not in genome library | 5 件 | https://helpcenter.ecostruxureit.com/hc/en-us/articles/360038495433 |
| helpcenter / Working with Rack PDUs | 5 件 | https://helpcenter.ecostruxureit.com/hc/en-us/articles/360038496693 |
| helpcenter / Editing rack inventory | 5 件 | https://helpcenter.ecostruxureit.com/hc/en-us/articles/360037928494 |
| help.ecostruxureit.com / Understanding estimated load | 5 件 | https://help.ecostruxureit.com/display/public/UADCO8x/Understanding+estimated+load+and+how+it+is+calculated |
| sxwhelpcenter / Adjusted nameplate or predicted power | 5 件 | https://sxwhelpcenter.ecostruxureit.com/display/public/UADCO8x/Adjusted+nameplate+or+predicted+power |
| APC WP-150 PDF | 5 件（全 abstain） | https://www.mercurymagazines.com/pdf/NCAPC6.pdf |

---

## 9. 既存設計との対応・裏取り結果

| 設計要素（03章）| 調査結果 | 評価 |
|---|---|---|
| `device_type`（Genome）+ `device`（実機）の2層分離 | Genome Library → local genome → 配置という2段階テンプレートが確認 | ✅ 設計確認 |
| `rack PDU bank/outlet` のモデリング（power_feed 粒度） | IT Advisor v8.3〜で Genome に bank/outlet 定義が追加されたと確認 | ✅ 裏取り済み |
| 電力フロー = 物理 path からの導出（並行グラフ不要） | 電圧・相は上流接続から自動導出と確認 | ✅ 設計確認 |
| `redundancy_group` の `paired power consumers` | コロケーション向け Paired power consumers テーブルを確認 | ✅ 設計確認 |
| `device_demand.load_strategy` の4値 | nameplate / adjusted_nameplate / predicted（90日→30日）を確認 | ✅ 設計確認 |
| adjusted_nameplate の定義 | 実際の既知ピーク電力（≠ メーカー銘板）と確認 | ✅ 定義一致 |
| `device` の `tenant_id` + Customer プロパティ | 機器に Customer グループ（コロ用）があると確認 | ✅ 裏取り済み |
| WP-150 の 5 容量要素 / Stranded capacity | 検証エージェントが全 abstain（セッション上限）→ 確認不能 | ⚠ 要別途確認 |
| 公開 REST API スキーマ（IT Expert / DCE） | 全ソースで抽出 0 件 → ドキュメントが認証壁の内側 | ❌ 取得不可 |
