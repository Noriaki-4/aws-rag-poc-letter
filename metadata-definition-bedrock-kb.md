# メタデータ定義 — Amazon Bedrock Knowledge Bases 前提

**対象構成**: ベクター検索KB ＋ GraphRAG（Amazon Neptune Analytics）
**設計原則**: メタデータを **hard filter / 検索補助 / トレース** の3層に分け、同じキーで役割を混ぜない。

## 凡例

| 列 | 意味 |
|---|---|
| **層** | `hard filter`=取得母集団を絞る / `検索補助`=チャンキング・埋め込み・後処理で精度に効く / `トレース`=引用・監査・障害解析用 |
| **フィルタ** | `必須`=常時 RetrievalFilter に注入 / `条件付き`=要件次第で注入 / `しない`=フィルタ化しない |
| **embed** | `includeForEmbedding` の推奨値。原則 `false`、短い静的属性だけ `true` |

> **重要**: アクセス・状態・日付の各キーは、**ベクターKBとGraphRAGで型・統制値まで完全一致**させること。型の不一致は取得母集団のズレ＝アクセス漏れの穴になる。

---

## 1. 文書単位（document-level）

### 1.1 識別・版管理

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `doc_id` | STRING | `POL-OPS-000184` | トレース | しない | false | 全系統の主キー・引用 |
| `canonical_doc_id` | STRING | `POL-OPS-SECURE-DOC` | トレース | しない | false | 改訂版を束ねる正準ID |
| `version` | STRING | `v3.2` | トレース | 条件付き | false | 版管理 |
| `record_status` | STRING | active / pending / retired / suspended | hard filter | **必須** | false | 常に `=active` を注入 |
| `approval_state` | STRING | `approved` | トレース | 条件付き | false | 承認状態の管理 |

### 1.2 アクセス・スコープ（両系で完全一致）

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `access_group` | STRING（単一値 ※1） | `ORG_A` / `COMMON_ALL` | hard filter | **必須**（共有KB） | false | アクセス境界 |
| `source_scope` | STRING | ORG_PRIVATE / SHARED_COMMON | hard filter | **必須**（共有KB） | false | 母集団の分離 |
| `owner_org` | STRING | `ORG_A` | トレース | 条件付き | false | 監査・課金按分 |
| `confidentiality` | STRING | `internal` | トレース | 条件付き | false | 機密区分 |

※1 Neptune Analytics はリスト型非対応のため単一値 STRING に統一。クエリ側の `in [自グループ, COMMON_ALL]` は単一値キーに対して有効。

### 1.3 分類・有効性

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `doc_type` | STRING | policy / manual / faq | hard filter | **必須** | **true可** | 種別で絞る／曖昧さ回避 |
| `language` | STRING | ja / en | hard filter | **必須** | false | 言語切替 |
| `jurisdiction` | STRING | JP / US | hard filter | 条件付き | false | 法域で絞る |
| `effective_from` | NUMBER (YYYYMMDD) | `20250401` | hard filter | 条件付き | false | 有効期間（`<= 今日`） |
| `effective_to` | NUMBER (YYYYMMDD) | `99991231`（無期限） | hard filter | 条件付き | false | 有効期間（`>= 今日`） |

### 1.4 解析・ルーティング

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `contains_table` | BOOLEAN | `true` | 検索補助 | しない | false | パーサ分岐 |
| `contains_figure` | BOOLEAN | `true` | 検索補助 | しない | false | 図説明生成の要否判断 |
| `ocr_required` | BOOLEAN | `false` | トレース | しない | false | OCR経路の可視化 |

### 1.5 出自・ガバナンス

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `source_uri` | STRING | `s3://.../manual.pdf` | トレース | しない ※2 | false | 出典リンク |
| `source_system` | STRING | `sharepoint` | トレース | しない | false | 登録元システム |
| `parser_name` | STRING | `bedrock_bda` | トレース | しない | false | 解析再現性 |
| `parser_version` | STRING | `2026-04` | トレース | しない | false | 解析再現性 |
| `checksum_sha256` | STRING | `a4f8...` | トレース | しない | false | 重複排除・完全性確認 |
| `ingestion_batch_id` | STRING | `2026-06-22T1200Z` | トレース | しない | false | 同期単位の追跡 |
| `embedding_model_version` | STRING | `titan-v2` | トレース | しない | false | ベクトルドリフト管理 |
| `index_version` | STRING | `v46` | トレース | しない | false | バージョニング・ロールバック |
| `chunking_strategy` | STRING | `hierarchical` | トレース | しない | false | 再現性 |

※2 `source_uri` への `startsWith` フィルタは GraphRAG（Neptune Analytics）でレイテンシを悪化させるため避け、専用属性＋`equals` を使う。

---

## 2. チャンク単位（chunk-level）

