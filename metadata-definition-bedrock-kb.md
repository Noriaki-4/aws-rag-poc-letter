# メタデータ定義 マスター表 — Amazon Bedrock Knowledge Bases

全キーを1行、全軸を1列にした統合表。キーは必ず1行に収まり、粒度・役割・2軸・優先度・制約がその行の各列値として決まる（＝同じキーが複数分類に重複しない）。

## 凡例

| 列             | 値と意味                                                                                                                                                  |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **粒度**（①）  | `文書`=文書全体に1つ（標準 `.metadata.json` で素直に載る） / `chunk`=分割断片ごと（標準 `.metadata.json` では**不可**。後述の物理化ルート A か B が前提） |
| **役割**（②）  | `hard`=母集団を絞る / `補助`=埋め込み・後処理で精度に効く / `トレース`=引用・監査・運用                                                                   |
| **フィルタ**   | `必須`=常時注入 / `条件付`=要件次第 / `しない`                                                                                                            |
| **embed**（③） | `includeForEmbedding`。`yes`/`可`/`no`                                                                                                                    |
| **注入**（④）  | フィルタ値を query 時に差すか。`yes(出所)` / `no`                                                                                                         |
| **優先度**     | `必須`=破ると漏洩 / `機能`=外すと精度低下 / `品質`=運用改善                                                                                               |
| **制約**       | `非null`=完全性 / `V/G`=Vector/Graph契約一致 / `pos`=positiveオペレータ限定 / `—`=該当なし                                                                |

---

## マスター表

