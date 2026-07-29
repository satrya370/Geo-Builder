# CP 015 — Concept Brief Architecture (ganti Researcher-first jadi Concept-first)

## Latar belakang & keputusan

Niche situs ini adalah deep-dive filsafat & perspektif psikologis — topiknya **timeless, konseptual, dan opini**, bukan pelaporan data terbaru. Arsitektur sekarang memaksa setiap artikel lewat web search penuh dulu, dan itu terbukti salah sasaran:

- Researcher memakan **108-275 detik dari total ~278 detik pipeline** — bottleneck terbesar.
- Dari 10-12 "fakta" yang dikembalikan, mayoritas adalah klaim konseptual yang **tidak butuh verifikasi web** ("Piers Steel bilang prokrastinasi bukan semata karena malas", "TMT punya 4 faktor", "Solomon & Rothblum 1984 mendefinisikan...").
- Sumber yang didapat untuk topik kanonik justru **lemah** (RRI.co.id, Kompasiana, Liputan6, BCA Karir). Menyitasi "menurut Aristoteles, akrasia adalah..." ke blog Kompasiana = **kredibilitas palsu**, nilai verifikasinya nol.
- Web search adalah alat yang tepat untuk data berubah, dan alat yang **buruk** untuk pengetahuan kanonik. Untuk isi *Nicomachean Ethics* Buku VII, pengetahuan parametrik model lebih akurat daripada blog acak.

Bukti kegagalan paling telak: di test terminal, Researcher (hy3-preview) menulis di `notes` bahwa "hasil pencarian tidak memuat informasi tentang Aristoteles atau akrasia" — tapi pipeline tetap jalan, Outline tetap membuat section tentang Aristoteles, dan Writer terpaksa mengisinya dari ingatan. **Di situlah kutipan Aristoteles palsu muncul di test gpt-5-nano.** Gap terdeteksi, dilaporkan, lalu diabaikan.

### Keputusan user (2026-07-27)
1. **Pemicu research: hybrid** — field form jadi kebijakan, Concept Brief menentukan di mode Auto.
2. **Scope saat menyala: terarah saja** — hanya `empiricalClaims` yang dideklarasikan brief, pakai `searchQuery`-nya.
3. **AI Safety Guard tetap dipisah**, tidak digabung ke Concept Brief.
4. **Rubrik GEO: turunkan bobot citation**, geser ke kualitas opini.

### Addendum — hasil riset 4 model AI independen (2026-07-27)
Sebelum implementasi, arsitektur ini diuji lewat riset mendalam ke 4 model AI berbeda (GPT, GLM, DeepSeek, Claude). Tiga dari empat (GPT/GLM/DeepSeek) konvergen merekomendasikan pipeline verifikasi berat (atomic claim ledger, retrieval router ke 5+ API, 3-way critic split, post-draft audit). Model ke-4 (Claude), diminta secara eksplisit menguji premis itu, menunjukkan bahwa proyek ini **sudah punya gerbang manusia** (artikel publish sebagai WordPress draft, bukan langsung live) — dan literatur yang dirujuk 3 sumber lain (FActScore, SelfCheckGPT, CoVe, dst.) mengevaluasi otomasi sebagai **pengganti** manusia, bukan sebagai lapisan tambahan di atas manusia yang sudah ada. Menerapkan bukti itu ke situasi kita adalah category error.

**Keputusan final**: arsitektur CP015 di dokumen ini (Concept Brief + targeted research) **tidak berubah** — dari awal sudah dirancang minimal, tidak memakai claim-ledger/retrieval-router yang diusulkan 3 sumber itu. Yang berubah adalah CP016 (grounding), lihat `016_citation-quote-verification_plan.md` — dirombak jadi jauh lebih kecil: hanya verifikasi kutipan + eksistensi sitasi, karena itu satu-satunya kelas kegagalan yang terbukti *tidak* tertangkap manusia yang teliti sekalipun (kutipan Aristoteles palsu, "DSM-5"/"ADHD" yang disisipkan tanpa ada di data — dua-duanya ditemukan di test kita sendiri, nol kegagalan ditemukan di kategori "salah paham konsep filsafat").