### 2.1 識別・階層

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `chunk_id` | STRING | `POL-OPS-000184#c014` | トレース | しない | false | チャンク主キー |
| `doc_id` | STRING | `POL-OPS-000184` | トレース | しない | false | 文書との結合キー |
| `chunk_index` | NUMBER | `14` | トレース | しない | false | 並び順・再構成 |
| `parent_chunk_id` | STRING | `...#p003` | 検索補助 | しない | false | 階層チャンクの親参照 |

### 2.2 構造・見出し（埋め込みに入れて精度に効かせる＋引用）

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `section_path` | STRING | `3.2 > 3.2.1 > 例外処理` | 検索補助 | しない | **true推奨** | contextual enrichment ＋引用 |
| `heading_l1` | STRING | `運用手順` | 検索補助 | しない | **true** | 見出し手がかり |
| `heading_l2` | STRING | `申請処理` | 検索補助 | しない | **true** | 見出し手がかり |
| `page_from` | NUMBER | `12` | トレース | しない | false | 引用ページ |
| `page_to` | NUMBER | `13` | トレース | しない | false | 引用ページ |

### 2.3 コンテンツ種別（hard filter にしない）

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `content_type` | STRING | paragraph / table / figure / list / faq | 検索補助 | しない ※3 | false | ソフトルーティング・後処理 |

※3 ハード絞り込みは表を説明する前後段落まで落として recall を削るため不可。表QA専用パスか rerank のソフト信号としてのみ使う。

### 2.4 表・図（再構成・引用）

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `table_id` | STRING | `tbl-07` | トレース | しない | false | 表のまとまり維持・再構成 |
| `figure_id` | STRING | `fig-04` | トレース | しない | false | 図のまとまり維持・再構成 |
| `table_caption` | STRING | `権限一覧` | 検索補助 | しない | **true検討** | 表検索の補助＋再構成 |
| `figure_caption` | STRING | `申請フロー` | 検索補助 | しない | **true検討** | 図検索の補助＋再構成 |

### 2.5 品質（post-check 用）

| キー | 型 | 統制値 / 例 | 層 | フィルタ | embed | 用途 |
|---|---|---|---|---|---|---|
| `token_count` | NUMBER | `384` | トレース | しない | false | 過大チャンク検知 |
| `ocr_confidence` | NUMBER | `0.82` | トレース | しない ※4 | false | 低信頼の抑制 |
| `extraction_method` | STRING | native_text / ocr / layout_parser | トレース | しない | false | 抽出方式 |
| `bbox_ref` | STRING/JSON | `p12:[x1,y1,x2,y2]` | トレース | しない | false | UIハイライト位置（任意） |

※4 retrieval 前の固定閾値フィルタではなく、低信頼チャンクのみで構成された回答を抑制／警告する post-check として使う。

---

## 3. GraphRAG（管理版）でのメタデータ定義の前提

- **エンティティ・関係は取り込み時に LLM が自動抽出**して Neptune に格納される。**エッジ／ノードのメタデータ（relationship_type・confidence・valid_from/to・provenance）をユーザーが定義することはできない**。上記は全て「文書／チャンクに付ける属性」で、グラフのエッジ属性は設計対象外（制御したいなら Neptune 直叩きのプリミティブ構成へ）。
- **Neptune Analytics の型は String / Number / Boolean のみ**。リスト型は使わない。
- メタデータフィルタは「入口のベクトル検索」にのみ効き、その後のグラフ探索は再フィルタされない。入口の hard filter が甘いとサブグラフ全体が汚染されるため、status / access / effective の精度がベクター以上に重要。

## 4. 実装上の制約

- `metadata.json` は **10KB 制限**。チャンク単位のこの量を全部ファイルメタデータに入れるのは不可。チャンク単位の付与は **custom transformation Lambda** で行い、文書単位メタデータとは分けて管理する。
- **chunking 戦略は後から変更不可**（data source 作り直し）。本番投入前にサンプルで確定。
- **最優先タスク**: hard filter キー（record_status / access_group / source_scope / doc_type / language / jurisdiction / effective_from / effective_to）を**全文書に欠損なく付与する取り込みバリデーション**を先に組む。フィルタ式より、付与漏れを弾く仕組みの方が事故を防ぐ。

---

## 付録: hard filter の RetrievalFilter 例（ベクター・グラフ共通構文）

共有KB・有効期間つき（today = 20260622）。`startsWith` を使わない限り GraphRAG でも同一に使える。

```json
{ "andAll": [
  { "equals": { "key": "record_status", "value": "active" } },
  { "in":     { "key": "access_group",  "value": ["ORG_A", "COMMON_ALL"] } },
  { "in":     { "key": "doc_type",      "value": ["policy", "manual"] } },
  { "equals": { "key": "language",      "value": "ja" } },
  { "lessThanOrEquals":    { "key": "effective_from", "value": 20260622 } },
  { "greaterThanOrEquals": { "key": "effective_to",   "value": 20260622 } }
]}
```

**注意**: アクセス境界・状態には必ず positive オペレータ（`equals` / `in`）を使う。`notEquals` / `notIn` はキー欠損文書を巻き込む挙動があるためセキュリティ条件に使わない。
