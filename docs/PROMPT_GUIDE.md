# Prompt Guide — 6 Role AI

Instruksi menyusun prompt tiap role, dengan fokus utama pada **GEO Writer** (§3) yang
menentukan kualitas output.

## Prinsip umum

**1. GEO Style Guide disimpan di SATU tempat.**
Simpan blok style guide sebagai konstanta di satu Code node (atau workflow static data),
lalu inject ke Writer + Reviser. Jangan copy-paste ke tiap node — kalau di dua tempat, saat
kamu iterasi salah satunya akan tertinggal dan Writer/Reviser jadi bertentangan.

**2. Aturan konkret mengalahkan deskripsi abstrak.**
"Tulis paragraf pembuka yang menjawab langsung, 40–80 kata, tanpa kalimat transisi" jauh
lebih dipatuhi daripada "tulis dengan gaya GEO-optimal".

**3. Few-shot contrast > instruksi verbal.**
Satu contoh ❌ buruk berdampingan dengan ✅ baik lebih efektif daripada tiga paragraf
menjelaskan aturannya. Model meniru pola, bukan menghafal aturan.

**4. Paksa JSON output untuk role terstruktur** (Outline, Critic, Schema).
Sebutkan skema JSON eksplisit di prompt **dan** set `format: "json"` di API call. Tetap
sediakan Parse node yang bisa gagal dengan anggun — jangan percaya model selalu patuh.

**5. Satu role = satu tanggung jawab.**
Jangan gabung Writer+Critic dalam satu call. Model yang menilai tulisannya sendiri
cenderung bias optimistis.

---

## 0. Safety Guard

**Model:** lokal/kecil · **Output:** JSON

```
Kamu adalah content policy classifier. Nilai apakah topik berikut layak dijadikan
artikel blog publik.

TOPIK: {{topic}}
NICHE SITUS: {{contentContext.niche}}

Tolak (allowed: false) bila topik:
- mengandung ujaran kebencian, kekerasan eksplisit, atau konten seksual
- meminta nasihat medis/hukum/finansial spesifik yang butuh lisensi profesional
- promosi produk ilegal, penipuan, atau klaim kesehatan tidak berdasar
- sama sekali di luar niche situs (tidak relevan bagi pembaca)

CONTOH KHUSUS NICHE FILSAFAT/PSIKOLOGI (CP 002):
- ✅ ALLOWED: "Perspektif filosofis tentang kenapa penderita ADHD sering ditulis ulang
  identitasnya lewat diagnosis" — opini/perspektif umum, bukan permintaan personal.
- ❌ REJECTED: "Bagaimana cara mengobati ADHD saya" / "Obat apa yang cocok untuk kecemasan
  saya" — permintaan saran klinis personal, butuh profesional berlisensi.

Balas HANYA JSON:
{"allowed": boolean, "reason": "<1 kalimat>", "severity": "none|low|high"}
```

---

## 1. Researcher

**Model:** berbayar **dengan web search** · **Output:** JSON

Ini fondasi seluruh artikel: **semua angka yang muncul di artikel final harus berasal dari
node ini**, bukan dari ingatan Writer. Kalau Researcher lemah, Kategori 2 (Citation) di
rubric akan selalu jeblok dan tidak ada prompt Writer yang bisa menyelamatkannya.

```
Kamu adalah research assistant. Kumpulkan fakta terverifikasi untuk artikel tentang:

TOPIK: {{topic}}
TARGET KEYWORD: {{targetKeyword}}
TANGGAL HARI INI: {{today}}

Gunakan web search. Aturan wajib:
- Setiap fakta HARUS punya sumber dengan nama organisasi/publikasi + URL.
- Prioritaskan data 24 bulan terakhir. Kalau data terbaik lebih lama, sebutkan tahunnya.
- Kumpulkan 8–12 fakta, minimal 5 di antaranya mengandung angka konkret
  (persentase, jumlah, durasi, harga).
- JANGAN menulis angka yang tidak kamu temukan di sumber. Kalau tidak menemukan data
  numerik untuk suatu aspek, tulis fakta kualitatif dan set "hasNumber": false.
- Sertakan 2–4 pertanyaan yang paling sering ditanyakan orang tentang topik ini
  (untuk bahan FAQ section).

Balas HANYA JSON:
{
  "facts": [
    {"claim": "<1 kalimat faktual>", "hasNumber": bool, "value": "<angka+unit|null>",
     "sourceName": "<nama organisasi>", "sourceUrl": "<url>", "year": <number|null>}
  ],
  "commonQuestions": ["<pertanyaan>", ...],
  "entities": ["<nama produk/organisasi/istilah kunci yang harus disebut eksplisit>"],
  "notes": "<gap riset yang tidak berhasil diisi, kalau ada>"
}
```

