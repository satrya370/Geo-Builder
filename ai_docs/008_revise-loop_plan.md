# CP 008 — Revise Loop (Node 32–36)

**Mode:** Planner · **Asal:** `ai_docs/todos.md` item 15 — "Bangun node 32–36: revise loop, pastikan berhenti tepat di 2x dan `reviseCount` naik benar"
**Dokumen rujukan wajib:** `docs/IMPLEMENTATION_PLAN.md` §3 Fase 4 (node 32-36, peringatan loop-back n8n), `docs/PROMPT_GUIDE.md` §5 (prompt Reviser, copy PERSIS), `docs/GEO_SCORING_RUBRIC.md` (threshold, `failedCategories`), workflow `rwIbdIkIhoVE8nkG` (node 30 output `verdict`/`failedCategories`/`critique.topFixes`, node 26 input `draft`)

## Ringkasan CP (gambaran besar)

- **Confidence:** medium — pola Reviser (`Build → AI → Parse → loop-back`) sudah jelas dari plan + `PROMPT_GUIDE.md` §5, tapi loop-back di n8n tanpa state native adalah sumber bug paling rawan (loop tak berhenti = bakar token terus-menerus)
- **Risk:** medium-high — `reviseCount` yang salah inkremen atau lupa di-reset berarti loop bisa jalan selamanya; Reviser yang "memperbaiki" section yang sudah lolos justru akan menurunkan skor antar iterasi; credential Reviser perlu disiapkan
- **Token/limit usage estimate:** medium saat build, **high saat uji** — 1 Reviser AI call per iterasi (~4.5–5.5k token per call), uji loop 2× berarti 2 call (~9–11k token)
- **Kesimpulan:** risk medium-high + confidence medium → **tidak** memenuhi syarat split sub-CP A/B (butuh risk tinggi DAN confidence rendah). Tapi uji loop rawan bakar token; Worker wajib stop jika estimasi token melebihi batas

## Langkah

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Node 32 `Code: Check Retry Limit` — increment `reviseCount` (di dalam item data, BUKAN variable global), cek apakah sudah mencapai max (2). Jika > 2 → normalkan ulang count dan teruskan (jalur reject nanti ditentukan di node 33). Output WAJIB membawa ulang `draft`, `geoScore`, `verdict`, `failedCategories`, `critique.topFixes`, `facts` dari node sebelumnya | high | medium | low |
| 2 | Node 33 `If - Retry Available?` — cek `reviseCount <= 2`. True → lanjut ke node 34 (Reviser). False → jalur reject (node 51+, belum ada — CUKUP CATAT sebagai dependency, jangan dibangun di sini). Output false dibiarkan menggantung | high | low | low |
| 3 | Node 34 `Code: Build AI Reviser Body` — prompt **copy PERSIS** dari `PROMPT_GUIDE.md` §5. Inject `draft`, `geoScore`, `failedCategories`, `critique` (output mentah Critic dari node 29), `critique.topFixes`, `facts` (dari `$('Code: Parse Research').first().json.facts`). Ambil `facts` lewat referensi bernama — jangan asumsikan ada di `$input` | high | low | low |
| 4 | Node 35 `HTTP: AI Reviser` — POST ke OpenRouter, model **berbayar setara Writer** (default: `openai/gpt-oss-120b` atau `openai/gpt-5-nano` — ikuti user decision untuk Writer. Gunakan model yang sama kecuali user spesifikasikan lain). Temperature 0.5, max_tokens = draftLength × 1.2. Credential `genericCredentialType`+`httpHeaderAuth` → `oSM2yzrS4VtlyIcj`. Timeout 180s, retry 2× @ 3s | medium | medium | medium |
| 5 | Node 36 `Code: Parse Revised Draft` — ekstrak markdown bersih (sama seperti node 25 `Parse Draft` — hapus preamble, code fence, ambil dari awal heading pertama). **Output WAJIB pakai field bernama `draft`** (ini kontrak dengan node 26 yang membaca `$input.first().json.draft`). Output juga bawa `reviseCount` (dari node 32) + `geoScore`, `failedCategories` dari input | high | low | low |
| 6 | Connect loop-back: node 36 → node 26 (`Code: GEO Rule Checker`) — ini menciptakan **loop di workflow**. Pastikan `reviseCount` terbawa DI DALAM item data (step 1) supaya loop punya state dan berhenti di iterasi ke-2 | high | high | low |
| 7 | Uji loop dengan 1 draft REVISE: validasi `reviseCount` naik 1→2, loop berhenti tepat di 2×, skor iterasi ke-2 masuk akal (tidak lebih rendah dari iterasi ke-1) | medium | medium | medium |
| 8 | Verifikasi via `get_workflow_details`: koneksi 30→32→33→34→35→36→26 benar, loop-back terpasang, credential node 35 tertaut | high | low | low |
| 9 | Tulis `008_revise-loop_log.md`, update `index.md`, update `todos.md` | high | low | low |

