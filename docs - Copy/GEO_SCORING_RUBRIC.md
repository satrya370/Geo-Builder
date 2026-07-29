# GEO Scoring Rubric

Rubric untuk menilai seberapa "citable" sebuah artikel oleh AI answer engines. Dipakai di
dua tempat:

1. **`Code: GEO Rule Checker`** (node 26) — kriteria `RULE`, jsCode murni, **0 token**.
2. **`HTTP: AI GEO Critic`** (node 28) — kriteria `AI`, judgment bahasa.

Lalu digabung di **`Code: Compute GEO Score`** (node 30).

**Total bobot 100.** Rule-based menutup **45 poin**, AI menutup **55 poin**.

> ⚠️ **Catatan penting:** "GEO score" bukan metrik standar industri — tidak ada angka
> resmi dari OpenAI/Google/Perplexity. Rubric ini adalah **proxy heuristik** yang kami
> definisikan sendiri berdasarkan bagaimana sistem retrieval (RAG) memotong dan memilih
> konten. Skor 85 tidak menjamin artikel dikutip AI; ia berarti artikel itu memenuhi
> karakteristik struktural yang membuatnya *lebih mungkin* terekstrak. Perlakukan sebagai
> alat kontrol kualitas internal yang konsisten, bukan ukuran kebenaran absolut.

---

## Ringkasan bobot

| # | Kategori | Bobot | Rule | AI |
|---|---|---|---|---|
| 1 | Answer-First Structure | 20 | 5 | 15 |
| 2 | Citation & Fact Density | 20 | 10 | 10 |
| 3 | Struktur & Format | 15 | 15 | 0 |
| 4 | Self-Containment & Entity Clarity | 15 | 3 | 12 |
| 5 | Bahasa & Tone | 15 | 5 | 10 |
| 6 | Metadata & Schema Signals | 10 | 10 | 0 |
| 7 | Freshness Signal | 5 | 2 | 3 |
| | **Total** | **100** | **50** | **50** |

---

## Kategori 1 — Answer-First Structure (20)

Inti GEO. Answer engine sering mengekstrak 1–2 kalimat pertama sebuah section sebagai
jawaban. Section yang dibuka dengan basa-basi tidak akan pernah terpilih.

### 1a. Direct-answer paragraph per H2 (AI, 15)

Tiap H2 harus dibuka paragraf 40–80 kata yang menjawab heading-nya secara **mandiri**,
sebelum elaborasi.

Skor per section: `2` = jawaban lengkap & mandiri, `1` = menjawab tapi tidak lengkap /
butuh konteks, `0` = tidak menjawab (basa-basi, definisi berputar, atau langsung
elaborasi tanpa kesimpulan).

```
skor_1a = (Σ skor_section / (jumlah_section × 2)) × 15
```

### 1b. Bebas filler pembuka (RULE, 5)

```js
const FILLER_PATTERNS = [
  /\bdi\s+(era|zaman|dunia)\s+(yang\s+)?(serba\s+)?(digital|modern|terus\s+berkembang)/i,
  /\bseiring\s+(dengan\s+)?(perkembangan|kemajuan|berjalannya)/i,
  /\btidak\s+dapat\s+dipungkiri\b/i,
  /\bsemakin\s+(hari|berkembangnya)\b/i,
  /\bpada\s+artikel\s+(ini|kali\s+ini)\b/i,
  /\bmari\s+kita\s+(bahas|simak|lihat)\b/i,
  /\bsebelum\s+(kita\s+)?(masuk|membahas)\s+lebih\s+(jauh|dalam)/i,
  /\bdalam\s+dunia\s+yang\b/i,
];

// Cek HANYA kalimat pertama tiap section — filler di tengah body lebih dimaafkan
function scoreNoFiller(sections) {
  const flagged = sections.filter(s => {
    const firstSentence = (s.body.split(/(?<=[.!?])\s+/)[0] || '');
    return FILLER_PATTERNS.some(p => p.test(firstSentence));
  });
  const ratio = sections.length ? flagged.length / sections.length : 0;
  return { score: (1 - ratio) * 5, flaggedSections: flagged.map(s => s.heading) };
}
```