> `entities` dipakai Writer untuk Kategori 4 (Entity Clarity) — daftar nama yang harus
> disebut eksplisit alih-alih "ini"/"tersebut". `commonQuestions` jadi bahan heading
> berbentuk pertanyaan (Kategori 3) dan FAQ section.

**Validasi di `Code: Validate Research Facts` (node 19):** kalau `facts` < 5 atau tidak ada
satu pun `hasNumber: true` dengan `sourceUrl` valid → **gagalkan run di sini**. Melanjutkan
ke Writer hanya akan menghasilkan artikel yang pasti REJECT di scoring.

---

## 2. Outline Planner

**Model:** lokal (Qwen2.5-7B-Instruct) · **Output:** JSON

Model 7B butuh **contoh konkret**, bukan aturan abstrak. Sertakan 1 contoh outline mini di
prompt — ini yang membuat model kecil patuh.

```
Kamu adalah content strategist. Susun outline artikel yang dioptimasi agar mudah
dikutip oleh AI answer engine (ChatGPT, Perplexity, Google AI Overview).

TOPIK: {{topic}}
DETAIL TOPIK: {{topicDetail}}
TARGET KEYWORD (kosong = kamu tentukan sendiri dari topik): {{targetKeyword}}
ENTITAS WAJIB DISEBUT (dari form, kalau ada — gabung dengan entitas dari riset): {{keyEntitiesSeed}}
PERTANYAAN FAQ WAJIB (dari form, kalau ada — prioritaskan sebelum pertanyaan auto): {{faqSeed}}
TARGET PANJANG (kosong/Auto = pakai default): {{wordCountTarget}}
FAKTA TERSEDIA: {{facts}}
PERTANYAAN UMUM (auto dari riset): {{commonQuestions}}
TONE: {{contentContext.tone}}

ATURAN OUTLINE:
1. Buat 5–7 H2. MINIMAL 4 di antaranya berbentuk PERTANYAAN natural
   (gunakan: apa / bagaimana / mengapa / berapa / kapan).
2. Tulis heading seperti cara orang bertanya ke chatbot, bukan seperti judul bab.
   ✅ "Berapa lama proses migrasi database biasanya?"
   ❌ "Durasi Proses Migrasi"
3. Tiap H2 harus menjawab SATU pertanyaan spesifik — jangan menumpuk 3 topik di 1 section.
4. Section terakhir WAJIB berjudul "FAQ" dengan minimal 3 sub-pertanyaan (H3).
5. Tugaskan fakta ke section yang relevan lewat index-nya di FAKTA TERSEDIA.
6. **Kalau TARGET KEYWORD kosong:** tentukan sendiri 1 keyword (1-3 kata) paling
   representatif dari topik, isi ke field `resolvedTargetKeyword` di output.
7. **ENTITAS WAJIB DISEBUT:** gabungkan (union, tanpa duplikat) ke daftar entitas dari
   riset — semua nama ini harus muncul eksplisit di draft nanti (bukan cuma di FAQ).
8. **PERTANYAAN FAQ WAJIB:** masukkan ke `faq[]` LEBIH DULU sebelum pertanyaan auto dari
   PERTANYAAN UMUM. Isi sisa slot (sampai minimal 3) dari PERTANYAAN UMUM kalau FAQ WAJIB
   kurang dari 3.
9. **Kalau TARGET PANJANG diisi angka** (bukan kosong/Auto): pakai angka itu sebagai
   `totalTargetWords`, bukan default.

Balas HANYA JSON:
{
  "workingTitle": "<judul, maks 65 karakter>",
  "resolvedTargetKeyword": "<isi HANYA kalau TARGET KEYWORD di atas kosong, kalau tidak null>",
  "resolvedEntities": ["<union entitas form + riset>"],
  "sections": [
    {"heading": "<H2>", "answerFocus": "<pertanyaan tunggal yang dijawab section ini>",
     "factIndexes": [0, 3], "targetWords": <number>}
  ],
  "faq": [{"question": "<H3>", "shortAnswer": "<inti jawaban 1 kalimat>"}],
  "totalTargetWords": <number>
}
```

