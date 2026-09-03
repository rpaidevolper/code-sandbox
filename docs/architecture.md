# Lecturer Studio 架構

## 1. 這個產品在解什麼問題

講師要準備一場課，通常得分別產出課程大綱、簡報、學員手冊，而這三樣東西本質上共用同一份 context：受眾是誰、目標是什麼、時間多長、要講哪些段落。分開做就會重複問三次、彼此不同步。

所以產品的核心不是「AI 幫你做簡報」，而是：

> 先跟 AI 把一份**教學藍圖**討論清楚，之後所有教材都從這份藍圖生成，藍圖改了下游就知道自己過期。

這條 pipeline 在 RPAI 既有的 skill 裡已經驗證過（`rpai-talk-blueprint` → `rpai-course-outline` / `rpai-course-content` / `rpai-pptx-workflow`），本專案是把它產品化成講師後台。

## 2. 資料模型

```
Course（一場課）
 ├─ Conversation（對話串；purpose = blueprint | outline | slides | handbook）
 │   └─ Message（Anthropic content blocks 原樣存）
 ├─ Blueprint v1, v2, v3 …（結構化 JSON，唯一真相來源）
 └─ Artifact（產物；每一筆綁定「基於哪一版藍圖」）
      ├─ outline  v2  ← blueprint v3
      ├─ slides   v1  ← blueprint v2   ← 過期：藍圖已到 v3
      └─ handbook v1  ← blueprint v3
```

關鍵設計：

- **對話不是真相來源，藍圖才是。** 對話結束（或講師按「更新藍圖」）時，用結構化輸出把對話抽成 `Blueprint` JSON 存成新版本。之後生成任何產物都只讀藍圖，不讀對話。
- **藍圖版本不可變。** 修改 = 新增一版，舊版保留，產物才能可靠地指向它出生時的那一版。
- **產物不自動重生。** 藍圖升版後，UI 標示過期產物；講師決定要不要重生（重生要花錢也要時間）。
- **五元素是 schema 的一部分。** `elements.{theory, handsOn, takeaway, quote, story}` 哪一格是空的，`missingElements()` 直接算得出來，UI 用它提示講師還缺什麼。

zod schema：`src/lib/blueprint/schema.ts`。DB schema：`supabase/migrations/0001_init.sql`。

## 3. AI 執行層

| 階段 | 用什麼 | 為什麼 |
|---|---|---|
| 藍圖對話 | Messages API streaming（`client.messages.stream`） | 即時回覆、自己掌控對話與存檔 |
| 抽藍圖 | `client.messages.parse` + `zodOutputFormat(BlueprintSchema)` | 直接拿到通過 schema 驗證的 JSON，不用自己 parse |
| 生成大綱／簡報／手冊 | `client.beta.messages.stream` + `container.skills` + code execution | pptx/docx 在 Anthropic 沙箱用 skill 產出，本專案不維護產檔引擎；既有的 rpai-* skill 上傳成自訂 skill 後可直接沿用 |
| 取回檔案 | Files API 下載 → 存 Supabase Storage | 沙箱產出的檔案先落地到自己的儲存，再給講師下載 |

程式碼都在 `src/lib/claude/`：`client.ts`（模型與 client）、`blueprint-chat.ts`（對話與抽藍圖）、`generate-artifact.ts`（產檔）。

### 為什麼不用 Managed Agents

Managed Agents 程式碼更少（一門課一個 session，容器、迴圈、記憶都由 Anthropic 管），但藍圖狀態會住在 Anthropic 端而不是自己的資料庫，而且目前仍是 beta。藍圖是這個產品的核心資產，掌控權優先。之後若想降低伺服器端維護成本可再評估遷移，資料模型不用改。

### Prompt 組裝順序（影響快取命中）

```
system[0]  品牌規範 + 角色設定            ← 最穩定，加 cache_control
system[1]  藍圖 JSON（某一版）             ← 同版本內穩定
messages   該產物的指示 + 講師這次的要求    ← 每次都變
```

會變的內容（時間戳、request id）永遠放在最後，不然整段快取失效。

## 4. 生成流程

```
講師點「生成簡報」
  → 讀最新藍圖 v3 + 品牌規範
  → artifacts 新增一筆 (kind=slides, blueprint_id=v3, status=generating)
  → generateArtifact()：beta messages + skills(pptx, 自訂 rpai skill) + code execution
  → 從回應的 bash_code_execution_tool_result 取 file_id
  → Files API 下載 → 上傳 Supabase Storage
  → artifacts.status = ready, storage_path = …
```

生成一份簡報可能要跑好幾分鐘，API route 不能同步等；要決定用 Next.js 背景任務、Supabase Edge Function，還是外部 queue（見第 6 節）。

## 5. 已做的決策

| 決策 | 選擇 | 日期 |
|---|---|---|
| 技術棧 | Next.js 16 + TypeScript，單一 repo | 2026-09-03 |
| AI 執行層 | Messages API + Agent Skills + code execution | 2026-09-03 |
| 資料庫 | Supabase Postgres，RLS 依 owner_id | 2026-09-03 |
| 模型 | `claude-opus-5`，adaptive thinking，用 effort 調深度 | 2026-09-03 |
| 藍圖版本 | 不可變、只新增；產物綁定版本 | 2026-09-03 |

## 6. 還沒決定的事

- **長時間生成怎麼跑**：Vercel function 上限不夠跑幾分鐘的產檔。候選：Supabase Edge Function + pg queue、Inngest / Trigger.dev、自架 worker。
- **自訂 skill 的上傳流程**：`rpai-brand-guidelines`、`rpai-pptx-workflow` 要用 Skills API 上傳成自訂 skill，skill id 放環境變數。要決定版本更新由誰、何時做。
- **結構化輸出的 schema 相容性**：`BlueprintSchema` 用了 `min(1)`、`min(2)` 這類限制，第一次真的呼叫 `extractBlueprint()` 時要確認 API 接受；不接受就把限制移到 `checkBlueprint()` 做。
- **講師登入**：Supabase Auth；Next.js 端要用 `@supabase/ssr` 處理 cookie session（尚未安裝）。
- **藍圖 diff UI**：v2 → v3 改了什麼要讓講師看得到，才能判斷哪些產物真的需要重生。
- **多講師／團隊**：目前 schema 只有單一 owner，團隊共享要加 membership 表。
