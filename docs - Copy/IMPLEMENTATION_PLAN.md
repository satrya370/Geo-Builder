# Website GEO Builder — Implementation Plan

n8n workflow untuk generate artikel yang dioptimasi untuk **GEO (Generative Engine
Optimization)** — konten yang mudah dikutip/direferensikan oleh AI answer engines
(ChatGPT, Perplexity, Google AI Overview) — lalu auto-publish ke WordPress (WordPress.com).

Diturunkan dari pola arsitektur `ig Content Builder For Demo` (n8n ID `V8VbQm1KWpe7jj7Y`,
82 node) yang sudah terbukti jalan. Lihat [ARCHITECTURE_NOTES.md](ARCHITECTURE_NOTES.md)
untuk pemetaan pola yang direuse.

> **Workflow n8n target (tempat plan ini dibangun):**
> `http://localhost:5678/workflow/rwIbdIkIhoVE8nkG` — semua operasi n8n MCP (buat/update
> node, validasi) untuk implementasi plan ini mengarah ke workflow ID `rwIbdIkIhoVE8nkG`.

> **Untuk AI (Planner/Worker/Reviewer):** dokumen ini adalah rujukan desain, bukan daftar
> kerja. Proses eksekusinya — checkpoint, log, index — mengikuti [`../rules.md`](../rules.md).
> Backlog kerja ada di [`../ai_docs/todos.md`](../ai_docs/todos.md), diturunkan dari
> §6 di bawah. Baca `rules.md` §8 sebelum membuat/mengerjakan/mereview CP apa pun.

---

## 1. Keputusan desain (sudah difinalisasi)

| Aspek | Keputusan |
|---|---|
| Definisi GEO | Generative Engine Optimization (bukan geo-targeting lokasi) |
| Trigger | **Dual**: Schedule (cron) + Form Trigger manual — pola `ig Content Builder` |
| Approval gate | **Di-skip.** Tidak ada Telegram Wait / human-in-the-loop |
| Publish mode | **Auto-publish live** ke WordPress **saja** (Medium di-drop, lihat §1.1 — CP 001) |
| Telegram | Hanya notifikasi hasil (skor GEO + link artikel) + alert error |
| Featured image | **Skip** — text-only dulu |
| Content context | File config: niche, kategori/tag WP existing, tone, author/E-E-A-T |
| Revise loop | Max 2x, dibatasi node `Check Retry Limit` |
| Model split | Outline + Schema pakai model gratis/lokal; sisanya model berbayar |

### 1.1 Keputusan final publish target (CP 001, selesai)

**❌ Medium — DI-DROP permanen.** Diverifikasi lewat CP 001
(`ai_docs/001_verifikasi-medium-token_plan.md`): integration token tidak bisa didapat.
**Fase 6 lama (node 43–45, Medium publish) tidak dibangun.** Semua referensi "WordPress +
Medium" di dokumen ini dianggap usang kecuali diperbarui eksplisit — WordPress **satu-satunya**
target publish.

**✅ WordPress — dipakai, TAPI mekanismenya beda dari asumsi awal.**
Situs ternyata **WordPress.com hosted** (`satryapudja.wordpress.com`), bukan self-hosted
wordpress.org — jadi **Application Password/Basic Auth tidak tersedia** (fitur itu cuma ada
di wordpress.org). Yang dipakai:

- **Auth:** OAuth2 generic credential di n8n (bukan node native "Wordpress"), dengan
  `Authentication: Body` (WordPress.com menolak client auth lewat Basic header — ini
  penyebab error `invalid_client`/`client_id missing` yang sempat muncul saat setup).
  - Authorization URL: `https://public-api.wordpress.com/oauth2/authorize`
  - Access Token URL: `https://public-api.wordpress.com/oauth2/token`
  - Scope: `global`