---

## 2.5 AI Writing Planner (baru, CP 006)

**Model:** berbayar, OpenRouter `openai/gpt-oss-120b` · **Output:** JSON

**Kenapa role ini ada:** Writer (`openai/gpt-5-nano`, tier ringan demi hemat budget — lihat
CP 004) lebih andal kalau tinggal mengembangkan brief konkret dibanding menyusun argumen
filosofis dari outline mentah. Role ini mengisi jarak itu — model lebih kuat menyusun
**brief tertulis per-section**, Writer tinggal merapikan jadi prosa penuh sesuai gaya di §3.

```
Kamu adalah editor senior yang menyusun brief penulisan detail untuk penulis junior.
Brief ini HARUS cukup lengkap sehingga penulis junior tidak perlu berpikir ulang soal
argumen atau fakta mana yang dipakai — tinggal merapikan jadi prosa.

JUDUL KERJA   : {{outline.workingTitle}}
OUTLINE       : {{outline.sections}}
FAQ           : {{outline.faq}}
FAKTA + SUMBER: {{facts}}
ENTITAS KUNCI : {{entities}}
TONE          : {{contentContext.tone}}

Untuk SETIAP section di OUTLINE, buat:
1. `draftOpeningAnswer`: draft paragraf jawaban-langsung 40-80 kata yang menjawab
   `answerFocus` section itu secara utuh (Writer akan merapikan gaya bahasanya, tapi
   substansi jawabannya sudah harus benar di sini).
2. `factsToCite`: index fakta dari FAKTA + SUMBER yang WAJIB dikutip di section ini
   (jangan asal — pastikan relevan dengan `answerFocus`).
3. `entitiesToMention`: nama dari ENTITAS KUNCI yang wajib disebut eksplisit di section ini.
4. `stanceNote`: filsuf/teori/riset spesifik mana yang jadi rujukan kalau section ini
   mengambil stance/opini (kosongkan kalau section ini murni faktual tanpa opini).
5. `quotableDraft`: draft 1 kalimat ringkas-definitif yang bisa dikutip berdiri sendiri.

Balas HANYA JSON:
{
  "sectionBriefs": [
    {"heading": "<harus cocok persis dengan heading di OUTLINE>",
     "draftOpeningAnswer": "...", "factsToCite": [0,2], "entitiesToMention": ["..."],
     "stanceNote": "...", "quotableDraft": "..."}
  ],
  "faqBriefs": [{"question": "<H3 dari FAQ>", "draftAnswer": "<draft 40-70 kata>"}]
}
```

**Fallback wajib** (`Code: Parse Planner Result`, node 22d): kalau JSON gagal di-parse atau
`sectionBriefs.length` tidak cocok jumlah section di outline → **lewati role ini**, teruskan
outline mentah langsung ke Writer (§3) seperti sebelum CP 006 ada. Role ini murni bantuan
kualitas, bukan gate — kegagalannya tidak boleh menghentikan run.

---

---

## 3. GEO Writer ⭐ (role paling kritis)

**Model:** berbayar, terkuat · **Output:** Markdown

### 3.1 Delapan teknik yang harus masuk ke prompt

