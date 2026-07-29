# Optional: Standalone GEO Scoring Workflow

**Status: opsional, prioritas kedua.** Article Builder (`IMPLEMENTATION_PLAN.md`, Fase 0–10)
adalah workflow utama dan harus stabil dulu sebelum bagian ini dikerjakan. Jangan mulai
dokumen ini sebelum Article Builder lolos Fase 8 (WordPress publish jalan).

## Tujuan

Entry point kedua: kamu upload/paste artikel yang **sudah ada** (bukan hasil generate
workflow ini — bisa tulisan manual, draft dari tools lain, atau artikel lama yang mau
dicek ulang), lalu langsung dapat skor GEO + rekomendasi perbaikan lewat Telegram.
**Tidak ada publish, tidak ada revise loop** — murni read-only scoring report.

## Keputusan arsitektur: sub-workflow, bukan trigger kedua di workflow yang sama

Seperti dibahas: logic scoring (Rule Checker + Critic + Compute Score) di-**ekstrak jadi
workflow terpisah** (`GEO Scoring Core`), dipanggil via node **Execute Workflow** dari dua
tempat:

```
Article Builder (utama)              GEO Scoring Upload (opsional, baru)
  ...Writer/Reviser draft...           Form Trigger - Upload
        │                                    │
        ▼                                    ▼
  ┌─────────────────────────────────────────────────┐
  │         GEO Scoring Core (sub-workflow)          │
  │  Rule Checker → Critic → Compute Score           │
  └─────────────────────────────────────────────────┘
        │                                    │
        ▼                                    ▼
  lanjut ke Reviser/Schema/Publish     Telegram Report → selesai
```

**Kenapa bukan 1 workflow dengan 2 trigger:** menghindari flag `runMode` yang harus dibawa
sepanjang graph + switch besar di ujung. Dua workflow kecil yang manggil 1 sub-workflow
bersama lebih gampang dibaca dan di-debug terpisah, dan tidak menambah risiko ke workflow
utama yang sudah jalan.

**Konsekuensi refactor di Article Builder:** node 26–31 (`Code: GEO Rule Checker` s/d
`Switch - GEO Score Gate`) di `IMPLEMENTATION_PLAN.md` diganti jadi 1 node
`Execute Workflow: GEO Scoring Core`, dilanjutkan `Switch - GEO Score Gate` di workflow
utama seperti biasa (switch tetap di pemanggil, bukan di sub-workflow — sub-workflow cukup
mengembalikan skor & breakdown, keputusan alur tetap milik masing-masing pemanggil).

---

## Workflow A: `GEO Scoring Core` (sub-workflow)

| # | Node | Type | Catatan |
|---|---|---|---|
| C1 | `Execute Workflow Trigger` | `executeWorkflowTrigger` | Input: `{ draft, targetKeyword, entities?, geoWeights? }` |
| C2 | `Code: GEO Rule Checker` | `code` | Sama persis dengan node 26 di plan utama — pindahan, bukan duplikat |
| C3 | `Code: Build AI Critic Body` | `code` | |
| C4 | `HTTP: AI GEO Critic` | `httpRequest` | |
| C5 | `Code: Parse Critique` | `code` | |
| C6 | `Code: Compute GEO Score` | `code` | Output: `{ geoScore, verdict, breakdown, failedCategories }` — lihat `GEO_SCORING_RUBRIC.md` |

Output C6 otomatis jadi return value ke pemanggil (perilaku default node terakhir di
sub-workflow n8n). **Tidak ada keputusan publish/reject di sini** — itu tetap tanggung
jawab pemanggil, karena Article Builder dan Scoring Upload butuh reaksi berbeda terhadap
skor yang sama.

`entities` bersifat opsional di sini karena Scoring Upload tidak selalu punya daftar
entitas dari tahap Researcher (lihat §Keterbatasan di bawah).

---

## Workflow B: `GEO Scoring Upload` (standalone, baru)

| # | Node | Type | Catatan |
|---|---|---|---|
| S1 | `Form Trigger - GEO Scoring Upload` | `formTrigger` | Field: lihat §Form fields |
| S2 | `If - Input Type: URL or Text?` | `if` | Cek apakah field URL diisi atau teks langsung dipaste/upload |
| S3 | `HTTP: Fetch URL Content` | `httpRequest` | Hanya kalau S2 = URL |
| S4 | `Code: Extract Main Content` | `code` | Strip nav/iklan/boilerplate dari HTML hasil fetch (readability-style regex/heuristik) |
| S5 | `Code: Normalize & Parse Sections` | `code` | Konversi ke `sections[]` (heading+body) — format yang dibutuhkan Rule Checker |
| S6 | `If - Input Valid?` | `if` | Minimal 300 kata & minimal 1 section terparse, kalau tidak → S7 |
| S7 | `Respond - Input Invalid` | `respondToWebhook` | "Teks terlalu pendek / tidak bisa diparse" |
| S8 | `Execute Workflow: GEO Scoring Core` | `executeWorkflow` | Panggil Workflow A |
| S9 | `Code: Build Scoring Report` | `code` | Format breakdown + topFixes jadi teks Telegram |
| S10 | `Telegram - Send Scoring Report` | `telegram` | |
| S11 | `Respond - Report Sent` | `respondToWebhook` | Thank-you page/pesan konfirmasi |

