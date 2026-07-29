# CP 006 — AI Writing Planner (Node 22b-22d)

**Mode:** Planner · **Asal:** instruksi langsung user (kompensasi Writer tier ringan), dicatat
di `ai_docs/todos.md`
**Dokumen rujukan wajib dibaca sebelum eksekusi:** `docs/PROMPT_GUIDE.md` §2.5 (baru),
`docs/IMPLEMENTATION_PLAN.md` §3 Fase 2 (baru), `docs/IMPLEMENTATION_PLAN.md` §3.5 (pola
credential httpRequest — WAJIB diikuti supaya tidak ulang bug CP 003/004)

## Ringkasan CP (gambaran besar)

- **Confidence:** high — pola node (`Build Body` → `HTTP` → `Parse`) sudah established dari
  CP 004, endpoint/model sudah dikonfirmasi dari `ig Content Builder`, credential OpenRouter
  yang cocok (`OpenRouter - GPT OSS 120B`, id `IgyTvPzKEgQWHXCk`) sudah ada.
- **Risk:** low-medium — 1 AI call baru, tapi **bukan gate** (kegagalannya fallback ke outline
  mentah, bukan menghentikan run) — beda dari Safety Guard/Validate Facts yang harus benar.
- **Token/limit usage estimate:** medium — 1 AI call tambahan per artikel (~2-3k token,
  model lebih besar dari Outline tapi lebih kecil dari Writer/Researcher).
- **Kesimpulan:** tidak perlu split sub-CP A/B.

## Langkah

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Node 22b `Code: Build AI Writing Planner Body` — prompt persis dari `PROMPT_GUIDE.md` §2.5, ambil context via `$('Code: Build Run Context')` untuk `contentContext.tone`, dan `$input`/`$('Code: Parse Outline')` untuk `outline`/`facts`/`entities` | high | low | low |
| 2 | Node 22c `HTTP: AI Writing Planner` — POST `https://openrouter.ai/api/v1/chat/completions`, model `openai/gpt-oss-120b`, **WAJIB** pakai pola credential §3.5 (`genericCredentialType`/`httpHeaderAuth`, `setNodeCredential` ke `IgyTvPzKEgQWHXCk` — cek dulu tipe credential itu, kalau bukan `httpHeaderAuth` pakai `OpenRouter Authorization - GPT OSS 120B` id `oSM2yzrS4VtlyIcj` yang sudah pasti `httpHeaderAuth`) | medium | medium | low |
| 3 | Node 22d `Code: Parse Planner Result` — parse JSON, validasi `sectionBriefs.length` cocok jumlah section outline. **Gagal → jangan throw, teruskan `sectionBriefs: null`** supaya Writer fallback ke outline mentah (lihat §2.5 "Fallback wajib") | high | low | low |
| 4 | Update node 23 `Code: Build AI Writer Body` — tambah field `sectionBriefs` dari node 22d ke prompt Writer sesuai `PROMPT_GUIDE.md` §3.2 yang sudah diperbarui (bagian "CARA PAKAI BRIEF PER-SECTION") | high | low | low |
| 5 | Rewire koneksi: `Code: Parse Outline` (22) → `Code: Build AI Writing Planner Body` (22b) → `HTTP: AI Writing Planner` (22c, main+error ke 22d) → `Code: Parse Planner Result` (22d) → `Code: Build AI Writer Body` (23) — putuskan koneksi lama `22 → 23` langsung | high | low | low |
| 6 | Verifikasi via `get_workflow_details`: credential benar-benar tertaut (bukan cuma `nodeCredentialType` di parameters — pelajaran dari CP 003/004), koneksi benar | high | low | low |
| 7 | Tulis log, update `index.md`, update `todos.md` | high | low | low |

## Catatan teknis

- **Ini BUKAN gate/validasi wajib** — beda dari Safety Guard (node 12-14) atau Validate
  Research Facts (node 19) yang boleh menghentikan run. Kalau Planner gagal/timeout, Writer
  tetap harus bisa jalan dengan outline mentah seperti sebelum CP 006 ada. Jangan buat node
  22d melempar error yang menghentikan pipeline.
- **Jangan ambil format JSON/config dari `ig Content Builder`** — instruksi eksplisit user:
  cuma model+provider+endpoint yang direplikasi. Skema JSON brief (`sectionBriefs`,
  `factsToCite`, dll) dirancang khusus untuk kebutuhan GEO Builder, beda total dari skema
  planner ig-content-builder (yang untuk carousel Instagram).