---

## Kategori 2 — Citation & Fact Density (20)

Konten tanpa sumber jelas cenderung tidak dikutip — dan lebih buruk lagi, statistik
halusinasi yang terpublikasi merusak kredibilitas situs.

### 2a. Rasio klaim numerik bersumber (RULE, 10)

```js
const NUMERIC_CLAIM = /(\b\d{1,3}(?:[.,]\d+)?\s*%|\b\d{4}\b|\b\d+(?:[.,]\d+)?\s*(juta|miliar|ribu|x|kali|jam|hari|bulan|tahun|detik|menit)\b)/gi;

// Sitasi dianggap ada bila di paragraf yang sama terdapat markdown link,
// atau pola atribusi bernama (bukan atribusi kabur).
const CITATION_MARKER = /\[([^\]]+)\]\((https?:\/\/[^)]+)\)/;
const NAMED_ATTRIBUTION = /\b(menurut|berdasarkan|laporan|survei|studi|riset|data)\s+(dari\s+)?[A-Z][\w&.\-]*/;

function scoreCitationRatio(paragraphs) {
  let withClaim = 0, cited = 0;
  const uncited = [];
  for (const p of paragraphs) {
    if (!NUMERIC_CLAIM.test(p)) { NUMERIC_CLAIM.lastIndex = 0; continue; }
    NUMERIC_CLAIM.lastIndex = 0;
    withClaim++;
    if (CITATION_MARKER.test(p) || NAMED_ATTRIBUTION.test(p)) cited++;
    else uncited.push(p.slice(0, 120));
  }
  if (withClaim === 0) return { score: 0, reason: 'tidak ada klaim numerik sama sekali', uncited };
  return { score: (cited / withClaim) * 10, ratio: `${cited}/${withClaim}`, uncited };
}
```

> **Skor 0 kalau tidak ada klaim numerik sama sekali** — itu disengaja. Artikel tanpa satu
> pun data konkret secara definisi tidak punya apa pun untuk dikutip AI.

### 2b. Tidak ada atribusi kabur (AI, 10)

Model men-flag kalimat dengan atribusi tanpa nama: "banyak ahli mengatakan", "menurut
penelitian" (penelitian apa?), "sebuah studi menunjukkan", "dipercaya bahwa".

```
skor_2b = max(0, (1 - jumlah_vague / max(1, jumlah_paragraf)) ) × 10
```

---

## Kategori 3 — Struktur & Format (15) — 100% RULE

```js
function scoreStructure(md, sections) {
  // 3a. Heading berbentuk pertanyaan (7)
  const QUESTION_WORD = /\b(apa|apakah|bagaimana|kenapa|mengapa|berapa|kapan|siapa|mana|manakah)\b/i;
  const qHeadings = sections.filter(s => s.heading.trim().endsWith('?') || QUESTION_WORD.test(s.heading));
  // target: minimal 60% heading berbentuk pertanyaan → skor penuh
  const qRatio = sections.length ? qHeadings.length / sections.length : 0;
  const s3a = Math.min(qRatio / 0.6, 1) * 7;

  // 3b. FAQ section di akhir (5)
  const tail = md.slice(-Math.floor(md.length * 0.3));
  const hasFaqHeading = /^#{2,3}\s*(faq|pertanyaan\s+(yang\s+)?(sering|umum))/im.test(tail);
  const qaPairs = (tail.match(/^#{3,4}\s*[^\n]*\?\s*$/gm) || []).length;
  const s3b = hasFaqHeading && qaPairs >= 3 ? 5 : hasFaqHeading ? 3 : 0;

  // 3c. Ada list / tabel untuk data terstruktur (3)
  const hasList  = /^\s*(?:[-*+]|\d+\.)\s+\S/m.test(md);
  const hasTable = /^\s*\|.+\|\s*$/m.test(md) && /^\s*\|[\s:|-]+\|\s*$/m.test(md);
  const s3c = (hasList ? 1.5 : 0) + (hasTable ? 1.5 : 0);

  return { score: s3a + s3b + s3c, detail: { s3a, s3b, s3c, qRatio, qaPairs } };
}
```