**Satu koreksi ke §9 di bawah**: node `Code: Build AI Writer Body` sekarang juga harus mengeluarkan `quotes[]` dan `citations[]` terstruktur bersamaan draft (bukan cuma markdown) — ini kebutuhan CP016, detail skema ada di plan CP016 §2.

## Arsitektur target

```
Form → Set Context → Load Content Context → Build Run Context
  → Build Guard Body → AI Safety Guard → Parse Guard → If - Topic Allowed?
      ├─[blocked]→ Respond - Topic Blocked
      └─[allowed]→ Code: Build Concept Brief Body
                   → HTTP: AI Concept Brief
                   → Code: Parse Concept Brief
                   → If - Need Empirical Data?
                       ├─[tidak]→ ──────────────────────────────┐
                       └─[ya]  → Code: Build Targeted Research Body
                                 → HTTP: AI Researcher (:online)  │
                                 → Code: Parse Research ──────────┤
                                                                  ↓
                                        Code: Normalize Research Payload
                                                                  ↓
                                        Code: Build AI Outline Body → (sisanya tidak berubah)
```

Node lama `Code: Build AI Research Body` diubah jadi `Code: Build Targeted Research Body` (prompt-nya diganti jadi terarah). `HTTP: AI Researcher` dipertahankan apa adanya (`tencent/hy3-preview:online` — sudah terbukti terbaik dari 7 model yang dites).

## Perubahan per komponen

### 1. Form Trigger — field baru "Research Mode"
Ikuti pola `Source Language` dari CP 014. Tempatkan setelah field `Source Language`.
```json
{
  "fieldLabel": "Research Mode",
  "fieldType": "dropdown",
  "requiredField": false,
  "fieldName": "research_mode",
  "fieldOptions": { "values": [
    { "option": "Auto", "value": "Auto" },
    { "option": "Concept Only", "value": "Concept Only" },
    { "option": "Force Research", "value": "Force Research" }
  ]}
}
```
Catatan: di sini "Auto" **bukan** nilai yang di-drop seperti field lain — Auto adalah mode bermakna (brief yang memutuskan). Jadi di `Code: Set Form Run Context` pakai `String(f.research_mode || 'Auto').trim()`, **bukan** helper `dropAuto`.

### 2. Node baru: `Code: Build Concept Brief Body`
Model rekomendasi: **`openai/gpt-5.4-nano` @ KobiLLM** (`https://api.koboillm.com/v1/chat/completions`) — dari test terminal: 19 detik, `reasoning_tokens: 0`, output terstruktur rapi, tidak fabrikasi. **Tanpa `:online`.**

Prompt harus meminta output JSON persis:
```json
{
  "coreQuestion": "<pertanyaan inti artikel>",
  "thinkers": [{"name":"...","position":"...","work":"<judul karya kalau yakin, null kalau ragu>"}],
  "keyConcepts": [{"term":"...","definition":"...","contrastWith":"<konsep yang sering tertukar|null>"}],
  "debates": [{"claim":"...","counterClaim":"..."}],
  "angle": "<sudut pandang khas artikel ini>",
  "sectionSeeds": [{"question":"...","evidenceType":"conceptual|empirical"}],
  "empiricalClaims": [{"claim":"...","whyNeeded":"...","searchQuery":"..."}],
  "needsResearch": true|false,
  "commonQuestions": ["..."]
}
```
Aturan wajib di prompt:
- `thinkers[].work` diisi HANYA kalau yakin; kalau ragu → `null`. Jangan mengarang judul karya.
- `empiricalClaims` HANYA untuk klaim yang benar-benar butuh angka/data kontemporer. Pengetahuan kanonik (isi teori, argumen filsuf, definisi konsep) **tidak boleh** masuk ke sini.
- `needsResearch` = `true` hanya kalau `empiricalClaims` tidak kosong.
- DILARANG menuliskan kutipan verbatim dari tokoh mana pun.

### 3. Node baru: `Code: Parse Concept Brief`
Pakai pola parsing robust yang sama seperti node Parse lain (ekstrak substring dari `{` pertama sampai `}` terakhir sebelum `JSON.parse` — lihat CP 011). Validasi minimal: `coreQuestion` ada, `sectionSeeds` array non-kosong. Kalau gagal parse → throw (jangan fail-open ke Writer tanpa brief).