- **Model Writer (`gpt-5-nano`) TIDAK diganti** — CP ini kompensasi lewat brief yang lebih
  detail, bukan upgrade model Writer.
- **Konsekuensi ke estimasi token/biaya** (`IMPLEMENTATION_PLAN.md` §5): total token per
  artikel naik karena ada 1 AI call baru — perlu update tabel estimasi token setelah CP ini
  jalan dan datanya ada.

## Definition of done

- [ ] Node 22b-22d ada di workflow `rwIbdIkIhoVE8nkG`, koneksi benar (22→22b→22c→22d→23)
- [ ] Credential node 22c benar-benar tertaut (diverifikasi via `get_workflow_details`, bukan
      cuma diasumsikan dari kode)
- [ ] Node 23 (`Build AI Writer Body`) menyerap `sectionBriefs` sesuai prompt §3.2 yang baru
- [ ] Node 22d tidak menghentikan pipeline saat parse gagal — fallback ke outline mentah teruji
- [ ] `IMPLEMENTATION_PLAN.md` §5 (estimasi token) dicatat untuk update setelah data nyata ada
- [ ] Log ditulis, `index.md` baris CP 006 ditambahkan, `todos.md` item dicoret `-> 006`

## Next mode

Worker — brief dulu sebelum eksekusi (§3 rules.md).

---

## Draft siap eksekusi (disiapkan, BELUM di-push ke n8n)

**Status: draft, menunggu CP 005 selesai** (agent lain sedang kerja di workflow yang sama —
sengaja ditahan supaya tidak ada edit bersamaan yang saling menimpa). Begini konten
node-nya, siap dieksekusi via `update_workflow` begitu CP 005 dikonfirmasi selesai:

**Koreksi dari plan awal:** credential `IgyTvPzKEgQWHXCk` ("OpenRouter - GPT OSS 120B")
ternyata tipe `openAiApi`, **BUKAN** `httpHeaderAuth` — jadi yang dipakai untuk node 22c
adalah `OpenRouter Authorization - GPT OSS 120B` (id `oSM2yzrS4VtlyIcj`, tipe `httpHeaderAuth`,
sudah dikonfirmasi lewat `list_credentials`).

### Node 22b — `Code: Build AI Writing Planner Body`

```js
const ctx = $('Code: Build Run Context').first().json;
const outlineData = $input.first().json;
const outline = outlineData.outline;
const facts = outlineData.facts || [];
const entities = outlineData.entities || [];
const tone = ctx.contentContext?.tone || {};

const prompt = `Kamu adalah editor senior yang menyusun brief penulisan detail untuk penulis junior.
Brief ini HARUS cukup lengkap sehingga penulis junior tidak perlu berpikir ulang soal
argumen atau fakta mana yang dipakai - tinggal merapikan jadi prosa.

JUDUL KERJA   : ${outline.workingTitle}
OUTLINE       : ${JSON.stringify(outline.sections)}
FAQ           : ${JSON.stringify(outline.faq)}
FAKTA + SUMBER: ${JSON.stringify(facts)}
ENTITAS KUNCI : ${JSON.stringify(entities)}
TONE          : ${JSON.stringify(tone)}

Untuk SETIAP section di OUTLINE, buat:
1. draftOpeningAnswer: draft paragraf jawaban-langsung 40-80 kata yang menjawab
   answerFocus section itu secara utuh.
2. factsToCite: index fakta dari FAKTA + SUMBER yang WAJIB dikutip di section ini.
3. entitiesToMention: nama dari ENTITAS KUNCI yang wajib disebut eksplisit di section ini.
4. stanceNote: filsuf/teori/riset spesifik mana yang jadi rujukan kalau section ini
   mengambil stance/opini (kosongkan kalau section ini murni faktual).
5. quotableDraft: draft 1 kalimat ringkas-definitif yang bisa dikutip berdiri sendiri.

Balas HANYA JSON:
{
  "sectionBriefs": [
    {"heading": "<harus cocok persis dengan heading di OUTLINE>",
     "draftOpeningAnswer": "...", "factsToCite": [0,2], "entitiesToMention": ["..."],
     "stanceNote": "...", "quotableDraft": "..."}
  ],
  "faqBriefs": [{"question": "<H3 dari FAQ>", "draftAnswer": "<draft 40-70 kata>"}]
}`;

return [{ json: {
  _plannerBody: {
    model: 'openai/gpt-oss-120b',
    messages: [
      { role: 'system', content: 'Anda adalah editor senior. Balas HANYA JSON valid.' },
      { role: 'user', content: prompt }
    ],
    max_tokens: 3000,
    temperature: 0.4,
    response_format: { type: 'json_object' }
  },
  _outline: outline,
  _facts: facts,
  _entities: entities
} }];
```

