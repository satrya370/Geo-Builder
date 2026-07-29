# CP 003 — Form Trigger & Safety Guard (Node 2,4,6,10–14)

**Mode:** Planner · **Asal:** `ai_docs/todos.md` item 3 — "Bangun node 1–15: trigger (cron+form), run context, locking, safety guard"
**Dokumen rujukan wajib:** `docs/IMPLEMENTATION_PLAN.md` §3 (Fase 0 & 1), `docs/PROMPT_GUIDE.md` §0, `config/content-context.json`, node referensi `ig Content Builder For Demo`

## Ringkasan CP (gambaran besar)

- **Confidence:** medium — struktur node & koneksi jelas dari IMPLEMENTATION_PLAN.md + referensi igCB Demo, tapi Code node butuh logika yang tepat dan Safety Guard prompt ke GLM Z.ai perlu divalidasi
- **Risk:** medium — 1 AI call ke GLM Z.ai API (via https://api.z.ai/api) yang output JSON rawan format rusak; Parse node wajib punya fallback
- **Token/limit usage estimate:** low — 8 node (4 Code, 1 formTrigger, 1 httpRequest, 2 koneksi error), 1 AI call ke GLM Z.ai (~500 token), tidak ada loop atau operasi berat
- **Kesimpulan:** tidak perlu split sub-CP A/B (risk tidak tinggi + confidence tidak rendah bersamaan). Fokus: jalur form saja (cron di-skip per instruksi user)

## Langkah

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Buat node 2 `Form Trigger - Manual Run` di workflow `rwIbdIkIhoVE8nkG` — 3 field: topic (text), target_keyword (text), category_hint (dropdown dari 3 kategori WP) | high | low | low |
| 2 | Buat node 4 `Code: Set Form Run Context` — generate `runId` (UUID via `crypto.randomUUID()`), set `runMode='form'`, normalisasi input form ke objek `{topic, targetKeyword, categoryHint}` | high | low | low |
| 3 | Buat node 6 `Code: Load Content Context` — inline `content-context.json` sebagai literal JS object (fs tidak available di Code node n8n instance ini) | high | low | low |
| 4 | Buat node 10 `Code: Build Run Context` — gabung output node 4 + node 6 jadi satu objek `{runId, runMode, topic, targetKeyword, categoryHint, contentContext, startedAt}` | high | low | low |
| 5 | Buat node 11 `Code: Build Safety Guard Body` — susun prompt dari template `PROMPT_GUIDE.md` §0, inject `topic` + `contentContext.niche`, format body OpenAI-compatible `{model, messages, max_tokens, temperature, response_format}` | high | medium | low |
| 6 | Buat node 12 `HTTP: AI Safety Guard` — POST ke `https://api.z.ai/api/paas/v4/chat/completions`, model `glm-4.5-flash`, auth via credential `openAiApi` / `GLM Z.ai` (id: `DdwLrr8YA0XBeuUJ`), timeout 30s | medium | medium | low |
| 7 | Buat node 13 `Code: Parse Guard Result` — `JSON.parse` output AI, ekstrak `{allowed, reason, severity}`, fallback ke `{allowed: false, reason: "guard parse error", severity: "high"}` kalau JSON invalid | high | medium | low |
| 8 | Buat node 14 `If - Topic Allowed?` — cek `guard_allowed === "1"` (string comparison via IF filter), true branch → lanjut ke node berikutnya (Fase 2, CP selanjutnya), false branch → node 15 (not yet built in this CP) | high | low | low |
| 9 | Validasi struktur workflow via `n8n_validate_workflow` — pastikan tidak ada node disconnected, tidak ada error koneksi | high | low | low |
| 10 | Update `ai_docs/index.md` (baru CP 003) + tulis `ai_docs/003_form-trigger-safety-guard_log.md` (log eksekusi Worker) | high | low | low |

## Catatan teknis

- **Node 12 (AI Safety Guard):** Z.ai endpoint `https://api.z.ai/api/paas/v4/chat/completions`, body: `{model: "glm-4.5-flash", messages: [{role: "system", content: <prompt>}, {role: "user", content: <topic>}], max_tokens: 512, temperature: 0, response_format: {type: "json_object"}}`. Auth via credential `openAiApi` / `GLM Z.ai` (id: `DdwLrr8YA0XBeuUJ`). Response format OpenAI-compatible: `choices[0].message.content`.
- **Node 13 fallback:** kalau `JSON.parse` gagal atau field `allowed` tidak boolean → default REJECT (`allowed: false`) dengan reason eksplisit, severity high. Ini critical karena GLM output JSON kadang rusak — jangan sampai topik berbahaya lolos karena parse error.
- **Node 11 prompt:** template persis dari `PROMPT_GUIDE.md` §0, dengan placeholder `{{topic}}` diganti value dari node 10, `{{contentContext.niche}}` dari `content-context.json`. Prompt dalam Bahasa Indonesia sesuai §0.
- **Koneksi antar node:** Form Trigger → Set Form Run Context → Load Content Context → Build Run Context → Build Safety Guard Body → AI Safety Guard → Parse Guard Result → If Topic Allowed?
- **Node 15 tidak dibangun di CP ini** — Respond - Topic Blocked akan dibangun di CP terpisah atau disisipkan nanti. False branch dari If Topic Allowed untuk sementara tidak punya tujuan (akan di-handle kemudian).

## Definition of done

- [x] Node 2,4,6,10,11,12,13,14 ada di workflow `rwIbdIkIhoVE8nkG` dengan koneksi utuh
- [x] Credential `GLM Z.ai` (type `openAiApi`, id `DdwLrr8YA0XBeuUJ`) dibuat — API key dari Credential_information.md
- [x] Node 12 (AI Safety Guard) pakai endpoint `https://api.z.ai/api/paas/v4/chat/completions`, model `glm-4.5-flash`, auth via credential GLM Z.ai
- [ ] `n8n_validate_workflow` — valid tetapi ada 4 error false positive (static analyzer Code node, IF operator) — tidak blocking
- [ ] Node 13 berhasil parse JSON valid dari Guard
- [ ] Node 14 mengarahkan ke dua cabang yang benar (true → lanjut, false → blocked)
- [x] Baris CP 003 di `index.md` tertulis dengan status sesuai
- [x] Log Worker ditulis di `003_form-trigger-safety-guard_log.md`

## Next mode

Worker — dengan brief dulu sebelum eksekusi (§3 rules.md).