### 4. Node baru: `If - Need Empirical Data?`
Kondisi gabungan mode form + deklarasi brief:
```js
// true kalau research harus jalan
const mode = ctx.researchMode || 'Auto';
mode === 'Force Research' || (mode === 'Auto' && brief.needsResearch === true && (brief.empiricalClaims||[]).length > 0)
// mode === 'Concept Only' → selalu false
```
**PERINGATAN**: urutan port If node TIDAK bisa diasumsikan true=port0/false=port1. Di CP 011 sudah ketemu kasus nyata kondisi `true` mendarat di **port 1**. Wiring wajib diverifikasi lewat eksekusi nyata, bukan asumsi.

### 5. `Code: Build AI Research Body` → jadi terarah
Ganti prompt: alih-alih "kumpulkan 8-12 fakta tentang topik", jadi "cari data untuk daftar klaim spesifik berikut" dengan menyuntikkan `empiricalClaims[].searchQuery`. Tetap pertahankan instruksi `sourceLanguage` dari CP 014 dan aturan anti-fabrikasi angka. Target fakta menyesuaikan jumlah `empiricalClaims`, bukan angka tetap 8-12.

### 6. Node baru: `Code: Normalize Research Payload`
Kedua cabang WAJIB keluar dengan bentuk JSON identik, kalau tidak `Code: Build AI Outline Body` pecah:
```js
{ facts: [...], commonQuestions: [...], entities: [...], conceptBrief: {...}, researchRan: true|false }
```
Di jalur concept-only: `facts: []`, `commonQuestions`/`entities` diambil dari brief. Kabar baik: prompt Writer sudah punya fallback `'(tidak ada fakta tersedia)'`, jadi kontrak lama tidak pecah.

### 7. `Code: Validate Research Facts` — jadi mode-aware
Config sekarang `minFactsRequired: 5`, `minNumericFactsRequired: 3` (di `config/content-context.json`). Di mode concept-only ini akan **memblokir semua eksekusi**. Threshold hanya boleh berlaku kalau `researchRan === true`. Di mode concept-only, ganti validasi jadi: `conceptBrief.sectionSeeds.length >= 5` dan `thinkers.length >= 1`.

### 8. `Code: Build AI Outline Body` & `Code: Build AI Writing Planner Body`
Keduanya harus ikut membaca `conceptBrief` — Outline memakai `sectionSeeds` + `angle` sebagai kerangka, Writing Planner memakai `thinkers`/`keyConcepts`/`debates` sebagai bahan rujukan stance saat `facts` kosong.

### 9. `Code: Build AI Writer Body` — perbaikan aturan (KRITIS di mode concept-only)
Risiko fabrikasi **naik** saat `facts: []`, jadi aturan harus ganti bentuk, bukan hilang:

- **Rule 9 diarahkan ulang**: "setiap stance wajib dirujuk ke pemikir/teori spesifik" sekarang menunjuk ke `conceptBrief.thinkers` dan `keyConcepts`, bukan ke array facts. Tanpa ini, Writer tidak punya rujukan sah di mode concept-only dan akan mengarang lagi.
- **Pisahkan mengutip vs merujuk ide** (aturan baru): tanda kutip (`"`) HANYA boleh dipakai kalau kalimat itu ada verbatim di FAKTA. Merujuk gagasan seseorang → parafrase tanpa tanda kutip, format "menurut pandangan X...". DILARANG menulis kalimat dalam tanda kutip lalu mengatribusikannya ke tokoh.
- **Closed-world rule untuk nama** (aturan baru): Writer hanya boleh menyebut studi/organisasi/manual diagnostik/tokoh yang ada di FAKTA atau `conceptBrief`. Di test kemarin deepseek menyelipkan "DSM-5" dan big-pickle menyelipkan "ADHD" — dua-duanya tidak ada di data yang diberikan.
- **Perbaiki rule 8**: ada bug nyata — draft deepseek menuliskan label scaffolding ke artikel (`*Kalimat quotable:* **"..."**`). Tegaskan kalimat quotable adalah kalimat penulis sendiri yang **menyatu dalam prosa**, bukan blok berlabel, dan bukan diklaim sebagai ucapan orang lain.
- **Rule 5-6** (angka wajib bersumber) tetap dipertahankan apa adanya — di mode concept-only dengan `facts: []`, efeknya jadi "dilarang menulis angka sama sekali", dan itu memang perilaku yang benar.

