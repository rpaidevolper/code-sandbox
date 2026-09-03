# Lecturer Studio

講師教材後台：跟 AI 對話定出一份「教學藍圖」，再從同一份藍圖生成課程大綱、簡報與學員手冊。

## 開始

```bash
pnpm install
cp .env.example .env.local   # 填入 Anthropic 與 Supabase 金鑰
pnpm dev
```

## 文件

- `CLAUDE.md` — 給 Claude Code 的專案規範（也是給人看的最短摘要）
- `docs/architecture.md` — 資料模型、生成流程與決策記錄
- `supabase/migrations/` — 資料庫 schema