- **Node publish:** `HTTP Request` node (generic), bukan node native "Wordpress" —
  credential OAuth2 generic + endpoint `POST
  https://public-api.wordpress.com/rest/v1.1/sites/satryapudja.wordpress.com/posts/new`.
  Body payload beda dari `/wp-json/wp/v2/posts` (self-hosted) — perlu disesuaikan saat
  node 40–41 dibangun (lihat Fase 6 di §3, catatan sudah diperbarui).
- **Konsekuensi ke rubric/config:** Kategori 6 (Metadata & Schema Signals) di
  `GEO_SCORING_RUBRIC.md` tetap berlaku (JSON-LD tetap di-generate di jsCode, disisipkan ke
  body `content` sebagai `<script type="application/ld+json">` karena WordPress.com REST
  API tidak punya field terpisah untuk custom schema).

---

## 2. Arsitektur high-level

```
┌─ Schedule Trigger (cron) ─┐
│                            ├─→ Run Context ─→ Claim Topic Slot (Sheets lock)
└─ Form Trigger (manual) ───┘                            │
                                                          ▼
                                        [GATE 0] AI Safety Guard ─→ blocked? ─→ Respond/Alert
                                                          │ allowed
                                                          ▼
                                  ┌───────────────────────────────────────┐
                                  │  ROLE 1  Researcher   (web search)    │
                                  │  ROLE 2  Outline      (LOKAL/gratis)  │
                                  │  ROLE 3  Writer       (model utama)   │
                                  └───────────────────────────────────────┘
                                                          │ draft
                                                          ▼
                                  ┌───────────────────────────────────────┐
                                  │  GEO Rule Checker  (jsCode, 0 token)  │
                                  │  ROLE 4  Critic    (AI judgment)      │
                                  │  Compute GEO Score (gabung keduanya)  │
                                  └───────────────────────────────────────┘
                                                          │
                        ┌─────────────────────────────────┼──────────────────────┐
                   score < 60                       60 ≤ score < 80          score ≥ 80
                        │                                 │                      │
                  reject → alert          ROLE 5 Reviser (max 2x) ──loop──┘      │
                                                                                  ▼
                                                    ROLE 6 Schema/Metadata (LOKAL/gratis)
                                                                                  │
                                                                                  ▼
                                              WordPress.com publish (live, OAuth2) ─→ dapat URL
                                                                                  │
                                                                                  ▼
                                        Log ke Sheets ─→ Telegram Send Result (skor + link)
```

Error di jalur mana pun → `Build Failure Context` → `Telegram Operational Alert` +
`Mark Topic Failed` di Sheets (krusial karena cron jalan tanpa pengawasan).

---

## 3. Node mapping lengkap

Konvensi nama mengikuti `ig Content Builder` (`Code: X`, `Build X Body`, `Parse X`) supaya
konsisten kalau kamu bolak-balik antar dua workflow. Estimasi **~54 node** (Fase 0–8).

### Fase 0 — Trigger, Run Context & Locking

| # | Node | Type | Fungsi |
|---|---|---|---|
| 1 | `Schedule Trigger - Daily Cron` | `scheduleTrigger` | Jadwal harian |
| 2 | `Form Trigger - Manual Run` | `formTrigger` | **Update (CP 003 fix):** 8 field — lihat rincian di bawah |
| 3 | `Code: Set Cron Run Context` | `code` | Tandai `runMode='cron'`, generate `runId` |
| 4 | `Code: Set Form Run Context` | `code` | Tandai `runMode='form'`, normalisasi input form (termasuk parsing 4 field baru — lihat di bawah) |
| 5 | `Google Sheets: Load Topic Queue` | `googleSheets` | Ambil daftar topik `status=pending` (cron only) |
| 6 | `Code: Load Content Context` | `code` | Baca `config/content-context.json` → niche, kategori WP, tone, author |
| 7 | `Google Sheets: Load Article History` | `googleSheets` | Judul/slug yang sudah pernah dibuat → cegah duplikat topik |
| 8 | `If - Run Has Topic?` | `if` | Kalau queue kosong & bukan form run → keluar tanpa error |
| 9 | `Google Sheets: Claim Topic Slot` | `googleSheets` | **Lock**: set `status=processing` + `runId` sebelum kerja dimulai |
| 10 | `Code: Build Run Context` | `code` | Gabung semua jadi satu objek konteks yang dibawa sepanjang workflow |