### 10. Rubrik GEO — rebalance
`Code: Compute GEO Score`, `GATE_WEIGHTS` sekarang:
```
answerFirst 20 | citation 20 | structure 15 | selfContained 15 | language 15 | freshness 5   (total 90)
```
Citation = 22% dari total. Artikel concept-only yang bagus kehilangan ~22 poin — eksekusi 936 yang skornya 85 akan jatuh ke ~63 = zona REVISE (60-79), lalu revise loop berusaha menambah sitasi yang tidak ada bahannya. **Rubrik lama secara aritmetis memaksa fabrikasi.**

Usulan bobot baru (**total tetap 90**, supaya `NORM = 100/90` dan sisa kode tidak perlu diubah):

| Kategori | rule | ai | total | sebelumnya |
|---|---|---|---|---|
| answerFirst | 5 | 20 | **25** | 20 |
| citation | 4 | 4 | **8** | 20 |
| structure | 15 | 0 | **15** | 15 |
| selfContained | 4 | 16 | **20** | 15 |
| language | 7 | 13 | **20** | 15 |
| freshness | 1 | 1 | **2** | 5 |
| | 36 | 54 | **90** | 90 |

Rasionalnya: yang bikin konten opini dikutip mesin AI adalah jawaban-langsung di awal section, section yang berdiri sendiri, dan perumusan yang jernih/khas — bukan jumlah link. Freshness turun karena topik filsafat memang timeless.

**PENTING — tiga tempat harus sinkron**, kalau tidak skornya jadi ngaco:
1. `GATE_WEIGHTS` di `Code: Compute GEO Score`
2. Nilai maksimum tiap fungsi di `Code: GEO Rule Checker` (mis. `scoreCitationRatio` sekarang mengembalikan maks 10, harus jadi maks 4) — kalau tidak diskalakan, kategori itu akan selalu mentok di `clamped`
3. Angka maksimum yang diminta di prompt `Code: Build AI Critic Body`

## Yang HARUS diverifikasi (jangan asumsi)
1. **Bandingkan head-to-head**: 1 artikel jalur concept-only vs output eksekusi 936 (jalur research penuh). Nilai mana yang lebih kuat sebagai tulisan, bukan cuma skor angkanya.
2. **Verifikasi anti-fabrikasi di mode concept-only** — ini mode paling berisiko. Cek manual draft: ada kutipan bertanda kutip yang diatribusikan ke tokoh? Ada angka tanpa sumber? Ada nama studi/organisasi di luar `conceptBrief`?
3. **Wiring If node diverifikasi lewat eksekusi nyata** (lihat peringatan port di §4).
4. **Uji ketiga mode**: Concept Only, Auto (topik konseptual → research mati), Auto (topik yang butuh data → research menyala), Force Research.
5. Pastikan skor artikel concept-only mendarat di zona PASS dengan rubrik baru — kalau masih jatuh ke REVISE, bobotnya belum benar.

## Definition of done
- [ ] Field "Research Mode" di form + propagasi di `Code: Set Form Run Context` (tanpa `dropAuto`)
- [ ] Node `Build Concept Brief Body` + `HTTP: AI Concept Brief` + `Parse Concept Brief` jalan
- [ ] `If - Need Empirical Data?` terwiring benar (**diverifikasi lewat eksekusi, bukan asumsi port**)
- [ ] `Build Targeted Research Body` memakai `empiricalClaims[].searchQuery`
- [ ] `Normalize Research Payload` menyatukan kedua cabang dengan bentuk identik
- [ ] `Validate Research Facts` mode-aware (tidak memblokir concept-only)
- [ ] Outline & Writing Planner membaca `conceptBrief`
- [ ] 4 perbaikan aturan Writer diterapkan (rule 9 redirect, kutip-vs-parafrase, closed-world nama, rule 8 anti-scaffolding)
- [ ] Rubrik direbalance di **ketiga** tempat yang sinkron
- [ ] 4 eksekusi live (3 mode + 1 head-to-head) dicatat hasilnya di log
- [ ] `ai_docs/index.md` diperbarui + log ditulis di `015_concept-brief-architecture_log.md`
