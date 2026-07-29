# Log CP 004 — Researcher, Outline, Writer (Node 16–25)

**Mengikuti:** `004_researcher-outline-writer_plan.md`
**Mode:** Worker

## Progress

### ✅ Node 16 — `Code: Build AI Research Body`

| | |
|---|---|
| **Model** | `tencent/hy3-preview` (Tencent HunYuan, 262K ctx, MoE) |
| **Provider** | OpenRouter (`https://openrouter.ai/api/v1/chat/completions`) |
| **Prompt** | `PROMPT_GUIDE.md` §1 + `topicDetail` + `keyEntitiesSeed` |
| **Params** | temp 0.3, max_tokens 2048, `response_format: json_object` |
| **Connection** | `If - Topic Allowed?` (true) → node 16 |

### ✅ Node 17 — `HTTP: AI Researcher`

| | |
|---|---|
| **Credential** | explicit Bearer token (same as ig Content Builder) |
| **Timeout** | 120s, retry 2x @ 5s, `continueErrorOutput` |
| **Connection** | node 16 → node 17 |

### ⚠️ Node 24 — `HTTP: AI Writer`

| | |
|---|---|
| **Credential** | `OYN6H3ReLc0kzTYb` — "Koboillm API" (`openAiApi`) |
| **Endpoint** | `https://api.koboillm.com/v1` |
| **Model** | `openai/gpt-5-nano` (akan diisi di `_writerBody` oleh node 23) |
| **Timeout** | 180s, retry 2x @ 3s, `continueErrorOutput` |
| **Connection** | ⚠️ **sementara** ke Form Trigger — perlu di-rewire ke node 23 saat dibangun |

## Checklist

- [x] Node 16 — `Code: Build AI Research Body`
- [x] Node 17 — `HTTP: AI Researcher`
- [ ] Node 18 — `Code: Parse Research`
- [ ] Node 19 — `Code: Validate Research Facts`
- [ ] Node 20 — `Code: Build AI Outline Body`
- [ ] Node 21 — `HTTP: AI Outline Planner` (Ollama local)
- [ ] Node 22 — `Code: Parse Outline`
- [ ] Node 23 — `Code: Build AI Writer Body`
- [x] Node 24 — `HTTP: AI Writer` (credential siap, perlu rewire)
- [ ] Node 25 — `Code: Parse Draft`

## Status

**in-progress** — 3/10 node built (16, 17, 24). Node 24 credential sudah benar (Koboillm, `openai/gpt-5-nano`), tapi koneksi masih temp ke Form Trigger. Sisa: parse research (18), validate (19), outline (20-22), writer body (23), parse draft (25).

---

## Re-run — Fix Reject (Worker)

**Mengikuti:** `004_researcher-outline-writer_plan.md` (Update pasca-review), `004_researcher-outline-writer_report.md`
**Mode:** Worker (re-run CP 004, per `rules.md` §6 reject fix)

### Fix diterapkan & diverifikasi via `get_workflow_details`

| # | Item | Bukti |
|---|------|-------|
| 1 | Pola credential `genericCredentialType`+`httpHeaderAuth` | Node 17,24: `credentials.httpHeaderAuth.id=oSM2yzrS4VtlyIcj` ("OpenRouter Authorization - GPT OSS 120B"), tidak ada `nodeCredentialType` invalid |
| 2 | Hapus key literal | Node 17: `headerParameters` hanya `Content-Type`, `Authorization` hilang |
| 3 | Model web search | Node 16: `tencent/hy3-preview:online` + `tools: [{type:'web_search', max_results:5}]` + `tool_choice:'auto'` |
| 4 | Hapus koneksi salah | `Form Trigger → HTTP: AI Writer` sudah terputus |
| 5 | Fix context | Node 16,20,23: `$('Code: Build Run Context').first().json`, bukan `$input.first().json` |
| 6 | Semua node | 18 total: 16 (Research Body), 17 (Researcher), 18 (Parse), 19 (Validate), 20 (Outline Body), 21 (Outline Planner), 22 (Parse Outline), 23 (Writer Body), 24 (Writer), 25 (Parse Draft) |
| 7 | Koneksi | 17/18 connections: semua HTTP error branch tersambung ke parse node |

### Pipeline final
```
Form → Set Context → Load Context → Build Context → Safety Guard Body → 
AI Safety Guard (main+error) → Parse Guard → If-Allowed? (true) →
Build Research Body → AI Researcher (main+error) → Parse Research → 
Validate Facts → Build Outline Body → AI Outline Planner (main+error) → 
Parse Outline → Build Writer Body → AI Writer (main+error) → Parse Draft
```

### Model summary
| Role | Model | Provider | Auth |
|------|-------|----------|------|
| Safety Guard | `glm-4.5-flash` | Z.ai | `openAiApi` (GLM Z.ai) |
| Researcher | `tencent/hy3-preview:online` | OpenRouter | `httpHeaderAuth` (OpenRouter Auth - GPT OSS 120B) |
| Outline | `qwen2.5:7b-instruct-q4_K_M` | Ollama local | none |
| Writer | `openai/gpt-5-nano` | OpenRouter | `httpHeaderAuth` (OpenRouter Auth - GPT OSS 120B) |

## Status (re-run)

**done** — 10/10 node built & connected, 7 perbaikan diverifikasi via MCP.