> **Kenapa locking wajib:** cron dan form-trigger bisa jalan bersamaan. Tanpa claim-slot,
> topik yang sama bisa diproses dobel → dobel biaya token dan dobel artikel published.
> Pola ini diambil dari `Google Sheets: Claim Content Plan Slot` di ig-content-builder.

#### Field Form Trigger (node 2) — final, diputuskan setelah review CP 003

Referensi pola dari `ig Content Builder For Demo` (`topic_override`+`topic_detail`+`keywords`
optional+dropdown dengan opsi "Auto"), disesuaikan untuk GEO:

| Field | Tipe | Wajib? | Catatan |
|---|---|---|---|
| `topic` | text | **Wajib** | Judul kerja singkat, 1 kalimat |
| `topic_detail` | textarea | **Wajib** | Deskripsi detail aspek/sudut pandang yang mau dibahas — bahan riset Researcher, bukan sekadar 1 kalimat |
| `target_keyword` | text | Opsional | Helper text: "1-3 kata. Auto-extract dari topic_detail kalau kosong." Diselesaikan di Outline Planner (Role 2) kalau kosong, bukan node baru |
| `category_hint` | dropdown | Opsional, default `Auto` | Opsi: `Auto` / 3 nama kategori dari `content-context.json`. Kalau `Auto` → diteruskan kosong ke Schema Generator (Role 6), yang **sudah** punya instruksi pilih dari daftar tersedia — reuse, bukan node klasifikasi baru |
| `key_entities` | textarea | Opsional | Filsuf/teori/istilah yang WAJIB disebut eksplisit, satu per baris. Digabung (union) ke `entities` hasil Researcher di Outline Planner — bukan pengganti riset |
| `faq_seed` | textarea | Opsional | Pertanyaan spesifik yang ingin dijawab di FAQ, satu per baris. Diprioritaskan di atas `commonQuestions` auto dari Researcher saat Outline Planner menyusun `faq[]`, sisa slot (sampai `minFaqPairs`) diisi dari hasil riset |
| `word_count_target` | dropdown | Opsional, default `Auto` | Opsi: `Auto`/`1500`/`2000`/`2500`. Kalau `Auto` → pakai `geo.targetWordCount` dari `content-context.json` seperti biasa; kalau diisi angka → override `totalTargetWords` di Outline Planner |
| `related_article_url` | text | Opsional | URL artikel lain di situs ini untuk internal link. Kalau diisi, Writer menyisipkan satu rujukan alami di section yang relevan (bukan sekadar list "baca juga" di akhir) — sinyal topical authority. Field disiapkan sekarang meski belum ada artikel untuk di-link |

**Konsekuensi ke node lain** (bukan cuma form): `Code: Set Form Run Context` (node 4) parsing
4 field baru jadi array/null; `Outline Planner` (Role 2, lihat `PROMPT_GUIDE.md` §2) yang
menyerap `target_keyword`/`key_entities`/`faq_seed`/`word_count_target` kosong-atau-tidak;
`Schema Generator` (Role 6) yang menangani `category_hint=Auto`; `Writer` (Role 3) yang
menangani `related_article_url` kalau ada.

### 3.5 Aturan global: credential di node `httpRequest` (WAJIB dibaca sebelum bangun `HTTP: AI *` mana pun)

**Ditemukan saat review CP 003/CP 004 — bug yang sama terjadi 2x karena pola ini belum
didokumentasikan.** Node generic `httpRequest` **TIDAK mendukung**
`authentication: "predefinedCredentialType"` dengan `nodeCredentialType: "openAiApi"` — itu
cuma valid untuk node native khusus (mis. ChatOpenAI). Untuk semua `HTTP: AI *` di plan ini
(Safety Guard, Researcher, Outline, Writer, Critic, Reviser, Schema — semua pakai node
`httpRequest` generic), pola yang benar:

```json
{
  "authentication": "genericCredentialType",
  "genericAuthType": "httpHeaderAuth"
}
```

**⚠️ Penautan credential-nya TIDAK BISA lewat n8n MCP** (diverifikasi 3 cara di CP 006:
`setNodeCredential` dengan `openAiApi`, dengan `httpHeaderAuth`, dan `credentials` inline di
`addNode` — semua ditolak `node type 'n8n-nodes-base.httpRequest' does not accept credential`).
n8n sendiri mengonfirmasi: *"HTTP Request nodes were skipped during credential auto-assignment.
Their credentials must be configured manually."*

Jadi urutan yang benar: AI membangun node dengan `genericCredentialType` + `genericAuthType`,
lalu **credential dipasang manual di UI n8n** (atau oleh agent yang punya akses credential
langsung). Selalu tulis `notes` di node yang credential-nya belum terpasang supaya tidak
terlupa.

**Jangan pernah** taruh API key literal di `headerParameters`/`jsonBody` sebagai jalan pintas —
walau ada budget cap di provider, itu tetap kebocoran credential yang tidak perlu.

### Fase 1 — Safety Guard (gate sebelum bakar token)

| # | Node | Type |
|---|---|---|
| 11 | `Code: Build Safety Guard Body` | `code` |
| 12 | `HTTP: AI Safety Guard` | `httpRequest` |
| 13 | `Code: Parse Guard Result` | `code` |
| 14 | `If - Topic Allowed?` | `if` |
| 15 | `Respond - Topic Blocked` | `respondToWebhook` |

Guard jalan **sebelum** Researcher — menolak topik setelah riset berarti sudah buang
~3k token sia-sia.

### Fase 2 — Content Generation (ROLE 1–3 + ROLE 2.5 baru)

| # | Node | Type | Model |
|---|---|---|---|
| 16 | `Code: Build AI Research Body` | `code` | — |
| 17 | `HTTP: AI Researcher` | `httpRequest` | **Berbayar + web search** |
| 18 | `Code: Parse Research` | `code` | — |
| 19 | `Code: Validate Research Facts` | `code` | Reject kalau 0 fakta bersumber → tidak ada gunanya lanjut |
| 20 | `Code: Build AI Outline Body` | `code` | — |
| 21 | `HTTP: AI Outline Planner` | `httpRequest` | **Lokal (Ollama) / Groq free** |
| 22 | `Code: Parse Outline` | `code` | Validasi JSON + minimal N heading |
| 22b | `Code: Build AI Writing Planner Body` | `code` | **Baru (CP 006).** Lihat ROLE 2.5 di bawah |
| 22c | `HTTP: AI Writing Planner` | `httpRequest` | **Berbayar, OpenRouter `openai/gpt-oss-120b`** |
| 22d | `Code: Parse Planner Result` | `code` | Validasi brief per-section, fallback ke outline mentah kalau gagal |
| 23 | `Code: Build AI Writer Body` | `code` | Inject brief dari ROLE 2.5 (bukan outline mentah lagi) |
| 24 | `HTTP: AI Writer` | `httpRequest` | **Berbayar (`openai/gpt-5-nano`, tier ringan — lihat catatan ROLE 2.5)** |
| 25 | `Code: Parse Draft` | `code` | — |

> **Penomoran `22b/22c/22d`** sengaja tidak menggeser nomor node lain — pola sama dengan
> `39b` dan nomor bolong 43–45 (lihat §3 Fase 3/5/6).

#### ROLE 2.5 — AI Writing Planner (baru, CP 006)