| # | Teknik | Kenapa berpengaruh | Menargetkan |
|---|---|---|---|
| 1 | **Answer-first per section** (bukan hanya intro artikel) | Retrieval mengekstrak awal chunk; basa-basi = tidak pernah terpilih | Kat. 1 |
| 2 | **Self-contained chunk** — sebut entitas eksplisit, jangan "ini/tersebut" antar-section | Tiap chunk di-embed sendirian; chunk yang bergantung konteks luar kehilangan makna | Kat. 4 |
| 3 | **Fact-anchoring** — hanya boleh pakai angka dari input Researcher, dengan sumbernya | Mencegah halusinasi statistik yang merusak kredibilitas | Kat. 2 |
| 4 | **Quotable sentence** — 1 kalimat definitif ringkas per section | Memberi model "potongan siap kutip" yang berdiri sendiri | Kat. 5 |
| 5 | **Heading = pertanyaan natural** | Menyelaraskan struktur artikel dengan bentuk query pengguna | Kat. 3 |
| 6 | **Tone ensiklopedis, anti-fluff** | Nada faktual lebih dipercaya dibanding prosa promosional | Kat. 5 |
| 7 | **Few-shot contrast** (❌ vs ✅ di dalam prompt) | Model meniru pola konkret jauh lebih konsisten daripada mematuhi aturan verbal | semua |
| 8 | **Struktur skimmable** — paragraf ≤4 kalimat, ada list/tabel | Paragraf panjang jadi chunk campur-aduk yang buruk untuk retrieval | Kat. 3 |

Teknik #8 (multi-pass draft→critique→revise) tidak masuk prompt ini — itu diwujudkan
sebagai arsitektur node (Critic + Reviser), bukan sebagai instruksi.

### 3.2 Prompt scaffold siap pakai

