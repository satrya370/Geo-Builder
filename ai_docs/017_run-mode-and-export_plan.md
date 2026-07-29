# CP 017 — Run Mode (Live Publish / Live Draft / Test) + Export to TXT + Sederhanakan Source Language → Article Language

## Latar belakang
Empat kebutuhan operasional, digabung jadi satu CP karena saling terkait di area form + tail pipeline:

1. Sekarang **tidak ada cara memilih publish beneran** — `Code: Build WordPress Payload` hardcode `status: 'draft'` selalu (safety default). Perlu opsi eksplisit untuk publish live saat sudah percaya diri, TAPI default tetap draft supaya tidak kepencet publish tidak sengaja.
2. **Tidak ada mode test cepat** — setiap kali mau uji perubahan pipeline (misalnya abis fix bug kayak CP015/016), harus jalan run penuh (~4-5 menit, riset+outline+writer+critic+revise). Untuk sekadar "apakah semua node masih nyambung", ini boros waktu & (kalau research menyala) boros biaya API.
3. **Tidak ada rekam jejak lokal** — hasil akhir cuma ada di WordPress atau di n8n execution log yang gampang ke-scroll hilang. Butuh file `.txt` tersimpan tiap run selesai, apa pun mode-nya.
4. **Field "Source Language" (CP014) terlalu sempit cakupannya** — dulu isinya soal "prioritaskan sumber riset internasional/Indonesia/campuran", tapi sejak CP015 (Concept Brief architecture) web search jadi jarang dipakai (hanya untuk `empiricalClaims` yang di-declare Concept Brief), jadi instruksi itu sudah tidak relevan buat riset. **Tapi kebutuhan aslinya — kontrol bahasa — tetap valid**, cuma salah sasaran: yang sebetulnya dibutuhkan adalah kontrol **bahasa artikel yang ditulis** (Indonesia/Inggris), bukan bahasa sumber riset. Field ini disederhanakan jadi "Language" yang HANYA mengatur bahasa tulisan Writer/Outline/Schema — Concept Brief dan Research TETAP mencari referensi paling kredibel (mayoritas internasional/akademik) apa pun pilihan bahasa artikelnya.
5. **Field "Research Mode" (CP015) — diaudit 2026-07-27, ketemu 2 bug yang bikin error/hasil jelek** (lihat §8 di bawah untuk detail & fix).

## Keputusan desain

### 0. Field "Language" (sederhanakan dari "Source Language", posisi tetap)
Ganti opsi lama (Auto/International/Force Indonesia/Bilingual — semuanya soal SUMBER riset) jadi field baru yang isinya soal BAHASA ARTIKEL:
```json
{
  "fieldLabel": "Language",
  "fieldType": "dropdown",
  "requiredField": false,
  "fieldName": "language",
  "fieldOptions": { "values": [
    { "option": "Indonesia", "value": "Indonesia" },
    { "option": "English", "value": "English" }
  ]}
}
```
Default `Indonesia` (audiens situs ini orang Indonesia — lihat `ctx.contentContext.site.audience`). Tidak perlu opsi "Auto" — ini pilihan sadar user tentang bahasa tulisan, bukan fallback.

**Batasan scope yang wajib dijaga**: field ini HANYA boleh memengaruhi node yang MENULIS untuk pembaca akhir — `Code: Build AI Outline Body` (heading harus natural dalam bahasa itu), `Code: Build AI Writing Planner Body` (draft jawaban per-section), `Code: Build AI Writer Body` (draft artikel penuh), `Code: Build AI Schema Body` (title/metaDescription/slug/excerpt/FAQ — metadata harus konsisten bahasa dengan artikelnya). **JANGAN** disuntikkan ke `Code: Build Concept Brief Body` atau `Code: Build AI Research Body` (targeted) — dua node itu tetap independen, tetap mencari referensi paling kredibel (yang secara empiris kita temukan mayoritas berbahasa Inggris/internasional untuk topik filsafat/psikologi kanonik, lihat riset CP015) apa pun bahasa artikel yang dipilih user.