**Kenapa ditambahkan:** model Writer (`openai/gpt-5-nano`) dipilih tier ringan demi hemat
budget (keputusan sadar, lihat CP 004). Model ringan lebih butuh instruksi konkret per-section
dibanding menyusun argumen dari nol. **AI Writing Planner** (model lebih kuat, `openai/
gpt-oss-120b` via OpenRouter — pola diambil dari node "AI Planner (MiMo)" di
`ig Content Builder`, cuma endpoint+model+credential yang direplikasi, bukan format JSON-nya)
mengambil outline + fakta + entitas, lalu menyusun **brief tertulis per-section** (paragraf
jawaban-langsung draft, fakta mana yang dikutip di mana, entitas mana yang disebut, kalimat
quotable draft, filsuf/teori mana yang jadi rujukan stance). Writer (`gpt-5-nano`) tinggal
mengembangkan/merapikan brief itu jadi prosa penuh — bukan menyusun dari outline mentah.

Detail prompt: [PROMPT_GUIDE.md](PROMPT_GUIDE.md) §2.5.

### Fase 3 — GEO Scoring (ROLE 4)

| # | Node | Type | Fungsi |
|---|---|---|---|
| 26 | `Code: GEO Rule Checker` | `code` | **0 token.** Rule-based, **6 kategori** (Kategori 6/Metadata di-exclude — lihat catatan) |
| 27 | `Code: Build AI Critic Body` | `code` | Kirim draft + rubric AI-only |
| 28 | `HTTP: AI GEO Critic` | `httpRequest` | Berbayar (judgment bahasa) |
| 29 | `Code: Parse Critique` | `code` | — |
| 30 | `Code: Compute GEO Score` | `code` | Gabung skor rule + AI dari **6 kategori** (dinormalisasi ke 0–100) |
| 31 | `Switch - GEO Score Gate` | `switch` | `≥80` pass / `60–79` revise / `<60` reject |

Detail rubric & formula: [GEO_SCORING_RUBRIC.md](GEO_SCORING_RUBRIC.md).

> **Kategori 6 (Metadata & Schema) di-exclude dari node 26/30/31 (keputusan CP 005):**
> Kategori ini butuh output Schema Generator (node 37–39) yang belum ada di titik ini.
> Bobotnya (10 poin) didistribusikan proporsional ke 6 kategori lain untuk gate score.
> Dicek terpisah setelah node 39, sebelum node 40 — lihat Fase 5 & `GEO_SCORING_RUBRIC.md`
> §"Kategori 6 — timing khusus".

> **Catatan refactor opsional:** node 26–31 bisa diekstrak jadi sub-workflow tersendiri
> (`GEO Scoring Core`), dipanggil via `Execute Workflow`, supaya logic-nya bisa dipakai
> ulang oleh workflow scoring-only terpisah (upload artikel manual → langsung skor,
> tanpa generation/publish). Ini **opsional, prioritas kedua** — kerjakan setelah plan
> ini stabil sampai Fase 8. Detail: [OPTIONAL_SCORING_WORKFLOW.md](OPTIONAL_SCORING_WORKFLOW.md).

> **Kenapa Critic dipisah dari Writer:** model yang menulis lalu menilai tulisannya sendiri
> dalam satu call cenderung bias optimistis terhadap outputnya. Call terpisah dengan
> instruksi "kamu auditor, bukan penulis" memberi evaluasi yang jauh lebih tajam.

### Fase 4 — Revise Loop (ROLE 5)

| # | Node | Type |
|---|---|---|
| 32 | `Code: Check Retry Limit` | `code` |
| 33 | `If - Retry Available?` | `if` |
| 34 | `Code: Build AI Reviser Body` | `code` |
| 35 | `HTTP: AI Reviser` | `httpRequest` |
| 36 | `Code: Parse Revised Draft` | `code` |

Output node 36 → balik ke node 26 (`GEO Rule Checker`) untuk re-score. Max **2 loop**;
lewat itu → jalur reject.

> **Peringatan loop di n8n:** n8n tidak punya loop-back native yang aman tanpa state.
> Counter `reviseCount` **wajib** dibawa di dalam item data (di-increment di node 32),
> bukan diandalkan ke struktur koneksi. Tanpa ini, loop bisa jalan tak berhenti dan
> membakar token — ini pola yang sama dengan `Check Retry Limit` di ig-content-builder.