| キー                              | 粒度  | 役割     | フィルタ  | embed   | 注入          | 優先度   | 制約             | 型               | 例                       | 用途                         |
| --------------------------------- | ----- | -------- | --------- | ------- | ------------- | -------- | ---------------- | ---------------- | ------------------------ | ---------------------------- |
| **▼ 文書単位（document-level）**  |       |          |           |         |               |          |                  |                  |                          |                              |
| `record_status`                   | 文書  | hard     | 必須      | no      | no            | **必須** | 非null・V/G・pos | STRING           | `active`                 | 常に `=active`               |
| `access_group`                    | 文書  | hard     | 必須      | no      | **yes**(ID)   | **必須** | 非null・V/G・pos | STRING（単一値） | `ORG_A`                  | アクセス境界                 |
| `source_scope`                    | 文書  | hard     | 必須      | no      | no            | **必須** | 非null・V/G・pos | STRING           | `SHARED_COMMON`          | 母集団分離                   |
| `doc_type`                        | 文書  | hard     | 必須      | **可**  | no            | 機能     | 非null・V/G      | STRING           | `manual`                 | 種別・曖昧さ回避             |
| `language`                        | 文書  | hard     | 必須      | no      | no            | 機能     | 非null・V/G      | STRING           | `ja`                     | 言語切替                     |
| `jurisdiction`                    | 文書  | hard     | 条件付    | no      | no            | 機能     | V/G              | STRING           | `JP`                     | 法域                         |
| `effective_from`                  | 文書  | hard     | 条件付    | no      | **yes**(日付) | **必須** | 非null・V/G・pos | NUMBER(YYYYMMDD) | `20250401`               | 有効開始 `≤今日`             |
| `effective_to`                    | 文書  | hard     | 条件付    | no      | **yes**(日付) | **必須** | 非null・V/G・pos | NUMBER(YYYYMMDD) | `99991231`               | 有効終了 `≥今日`             |
| `approval_state`                  | 文書  | トレース | 条件付    | no      | no            | 品質     | —                | STRING           | `approved`               | 承認管理                     |
| `confidentiality`                 | 文書  | トレース | 条件付    | no      | no            | 機能     | V/G（該当時）    | STRING           | `internal`               | 機密区分                     |
| `owner_org`                       | 文書  | トレース | 条件付    | no      | no            | 品質     | —                | STRING           | `ORG_A`                  | 監査・課金按分               |
| `doc_id`                          | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `POL-OPS-000184`         | 主キー・引用                 |
| `canonical_doc_id`                | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `POL-OPS-SECURE`         | 正準ID（版束ね）             |
| `version`                         | 文書  | トレース | 条件付    | no      | no            | 品質     | —                | STRING           | `v3.2`                   | 版管理                       |
| `contains_table`                  | 文書  | 補助     | しない    | no      | no            | 品質     | —                | BOOLEAN          | `true`                   | パーサ分岐                   |
| `contains_figure`                 | 文書  | 補助     | しない    | no      | no            | 品質     | —                | BOOLEAN          | `false`                  | 図説明生成の要否             |
| `ocr_required`                    | 文書  | トレース | しない    | no      | no            | 品質     | —                | BOOLEAN          | `false`                  | OCR経路可視化                |
| `source_uri`                      | 文書  | トレース | しない ※1 | no      | no            | 品質     | —                | STRING           | `s3://kb/ops/manual.pdf` | 出典リンク                   |
| `source_system`                   | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `sharepoint`             | 登録元                       |
| `parser_name`                     | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `bedrock_bda`            | 解析再現性                   |
| `parser_version`                  | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `2026-04`                | 解析再現性                   |
| `checksum_sha256`                 | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `a4f89d3c…`              | 重複排除・完全性             |
| `ingestion_batch_id`              | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `2026-06-22T12Z`         | 同期追跡                     |
| `embedding_model_version`         | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `titan-v2`               | ドリフト管理                 |
| `index_version`                   | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `v46`                    | バージョニング・ロールバック |
| `chunking_strategy`               | 文書  | トレース | しない    | no      | no            | 品質     | —                | STRING           | `hierarchical`           | 再現性                       |
| **▼ チャンク単位（chunk-level）** |       |          |           |         |               |          |                  |                  |                          |                              |
| `chunk_id`                        | chunk | トレース | しない    | no      | no            | 品質     | —                | STRING           | `POL-OPS-000184#c014`    | チャンク主キー               |
| `doc_id`(FK)                      | chunk | トレース | しない    | no      | no            | 品質     | —                | STRING           | `POL-OPS-000184`         | 文書との結合キー             |
| `chunk_index`                     | chunk | トレース | しない    | no      | no            | 品質     | —                | NUMBER           | `14`                     | 並び順・再構成               |
| `parent_chunk_id`                 | chunk | 補助     | しない    | no      | no            | 品質     | —                | STRING           | `…#p003`                 | 階層チャンク親参照           |
| `section_path`                    | chunk | 補助     | しない    | **yes** | no            | 機能     | —                | STRING           | `3.2 > 3.2.1 例外処理`   | 構造手がかり＋引用           |
| `heading_l1`                      | chunk | 補助     | しない    | **yes** | no            | 機能     | —                | STRING           | `運用手順`               | 見出し                       |
| `heading_l2`                      | chunk | 補助     | しない    | **yes** | no            | 機能     | —                | STRING           | `申請処理`               | 見出し                       |
| `content_type`                    | chunk | 補助     | しない ※2 | no      | no            | 機能     | —                | STRING           | `table`                  | ソフトルーティング・後処理   |
| `table_caption`                   | chunk | 補助     | しない    | yes検討 | no            | 機能     | —                | STRING           | `権限一覧`               | 表検索補助＋再構成           |
| `figure_caption`                  | chunk | 補助     | しない    | yes検討 | no            | 機能     | —                | STRING           | `申請フロー`             | 図検索補助＋再構成           |
| `table_id`                        | chunk | トレース | しない    | no      | no            | 品質     | —                | STRING           | `tbl-07`                 | 表のまとまり維持             |
| `figure_id`                       | chunk | トレース | しない    | no      | no            | 品質     | —                | STRING           | `fig-04`                 | 図のまとまり維持             |
| `page_from`                       | chunk | トレース | しない    | no      | no            | 品質     | —                | NUMBER           | `12`                     | 引用ページ                   |
| `page_to`                         | chunk | トレース | しない    | no      | no            | 品質     | —                | NUMBER           | `13`                     | 引用ページ                   |
| `token_count`                     | chunk | トレース | しない    | no      | no            | 品質     | —                | NUMBER           | `384`                    | 過大チャンク検知             |
| `ocr_confidence`                  | chunk | トレース | しない ※3 | no      | no            | 品質     | —                | NUMBER           | `0.82`                   | 低信頼抑制（post-check）     |
| `extraction_method`               | chunk | トレース | しない    | no      | no            | 品質     | —                | STRING           | `layout_parser`          | 抽出方式                     |
| `bbox_ref`                        | chunk | トレース | しない    | no      | no            | 品質     | —                | STRING/JSON      | `p12:[x1,y1,x2,y2]`      | UIハイライト位置（任意）     |

---

## 物理化ルート（chunk 粒度キーの載せ方）

文書構造を活かした検索（粒度=`chunk` のキーをチャンクごとに別値で持つ）には、標準のファイル単位 `.metadata.json` は使えない。チャンク分割を前処理に移し、各チャンクに個別メタを与える。3 ルートあり、**A を推奨**。

