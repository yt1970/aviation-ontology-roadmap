# 航空業務Ontology構想メモ
_2026-04-24 ゆーちゃ × Claude 対話まとめ_

---

## 1. きっかけ：エアクロ記事から始まった話

Zenn記事「GraphRAGで全DBを自然言語で横断検索できるMCPサーバーを作った話」（エアークローゼット）を読んで、パランティアのOntologyとの類似性に気づいたところから議論スタート。

---

## 2. エアクロ記事の本質

> **既存DBを後付けでOntology化するラッパーをMCPサーバーとして実装した話**

### やってること（4ステップ）

```
既存DB（暗黙知が埋まってる）
    ↓
① ORM解析 + AI説明文生成    ← 意味を半自動で起こす
② 人間レビューで補正         ← 暗黙知を明示知に変換
③ グラフ構造で保存           ← Ontology化
④ MCPサーバーで公開          ← AIから操作可能に
```

### 技術スタック
- BigQuery（グラフ + ベクトル検索 + JSON を1ストアで）
- Vertex AI Embedding（768次元）+ VECTOR_SEARCH
- Cloud Run（MCPサーバー）
- Firestore（レビューデータの永続化）

---

## 3. パランティアAIPとの概念的な違い

| | エアクロ記事 | Palantir AIP |
|---|---|---|
| 本質 | DB辞書 + クエリMCP | 業務Ontology + AIエージェント基盤 |
| アプローチ | 既存DB → 後付けOntology化 | Ontology先設計 → DBは射影 |
| 抽象レイヤー | 物理層に近い | 業務概念層 |
| AIの役割 | 検索・クエリ補助 | 業務判断・アクション実行 |

### Palantir AIPの3層構造

**① Ontology Layer（業務概念の定義）**
- Object Type: Aircraft, Component, WorkOrder...
- Link Type: 「機体は部品を持つ」「作業指示書は整備士に割り当て」
- Action Type: 「部品交換を起票」「整備完了を記録」

**② AIP Logic（AIの推論層）**
- Ontologyを通じてデータを読み書き
- LLMがOntologyのコンテキストを受け取る

**③ AIP Agent（実行層）**
- Human-in-the-loop（提案→承認→実行）

---

## 4. 核心的な理解：DBと業務ロジックの分離

```
// 物理層アクセス（エアクロ的）
SELECT * FROM maintenance_records WHERE aircraft_id = 123

// Ontology経由（Palantir的）
aircraft.getMaintenanceRecords()
```

**DBが変わっても、Ontologyのインターフェースは変わらない = 疎結合**

→ アメーバのようにいろんなデータソースに繋がれる
→ DBと業務ロジックの分離がPalantirの本質的な価値

---

## 5. 物理層と概念層の関係（重要な整理）

概念層（Ontology）は最終的に必ず物理化される。違いは「どの抽象度で設計・操作するか」だけ。

MCPが2段構成になる：

```
AI → MCP → DB              （エアクロ的）
AI → MCP → Ontology → DB   （Palantir的）
```

---

## 6. ゆーちゃの状況整理

| ユースケース | アプローチ |
|---|---|
| 日常タスク・知識管理 | Ontology先設計 → DB作る（先付け） |
| コンサル野望・航空業務 | SAP S/4HANA（既存）→ Ontology化（後付け） |

### S/4HANAが難しい理由
- テーブル数：数万〜数十万
- 命名規則：MARA, MARC, MARD...（人間が読めない）
- 暗黙知：SAPコンサルが何年もかけて習得する世界

### でも有利な点
- ATA章番号：部品・システムの標準分類体系（既存のOntologyに近い）
- S1000D：整備マニュアルの構造化標準
- 航空業務知識 × SAP知識が既にある

### SIerとの関係
SIerはSAPを知ってるがOntologyの概念がない。ただしSIerが作ったBIの出力をOntologyへの入口として使えば競合しない。

---

## 7. コンサルとしてやりたいこと（データ分析手法）

### やりたい分析の全体像

```
故障データ（指図）
    ↓
どの部品が・どの頻度で・どの機体で壊れるか
    ↓
必要な部品・数量・タイミングが見える
    ↓
購買依頼 → サプライヤーに発注
    ↓
受注できない相手先の特定           ← ここを掘りたい
    ↓
なぜ受注できないか（能力限界・在庫・リードタイム・独占供給）
    ↓
調達リスク → 運航インパクト試算
```

### 具体的な分析軸
- 故障が多い指図の傾向 → アラート
- 購買未成立 → 運航へのインパクト試算
- 受注できない相手先の特定
- サプライヤーの対応限界の要因分析
- 購買依頼数のAIレコメンド

---

## 8. Ontology オブジェクト設計（骨格）

```
Component（部品）
  ├── FailureHistory（故障履歴）→ 故障率・予測 → アラート
  ├── PurchaseOrder（購買依頼）→ 発注状況
  └── Supplier（サプライヤー）
        ├── 受注率
        ├── リードタイム実績
        ├── 供給能力限界
        └── 代替サプライヤーの有無

Aircraft（機体）
  └── MaintenanceSchedule（整備計画）
        └── 必要部品 → Component に繋がる

WorkOrder / 指図（整備作業）
  └── Component → Supplier → PurchaseOrder
```

---

## 9. 技術的な実装枠組み

| Palantirの概念 | OSS/自前実装 |
|---|---|
| Ontology定義 | JSON-LD or Neo4j / FalkorDB |
| Object/Link Storage | Neo4j / PostgreSQL + pgvector |
| セマンティック検索 | Embedding + ベクトル検索 |
| AIP Logic | LangGraph / LlamaIndex Workflows |
| MCPサーバー | Ontologyへのアクセス口 |
| Action実行 | MCP Tools として定義 |
| Human-in-the-loop | Slack通知 + 承認フロー |

---

## 10. ゆーちゃが学ぶべきこと

### 追加で学ぶべきこと

**Ontology・グラフ設計**
- グラフDBの基礎（Neo4j入門）
- RDF/OWL or JSON-LDの概念
- ATA章番号・S1000Dの構造理解

**データエンジニアリング**
- ETLの基礎（DWH設計、データパイプライン）
- pgvector or BigQuery VECTOR_SEARCHの使い方
- S/4HANAのテーブル構造

**AIエージェント**
- LangGraph（ワークフロー型エージェント）
- RAG → GraphRAGへの発展
- Human-in-the-loopパターン

---

## 11. ネクストアクションリスト

### 短期（〜1ヶ月）
- [ ] 日常タスク・知識管理用のOntologyを小さく設計してみる
- [ ] pgvector or BigQueryでベクトル検索を試す
- [ ] Neo4jサンドボックスでグラフDB入門

### 中期（〜3ヶ月）
- [ ] 指図一覧BIのデータ項目を棚卸しして、Ontologyオブジェクト定義を具体化
- [ ] 故障アラートのプロトタイプ（CSV → グラフDB → MCPサーバー）
- [ ] LangGraph入門

### 長期（退職前後）
- [ ] S/4HANAのOntology化マッピング定義を設計
- [ ] サプライヤー分析・購買レコメンドのデモを作る
- [ ] コンサル提案資料としてまとめる

---

## 12. 差別化ポイント（コンサルとして）

```
SAPドメイン知識
  × 航空業務知識（ATA/S1000D）
  × Ontology設計力
  × AI実装力（MCP/LangGraph）
```

SIerには絶対できない「データの意味を業務に繋げる」提案力が差別化の核心。