### Fase 5 — Schema & Metadata (ROLE 6)

| # | Node | Type | Model |
|---|---|---|---|
| 37 | `Code: Build AI Schema Body` | `code` | — |
| 38 | `HTTP: AI Schema Generator` | `httpRequest` | **Lokal (Ollama) / Groq free** |
| 39 | `Code: Parse & Validate Schema` | `code` | `JSON.parse` + cek field wajib JSON-LD |

Kalau JSON-LD dari model lokal invalid → fallback ke schema minimal yang di-generate
deterministik oleh jsCode (headline, datePublished, author dari content-context).
**Jangan gagalkan seluruh run** hanya karena model 7B keliru format JSON.

| 39b | `Code: Validate Metadata Rules` | `code` | **Baru (CP 005).** Cek Kategori 6 rubric (yang di-exclude dari node 26/30) sekarang setelah schema ada. Gagal → fallback deterministik (bukan balik ke Reviser). Lihat `GEO_SCORING_RUBRIC.md` §"Kategori 6 — timing khusus" |

Penomoran `39b` sengaja tidak menggeser Fase 6–8 (sama seperti pola nomor bolong 43–45 yang
dihapus di CP 001) — supaya referensi nomor node di dokumen lain tidak perlu diubah semua.

### Fase 6 — Publish

| # | Node | Type | Catatan |
|---|---|---|---|
| 40 | `Code: Build WordPress Payload` | `code` | Map kategori/tag ke ID, inject JSON-LD sebagai `<script type="application/ld+json">` di dalam `content` (WordPress.com REST API tidak punya field schema terpisah), `status: 'publish'` |
| 41 | `HTTP: WordPress Create Post` | `httpRequest` | **Update (CP 001):** `POST https://public-api.wordpress.com/rest/v1.1/sites/satryapudja.wordpress.com/posts/new`, auth = credential **OAuth2 generic** (`Authentication: Body`) — bukan Basic Auth, bukan node native "Wordpress" |
| 42 | `Code: Extract WP Result` | `code` | Ambil `link`/`URL` + `id` untuk log & notifikasi Telegram |
| ~~43~~ | ~~`Code: Build Medium Payload`~~ | — | **DIHAPUS (CP 001) — Medium di-drop, token tidak tersedia** |
| ~~44~~ | ~~`HTTP: Medium Create Post`~~ | — | **DIHAPUS (CP 001)** |
| ~~45~~ | ~~`Code: Extract Medium Result`~~ | — | **DIHAPUS (CP 001)** |

Node 43–45 dihapus permanen dari scope build (bukan sekadar dinonaktifkan) — keputusan final
di CP 001. Nomor node dibiarkan bolong (tidak di-renumber ke Fase 7/8) supaya referensi nomor
node di dokumen lain (`todos.md`, `ai_docs/index.md`, `ARCHITECTURE_NOTES.md`) tidak perlu
diubah semua sekaligus.

### Fase 7 — Logging & Notifikasi

| # | Node | Type |
|---|---|---|
| 46 | `Google Sheets: Log Article + GEO Score` | `googleSheets` |
| 47 | `Google Sheets: Mark Topic Success` | `googleSheets` |
| 48 | `Code: Build Result Notification` | `code` |
| 49 | `Telegram - Send Result` | `telegram` |
| 50 | `Respond - Success` | `respondToWebhook` |

### Fase 8 — Error Path

| # | Node | Type |
|---|---|---|
| 51 | `Code: Build Failure Context` | `code` |
| 52 | `Google Sheets: Mark Topic Failed` | `googleSheets` |
| 53 | `Telegram - Operational Alert` | `telegram` |
| 54 | `Respond - Error` | `respondToWebhook` |

Format notifikasi sukses (node 48):