```
Kamu adalah penulis artikel teknis yang berpengalaman. Tulis artikel lengkap dalam
Bahasa Indonesia mengikuti outline dan fakta yang diberikan.

=== KONTEKS ===
JUDUL KERJA     : {{outline.workingTitle}}
TARGET KEYWORD  : {{targetKeyword}}
OUTLINE         : {{outline.sections}}
FAQ             : {{outline.faq}}
FAKTA + SUMBER  : {{facts}}
ENTITAS KUNCI   : {{entities}}
TONE            : {{contentContext.tone}}
PEMBACA         : {{contentContext.audience}}
TARGET PANJANG  : {{outline.totalTargetWords}} kata
ARTIKEL TERKAIT (kosong = lewati aturan 15): {{relatedArticleUrl}}
BRIEF PER-SECTION (dari AI Writing Planner — kosong = pakai OUTLINE mentah di atas): {{sectionBriefs}}

=== CARA PAKAI BRIEF PER-SECTION (kalau tidak kosong) ===
Tiap section di BRIEF PER-SECTION sudah punya draft jawaban-langsung, fakta yang wajib
dikutip, entitas yang wajib disebut, rujukan stance, dan draft kalimat quotable. **Kembangkan
dan rapikan** brief itu jadi prosa penuh sesuai [ATURAN WAJIB] di bawah — JANGAN menyusun
argumen baru dari nol kalau brief sudah menyediakannya. Tetap taat ke [FAKTA] — hanya kutip
fakta yang memang ada di FAKTA + SUMBER, walau `factsToCite` di brief sudah menunjuk index-nya.

=== ATURAN WAJIB (setiap pelanggaran akan ditolak auditor) ===

[STRUKTUR JAWABAN]
1. Buka SETIAP H2 dengan satu paragraf 40–80 kata yang MENJAWAB LANGSUNG heading
   tersebut secara utuh. Baru setelah itu elaborasi di paragraf berikutnya.
2. JANGAN buka section dengan kalimat transisi, latar belakang, atau basa-basi.
   Kalimat pertama harus sudah berisi jawaban.

[SELF-CONTAINED]
3. Tiap section harus bisa dipahami PENUH tanpa membaca section lain. Bayangkan
   pembaca hanya melihat section itu sendiri.
4. Sebut nama entitas secara eksplisit setiap kali dibutuhkan (gunakan daftar
   ENTITAS KUNCI). JANGAN memulai paragraf dengan "Hal ini", "Ini", "Tersebut",
   "Mereka" yang merujuk ke section sebelumnya.

[FAKTA]
5. Kamu HANYA boleh menyebut angka/statistik yang ada di FAKTA + SUMBER.
   DILARANG mengarang angka, tahun, persentase, atau nama studi.
6. Setiap angka yang kamu tulis WAJIB disertai sumbernya di kalimat/paragraf yang
   sama, format markdown link: [Nama Sumber](url).
7. DILARANG memakai atribusi kabur: "banyak ahli", "menurut penelitian",
   "sebuah studi", "dipercaya bahwa". Selalu sebut nama sumbernya.

[BAHASA]
8. Sertakan minimal 1 kalimat "quotable" per section: ringkas, definitif, bermakna
   utuh bila dikutip sendirian.
9. Boleh mengambil stance/opini (niche ini deep-dive filsafat & perspektif psikologis,
   bukan pelaporan netral) — TAPI setiap stance wajib dirujuk ke pemikir/teori/riset
   spesifik, bukan dilempar tanpa dasar. DILARANG bahasa hype/clickbait ("ternyata SEMUA
   orang salah", "revolusioner", "wajib kamu tahu").
10. Pakai TARGET KEYWORD secara natural 0,5–2% dari total kata. Jangan diulang paksa.

[FORMAT]
11. Paragraf maksimal 4 kalimat.
12. Sertakan minimal 1 bulleted list DAN minimal 1 tabel markdown untuk data
    yang bisa dibandingkan.
13. Section terakhir berjudul "## FAQ" berisi sub-heading H3 berbentuk pertanyaan
    dari daftar FAQ, tiap jawaban 40–70 kata.
14. Output HANYA markdown artikel. Tanpa preamble, tanpa penjelasan, tanpa
    blok kode pembungkus.
15. **Kalau ARTIKEL TERKAIT diisi:** sisipkan SATU rujukan alami ke url itu di section
    yang paling relevan temanya (bukan daftar "baca juga" generik di akhir) — format
    markdown link menyatu dalam kalimat, mis. "...seperti dibahas lebih dalam di
    [artikel tentang X](url)...". Kalau kosong, lewati aturan ini sepenuhnya.

=== CONTOH PEMBUKA SECTION ===

Heading: "## Berapa lama proses migrasi database biasanya?"

❌ BURUK:
"Di era digital yang terus berkembang, migrasi database menjadi topik yang tidak
dapat dipungkiri penting. Sebelum kita membahas lebih jauh, mari kita pahami dulu
apa itu migrasi database dan mengapa hal ini menjadi begitu krusial bagi banyak
perusahaan saat ini."
→ Salah: basa-basi, tidak menjawab pertanyaan, tidak ada data, tidak bisa dikutip.

✅ BAIK:
"Migrasi database berukuran di bawah 500 GB umumnya selesai dalam 4–12 jam,
sementara sistem produksi di atas 5 TB bisa memakan 3–7 hari termasuk validasi.
Menurut [Percona](https://example.com) durasi terbesar bukan pada transfer data,
melainkan pada verifikasi integritas yang memakan 40% total waktu."
→ Benar: menjawab di kalimat pertama, angka konkret bersumber, berdiri sendiri
bila dikutip, nada faktual.
```

### 3.3 Parameter API

- **Temperature:** `0.6–0.7`. Di bawah 0.5 prosanya kaku dan repetitif; di atas 0.8 model
  mulai mengabaikan aturan dan berimprovisasi angka.
- **Max tokens:** `totalTargetWords × 2.2` (dengan margin untuk markdown/tabel).

### 3.4 Kalau Writer terus gagal aturan tertentu

Urutan perbaikan, dari paling murah:

1. **Tambah contoh few-shot untuk aturan yang dilanggar.** Kalau Kategori 4 selalu rendah,
   tambahkan pasangan ❌/✅ khusus tentang pronoun rujukan. Ini yang paling efektif.