**Aturan wajib "jangan terjemahkan yang seharusnya tidak diterjemahkan"** — ini bagian paling penting, harus masuk ke prompt SEMUA 4 node yang terpengaruh (Outline/Writing Planner/Writer/Schema), bukan cuma disebut sekali:
```
ATURAN BAHASA:
- Tulis SELURUH prosa (kalimat, penjelasan, transisi) dalam Bahasa ${languageLabel}.
- JANGAN terjemahkan: (a) istilah teknis/filosofis yang tidak punya padanan baku di bahasa
  target (mis. "akrasia", "enkrateia", "phronesis" — istilah asal Yunani ini dipakai APA
  ADANYA di kedua bahasa, boleh dikasih penjelasan singkat di sebelahnya, tapi JANGAN
  diterjemahkan jadi kata buatan sendiri); (b) nama resmi teori/model (mis. "Temporal
  Motivation Theory") — pertahankan nama aslinya, boleh tambah padanan di kurung sekali saja
  saat pertama disebut; (c) KUTIPAN VERBATIM — kutipan dalam tanda kutip WAJIB tetap persis
  dalam bahasa aslinya, JANGAN diterjemahkan sama sekali (menerjemahkan kutipan artinya bukan
  kutipan asli lagi, dan akan GAGAL diverifikasi oleh sistem pengecekan kutipan otomatis
  yang mencari teks persis, lihat CP 016).
- Nama tokoh: pakai ejaan yang konvensional untuk bahasa target KALAU sudah baku (mis. dalam
  Bahasa Indonesia lazim ditulis "Aristoteles", dalam Bahasa Inggris "Aristotle" — ini BUKAN
  pelanggaran aturan "jangan terjemahkan", ini konvensi ejaan nama yang sudah mapan). Kalau
  tidak yakin ada konvensi baku, pertahankan ejaan aslinya.
```
Nilai `languageLabel` = `"Indonesia"` atau `"English"` sesuai `ctx.language`. Instruksi ini SATU set aturan yang sama dipakai di 4 node (boleh diekstrak jadi 1 helper string yang di-reuse, supaya konsisten dan gampang dirawat — bukan ditulis ulang manual 4x dengan risiko beda kata di tiap node).

### 1. Field baru "Run Mode" (dropdown, ditambahkan di akhir sebelum submit — BUKAN menggantikan field manapun)
Field baru menaruh 3 opsi. **Default harus "Live Run - Draft"** (safety-first, konsisten dengan perilaku sekarang):
```json
{
  "fieldLabel": "Run Mode",
  "fieldType": "dropdown",
  "requiredField": false,
  "fieldName": "run_mode",
  "fieldOptions": { "values": [
    { "option": "Live Run - Draft", "value": "Live Run - Draft" },
    { "option": "Live Run - Publish", "value": "Live Run - Publish" },
    { "option": "Test", "value": "Test" }
  ]}
}
```
Field ini pakai nama internal `publishMode` (BUKAN `runMode` — nama itu SUDAH DIPAKAI di `Code: Set Form Run Context` untuk field lain yang menandai asal trigger, hardcode `'form'`, tidak boleh ditimpa/collision).

### 2. `Code: Set Form Run Context` — tambah field, hapus field lama
```js
const f = $input.first().json;
const id = require('crypto').randomUUID();
const trim = (v) => { const s = String(v || '').trim(); return s || null; };
const dropAuto = (v) => { const s = String(v || '').trim(); return (s && s !== 'Auto') ? s : null; };

return [{ json: {
  runId: id,
  runMode: 'form',
  topic: String(f.topic || '').trim(),
  topicDetail: String(f.topic_detail || '').trim(),
  targetKeyword: trim(f.target_keyword),
  categoryHint: dropAuto(f.category_hint),
  keyEntitiesSeed: trim(f.key_entities),
  faqSeed: trim(f.faq_seed),
  wordCountTarget: dropAuto(f.word_count_target),
  language: String(f.language || 'Indonesia').trim(),
  researchMode: String(f.research_mode || 'Auto').trim(),
  publishMode: String(f.run_mode || 'Live Run - Draft').trim(),
  relatedArticleUrl: trim(f.related_article_url),
  startedAt: new Date().toISOString()
} }];
```
Perhatikan: field lama `sourceLanguage: dropAuto(f.source_language)` **diganti** jadi `language` (nilai deklaratif "Indonesia"/"English", BUKAN pola `dropAuto` — karena "Indonesia" adalah pilihan bermakna default, bukan nilai yang di-drop jadi `null`).

