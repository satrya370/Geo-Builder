# CP 004 — Researcher, Outline, Writer (Node 16–25)

**Mode:** Planner · **Asal:** `ai_docs/todos.md` item 12 — "Bangun node 16–25: Researcher → Outline → Writer, hasilkan draft 1500 kata dengan fakta bersumber"
**Dokumen rujukan wajib:** `docs/IMPLEMENTATION_PLAN.md` §3 Fase 2, `docs/PROMPT_GUIDE.md` §1–3, `docs/GEO_SCORING_RUBRIC.md`, `config/content-context.json`, workflow `rwIbdIkIhoVE8nkG`

## Ringkasan CP (gambaran besar)

- **Confidence:** medium — struktur node & pola `Build → AI → Parse` sudah jelas dari `ig Content Builder`, tapi prompt Writer (§3.2) adalah inti kualitas dan harus copy-paste persis dari `PROMPT_GUIDE.md`
- **Risk:** medium-high — 3 AI call berurutan; Researcher wajib web search + output JSON valid; Outline pakai model lokal 7B yang rawan format JSON; Writer membakar token paling banyak dan menentukan kualitas draft
- **Token/limit usage estimate:** high — ~4k–6k untuk Researcher, ~1.5k–2k untuk Outline, ~5k–6k untuk Writer; total ~10k–14k per artikel uji
- **Kesimpulan:** tidak perlu split sub-CP A/B (risk tidak tinggi dan confidence tidak rendah bersamaan untuk satu step), tapi Worker wajib **stop** jika estimasi token per run melebihi batas; uji end-to-end hanya dengan 1–2 topik sampel

## Langkah

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Node 16 `Code: Build AI Research Body` — susun prompt dari `PROMPT_GUIDE.md` §1, inject `topic`, `topicDetail`, `targetKeyword`, `keyEntitiesSeed`, `contentContext.niche` | high | low | low |
| 2 | Node 17 `HTTP: AI Researcher` — POST ke provider berbayar dengan web search, temperature 0.3. **Risiko:** credential/endpoint yang tersedia (GLM Z.ai) belum pasti support web search; Worker wajib verifikasi dan buat credential baru kalau perlu | medium | medium | medium |
| 3 | Node 18 `Code: Parse Research` — parse JSON hasil Researcher, fallback ke `facts: []` + reason jika AI gagal format | high | medium | low |
| 4 | Node 19 `Code: Validate Research Facts` — reject run jika `facts.length < contentContext.geo.minFactsRequired` atau jumlah fakta numerik bersumber `< contentContext.geo.minNumericFactsRequired` | high | low | low |
| 5 | Node 20 `Code: Build AI Outline Body` — susun prompt dari `PROMPT_GUIDE.md` §2, inject `topic`, `topicDetail`, `targetKeyword`, `keyEntitiesSeed`, `faqSeed`, `wordCountTarget`, `facts`, `commonQuestions`, `contentContext.tone` | high | low | low |
| 6 | Node 21 `HTTP: AI Outline Planner` — POST ke model lokal Ollama `qwen2.5:7b-instruct-q4_K_M` di `http://localhost:11434/api/chat` dengan `format: "json"`, `stream: false`, temperature 0.4. Fallback ke Groq free tier kalau Ollama tidak tersedia | medium | medium | low |
| 7 | Node 22 `Code: Parse Outline` — parse JSON, validasi field wajib (`workingTitle`, `sections[]` min 5, `faq[]` min 3, `totalTargetWords`), fallback graceful | high | medium | low |
| 8 | Node 23 `Code: Build AI Writer Body` — susun prompt dari `PROMPT_GUIDE.md` §3.2 persis, inject `outline`, `facts`, `entities`, `contentContext.tone`, `contentContext.audience`, `relatedArticleUrl` | high | low | low |
| 9 | Node 24 `HTTP: AI Writer` — POST ke model berbayar terkuat (GLM-4.5-flash atau setara), temperature 0.6–0.7, max_tokens = `outline.totalTargetWords × 2.2` | medium | high | high |
| 10 | Node 25 `Code: Parse Draft` — ekstrak markdown artikel bersih dari response; bersihkan preamble, code block wrapper, atau penjelasan di luar artikel | high | low | low |
| 11 | Uji end-to-end manual dengan 1–2 topik contoh dari niche; verifikasi draft ~1500 kata, fakta bersumber, struktur FAQ, dan tidak ada halusinasi angka | medium | medium | medium |
| 12 | Tulis log Worker `004_researcher-outline-writer_log.md`, update `index.md` baris CP 004, update `todos.md` item 12 | high | low | low |