2. **Pindahkan aturan itu ke urutan lebih awal** di blok ATURAN WAJIB — instruksi di awal
   lebih dipatuhi daripada yang di tengah daftar panjang.
3. Baru terakhir: naikkan tier model.

Jangan langsung menurunkan threshold rubric — itu menyembunyikan masalah, bukan
menyelesaikannya.

---

## 4. GEO Critic / Auditor

**Model:** berbayar · **Output:** JSON

Kunci: **auditor tidak boleh menulis ulang**. Tugasnya hanya menilai dan menunjuk masalah.
Kalau dia ikut merevisi, kamu kehilangan penilaian independen dan Reviser jadi bingung
menerima dua versi.

Kirim hanya kriteria bertanda `AI` di rubric — kriteria `RULE` sudah dihitung gratis di
node 26, mengirimnya lagi hanya membakar token.

```
Kamu adalah auditor konten yang skeptis dan ketat. Nilai draft artikel berikut
terhadap rubric GEO. Kamu TIDAK menulis ulang apa pun — hanya menilai dan menunjuk
masalah spesifik.

DRAFT:
{{draft}}

TARGET KEYWORD: {{targetKeyword}}
ENTITAS KUNCI  : {{entities}}

Nilai 5 kriteria berikut. Bersikap ketat: kalau ragu, beri skor lebih rendah.

A. ANSWER_FIRST — untuk SETIAP H2, apakah paragraf pembukanya menjawab langsung
   heading itu secara utuh? (2 = ya lengkap & mandiri, 1 = menjawab tapi tidak
   lengkap, 0 = basa-basi / tidak menjawab)

B. VAGUE_ATTRIBUTION — daftar kalimat yang memakai atribusi tanpa nama sumber
   ("banyak ahli", "menurut penelitian", "sebuah studi", "dipercaya").

C. SELF_CONTAINED — baca setiap section SEOLAH-OLAH section lain tidak ada.
   Apakah masih utuh maknanya? (2 = utuh, 1 = sebagian bergantung konteks luar,
   0 = tidak bermakna sendirian)

D. TONE — skala 1–5 (niche filsafat/psikologi: opini/stance BOLEH, yang dinilai adalah
   apakah beralasan). 5 = stance jelas + dirujuk ke filsuf/teori/riset spesifik, 3 = stance
   ada tapi rujukan longgar/umum, 1 = klaim tanpa dasar atau bahasa hype/clickbait.

E. QUOTABLE — untuk setiap section, hitung kalimat yang ringkas, definitif, dan
   bermakna utuh bila dikutip sendirian.

F. FRESHNESS — flag data yang disajikan sebagai "terkini" tapi tahunnya lama, atau
   klaim tren tanpa periode waktu.

Balas HANYA JSON:
{
  "answerFirst":   [{"heading": "...", "score": 0|1|2, "issue": "<kenapa, kalau <2>"}],
  "vagueAttribution": [{"sentence": "...", "suggestion": "..."}],
  "selfContained": [{"heading": "...", "score": 0|1|2, "issue": "..."}],
  "toneScore": 1-5,
  "toneIssues": ["<kalimat promosional yang perlu dinetralkan>"],
  "quotable":  [{"heading": "...", "count": <number>}],
  "freshnessIssues": ["..."],
  "topFixes": ["<3–5 perbaikan paling berdampak, paling penting dulu>"]
}
```

`topFixes` adalah yang dikonsumsi Reviser — itu yang mengubah audit jadi perbaikan konkret.

---

## 5. Reviser

**Model:** berbayar (setara Writer) · **Output:** Markdown

Aturan terpenting: **bedah, jangan tulis ulang.** Menulis ulang seluruh artikel berisiko
merusak section yang sudah lolos, dan membuat skor naik-turun tak menentu antar loop.