### 3. `Code: Build Run Context` — inject override Test mode di satu tempat pusat
Kode sekarang cuma 1 baris (`const a=...;const b=...;return[{json:{...a,contentContext:b.contentContext}}];`). Perluas supaya Test mode memaksa scope kecil TANPA harus ubah logic di tiap node Body satu-satu:
```js
const a = $('Code: Set Form Run Context').first().json;
const b = $('Code: Load Content Context').first().json;
let merged = { ...a, contentContext: b.contentContext };

if (merged.publishMode === 'Test') {
  // Paksa scope kecil supaya test SELALU cepat, apa pun pilihan field lain oleh user.
  merged.wordCountTarget = '300';
  merged.researchMode = 'Concept Only'; // otomatis skip web search (bottleneck terbesar, 100-275 detik)
}

return [{ json: merged }];
```
Kenapa di sini (bukan di tiap Body node): satu titik override, tidak ada risiko ada 1 node yang lupa di-update kalau nanti ada override baru. `researchMode: 'Concept Only'` otomatis membuat `If - Need Empirical Data?` selalu ke cabang "tidak" (lihat CP015 §4), jadi Researcher/web-search tidak pernah terpanggil di mode Test — inilah mekanisme "max kata ato apapun" yang diminta: dikombinasikan, targetWords kecil (300) membuat Outline/Writer/Critic semua menghasilkan output kecil → total waktu jauh lebih pendek, TAPI seluruh node tetap dilalui (Guard → Concept Brief → Outline → Writing Planner → Writer → RuleChecker → Critic → Score Gate → Schema → export) — cakupan test tetap penuh, cuma skalanya kecil.

### 4. Node baru: `If - Is Test Mode?`
Ditaruh setelah `Code: Validate Metadata Rules`, sebelum `Code: Build WordPress Payload`:
```js
{ "conditions": [{ "leftValue": "={{ $('Code: Build Run Context').first().json.publishMode }}", "rightValue": "Test", "operator": { "type": "string", "operation": "equal" } }] }
```
**PERINGATAN WAJIB** (sudah 2x jadi bug nyata di CP015 — lihat `015_concept-brief-architecture_plan.md` §4 dan riwayat perbaikan CP011): port True/False If node **TIDAK BOLEH diasumsikan** port0=true/port1=false. Wiring wajib diverifikasi lewat `execute_workflow` nyata sebelum dianggap benar.

