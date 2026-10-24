# CLAUDE.md

本檔定義本專案的 skill 分工規則。目的是避免「同名／同職責」的 skill 互相搶觸發，
導致每次都要重猜該用哪一套。

## 已安裝的 skill 來源

- **帳號層 RPAI 系列**（`rpai-*`、`grilling`、`care-record-quality-check` 等）
  —— 內容生產用：講座、課程、簡報、社群貼文、提案。
- **mattpocock-skills plugin**（marketplace: `mattpocock`）
  —— 軟體工程用：需求釐清、規格、TDD、除錯、架構、程式碼審查。

安裝方式（供日後重建環境參考）：

```bash
claude plugin marketplace add mattpocock/skills
claude plugin install mattpocock-skills@mattpocock
```

## 分工規則

### 1. 「拷問／釐清需求」——看產出物決定

兩邊都有名為 `grilling` 的 skill，且**都會被模型自動觸發**，這是最容易撞的一組。

| 情境 | 用哪個 |
|---|---|
| 要規劃講座／課程／簡報／提案等**內容交付物** | 帳號層的 `rpai-talk-blueprint`（其內部會用帳號層 `grilling`） |
| 要釐清**軟體功能需求／技術設計** | `/grill-with-docs`（會順帶產出 ADR 與 glossary） |
| 只是想單純被電一頓、壓力測試某個想法 | `/grill-me` |

判準：**產出物是投影片或文章 → RPAI 鏈；產出物是程式碼或 spec → mattpocock 鏈。**

### 2. 「程式碼審查」——看要審什麼

同樣有兩個 `code-review`，且都會自動觸發。

- **內建 `/code-review`**：抓 correctness bug、可簡化處、效率問題。日常改動預設用這個。
- **mattpocock `code-review`**：以「本 repo 的 coding standards」與「原始 issue/spec」兩軸並行審查。
  **有寫下 spec 或 ticket 的工作**完成後用這個，才能驗收「有沒有照規格做」。

規則：**沒有 spec 就用內建版；有 spec 就用 mattpocock 版。**

### 3. 軟體工程流程——預設走 mattpocock 鏈

需求 → 規格 → 工單 → 實作 → 審查：

```
/grill-with-docs  →  /to-spec  →  /to-tickets  →  /implement (內部走 tdd)  →  /code-review
```

- 工作量大到一個 session 裝不下時，先用 `/wayfinder` 畫決策地圖。
- 修 bug 走 `diagnosing-bugs`，不要直接猜。
- 動模組介面／切分層時，用 `codebase-design` 的詞彙（deep module、seam），不要自創「service」「component」等說法。
- merge / rebase 衝突一律解完，不要 `--abort`。先讀衝突兩邊原始的 commit、PR 與 issue，弄清楚各自要達成什麼，再合成同時保留兩邊意圖的結果，不要整段選 ours／theirs 了事；解完跑過專案的檢查（lint、型別、測試）再繼續 merge / rebase。這不需要專用 skill，直接處理。

`/to-spec`、`/to-tickets`、`triage` 需要先設定 issue tracker。第一次使用前跑 `/setup-matt-pocock-skills`。

### 4. 內容生產流程——維持原有 RPAI 鏈，不要換

```
rpai-talk-blueprint  →  rpai-course-outline / rpai-course-content  →  rpai-pptx-workflow / rpai-proposal
```

`rpai-talk-blueprint` 是這條鏈的唯一上游真相來源。**不要**用 `/grill-with-docs` 取代它——
那會產出 ADR 與領域模型，不是教學藍圖，下游 skill 讀不到需要的欄位。

品牌規範一律先讀 `rpai-brand-guidelines`。

## Git

開發分支見 session 指定的 `claude/*` 分支，不要直接推 default branch。