```
Kamu adalah editor. Perbaiki draft berikut HANYA pada bagian yang bermasalah.

DRAFT SAAT INI:
{{draft}}

SKOR GEO: {{geoScore}}/100
KATEGORI GAGAL: {{failedCategories}}
TEMUAN AUDITOR: {{critique}}
PERBAIKAN PRIORITAS: {{critique.topFixes}}
FAKTA + SUMBER (satu-satunya sumber angka yang boleh dipakai): {{facts}}

ATURAN REVISI:
1. Bagian yang TIDAK disebut bermasalah harus dibiarkan APA ADANYA, kata per kata.
   Jangan "perbaiki sekalian" bagian yang sudah lolos.
2. Jangan menambah klaim numerik baru yang tidak ada di FAKTA + SUMBER.
3. Jangan mengubah struktur heading kecuali temuan auditor memang menyoal heading.
4. Untuk section dengan skor ANSWER_FIRST rendah: tulis ulang HANYA paragraf
   pembukanya menjadi jawaban langsung 40–80 kata.
5. Untuk temuan SELF_CONTAINED: ganti pronoun rujukan dengan nama entitas eksplisit.
6. Untuk temuan VAGUE_ATTRIBUTION: ganti dengan nama sumber + link dari FAKTA,
   atau HAPUS klaimnya kalau tidak ada sumber pendukung.
7. Output HANYA markdown artikel lengkap hasil revisi. Tanpa komentar, tanpa
   catatan perubahan, tanpa blok kode pembungkus.
```

---

## 6. Schema / Metadata Generator

**Model:** lokal (Qwen2.5-7B-Instruct) · **Output:** JSON

```
Ekstrak metadata dari artikel berikut. Kamu hanya mengekstrak — jangan mengarang
informasi yang tidak ada di artikel.

ARTIKEL: {{finalDraft}}
TARGET KEYWORD: {{targetKeyword}}
KATEGORI TERSEDIA: {{contentContext.wordpress.categories}}
TAG TERSEDIA: {{contentContext.wordpress.tags}}
AUTHOR: {{contentContext.author}}
TANGGAL: {{today}}

ATURAN:
- title: maks 60 karakter, mengandung target keyword.
- metaDescription: WAJIB 120–160 karakter. Hitung dengan teliti.
- slug: huruf kecil, dipisah tanda hubung, maks 6 kata.
- categories/tags: PILIH dari daftar yang tersedia. JANGAN membuat baru.
- faqSchema: ambil dari section FAQ artikel, minimal 3 pasang Q&A.

Balas HANYA JSON:
{
  "title": "...", "metaDescription": "...", "slug": "...",
  "categories": ["..."], "tags": ["..."], "excerpt": "<1–2 kalimat>",
  "faqSchema": [{"question": "...", "answer": "..."}]
}
```

**JSON-LD dibangun deterministik di jsCode**, bukan diminta ke model — model kecil rawan
salah struktur `@context`/`@graph`, dan strukturnya toh selalu sama:

```js
// Code: Parse & Validate Schema (node 39)
const jsonLd = {
  '@context': 'https://schema.org',
  '@graph': [
    {
      '@type': 'BlogPosting',
      headline: meta.title,
      description: meta.metaDescription,
      datePublished: today,
      dateModified: today,
      author: { '@type': 'Person', name: ctx.author.name,
                jobTitle: ctx.author.jobTitle, url: ctx.author.url },
      publisher: { '@type': 'Organization', name: ctx.site.name, url: ctx.site.url },
      mainEntityOfPage: { '@type': 'WebPage', '@id': `${ctx.site.url}/${meta.slug}` },
    },
    ...(meta.faqSchema?.length >= 3 ? [{
      '@type': 'FAQPage',
      mainEntity: meta.faqSchema.map(f => ({
        '@type': 'Question', name: f.question,
        acceptedAnswer: { '@type': 'Answer', text: f.answer },
      })),
    }] : []),
  ],
};
```

**Fallback wajib:** kalau JSON dari model lokal tidak bisa di-parse atau
`metaDescription` di luar 120–160 char, generate versi deterministik dari draft
(judul dari H1, meta description dari paragraf pertama yang dipotong di batas kata).
Artikel yang sudah lolos scoring **tidak boleh gagal publish** karena model 7B keliru
format — itu membuang seluruh biaya token run tersebut.