### 5. Wiring tail pipeline — cabang Test skip WordPress, keduanya konvergen ke Export
```
Code: Validate Metadata Rules
  → If - Is Test Mode?
      ├─[Test]──────────────────────────────────────────┐
      └─[Live Run]→ Code: Build WordPress Payload        │
                    → HTTP: WordPress Create Post         │
                    → Code: Extract WP Result ────────────┤
                                                           ↓
                                        Code: Build Export Payload
                                                           ↓
                                        (node convert-to-file, lihat §7)
                                                           ↓
                                        (node write file ke disk, lihat §7)
```
`Code: Build Export Payload` punya 2 kemungkinan node sumber sebelumnya (Test vs Live Run) — JANGAN andalkan `$input`/`$json` langsung (bentuknya beda per cabang). Ambil semua data via referensi nama node eksplisit (pola yang sudah konsisten dipakai di seluruh workflow ini):
```js
const validated = $('Code: Validate Metadata Rules').first().json; // { meta, jsonLd, metadataScore, metadataPct, ... }
const draft = $('Code: Parse Draft').first().json.draft || '';
const ctx = $('Code: Build Run Context').first().json;
const score = $('Code: Compute GEO Score').first().json; // { geoScore, verdict, breakdown }

let wpSection = 'WordPress: (tidak dipublish — mode Test)';
try {
  const wp = $('Code: Extract WP Result').first().json;
  wpSection = `WordPress Post ID : ${wp.wpPostId}\nWordPress URL     : ${wp.wpUrl}\nWordPress Status  : ${wp.wpStatus}`;
} catch (e) { /* Test mode: node ini tidak jalan, biarkan default di atas */ }

const meta = validated.meta || {};
const today = new Date().toISOString();

const txt = `=== ARTIKEL GEO BUILDER — EXPORT ===
Run ID       : ${ctx.runId}
Run Mode     : ${ctx.publishMode}
Waktu Export : ${today}
Topik        : ${ctx.topic}

--- METADATA ---
Judul        : ${meta.title || ''}
Meta Desc.   : ${meta.metaDescription || ''}
Slug         : ${meta.slug || ''}
Kategori     : ${(meta.categories || []).join(', ')}
Tags         : ${(meta.tags || []).join(', ')}

--- SKOR GEO ---
Skor   : ${score.geoScore}/100
Verdict: ${score.verdict}

--- ${wpSection} ---

--- ARTIKEL (markdown) ---
${draft}

--- FAQ SCHEMA ---
${JSON.stringify(meta.faqSchema || [], null, 2)}
`;

return [{ json: { exportText: txt, exportFilename: `${ctx.runId}_${(ctx.publishMode || '').replace(/\s+/g, '-')}.txt` } }];
```

### 6. `Code: Build WordPress Payload` — status dinamis, bukan hardcode
Ganti baris `status: 'draft'` jadi:
```js
const ctx2 = $('Code: Build Run Context').first().json;
const status = ctx2.publishMode === 'Live Run - Publish' ? 'publish' : 'draft';
// ...
return [{ json: { _wpPayload: { ..., status, ... } } }];
```
Node ini HANYA dilalui di cabang Live Run (baik Draft maupun Publish) — di cabang Test node ini sama sekali tidak dieksekusi (lihat wiring §5), jadi tidak perlu ditambah pengecekan `publishMode==='Test'` di sini.

### 7. Node export ke file — WAJIB pakai `get_node_types`/`get_suggested_nodes` dulu, jangan tebak schema
Kebutuhan: ambil `exportText` (string) dari `Code: Build Export Payload`, tulis ke file `.txt` di disk lokal dengan nama dari `exportFilename`. Pola umum n8n: Code node (teks) → node **Convert to File** (ubah field teks jadi binary) → node **Read/Write Files from Disk** (mode write, path dari expression). **Sebelum membangun ini, WAJIB panggil `get_suggested_nodes`/`get_node_types` untuk 2 node itu** — jangan menebak nama parameter (proyek ini pernah rugi waktu karena menebak skema node, lihat `rules.md` §3.5 dan insiden Switch node `rules.values` vs `rules.rules` di CP007). Simpan ke folder `exports/` di working directory project (buat folder ini kalau belum ada — cek dulu apakah n8n punya izin tulis ke path itu, atau pakai path absolut yang pasti writable).

### 8. Perbaikan bug Research Mode (CP015) — ditemukan lewat audit, bukan dugaan

Dua node krusial belum pernah disesuaikan setelah CP015 mengubah Researcher dari "riset borongan 8-12 fakta" jadi "riset targeted, cuma cari klaim spesifik yang diminta Concept Brief". Akibatnya field Research Mode (Auto/Force Research) bisa **crash pipeline** atau **diam-diam kembali ke pola riset lemah yang sudah terbukti buruk**.

**Bug A — threshold validasi masih pakai angka lama, bikin crash di skenario paling normal.**