---

## Kategori 4 — Self-Containment & Entity Clarity (15)

**Ini kategori yang paling sering diabaikan, dan paling berpengaruh.** Retrieval memotong
artikel jadi chunk kecil lalu meng-embed masing-masing. Chunk yang bergantung pada section
sebelumnya kehilangan makna saat diambil sendirian — jadi tidak berguna sebagai jawaban.

### 4a. Section berdiri sendiri (AI, 12)

Model diminta membaca **satu section terisolasi** (tanpa section lain) lalu menilai:
apakah masih bisa dipahami? Entitas utama disebut eksplisit, atau merujuk ke sesuatu yang
tidak ada di chunk itu?

Skor per section: `2` = utuh mandiri, `1` = sebagian bergantung konteks luar, `0` = tidak
bermakna tanpa section lain.

```
skor_4a = (Σ skor_section / (jumlah_section × 2)) × 12
```

### 4b. Pronoun rujukan di awal paragraf (RULE, 3)

```js
const DANGLING_OPENER = /^(hal\s+ini|ini|itu|tersebut|hal\s+tersebut|mereka|dia|nya\b)/i;

function scoreEntityClarity(paragraphs) {
  const bad = paragraphs.filter(p => DANGLING_OPENER.test(p.trim()));
  const ratio = paragraphs.length ? bad.length / paragraphs.length : 0;
  // toleransi 10% — sedikit pronoun masih natural
  return { score: Math.max(0, 1 - Math.max(0, ratio - 0.10) / 0.30) * 3, danglingCount: bad.length };
}
```

---

## Kategori 5 — Bahasa & Tone (15)

### 5a. Reasoned/argued vs unsupported hype (AI, 7)

**Disesuaikan untuk niche filsafat/psikologi (CP 002)** — konten di sini kebanyakan opini
dan perspektif (deep-dive filsafat, opini fenomena psikologis), bukan pelaporan fakta netral
gaya ensiklopedia. Skor **bukan** soal "punya pendapat atau tidak", tapi soal apakah
pendapat itu **beralasan dan dirujuk** ke pemikiran/posisi filosofis atau psikologis
tertentu, versus klaim yang dilempar begitu saja tanpa dasar atau berbau clickbait/hype.

Model menilai skala 1–5:
- **5** = stance jelas, didukung rujukan eksplisit ke filsuf/teori/riset spesifik, bahasa
  tetap terukur.