```
✅ Artikel published
📝 {title}
📊 GEO Score: {score}/100  (revise: {reviseCount}x)
🔗 WordPress: {wpLink}
⏱️ {durationSec}s · {tokenEstimate} token
```

---

## 4. Kolom Google Sheets

**Sheet `topic_queue`** (sumber cron + locking):

| Kolom | Isi |
|---|---|
| `topic` | Topik/judul kerja |
| `target_keyword` | Keyword utama (untuk cek stuffing di rule checker) |
| `category_hint` | Kategori WP yang disarankan (opsional) |
| `status` | `pending` / `processing` / `success` / `failed` |
| `run_id` | Diisi saat claim — pemilik lock |
| `claimed_at` | Timestamp claim (untuk deteksi lock macet) |
| `note` | Pesan error kalau failed |

**Sheet `article_log`** (audit + tren skor):

| Kolom | Isi |
|---|---|
| `run_id`, `created_at`, `title`, `slug` | Identitas |
| `geo_score` | Skor akhir |
| `score_breakdown` | JSON per-kategori — untuk lihat kategori mana yang sering jeblok |
| `revise_count` | Berapa kali masuk revise loop |
| `wp_post_id`, `wp_url` | Hasil publish |
| `word_count`, `token_estimate` | Biaya |

> `score_breakdown` per-kategori itu yang bikin log ini berguna: kalau setelah 20 artikel
> ternyata kategori "Citation & Fact Density" selalu paling rendah, masalahnya ada di
> prompt Researcher — bukan di Writer. Tanpa breakdown, kamu cuma tahu "skornya jelek".

---

## 5. Konfigurasi model & estimasi biaya

| Role | Model | Alasan |
|---|---|---|
| Safety Guard | Lokal / model kecil | Klasifikasi biner, tugas ringan |
| 1. Researcher | Berbayar **dengan web search** | Wajib grounded; ini sumber semua fakta |
| 2. Outline | **Lokal** Qwen2.5-7B-Instruct Q4_K_M | Structured output, tugas terstruktur |
| 3. Writer | Berbayar, model terkuat | Kualitas prosa = inti produk |
| 4. Critic | Berbayar | Judgment bahasa yang halus |
| 5. Reviser | Berbayar | Menulis ulang, butuh kualitas Writer |
| 6. Schema | **Lokal** Qwen2.5-7B-Instruct Q4_K_M | Ekstraksi → JSON, bukan kreatif |

### Setup model lokal (8GB VRAM)

```bash
ollama pull qwen2.5:7b-instruct-q4_K_M    # ~4.4GB — aman di 8GB VRAM
ollama serve                               # listen di localhost:11434
```

n8n memanggilnya via `httpRequest` ke `http://localhost:11434/api/chat` dengan
`"format": "json"` dan `"stream": false`.

**Batas aman 8GB VRAM:** 7B–8B di Q4_K_M (~4–5GB bobot) menyisakan ~3–4GB untuk KV cache,
cukup untuk context ~8–16k token. **Hindari 13B+** — bobot Q4-nya saja sudah ~7–8GB,
sehingga KV cache terdorong ke CPU dan inferensi melambat drastis. Kalau GPU juga dipakai
untuk display desktop (bukan headless), sisakan margin lebih besar lagi.

**Fallback:** kalau model lokal kurang stabil menghasilkan JSON valid, Groq free tier
menyediakan model open-weight dengan latensi sangat rendah — swap URL-nya saja, struktur
node tidak berubah.

### Estimasi token per artikel (~1500 kata)

| Role | Input | Output | Total | Ditagih? |
|---|---|---|---|---|
| Safety Guard | ~0.3k | ~0.1k | ~0.4k | lokal → gratis |
| 1. Researcher | ~2–2.5k | ~0.6–0.8k | ~2.6–3.3k | ✅ |
| 2. Outline | ~1.2k | ~0.4–0.6k | ~1.6–1.8k | lokal → gratis |
| 3. Writer | ~2.5–3k | ~2–2.5k | ~4.5–5.5k | ✅ |
| 4. Critic | ~3–3.5k | ~0.4–0.6k | ~3.4–4.1k | ✅ |
| 5. Reviser | ~2.5–3k | ~2–2.5k | ~4.5–5.5k | ✅ (kalau kena) |
| 6. Schema | ~2.5k | ~0.3–0.5k | ~2.8–3k | lokal → gratis |