`Code: Validate Research Facts` sekarang:
```js
const minFacts = ctx.contentContext?.geo?.minFactsRequired || 5;      // dari config/content-context.json, angka lama
const minNumeric = ctx.contentContext?.geo?.minNumericFactsRequired || 3;
...
if (factCount < minFacts || numericCount < minNumeric) {
  throw new Error(`Validate Research: FAIL — facts ${factCount}/${minFacts}, numeric+source ${numericCount}/${minNumeric}`);
}
```
Tapi `Code: Build AI Research Body` sekarang cuma minta fakta untuk `empiricalClaims[]` spesifik — kalau Concept Brief cuma menandai 1-2 klaim (skenario PALING UMUM, karena memang itu tujuan targeted research), hasilnya wajar 1-3 fakta. Threshold 5 fakta/3 numerik akan REJECT hasil yang valid ini dan men-throw error, menghentikan seluruh pipeline. Ini bukan edge case langka — ini kemungkinan besar terjadi di kebanyakan run Auto/Force Research.

**Fix**: ganti threshold jadi dinamis, mengikuti jumlah `empiricalClaims` yang benar-benar diminta, bukan angka tetap:
```js
const d = $input.first().json;
const ctx = $('Code: Build Run Context').first().json;
const researchRan = d.researchRan !== false;

if (!researchRan) {
  const cb = d.conceptBrief || {};
  const seeds = (cb.sectionSeeds || []).length;
  const thinkers = (cb.thinkers || []).length;
  if (seeds < 5 || thinkers < 1) throw new Error(`Concept-only validation FAIL: sectionSeeds ${seeds}/5, thinkers ${thinkers}/1`);
  return [{ json: { ...d, _validated: true } }];
}

// AMBIL jumlah klaim yang memang diminta Concept Brief — bukan angka tetap dari config lama.
const brief = $('Code: Parse Concept Brief').first().json || {};
const claimsRequested = Math.max(1, (brief.empiricalClaims || []).length);
const minFacts = claimsRequested;           // minimal 1 fakta per klaim yang diminta
const minNumeric = Math.min(claimsRequested, 1); // minimal 1 fakta numerik kalau memang ada klaim yang diminta

const numericFacts = (d.facts || []).filter(f => f.hasNumber && f.sourceUrl);
const factCount = (d.facts || []).length;
const numericCount = numericFacts.length;
if (factCount < minFacts || numericCount < minNumeric) {
  throw new Error(`Validate Research: FAIL — facts ${factCount}/${minFacts}, numeric+source ${numericCount}/${minNumeric}`);
}
return [{ json: { ...d, _validated: true, _factCount: factCount, _numericCount: numericCount } }];
```
Catatan: kode di atas kerangka awal, bukan final — worker boleh menyesuaikan angka persisnya, tapi PRINSIPNYA harus dipegang: threshold validasi HARUS proporsional ke seberapa banyak yang memang diminta, bukan angka tetap peninggalan desain lama.

**Bug B — Force Research pada topik konseptual diam-diam jatuh ke pola riset broad yang sudah terbukti buruk.**

`Code: Build AI Research Body` sekarang, kalau `empiricalClaims` kosong (Concept Brief bilang topik ini murni konseptual, tidak butuh verifikasi apa pun) TAPI user tetap pilih Force Research:
```js
${empiricalClaims.length > 0 ? empiricalClaims.map(...).join('\n') : (d.topic + ' (tidak ada klaim spesifik, cari data numerik yang paling relevan)')}
...
Kumpulkan ${empiricalClaims.length > 0 ? empiricalClaims.length + '-12' : '8-12'} fakta, minimal 3 di antaranya mengandung angka
```
Fallback ini **persis** pola riset umum/broad lama yang CP015 dibangun untuk dihindari — dan kita punya bukti empiris (test 7+ model, dicatat di riset CP015) bahwa riset broad berbahasa Indonesia konsisten menghasilkan sumber blog SEO lemah (Kompasiana, RRI, dst), bukan akademik. Force Research pada topik konseptual diam-diam membatalkan seluruh tujuan CP015.