- **3** = stance ada tapi rujukannya longgar/umum ("beberapa filsuf berpendapat...").
- **1** = klaim tanpa dasar sama sekali, atau bahasa hype/clickbait ("ternyata SEMUA orang
  salah soal ini!").

`skor = (nilai − 1) / 4 × 7`

### 5b. Kalimat quotable per section (AI, 3)

Kalimat ringkas, definitif, bermakna utuh tanpa konteks. Minimal 1 per section → penuh.

```
skor_5b = (jumlah_section_dengan_count>=1 / max(1, jumlah_section)) × 3
```

Dihitung dari array `quotable[]` output Critic (`[{heading, count}]`) — yang dinilai adalah
**cakupan** (berapa section yang punya minimal 1 kalimat quotable), bukan total kalimat.
Section dengan 5 kalimat quotable tidak menutupi section yang punya nol.

### 5c. Tidak ada keyword stuffing (RULE, 5)

```js
function scoreKeywordDensity(md, targetKeyword) {
  if (!targetKeyword) return { score: 5, skipped: true };
  const words = md.toLowerCase().match(/[\p{L}\p{N}']+/gu) || [];
  const kw = targetKeyword.toLowerCase().trim();
  const kwWordCount = (kw.match(/[\p{L}\p{N}']+/gu) || []).length || 1;
  const occurrences = (md.toLowerCase().match(
    new RegExp(kw.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'g')
  ) || []).length;
  const density = words.length ? (occurrences * kwWordCount) / words.length : 0;

  // ideal 0.5%–2%. Di bawahnya kurang fokus, di atas 3% = stuffing.
  let score;
  if (density >= 0.005 && density <= 0.02) score = 5;
  else if (density < 0.005) score = Math.max(0, (density / 0.005) * 3);   // maks 3 kalau terlalu tipis
  else if (density <= 0.03) score = 5 - ((density - 0.02) / 0.01) * 2;    // 5 → 3
  else score = Math.max(0, 3 - ((density - 0.03) / 0.02) * 3);            // 3 → 0
  return { score, density: (density * 100).toFixed(2) + '%', occurrences };
}
```

> Berbeda dari SEO klasik, **keyword stuffing justru merugikan di GEO** — bahasa yang tidak
> natural menurunkan kualitas embedding dan membuat model kurang percaya pada teksnya.

---

## Kategori 6 — Metadata & Schema Signals (10) — 100% RULE

```js
function scoreMetadata(schemaJson, metaDescription, slug) {
  let score = 0; const issues = [];

  // 6a. JSON-LD Article valid (5)
  try {
    const s = typeof schemaJson === 'string' ? JSON.parse(schemaJson) : schemaJson;
    const graph = s['@graph'] || [s];
    const article = graph.find(x => /Article|BlogPosting/i.test(x['@type'] || ''));
    if (!article) issues.push('tidak ada node Article/BlogPosting');
    else {
      const required = ['headline', 'datePublished', 'author'];
      const missing = required.filter(f => !article[f]);
      score += missing.length === 0 ? 3 : Math.max(0, 3 - missing.length);
      if (missing.length) issues.push('Article kurang field: ' + missing.join(', '));
    }
    // 6b. FAQPage schema (2)
    const faq = graph.find(x => /FAQPage/i.test(x['@type'] || ''));
    if (faq && Array.isArray(faq.mainEntity) && faq.mainEntity.length >= 3) score += 2;
    else issues.push('FAQPage schema tidak ada / <3 pertanyaan');
  } catch (e) { issues.push('JSON-LD tidak bisa di-parse: ' + e.message); }

  // 6c. Meta description 120–160 char (3)
  const len = (metaDescription || '').trim().length;
  if (len >= 120 && len <= 160) score += 3;
  else if (len >= 100 && len <= 175) score += 1.5;
  else issues.push(`meta description ${len} char (target 120–160)`);

  // 6d. Slug bersih (2)
  if (/^[a-z0-9]+(?:-[a-z0-9]+)*$/.test(slug || '') && (slug || '').length <= 75) score += 2;
  else issues.push('slug tidak bersih / terlalu panjang');

  return { score: Math.min(score, 10), issues };
}
```

---

## Kategori 7 — Freshness Signal (5)

### 7a. Klaim numerik punya penanda waktu (RULE, 2)

```js
const YEAR_NEAR = /\b(20[12]\d)\b/;
// Untuk tiap paragraf berisi klaim numerik, cek ada tahun/penanda waktu di paragraf sama
```

### 7b. Data relevan & tidak kedaluwarsa (AI, 3)

Model men-flag data yang dipresentasikan sebagai "terkini" padahal tahunnya sudah lama,
atau klaim tren tanpa periode waktu.

```
skor_7b = max(0, 1 − jumlah_flag / max(1, jumlah_section)) × 3
```

Proporsional terhadap panjang artikel (jumlah section), konsisten dengan cara 2b menghitung
atribusi kabur — 2 temuan di artikel 7 section tidak sama beratnya dengan 2 temuan di artikel
3 section. Dihitung dari array `freshnessIssues[]` output Critic.

---

## Agregasi kriteria AI → `aiScores` per kategori (dipakai node 29)

Critic (node 28) mengembalikan **temuan mentah**, bukan skor per kategori — sengaja, karena
model lebih andal menilai per-item konkret ("section ini skor 1, alasannya X") daripada
mengarang angka agregat. Konversi temuan → skor kategori dilakukan **deterministik di jsCode**
(node 29 `Parse Critique`), bukan diserahkan ke AI.

| Field output Critic | Kriteria | Formula | Masuk kategori | Bobot AI |
|---|---|---|---|---|
| `answerFirst[].score` (0/1/2) | 1a | `(Σ score / (n_section × 2)) × 15` | `answerFirst` | 15 |
| `vagueAttribution[]` | 2b | `max(0, 1 − n_vague / max(1, n_paragraf)) × 10` | `citation` | 10 |
| `selfContained[].score` (0/1/2) | 4a | `(Σ score / (n_section × 2)) × 12` | `selfContained` | 12 |
| `toneScore` (1–5) | 5a | `(toneScore − 1) / 4 × 7` | `language` | 7 |
| `quotable[].count` | 5b | `(n_section_count≥1 / max(1, n_section)) × 3` | `language` | 3 |
| `freshnessIssues[]` | 7b | `max(0, 1 − n_flag / max(1, n_section)) × 3` | `freshness` | 3 |
| `topFixes[]` | — | tidak diskor — diteruskan ke Reviser (node 34) | — | — |
| — | — | kategori `structure` tidak punya kriteria AI | `structure` | 0 |

`n_paragraf` dan `n_section` **diambil dari `diagnostics` node 26** (Rule Checker), bukan
dihitung ulang di node 29 — supaya pembagi yang dipakai rule dan AI identik. Kalau dihitung
dua kali dengan parser berbeda, skor bisa tidak konsisten tanpa ketahuan.

**Fallback wajib:** kalau Critic gagal parse JSON, jangan set `aiScores` ke 0 — itu membuat
draft bagus langsung jatuh ke REJECT karena kegagalan teknis, bukan kualitas. Yang benar:
tandai `criticFailed: true`, isi `aiScores` dengan **nilai netral 60% dari bobot** tiap
kategori, dan paksa `verdict = 'REVISE'` (bukan PASS/REJECT) supaya run diulang lewat jalur
revise, bukan dipublish atau dibuang berdasarkan skor yang tidak valid.

---

## Kategori 6 — timing khusus (keputusan CP 005)

**Masalah:** Kategori 6 (Metadata & Schema Signals) butuh output Schema Generator (Role 6,
node 37–39) — tapi `Code: GEO Rule Checker` (node 26) dan `Compute GEO Score` (node 30) jalan
di Fase 3, **jauh sebelum** Fase 5 (Schema). Metadata/slug/JSON-LD belum ada saat gate score
dihitung.

**Keputusan:** Kategori 6 **di-exclude dari gate score** (node 26/30/31) dan bobotnya (10
poin) **didistribusikan proporsional** ke 6 kategori lain — pola yang sama persis dengan
penanganan Kategori 6 di `OPTIONAL_SCORING_WORKFLOW.md` (di sana N/A karena jalur upload tidak
lewat Schema; di sini N/A karena Schema belum jalan — beda alasan, solusi sama). Kategori 6
dicek **terpisah** setelah node 39 (Schema), **sebelum** node 40 (WordPress publish):

- Kalau lolos (`pct >= 70`) → lanjut publish seperti biasa.
- Kalau gagal → **jangan** masuk Reviser loop lagi (Reviser bukan role yang menghasilkan
  metadata — percuma). Pakai **fallback deterministik** yang sudah dirancang di
  `PROMPT_GUIDE.md` §6 (judul dari H1, meta description dari paragraf pertama), lalu tetap
  lanjut publish. Ini konsisten dengan prinsip yang sudah ada: "artikel yang lolos scoring
  tidak boleh gagal publish karena model 7B keliru format".

Konsekuensinya: `geoScore` yang menentukan PASS/REVISE/REJECT (node 31) **cuma mencerminkan
6 kategori** (maks 90 poin dinormalisasi ke 100). Kolom `score_breakdown` di `article_log`
tetap mencatat Kategori 6 terpisah (dari cek pasca-Schema) untuk keperluan kalibrasi, bukan
digabung ke `geoScore`.

## Formula skor akhir (gate, node 30 — TANPA Kategori 6)

```js
// Code: Compute GEO Score  (node 30) — dipanggil di Fase 3, sebelum Schema ada
const GATE_WEIGHTS = {
  answerFirst:    { rule: 5,  ai: 15 },
  citation:       { rule: 10, ai: 10 },
  structure:      { rule: 15, ai: 0  },
  selfContained:  { rule: 3,  ai: 12 },
  language:       { rule: 5,  ai: 10 },
  freshness:      { rule: 2,  ai: 3  },
};
// Total bobot mentah 6 kategori = 90. Normalisasi tiap kategori ke skala 100
// dengan faktor 100/90, supaya geoScore tetap 0-100 dan threshold (80/60) tidak berubah.
const NORM = 100 / 90;

const categories = Object.keys(GATE_WEIGHTS);
const breakdown = {};
let total = 0;

for (const k of categories) {
  const max = (GATE_WEIGHTS[k].rule + GATE_WEIGHTS[k].ai) * NORM;
  const got = ((ruleScores[k] ?? 0) + (aiScores[k] ?? 0)) * NORM;
  const clamped = Math.max(0, Math.min(got, max));
  breakdown[k] = { score: +clamped.toFixed(1), max: +max.toFixed(1), pct: Math.round((clamped / max) * 100) };
  total += clamped;
}

const geoScore = Math.round(total);
const verdict = geoScore >= 80 ? 'PASS' : geoScore >= 60 ? 'REVISE' : 'REJECT';

return [{ json: { geoScore, verdict, breakdown, failedCategories:
  categories.filter(k => breakdown[k].pct < 70) } }];
```

## Cek Kategori 6 terpisah (setelah node 39, sebelum node 40)

```js
// Code: Validate Metadata Rules — node baru setelah node 39 (Schema), sebelum node 40 (WordPress)
const m = scoreMetadata(schemaJson, metaDescription, slug); // fungsi sama persis dari Kategori 6 di atas
const pct = Math.round((m.score / 10) * 100);
if (pct < 70) {
  // trigger fallback deterministik (PROMPT_GUIDE.md §6) — TIDAK kembali ke Reviser
  // ... generate title dari H1, metaDescription dari paragraf pertama ...
}
return [{ json: { metadataCheck: { score: m.score, pct, issues: m.issues } } }];
```

### Threshold

| Skor | Verdict | Aksi |
|---|---|---|
| **≥ 80** | `PASS` | Lanjut ke Schema → publish |
| **60–79** | `REVISE` | Masuk Reviser loop (max 2x), perbaiki **hanya** kategori dengan `pct < 70` |
| **< 60** | `REJECT` | Alert Telegram + `Mark Topic Failed`. Biasanya indikasi outline atau riset yang bermasalah, bukan sekadar prosa — menulis ulang draft dari outline yang sama jarang menolong |

### Kalibrasi awal

Threshold 80 itu **titik awal, bukan angka sakral**. Setelah ~10 artikel pertama, cek
kolom `score_breakdown` di sheet `article_log`:

- Kalau **hampir semua draft langsung ≥80 tanpa revise** → rubric terlalu longgar, naikkan
  threshold ke 85 atau perketat kriteria AI.
- Kalau **hampir semua stuck di REVISE** → prompt Writer belum menyerap style guide;
  perbaiki prompt (lihat [PROMPT_GUIDE.md](PROMPT_GUIDE.md)) alih-alih menurunkan threshold.
- Kalau **satu kategori selalu rendah**, perbaiki role penyebabnya, bukan rubriknya:
  Citation rendah → prompt Researcher; Answer-First / Self-Containment rendah → prompt
  Writer; Metadata rendah → prompt Schema atau fallback deterministiknya.