## Catatan teknis

- **Input dari CP 003:** node 14 `If - Topic Allowed?` true branch → node 16. Run context sudah berisi: `runId`, `runMode`, `topic`, `topicDetail`, `targetKeyword`, `categoryHint`, `keyEntitiesSeed`, `faqSeed`, `wordCountTarget`, `relatedArticleUrl`, `contentContext`, `startedAt`.
- **Prompt copy-paste:** Semua prompt (Researcher §1, Outline §2, Writer §3.2) WAJIB copy persis dari `PROMPT_GUIDE.md` — tidak boleh diparafrase, sama seperti aturan yang diterapkan di CP 003 untuk Safety Guard.
- **Node 17 Researcher — web search:** Ini risiko utama CP. Jika GLM Z.ai tidak support web search, Worker harus membuat credential untuk provider berbayar yang support (mis. OpenAI `gpt-4o-search-preview`, Perplexity API, atau serupa). Endpoint dan model dipilih setelah credential tersedia; struktur node HTTP Request tetap sama.
- **Node 21 Outline — model lokal:** Ollama `qwen2.5:7b-instruct-q4_K_M` di `http://localhost:11434/api/chat`. Body n8n: `{model, messages, format: "json", stream: false, options: {temperature: 0.4, num_ctx: 8192}}`. Jika Ollama tidak tersedia atau sering gagal format, fallback ke Groq free tier dengan model Llama 3.1/Gemma — hanya URL dan model yang berubah, struktur node sama.
- **Node 24 Writer — max_tokens:** `outline.totalTargetWords × 2.2` per `PROMPT_GUIDE.md` §3.3. Jika `outline.totalTargetWords` null/Auto, default ke `contentContext.geo.targetWordCount` (1500). Total target words override juga boleh dari `wordCountTarget` form jika user pilih 1500/2000/2500.
- **Koneksi error:** Setiap `HTTP: AI *` wajib sambungkan error branch ke jalur error logging. Node 18/19 (Parse + Validate) juga perlu menangani error/invalid output dengan anggun; di Fase 8 nanti akan ada `Code: Build Failure Context` + `Telegram - Operational Alert`.
- **Tidak menyentuh:** node 26–31 (GEO Scoring), node 32–36 (Revise), node 37–39 (Schema), node 40–42 (WordPress), node 46–54 (logging/notifikasi) — tetap di CP berikutnya.

## Update pasca-review (ditambahkan setelah reject CP 004 pertama)

- **Pola credential HTTP Request yang BENAR** (lihat `IMPLEMENTATION_PLAN.md` §3.5 baru) —
  `authentication: "predefinedCredentialType"` + `nodeCredentialType: "openAiApi"` **TIDAK
  VALID** untuk node generic `httpRequest` (cuma valid untuk node native seperti ChatOpenAI).
  WAJIB pakai `authentication: "genericCredentialType"`, `genericAuthType: "httpHeaderAuth"`,
  lalu `setNodeCredential` dengan `credentialKey: "httpHeaderAuth"`. Ini bug yang sama persis
  terjadi di CP 003 (Safety Guard) dan percobaan pertama CP 004 (Researcher, Writer) — root
  cause-nya baru ketemu saat review CP 004, bukan sekadar "lupa nge-link".