### Form fields (S1)

| Field | Wajib? | Catatan |
|---|---|---|
| Judul/label artikel (bebas) | tidak | Untuk label di laporan saja |
| Teks artikel (textarea) ATAU file upload (.txt/.md) | ya, salah satu | |
| URL artikel (kalau mau cek yang sudah live) | tidak | Alternatif dari field di atas |
| **Target keyword** | **ya** | Rubric butuh ini untuk cek Kategori 2 (Citation) & 5c (keyword density) — tanpa ini skor kategori itu tidak bermakna |
| Email/kontak (kalau dipakai publik) | tidak | Hanya kalau workflow ini nanti dibuka untuk orang lain, bukan cuma kamu |

### S4/S5 — kenapa perlu, dan batasannya

Draft dari Writer (Article Builder) sudah markdown bersih dengan heading konsisten karena
memang di-generate sesuai outline. **Artikel upload tidak punya jaminan itu** — bisa HTML
kotor dari copy-paste, tanpa heading sama sekali, atau format Word/Google Docs yang
berantakan. `Code: Normalize & Parse Sections` harus:

```js
// Sketsa logic S5 — bukan implementasi final
function parseSections(rawText) {
  // 1. Kalau HTML: strip tag ke markdown kasar (heading <h2>/<h3> → ##/###)
  // 2. Split berdasarkan heading markdown (## atau ###)
  // 3. Kalau TIDAK ADA heading sama sekali: seluruh teks jadi 1 "section" tanpa nama
  //    → ini SAH, bukan error. Kategori 3 (Struktur) & sebagian Kategori 1
  //    (Answer-First per section) akan otomatis dapat skor rendah, dan itu
  //    mencerminkan kenyataan: artikel tanpa heading memang kurang GEO-friendly.
  return { sections, paragraphs: flattenToParagraphs(sections) };
}
```

**Jangan menganggap "tidak ada heading" sebagai kegagalan parsing yang harus di-reject.**
Itu justru temuan yang valid dan harus muncul di report sebagai salah satu `topFixes`.
Reject (S6/S7) hanya untuk kasus benar-benar tidak bisa diproses: teks kosong, terlalu
pendek (<300 kata), atau upload file yang gagal dibaca sama sekali.

### Keterbatasan dibanding scoring dalam Article Builder

| Aspek | Article Builder (internal) | Scoring Upload (eksternal) |
|---|---|---|
| `entities` (daftar entitas kunci) | Ada, dari Researcher | Tidak ada — Kategori 4 (Self-Containment) di-nilai AI tanpa daftar acuan, sedikit lebih longgar |
| Fakta bersumber untuk validasi silang | Ada (`facts[]` dari Researcher) | Tidak ada — Kategori 2 (Citation) hanya bisa cek "apakah ada sitasi", bukan "apakah sitasinya cocok dengan riset kita" |
| Target keyword | Otomatis dari topic queue/form | Harus diisi manual di form |

Ini bukan bug, tapi keterbatasan yang wajar — beri tahu di laporan Telegram kalau
`entities`/`facts` tidak tersedia untuk konteks ini, supaya skor tidak disalahartikan
sebagai "lebih akurat" atau "kurang akurat" dari yang dihasilkan internal.

### Format laporan Telegram (S9)

```
📊 GEO Scoring Report
📝 {label artikel / "Tanpa judul"}
🎯 Skor: {geoScore}/100 — {verdict}

Breakdown:
• Answer-First Structure    {pct}%
• Citation & Fact Density   {pct}%
• Struktur & Format         {pct}%
• Self-Containment          {pct}%
• Bahasa & Tone             {pct}%
• Metadata & Schema         {pct}% *(N/A — upload tidak melalui tahap schema)*
• Freshness Signal          {pct}%

🔧 Perbaikan prioritas:
1. {topFixes[0]}
2. {topFixes[1]}
3. {topFixes[2]}

ℹ️ Catatan: skor dihitung tanpa daftar entitas & fakta riset internal —
lihat batasan di atas.
```

Kategori Metadata & Schema (bobot 10) tidak relevan untuk artikel yang belum tentu akan
dipublish lewat pipeline ini — tampilkan sebagai N/A, dan **redistribusikan bobotnya**
secara proporsional ke 6 kategori lain saat menghitung skor total untuk jalur ini saja
(implementasi di `Code: Build Scoring Report`, bukan mengubah rubric utama).

---

## Urutan implementasi (setelah Article Builder stabil)

| Tahap | Cakupan | Kriteria selesai |
|---|---|---|
| R1 | Refactor: pindahkan node 26–31 dari Article Builder jadi Workflow A (`GEO Scoring Core`) | Article Builder tetap jalan identik lewat `Execute Workflow` — regresi nol |
| R2 | Workflow B node S1–S7 (input + parsing, tanpa scoring) | Artikel tanpa heading & artikel rapi sama-sama terparse tanpa error |
| R3 | Workflow B node S8–S11 (panggil Core + report) | Skor + breakdown + topFixes masuk Telegram sesuai format |

**R1 dulu sebelum R2/R3** — kalau refactor sub-workflow ternyata mengubah perilaku scoring
di Article Builder (regresi), lebih murah ketahuan sebelum Workflow B dibangun di atasnya.
