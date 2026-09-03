@AGENTS.md

# Lecturer Studio — 講師教材後台

講師在後台跟 AI 對話，先把一場課的「教學藍圖」討論清楚；之後所有教材（課程大綱、簡報、學員手冊）都從同一份藍圖生成，藍圖升版時下游產物會知道自己過期了。

## 核心原則（動任何功能前先讀）

1. **藍圖是唯一真相來源。** 對話只是產生／修改藍圖的手段；所有生成一律讀 `Blueprint`，不讀對話紀錄。
2. **藍圖有版本，產物綁版本。** `artifacts.blueprint_id` 指向產生它的那一版；藍圖升版不會自動重生產物，UI 顯示「已過期」讓講師決定。
3. **五元素完整性**（理論／動手／帶走／金句／故事）是藍圖 schema 的一部分，缺哪格 UI 要看得到。
4. **產檔的重活丟給 Anthropic 沙箱。** pptx/docx 由 Agent Skills + code execution 產出，本專案不自己跑 python-pptx。

完整設計與決策記錄見 `docs/architecture.md`。

## 技術棧

- Next.js 16（App Router、`src/` 目錄）+ TypeScript strict + Tailwind v4
- Anthropic TypeScript SDK `@anthropic-ai/sdk`，模型 `claude-opus-5`
- Supabase（Postgres + Auth + Storage），schema 在 `supabase/migrations/`
- zod v4（藍圖 schema、結構化輸出）、vitest

## 常用指令

| 目的 | 指令 |
|---|---|
| 安裝 | `pnpm install` |
| 開發 | `pnpm dev` |
| Lint | `pnpm lint` |
| 型別檢查 | `pnpm typecheck`（會先跑 `next typegen` 產生 Next.js 的全域型別） |
| 測試 | `pnpm test`（單檔：`pnpm test src/lib/blueprint/schema.test.ts`） |
| Build | `pnpm build` |

提交前必跑 `pnpm lint && pnpm typecheck && pnpm test`。套件管理一律用 pnpm。

## 目錄

```
src/app/              Next.js 路由（頁面、API route）
src/lib/blueprint/    藍圖 schema 與完整性檢查（純函式，可單測）
src/lib/claude/       所有 Anthropic API 呼叫集中在這裡
src/lib/supabase/     Supabase client
supabase/migrations/  SQL migration，只新增不改舊檔
docs/                 架構與決策記錄
.claude/              Claude Code 設定與 hooks
```

## Claude API 規範

動任何 `src/lib/claude/` 程式前，先載入 `claude-api` skill 確認 API 形狀，不要憑記憶寫。

- 模型固定用 `src/lib/claude/client.ts` 的 `MODEL`（`claude-opus-5`），不要為了省錢自行降級。
- thinking 預設 adaptive（可省略參數），用 `output_config.effort` 控制深度。**不要**用 `budget_tokens`、`temperature`，**不要**做 assistant prefill——這些在現行模型都會回 400。
- 對話一律 streaming（`client.messages.stream`）；非串流 `max_tokens` 約 16000，串流約 64000。
- 從對話抽藍圖用 `client.messages.parse` + `zodOutputFormat(BlueprintSchema)`；schema 只有一份，在 `src/lib/blueprint/schema.ts`，不要另外手刻 JSON schema。
- 生成 pptx/docx：`client.beta.messages.stream` + `container.skills` + `{ type: "code_execution_20260521", name: "code_execution" }` + beta `code-execution-2025-08-25`；產出檔案的 `file_id` 在 `bash_code_execution_tool_result` 裡，用 Files API 下載。
- Prompt 順序固定：品牌規範 → 藍圖 → 該產物的指示 → 講師這次的要求。穩定的前段加 `cache_control`；時間戳、UUID 這類會變的內容一律放最後。
- 錯誤處理用 SDK 的 typed error class（`Anthropic.RateLimitError` 等），不要字串比對。

## 資料規範

- `blueprints.content` 寫入前必須通過 `BlueprintSchema.parse()`。
- 新增欄位 = 新增一個 migration 檔（`supabase/migrations/NNNN_*.sql`），並同步更新 zod schema 與 `docs/architecture.md`。
- 所有表都開 RLS；資料只有 `owner_id` 本人可讀寫。

## 環境變數

名稱列在 `.env.example`；本機放 `.env.local`。永遠不要 commit、cat 或 echo 任何 `.env*`（`.env.example` 除外）。