### Node 22c — `HTTP: AI Writing Planner`

```json
{
  "method": "POST",
  "url": "https://openrouter.ai/api/v1/chat/completions",
  "authentication": "genericCredentialType",
  "genericAuthType": "httpHeaderAuth",
  "sendHeaders": true,
  "headerParameters": {"parameters": [{"name": "Content-Type", "value": "application/json"}]},
  "sendBody": true,
  "specifyBody": "json",
  "jsonBody": "={{ $json._plannerBody }}",
  "options": {"timeout": 90000},
  "onError": "continueErrorOutput",
  "retryOnFail": true,
  "maxTries": 2,
  "waitBetweenTries": 3000
}
```
Credential: `setNodeCredential` → `credentialKey: "httpHeaderAuth"`, `credentialId:
"oSM2yzrS4VtlyIcj"`, `credentialName: "OpenRouter Authorization - GPT OSS 120B"`.

### Node 22d — `Code: Parse Planner Result`

```js
const apiResp = $input.first().json;
const carried = $('Code: Build AI Writing Planner Body').first().json;
const r = apiResp.choices?.[0]?.message?.content || apiResp.content || "";

function pj(v) {
  if (v && typeof v === "object") return v;
  const t = String(v || "").replace(/```json?/gi, "").replace(/```/g, "").trim();
  try { return JSON.parse(t); } catch (e) { return null; }
}
const p = pj(r);
const outline = carried._outline;
const expectedCount = (outline?.sections || []).length;
const sectionBriefs = Array.isArray(p?.sectionBriefs) ? p.sectionBriefs : null;
const faqBriefs = Array.isArray(p?.faqBriefs) ? p.faqBriefs : null;
const valid = sectionBriefs && sectionBriefs.length === expectedCount;

// TIDAK throw — role ini bukan gate, gagal cukup teruskan null (Writer fallback ke outline mentah)
return [{ json: {
  outline: outline,
  facts: carried._facts || [],
  entities: carried._entities || [],
  sectionBriefs: valid ? sectionBriefs : null,
  faqBriefs: valid ? faqBriefs : null
} }];
```

### Node 23 — `Code: Build AI Writer Body` (UPDATE, bukan node baru)

Tambahkan di awal fungsi (setelah `const outlineData = $input.first().json;` yang sudah ada
dari CP 004): `const sectionBriefs = outlineData.sectionBriefs || null;`, lalu sisipkan
blok berikut ke prompt SEBELUM baris `=== ATURAN WAJIB ===`:

```js
const briefSection = sectionBriefs
  ? `\n\n=== CARA PAKAI BRIEF PER-SECTION (dari AI Writing Planner) ===\nTiap section di BRIEF PER-SECTION sudah punya draft jawaban-langsung, fakta yang wajib\ndikutip, entitas yang wajib disebut, rujukan stance, dan draft kalimat quotable. Kembangkan\ndan rapikan brief itu jadi prosa penuh sesuai [ATURAN WAJIB] di bawah - JANGAN menyusun\nargumen baru dari nol kalau brief sudah menyediakannya. Tetap taat ke [FAKTA] - hanya kutip\nfakta yang memang ada di FAKTA + SUMBER.\n\nBRIEF PER-SECTION: ${JSON.stringify(sectionBriefs)}`
  : '';
```
lalu tambahkan `${briefSection}` di akhir blok `=== KONTEKS ===` (setelah baris
`ARTIKEL TERKAIT`), sebelum `=== ATURAN WAJIB ===`. Sisa prompt (aturan 1-15) **tidak berubah**.

### Operasi `update_workflow` (urutan eksekusi)

1. `addNode` 22b, 22c, 22d (posisi antara node 22 dan 23 di canvas, mis. x=4400,4600,4800
   — geser node 23 dkk ke kanan kalau perlu `setNodePosition`)
2. `setNodeCredential` node 22c → `httpHeaderAuth` / `oSM2yzrS4VtlyIcj`
3. `updateNodeParameters` node 23 (`Code: Build AI Writer Body`) → jsCode baru dengan
   `sectionBriefs`/`briefSection`
4. `removeConnection` node 22 → node 23 (langsung)
5. `addConnection` 22→22b, 22b→22c, 22c→22d (main+error), 22d→23
6. Verifikasi via `get_workflow_details`

**Trigger publish:** begitu CP 005 dikonfirmasi selesai (status `done` di `index.md`), jalankan
langkah di atas sebagai Worker CP 006.