## Catatan teknis

- **Loop-back state:** n8n tidak punya loop-back native yang aman tanpa state di dalam item. `reviseCount` WAJIB di-increment dan dibawa di dalam item data sepanjang loop (node 32 tambah +1, node 36 teruskan ke node 26, node 26 teruskan ke downstream). Tanpa ini, `Check Retry Limit` tidak akan tahu ini iterasi ke berapa dan loop jalan selamanya. Pola ini diambil dari `Check Retry Limit` di `ig Content Builder` (`ARCHITECTURE_NOTES.md` #3).
- **Reviser "bedah, jangan tulis ulang":** bagian yang TIDAK disebut di `failedCategories`/`topFixes` harus dibiarkan apa adanya. Reviser menulis ulang section yang hanya gagal, bukan seluruh artikel — kalau ia tulis ulang semuanya, skor bisa naik-turun liar antar iterasi dan loop tidak akan konvergen.
- **Fakta untuk Reviser:** ambil dari `$('Code: Parse Research').first().json.facts` lewat referensi bernama — node 30 output tidak membawa `facts`, jadi Reviser body tidak bisa mengandalkan `$input.first().json.facts`. Pola yang sama dengan bug context-hilang di CP 004.
- **Koneksi loop-back:** node 36 output (main) → node 26 input. Ini menciptakan **loop fisik** di workflow n8n. n8n mendukung ini, tapi pastikan tidak ada infinite loop tanpa state. Node 26 nanti akan menerima input dari DUA sumber: node 25 (draft pertama) dan node 36 (draft revisi). Keduanya punya field `draft` — tidak ada konflik.
- **Jalur reject (>2 loop):** node 33 false branch → target node 51+ (`Code: Build Failure Context` + `Telegram - Operational Alert`). Node 51 belum dibangun (Fase 8). **DIBIARKAN MENGGANTUNG** — jangan buat koneksi sementara ke node lain (pengulangan bug CP 004 `Form Trigger → AI Writer`).
- **Credential node 35:** pola `genericCredentialType` + `genericAuthType: "httpHeaderAuth"` + credential `oSM2yzrS4VtlyIcj` (OpenRouter). Model default meniru Writer — dikonfirmasi user di CP 004 (`openai/gpt-5-nano`), bisa diganti ke model setara Writer lain kalau user spesifikasikan.
- **Tidak menyentuh:** node 37–39 (Schema), node 40–42 (WordPress), node 46–54 (logging/notifikasi), node 51+ (error path).

## Dependency

- **Node 31 (`Switch - GEO Score Gate`) belum ada** — stuck di MCP tool limitation (Switch node gagal dibangun, lihat `007_critic-compute-score-gate_log.md`). Node 32 menyambung dari node 31 output REVISE. Worker CP 008 **harus** memastikan node 31 sudah ada sebelum mulai membangun node 32. Kalau belum → buat node 31 dulu manual di n8n UI (3 output: PASS/REVISE/REJECT), atau gunakan workaround IF node bertingkat.

## Definition of done

- [ ] **Node 26 diperbaiki**: output menambahkan field `draft` (passthrough) — WAJIB sebelum step 1
- [ ] **Node 27 diperbaiki**: `draft` diambil dari `$input`, bukan referensi bernama `$('Code: Parse Draft')` — WAJIB sebelum step 1
- [ ] Diverifikasi: `geoScore` iterasi ke-2 benar-benar berubah mengikuti draft revisi (bukan identik dengan iterasi pertama)
- [ ] Node 32–36 ada di workflow `rwIbdIkIhoVE8nkG`, node 32 terhubung dari node 31 output REVISE
- [ ] `reviseCount` di-increment di node 32 dan dibawa dalam item data sepanjang loop
- [ ] Node 33 menghasilkan dua cabang: true (revise available) → node 34, false → gantung
- [ ] Node 34 prompt sesuai `PROMPT_GUIDE.md` §5 persis, `facts` diambil dari `$('Code: Parse Research')`
- [ ] Node 35 credential `oSM2yzrS4VtlyIcj` terpasang
- [ ] Node 36 output field `draft` terhubung balik ke node 26 (loop-back)
- [ ] Uji 1 draft REVISE: `reviseCount` naik dari 1 ke 2, loop berhenti di 2× (draft iterasi ke-3 tidak dikirim ke node 26)
- [ ] Skor iterasi ke-2 tidak lebih rendah dari iterasi ke-1 (atau kalau iya: tercatat di log sebagai data kalibrasi)
- [ ] `008_revise-loop_log.md` tertulis, `index.md` updated, `todos.md` item 15 dicoret `[x]` + `-> 008`

## Next mode

Worker — brief dulu sebelum eksekusi (§3 rules.md). Pastikan node 31 sudah ada sebelum mulai.

---

## Addendum Reviewer (WAJIB dibaca sebelum eksekusi — plan di atas TIDAK BOLEH dijalankan apa adanya)

**Status dependency sudah usang:** bagian "Dependency" di atas bilang node 31 belum ada
karena "stuck di MCP tool limitation". **Ini sudah tidak berlaku** — node 31
(`Switch - GEO Score Gate`) sudah dibangun dan terverifikasi (lihat
`010_cp005-cp007-review_revision.md`). Akar masalah sebelumnya: parameter `rules.rules`
(salah) alih-alih `rules.values` (benar, sesuai `get_node_types`). Worker CP 008 **tidak
perlu** membangun ulang node 31 — langsung sambung dari output `REVISE`-nya.

### BUG KRITIS yang ditemukan saat review — WAJIB diperbaiki SEBELUM step 1 plan ini

Diverifikasi langsung dari kode node yang sudah ada di `rwIbdIkIhoVE8nkG`:

- **Node 26 (`Code: GEO Rule Checker`)** — output-nya cuma `{ ruleScores, diagnostics }`,
  **tidak membawa `draft`** ke node berikutnya.
- **Node 27 (`Code: Build AI Critic Body`)** — mengambil draft lewat **referensi bernama
  tetap**: `$('Code: Parse Draft').first().json.draft`.

`Code: Parse Draft` (node 25) **cuma jalan sekali**, sebelum loop ada. Begitu CP 008
menyambungkan `node 36 → node 26`, Critic (lewat node 27) akan **selamanya membaca draft
PERTAMA**, tidak peduli berapa kali Reviser merevisi — karena referensi bernama itu tidak
pernah menunjuk ke eksekusi node 36 yang baru. **Akibatnya: skor GEO tidak akan pernah
berubah antar iterasi, dan seluruh revise loop jadi tidak berguna secara substansi** —
loop tetap "jalan" (reviseCount naik benar, berhenti di 2×), tapi Reviser bekerja untuk draft
yang tidak pernah benar-benar dievaluasi ulang.

**PERBAIKAN WAJIB, dikerjakan SEBAGAI BAGIAN dari CP 008 (bukan CP terpisah) sebelum step 1:**

1. **Node 26** — tambahkan `draft` ke object return, jadi:
   ```js
   return [{ json: {
     draft,  // <-- WAJIB ditambahkan, passthrough
     ruleScores: { ... },  // tidak berubah
     diagnostics: { ... }  // tidak berubah
   } }];
   ```
2. **Node 27** — ganti baris pertama dari
   `const draft = $('Code: Parse Draft').first().json.draft || '';`
   menjadi
   `const draft = $input.first().json.draft || '';`
   (mengikuti pola node 26 sendiri yang sudah benar pakai `$input`, bukan referensi bernama —
   ini yang membuat node 26 tahan terhadap dua sumber input, node 25 di run pertama dan node 36
   di iterasi loop).

**Verifikasi setelah fix:** jalankan 1 draft lewat loop 2 iterasi, pastikan `geoScore`
iterasi ke-2 **benar-benar dihitung dari draft yang direvisi** (mis. sengaja buat Reviser
mengubah sesuatu yang jelas terukur seperti menghapus 1 filler sentence, lalu cek apakah
`ruleScores.answerFirst` di iterasi ke-2 berubah dibanding iterasi ke-1). Kalau skor identik
persis antar iterasi walau draft berubah, fix belum benar — cek ulang referensi draft di
sepanjang chain 26→27.

### Temuan minor (perbaiki sekalian, tidak fatal tapi jangan diabaikan)

- **`max_tokens = draftLength × 1.2`** (step 4) — `draftLength` adalah jumlah **karakter**
  (`content.length` di node 25), bukan kata. Pola Writer (`PROMPT_GUIDE.md` §3.3) pakai
  `totalTargetWords × 2.2` berbasis kata. Ganti jadi berbasis kata juga, mis. estimasi
  `draft.split(/\s+/).length × 2.2`, supaya konsisten dengan role lain.
- **Model Reviser jangan ditebak sendiri** — plan bilang "gpt-oss-120b atau gpt-5-nano,
  ikuti user". **Tanyakan eksplisit ke user** sebelum step 4, jangan asumsikan salah satu.
- **Fallback `criticFailed`**: `critique.topFixes` jadi `[]` saat Critic gagal parse (lihat
  node 30). Reviser (node 34) akan dapat `failedCategories` tapi tanpa instruksi konkret di
  kasus ini — tambahkan fallback instruction generik di prompt Reviser untuk kasus
  `topFixes` kosong (mis. "kalau tidak ada perbaikan spesifik, revisi bagian dengan skor
  kategori terendah di failedCategories").