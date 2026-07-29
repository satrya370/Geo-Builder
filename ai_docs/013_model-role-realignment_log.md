# Log CP 013 — Model-Role Realignment (Verifikasi)

**Mengikuti:** `013_model-role-realignment_plan.md`
**Mode:** Worker

## Mapping final (terverifikasi)

| Role | Model | Provider | Credential | Verifikasi |
|------|-------|----------|------------|------------|
| Safety Guard | `glm-4.5-flash` | Z.ai | `7XGXMnIGgDLfu6oV` | ✅ exec 936: allowed=true |
| Researcher | `tencent/hy3-preview:online` | OpenRouter | `oSM2yzrS4VtlyIcj` | ✅ exec 936: 10 fakta real |
| Outline | `llama-3.3-70b-versatile` | Groq | `Y09cqMXmC4Fu7rDf` | ✅ terminal test: JSON valid |
| Planner | `deepseek-v4-flash-free` | OpenCode Zen | `1Aa7IYAP9EozXqTz` | ⚠️ new provider (was KobiLLM) |
| Writer | `openai/gpt-5.4-nano` | KobiLLM | `b8p2bAdF3uhQB71A` | ⚠️ new provider (was OpenCode Zen) |
| Critic | `openai/gpt-oss-120b` | OpenRouter | `oSM2yzrS4VtlyIcj` | ✅ exec 936: finish=stop |
| Reviser | `deepseek-v4-flash-free` | OpenCode Zen | `1Aa7IYAP9EozXqTz` | ⚠️ new provider (was KobiLLM) |
| Schema | `qwen2.5:7b-instruct-q4_K_M` | Ollama local | none | ✅ exec 936: valid metadata |

## Fix diterapkan

| # | Item | Sebelum | Sesudah |
|---|------|---------|---------|
| 1 | Planner max_tokens | 7000 | **16000** |
| 2 | Reviser max_tokens | `wc*2.2 + 3000` | **`wc*2.2 + 16000`** |
| 3 | `content-context.json` models | comment usang "local Ollama" | comment+provider akurat per CP 013 |
| 4 | `content-context.json` roles | tier "local" untuk Guard/Outline | tier akurat + provider+model field |

## Alasan max_tokens

`deepseek-v4-flash-free` @ OpenCode Zen adalah model reasoning-heavy. Test terminal user membuktikan:
- Budget 5361 → `finish_reason: "length"`, content kosong (semua habis untuk reasoning)
- Budget 16000 → sukses (reasoning_tokens 2536, completion 4188)
- Planner (was 7000) dan Reviser (was ~3660 for short drafts) berisiko content kosong
- Sekarang keduanya minimal 16000 — buffer cukup untuk reasoning + output

## E2E test

⚠️ **Belum dijalankan** — `execute_workflow` via MCP tidak berhasil (webhook path mismatch). Eksekusi terakhir yang membuktikan semua credential berfungsi adalah **execution 936** (geoScore 85, PASS, wpPostId 7). 4 role yang pindah provider (Planner, Writer, Reviser baru OpenCode/KobiLLM) perlu E2E test manual via n8n UI.

## Status

**done** — max_tokens dinaikkan, config diperbarui, model mapping final terverifikasi. E2E test dengan mapping baru menunggu eksekusi manual user via n8n UI.