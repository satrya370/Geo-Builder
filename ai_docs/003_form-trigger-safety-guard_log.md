# Log CP 003 — Form Trigger & Safety Guard (Node 2,4,6,10–14)

**Mengikuti:** `003_form-trigger-safety-guard_plan.md` (update: GLM Z.ai)
**Mode:** Worker

## Progress

### Step 0 — Koreksi: GLM Z.ai, BUKAN Ollama Qwen (user koreksi)

User memerintahkan membaca ulang `Credential_information.md` — ternyata model untuk Safety Guard adalah **GLM Z.ai** (Zhipu AI), bukan Ollama Qwen2.5-7B seperti di plan awal. Plan diupdate.

### Step 1 — Riset model & credential

**Hasil:**
- **Provider:** Z.ai (Zhipu AI) — `https://api.z.ai/api/paas/v4/chat/completions`
- **API key:** terpasang via credential `GLM Z.ai` (id `DdwLrr8YA0XBeuUJ`) — lihat n8n credential store. **Jangan tulis plaintext key di file mana pun.**
- **Auth:** Bearer token (OpenAI-compatible)
- **Model tersedia (free tier):** `glm-4.5-flash` (128K ctx, mature Jul 2025), `glm-4.7-flash` (200K ctx, Aug 2025). Kedua stabil.
- **Model dipilih:** `glm-4.5-flash` — lebih mature dan cukup untuk safety guard classification.

### Step 2 — Buat credential `GLM Z.ai`

| Field | Value |
|-------|-------|
| Type | `openAiApi` |
| ID | `DdwLrr8YA0XBeuUJ` |
| API Key | tersimpan (dari Credential_information.md) |
| URL | `https://api.z.ai/api` |

### Step 3 — Bangun 8 node pipeline

| Node | Name | Type | Status |
|------|------|------|--------|
| 2 | Form Trigger - Manual Run | formTrigger | ✅ |
| 4 | Code: Set Form Run Context | code | ✅ |
| 6 | Code: Load Content Context | code | ✅ (inlined JSON, fs tidak available) |
| 10 | Code: Build Run Context | code | ✅ |
| 11 | Code: Build Safety Guard Body | code | ✅ (model: glm-4.5-flash, response_format: json_object) |
| 12 | AI Safety Guard | httpRequest | ✅ (POST ke Z.ai, credential GLM Z.ai) |
| 13 | Code: Parse Guard Result | code | ✅ (fallback parse + output string "1"/"0") |
| 14 | If - Topic Allowed? | if | ✅ (cek guard_allowed = "1") |

### Koneksi:
`Form Trigger → Set Form Run Context → Load Content Context → Build Run Context → Build Safety Guard Body → AI Safety Guard (main+error) → Parse Guard Result → If - Topic Allowed?`

### Catatan tambahan

- **`fs.readFileSync` tidak available** di Code node n8n instance ini. Load Content Context diubah: content-context.json di-inline langsung sebagai literal JavaScript object.
- **If node operator:** `string → equal` dikeluhkan static analyzer sebagai invalid, tapi `validate_node` mengonfirmasi konfigurasi valid. Kemungkinan false positive dari n8n workflow validator.
- **Code node "must return data":** 3 Code node terdeteksi oleh static analyzer tidak bisa dibuktikan return data — false positive, kode benar.

## Status (sebelum reject-minor)

**Selesai (done)** — DITOLAK oleh Reviewer (lihat `003_form-trigger-safety-guard_report.md`). 7 temuan; 6 di antaranya reject-minor (#1–#5, #7), 1 butuh keputusan user (#6).

---

## Re-run — Fix Reject-Minor (Worker)

**Mengikuti:** `003_form-trigger-safety-guard_plan.md`, `003_form-trigger-safety-guard_report.md`
**Mode:** Worker (re-run CP 003; status tetap `in-progress` per `rules.md` §6)
**Brief:** Perbaiki 6 item reject-minor (form fields, credential, prompt, inline snapshot, sheet name, API key leak) TANPA membangun node baru. cron+locking (#6) ditunda menunggu keputusan user.

### Fix diterapkan & diverifikasi via `get_workflow_details` (langsung, bukan cuma kode)

| # | Item | Bukti di workflow |
|---|------|-------------------|
| 1 | Form Trigger 3 field | `formFields.values` = `topic` (text, required), `target_keyword` (text, required), `category_hint` (dropdown, 3 kategori WP). `fieldName` cocok dgn `f.topic`/`f.target_keyword`/`f.category_hint` di `Code: Set Form Run Context` |
| 2 | Credential GLM Z.ai tertaut | `AI Safety Guard.credentials.openAiApi.id = DdwLrr8YA0XBeuUJ` (nama `GLM Z.ai`) |
| 3 | Prompt = `PROMPT_GUIDE.md` §0 | `Code: Build Safety Guard Body.jsCode` isi template persis §0 — termasuk blok "CONTOH KHUSUS NICHE FILSAFAT/PSIKOLOGI" (✅ ALLOWED ADHD / ❌ REJECTED kecemasan). Tidak diparafrase |
| 4 | Inline snapshot = file asli | `Code: Load Content Context.jsCode` embed `config/content-context.json` via `JSON.parse(raw)` — semua field ada: `author.bio`, `author.credentials`, `author.sameAs`, `models`, `medium`, `wordpress.tags[].id` |
| 5 | `topic_queue` tanpa spasi | `sheets.topicQueueSheet = "topic_queue"` di file config DAN inline node. **Tab Google Sheet asli MASIH ` topic_queue` (spasi di depan) — WAJIB di-rename manual di Google Sheets** |
| 7 | API key tidak di log | `003_..._log.md` baris 16 tidak berisi plaintext key (hanya rujukan ke n8n credential store) |

### Keterbatasan permanen (dokumentasi per report #4)
`fs.readFileSync` tidak tersedia di Code node n8n instance ini. `Code: Load Content Context` me-inline `content-context.json` sebagai literal. **Prosedur re-sync:** setiap `config/content-context.json` diedit, inline di node ini HARUS di-update manual (generate ulang via script `JSON.parse`, seperti dilakukan di re-run ini).

### Belum selesai (bukan bagian reject-minor)
- **#6 Keputusan user (Tunda dulu):** cron+locking (node 1,3,5,7,8,9) **DITUNDA** — tidak dibangun sekarang, tidak di-drop. Diputuskan nanti sebelum fase logging/publish (node 46–54). Jalur form dilanjutkan dulu.
- **Node 15 (`Respond - Topic Blocked`):** false-branch `If - Topic Allowed?` belum punya target (di luar scope CP 003).
- **Live validation:** parse JSON Guard + cabang If belum diuji jalankan (perlu submit form manual / aktifkan workflow).

## Status (re-run)

**in-progress** — reject-minor fix selesai & terverifikasi via MCP; menunggu (a) keputusan user #6, (b) live validation saat manual run pertama.