**Fix**: kalau `empiricalClaims` kosong TAPI mode Force Research, JANGAN fallback ke pencarian generik 8-12 fakta. Bangun query dari `sectionSeeds`/`coreQuestion` Concept Brief supaya tetap terarah, dan turunkan ekspektasi jumlah fakta (mis. 3-5, bukan 8-12) karena topiknya memang sudah dinilai minim kebutuhan data numerik oleh Concept Brief:
```js
const fallbackSeeds = (brief.sectionSeeds || []).filter(s => s.evidenceType === 'empirical').map(s => s.question);
const searchTargets = empiricalClaims.length > 0
  ? empiricalClaims.map((ec, i) => `${i+1}. ${ec.claim} — query: ${ec.searchQuery}`).join('\n')
  : (fallbackSeeds.length > 0
      ? fallbackSeeds.map((q, i) => `${i+1}. ${q}`).join('\n')
      : `${d.topic} (Concept Brief menilai topik ini murni konseptual — cari HANYA data pendukung yang benar-benar relevan, JANGAN memaksakan angka kalau tidak ada yang otoritatif)`);
// lalu pakai searchTargets menggantikan baris "Gunakan web search UNTUK KLAIM SPESIFIK BERIKUT: ..."
// dan turunkan target jumlah fakta jadi 3-5 di jalur fallback ini (bukan 8-12)
```

## Bagian penyederhanaan: Source Language → Article Language (CP014 direvisi, bukan dicabut)

1. **Form Trigger** — field `Source Language` (4 opsi lama) diganti isinya jadi field `Language` (2 opsi baru: Indonesia/English) — lihat skema lengkap di §0. Posisi field TETAP SAMA (tidak dipindah), cuma isi opsi & `fieldName`/label yang berubah.
2. **`Code: Set Form Run Context`** — `sourceLanguage: dropAuto(f.source_language)` diganti `language: String(f.language || 'Indonesia').trim()` (sudah tercermin di kode §2 di atas).
3. **`Code: Build Concept Brief Body`** — hapus TOTAL blok lama (`sl`/`slInst`, soal prioritas sumber riset) — **TIDAK diganti apa pun**, node ini sengaja tetap bahasa-agnostik. Pastikan `${slInst}` juga dihapus dari template `prompt`.
4. **`Code: Build AI Research Body`** (targeted research, CP015) — sama, hapus TOTAL blok `sourceLanguage`/`sourceLanguageInstruction` dan `${sourceLanguageInstruction}` dari template `prompt`. Node ini juga sengaja tetap bahasa-agnostik.
5. **`Code: Build AI Outline Body`, `Code: Build AI Writing Planner Body`, `Code: Build AI Writer Body`, `Code: Build AI Schema Body`** — keempatnya TAMBAH instruksi bahasa baru (lihat §0 untuk teks aturan lengkapnya). Untuk Writer Body khususnya: system prompt sekarang hardcode `"Kamu adalah penulis artikel GEO. Tulis dalam Bahasa Indonesia."` — ganti jadi dinamis mengikuti `ctx.language`.
6. **`ai_docs/index.md`** — update baris CP 014: status jadi `revised (disederhanakan di CP017 — dari "kontrol sumber riset" jadi "kontrol bahasa artikel", scope riset tetap bahasa-agnostik)`. JANGAN hapus barisnya (riwayat kerja tetap tercatat, konsisten dengan gaya index.md yang sudah ada — lihat bagaimana CP 010/011 tetap dicatat sebagai revision, bukan dihapus).