- **Node 16 — tambahkan web search sungguhan**: model `tencent/hy3-preview` di request body
  WAJIB diberi suffix `:online` (`tencent/hy3-preview:online`) — konvensi OpenRouter untuk
  mengaktifkan plugin web search. Tanpa ini, Researcher cuma mengarang dari training data,
  bukan riset ber-sumber.
- **Jangan hardcode API key di node manapun** (`headerParameters`/`jsonBody`/dll) — SELALU
  lewat credential store n8n, walau ada pengaman budget cap. Credential OpenRouter yang valid
  sudah ada: `OpenRouter - GPT OSS 120B` (id `IgyTvPzKEgQWHXCk`, type `openAiApi` — TIDAK
  dipakai di sini karena bukan httpHeaderAuth) — pakai `OpenRouter Authorization - GPT OSS
  120B` (id `oSM2yzrS4VtlyIcj`, type `httpHeaderAuth`) atau `OpenRouter Fallback` (id
  `p3jbxROuZ1ZrDXA6`, type `httpHeaderAuth`) untuk node 17 dan node 24.
- **Fix koneksi**: putuskan `Form Trigger - Manual Run → HTTP: AI Writer` (koneksi sementara
  yang salah, harus hilang sebelum lanjut node 18-23).
- **Fix context hilang di tengah jalan** (bug baru ditemukan saat review, di luar temuan
  awal): node `Build X Body` yang dibangun setelah node 10 (`Code: Build Run Context`)
  masing-masing menimpa item dengan output sendiri (pola dari `ARCHITECTURE_NOTES.md` #7 yang
  belum diikuti). Node 16, 20, 23 **WAJIB** ambil context asli lewat referensi bernama
  `$('Code: Build Run Context').first().json`, BUKAN `$input.first().json` — karena item yang
  mengalir dari Safety Guard/Parse Guard Result sudah tidak membawa `topic`/`topicDetail`/
  `contentContext` lagi.
- **Model Writer (`openai/gpt-5-nano`) DIBIARKAN untuk sekarang** — keputusan sadar user demi
  hemat budget, evaluasi kualitas setelah MVP tercapai, JANGAN diganti sendiri oleh Worker.
- **Urutan build node boleh tidak sekuensial** (16→17→24 lalu isi tengah) — dikonfirmasi user
  sebagai instruksi yang memang diberikan, bukan bug proses.

## Definition of done

- [ ] Node 16–25 ada di workflow `rwIbdIkIhoVE8nkG` dengan koneksi utuh dari node 14 true branch
- [ ] Node 16 & 17 prompt sesuai `PROMPT_GUIDE.md` §1, output JSON valid dengan ≥5 fakta dan ≥3 fakta numerik bersumber
- [ ] Node 19 berhasil reject run saat fakta tidak memenuhi threshold (uji sengaja dengan topik yang hasil riset minim)
- [ ] Node 20–22 prompt & output sesuai `PROMPT_GUIDE.md` §2, validasi JSON + minimal section/FAQ terpenuhi
- [ ] Node 23–24 prompt sesuai `PROMPT_GUIDE.md` §3.2 persis, parameter API benar (temperature, max_tokens)
- [ ] Node 25 mengeluarkan markdown draft bersih tanpa preamble atau wrapper
- [ ] Uji end-to-end 1–2 topik menghasilkan draft ~1500 kata dengan fakta bersumber dan struktur FAQ
- [ ] Credential untuk Researcher/Writer tersedia dan terpasang; Outline pakai Ollama/Groq
- [ ] `004_researcher-outline-writer_log.md` tertulis, `index.md` baris CP 004 ditambahkan, `todos.md` item 12 dicoret `[x]` + `-> 004`

## Next mode

Worker — dengan brief dulu sebelum eksekusi (§3 rules.md).