- **Total semua role:** ~19.4k–23.4k token/artikel (1x revise).
- **Yang benar-benar ditagih** (Outline + Schema + Guard dipindah ke lokal):
  **~15k–18.4k token/artikel**.
- Skenario 2x revise loop: naik ke ~27k–33k total (~22k–27k ditagih).

Memindahkan Outline + Schema + Guard ke lokal menghemat **~4.8k–5.2k token/artikel**
(~22–25%) tanpa mengorbankan kualitas prosa, karena ketiganya tugas terstruktur —
bukan penulisan.

---

## 6. Urutan implementasi

Bertahap, tiap fase bisa dites sendiri sebelum lanjut — jangan bangun 50 node lalu tes
sekali di akhir.

| Tahap | Cakupan | Kriteria selesai |
|---|---|---|
| **0** | ~~Verifikasi Medium token~~ (§1.1) | **Selesai — CP 001.** Medium di-drop, WordPress-only via OAuth2 (WordPress.com) |
| 1 | Content context file + sheet `topic_queue`/`article_log` | Sheet terisi ≥3 topik uji |
| 2 | Node 1–15 (trigger, lock, guard) | Topik terklaim benar; topik terlarang tertolak |
| 3 | Node 16–25 (Research → Outline → Draft) | Draft 1500 kata keluar utuh, fakta ada sumbernya |
| 4 | Node 26 saja (`GEO Rule Checker`) | Skor rule-based masuk akal di 3 draft uji |
| 5 | Node 27–31 (Critic + Compute Score) | Skor gabungan stabil, tidak liar antar run |
| 6 | Node 32–36 (revise loop) | Loop berhenti tepat di 2x, `reviseCount` naik benar |
| 7 | Node 37–39 (schema, model lokal) | JSON-LD valid; fallback bekerja saat sengaja dirusak |
| 8 | Node 40–42 (WordPress) | Artikel live, kategori benar, schema tertanam |
| ~~9~~ | ~~Node 43–45 (Medium)~~ | **Dihapus dari scope (CP 001)** |
| 10 | Node 46–54 (log, notif, error path) | Telegram masuk; error path diuji dengan sengaja gagal |

**Tes tahap 4 sebelum 5 itu penting**: rule checker gratis dan deterministik. Kalau
skor rule-based-nya sudah tidak masuk akal, menambah Critic AI di atasnya hanya
menyamarkan masalah dengan biaya token.

---

## 7. Proses eksekusi

Tabel §6 di atas adalah **cakupan pekerjaan**, bukan urutan CP yang mengikat 1:1 — Planner
memecahnya jadi CP sesuai `rules.md` (bisa lebih granular kalau risk/token tinggi). Backlog
mentahnya ada di [`../ai_docs/todos.md`](../ai_docs/todos.md), sudah diseed dari 10 tahap
di §6. Status per-CP dilacak di [`../ai_docs/index.md`](../ai_docs/index.md).

## 8. Dokumen terkait

- [GEO_SCORING_RUBRIC.md](GEO_SCORING_RUBRIC.md) — rubric lengkap 7 kategori, bobot,
  logic deteksi tiap kriteria, formula skor
- [PROMPT_GUIDE.md](PROMPT_GUIDE.md) — instruksi menyusun prompt tiap role, terutama
  GEO Writer (teknik + scaffold siap pakai)
- [ARCHITECTURE_NOTES.md](ARCHITECTURE_NOTES.md) — pola yang direuse dari
  `ig Content Builder For Demo`
- `config/content-context.json` — niche, kategori/tag WP, tone, author/E-E-A-T