## Yang HARUS diverifikasi (jangan asumsi)
1. **Wiring `If - Is Test Mode?`** — verifikasi via eksekusi nyata mana port yang benar-benar true/false (lihat peringatan §4).
2. **Test mode benar-benar cepat** — ukur durasi total end-to-end run dengan `Run Mode = Test`, bandingkan dengan durasi run normal (target: signifikan lebih cepat, karena skip web search + target kata kecil).
3. **Test mode benar-benar tidak menyentuh WordPress** — cek tidak ada post baru muncul di WordPress setelah run Test.
4. **Live Run - Publish benar-benar set status publish** — cek response `HTTP: WordPress Create Post` menunjukkan post live, bukan draft (lalu, kalau ini cuma uji coba, cabut/unpublish manual dari WordPress setelah tes supaya tidak ada artikel asal-asalan nongol publik).
5. **Live Run - Draft (default) perilakunya SAMA seperti sebelum CP017** — tidak ada regresi ke jalur yang sudah teruji sebelumnya (execution 936 dkk).
6. **File export benar-benar tertulis ke disk** dengan isi yang benar, di ketiga mode (Test, Live Run - Draft, Live Run - Publish) — buka filenya, pastikan draft/metadata/skor terbaca jelas.
7. Pastikan sisa logic Source Language LAMA (soal sumber riset — `sl`, `slInst`, `sourceLanguageInstruction`) benar-benar hilang dari `Concept Brief Body`/`Research Body` — grep ulang setelah selesai. Tapi pastikan field BARU `ctx.language` memang dipakai di 4 node yang seharusnya (Outline/Writing Planner/Writer/Schema).
8. **Test paling penting**: jalankan 1x dengan `Language = English`, cek manual draft hasilnya — pastikan istilah seperti "akrasia"/nama teori TIDAK diterjemahkan jadi kata aneh, dan kalau ada kutipan verbatim, kutipan itu tetap dalam bahasa/kata asli sumbernya (bukan ikut diterjemahkan ke Inggris kalau sumber aslinya bahasa lain).
9. Cek metadata Schema (title/slug/meta description) konsisten bahasa dengan draft artikelnya — jangan sampai draft berbahasa Inggris tapi title/meta masih auto-generate dalam Bahasa Indonesia.

## Definition of done
- [ ] Field `Language` (Indonesia/English, default Indonesia) menggantikan isi field `Source Language` di posisi yang sama
- [ ] Field `Run Mode` baru ditambahkan (3 opsi, default `Live Run - Draft`)
- [ ] `Code: Set Form Run Context` propagasi `language` dan `publishMode`
- [ ] `Code: Build Run Context` override `wordCountTarget`/`researchMode` saat `publishMode === 'Test'`
- [ ] `If - Is Test Mode?` dibuat, wiring terverifikasi lewat eksekusi nyata (bukan asumsi port)
- [ ] `Code: Build WordPress Payload` status dinamis sesuai `publishMode`
- [ ] `Code: Build Export Payload` + node convert-to-file + node write-to-disk jalan di ketiga mode
- [ ] `Code: Build Concept Brief Body` & `Code: Build AI Research Body` TETAP bahasa-agnostik (tidak baca `ctx.language` sama sekali)
- [ ] `Code: Build AI Outline Body`, `Writing Planner Body`, `Writer Body`, `Schema Body` — keempatnya baca `ctx.language` dan menerapkan aturan "jangan terjemahkan istilah teknis/kutipan verbatim/nama baku" (lihat §0)
- [ ] Minimal 4 eksekusi live: 1 per Run Mode (3x) + 1 khusus test `Language = English` untuk cek native-term preservation — dicatat hasilnya di log
- [ ] `Code: Validate Research Facts` threshold dinamis (bukan angka tetap 5/3) — diverifikasi dengan run yang Concept Brief-nya cuma minta 1-2 empiricalClaims, TIDAK boleh crash
- [ ] `Code: Build AI Research Body` — fallback Force Research pada topik konseptual tidak lagi minta 8-12 fakta generik, dites 1x dengan topik yang Concept Brief-nya menghasilkan `empiricalClaims: []` + Research Mode = Force Research
- [ ] `ai_docs/index.md` diperbarui (baris 017 baru + baris 014 ditandai revised)