| ルート        | 方式                                                                                                                                                             | チャンク粒度メタ                                      | 採否                                                       |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| **A（推奨）** | カスタムデータソース ＋ `IngestKnowledgeBaseDocuments`（直接取り込み）＋ NoChunking。外部で構造aware分割し 1チャンク=1ドキュメントを**インラインメタ付き**で投入 | 型付きインライン属性で一体投入。`.metadata.json` 不要 | 構造活用の第一選択                                         |
| **B（代替）** | S3データソース ＋ カスタム変換 Lambda（`customTransformationConfiguration`）。文書単位キーは `.metadata.json`、chunk 単位キーは Lambda で付与                    | Lambda が付与                                         | S3 を source of truth＋管理同期を残したいとき              |
| **C（限定）** | 構造化 CSV/JSONL ＋ contentField/metadataField ＋ NoChunking（1 行=1 チャンク）                                                                                  | 行のメタ列                                            | 表形式データ限定。散文は前処理でファイル爆発するため不向き |

**A の具体手順**: ① 生データを外部パース（BDA 等）し、見出し・表・ページ境界で構造aware分割。各チャンクに doc_id・section_path・table_id・page・content_type＋文書単位キー（record_status・access_group・doc_type・effective_*）を付与 → ② NoChunking のカスタムデータソース作成 → ③ `IngestKnowledgeBaseDocuments` で 1ドキュメント=1チャンク・`metadata.type=IN_LINE_ATTRIBUTE` で投入 → ④ バルクはキュー＋レート制限（同時実行 ≤ 10、1 リクエスト documents ≤ 10 件）。生データ・派生データは S3 に保持し再構築可能に。

```json
{ "documents": [{
  "content": { "dataSourceType": "CUSTOM",
    "custom": { "customDocumentIdentifier": { "id": "POL-OPS-000184#c014" },
      "sourceType": "IN_LINE",
      "inlineContent": { "type": "TEXT", "textContent": { "data": "（3.2.1 例外処理 の本文）" } } } },
  "metadata": { "type": "IN_LINE_ATTRIBUTE", "inlineAttributes": [
    { "key": "section_path",  "value": { "type": "STRING", "stringValue": "3.2 > 3.2.1 例外処理" } },
    { "key": "page_from",     "value": { "type": "NUMBER", "numberValue": 12 } },
    { "key": "record_status", "value": { "type": "STRING", "stringValue": "active" } },
    { "key": "access_group",  "value": { "type": "STRING", "stringValue": "ORG_A" } },
    { "key": "doc_type",      "value": { "type": "STRING", "stringValue": "manual" } }
  ] } }
]}
```

> `STRING_LIST` はインラインでは使えるが、GraphRAG/Neptune との契約一致のため `access_group` は **STRING 単一値**で統一。A を採ると `.metadata.json` は基本不要、B のときだけ文書単位キーの担当として残る。

---

## 脚注（表に入らない横断ルール）

- **物理投影**: 粒度=`文書` の hard filter キーも、Bedrock は必ず**チャンクのメタデータを評価**するため、全チャンクに複製格納される（論理 doc → 物理 chunk）。ルート A ではこの複製を投入時に自分でインライン付与する（各チャンクに文書単位キーも併記）。粒度=`chunk` のキーは上記「物理化ルート」A/B が前提。
- **※1** `source_uri` への `startsWith` フィルタは GraphRAG（Neptune Analytics）でレイテンシ悪化。絞るなら専用属性＋`equals`。
- **※2** `content_type` はハード絞り込み不可（表＋前後段落を巻き込み落とす）。表QA専用パスか rerank のソフト信号のみ。
- **※3** `ocr_confidence` は固定閾値フィルタでなく、低信頼チャンクのみの回答を抑制する post-check として使う。
- **制約タグの意味**: `非null`=全文書/全チャンクに付与必須（完全性）／`V/G`=型・統制値を Vector と Graph で完全一致（契約一致）／`pos`=`equals`/`in`/`listContains` のみ。`notEquals` はキー欠損文書を巻き込むため ACL・status に使わない。
- **GraphRAG 前提**: エッジ／関係のメタデータ（relationship_type・confidence・temporal validity）は managed では定義不可。`access_group` は Neptune 制約で単一値 STRING に統一。

## 実装の最優先（優先度=必須の確実化）

1. `access_group` の **query 時注入**（呼び出し元IDから動的に差す）
2. hard filter キーの **完全性**（全文書・全チャンクに非 null で付与する取り込みバリデーション）
3. hard filter キーの **契約一致**（型・統制値を Vector/Graph で揃える）

この3つはどれが欠けても即アクセス事故になる。embed 設計（③）は精度・コストの問題で一段下、補助・トレースは品質改善レイヤー。
