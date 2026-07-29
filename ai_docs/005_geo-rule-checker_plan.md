# CP 005 — GEO Rule Checker (Node 26)

**Mode:** Planner · **Asal:** `ai_docs/todos.md` item 13 — "Bangun node 26 (`GEO Rule
Checker`) saja, validasi skor rule-based di 3 draft uji sebelum lanjut ke Critic"
**Dokumen rujukan wajib dibaca sebelum eksekusi:** `docs/GEO_SCORING_RUBRIC.md` (seluruh
kriteria RULE + bagian baru "Kategori 6 — timing khusus"), `docs/IMPLEMENTATION_PLAN.md` §3
Fase 3 (sudah diperbarui), `config/content-context.json` (blok `geo`), workflow `rwIbdIkIhoVE8nkG`

## Keputusan arsitektur yang mendahului CP ini

Sebelum plan ini ditulis, ditemukan celah desain: Kategori 6 (Metadata & Schema) butuh output
Schema Generator (node 37–39) yang belum ada saat node 26 jalan (Fase 3, jauh sebelum Fase 5).
**Diputuskan:** Kategori 6 di-exclude dari node 26/30/31, bobotnya didistribusikan proporsional
ke 6 kategori lain, dan dicek terpisah nanti setelah node 39 (CP terpisah, bukan bagian CP 005
ini). Detail penuh + kode formula baru: `GEO_SCORING_RUBRIC.md` §"Kategori 6 — timing khusus"
dan `IMPLEMENTATION_PLAN.md` §3 Fase 3 & 5 (sudah diperbarui). **CP 005 HANYA membangun node 26
untuk 6 kategori (1–5, 7)** — jangan implementasi Kategori 6 di sini.

## Ringkasan CP (gambaran besar)

- **Confidence:** high — semua fungsi RULE untuk 6 kategori sudah tersedia sebagai jsCode
  runnable persis di `GEO_SCORING_RUBRIC.md` (`scoreNoFiller`, `scoreCitationRatio`,
  `scoreStructure`, `scoreEntityClarity`, `scoreKeywordDensity`, freshness 7a) — tinggal
  digabung + parsing markdown jadi `sections[]`/`paragraphs[]` yang belum ada snippet-nya.
- **Risk:** low — 100% deterministik, 0 AI call, 0 request eksternal. Kalau salah, gampang
  dites ulang tanpa biaya token.
- **Token/limit usage estimate:** low — 1 node jsCode, tidak ada AI call sama sekali.
- **Kesimpulan:** tidak perlu split sub-CP A/B. CP paling rendah-risiko sejauh ini di project.

## Langkah

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Tulis fungsi parsing markdown → `sections[]` (split by `## ` heading, ambil `heading`+`body`) dan `paragraphs[]` (flatten semua body, split blank line) dari `draft` (output node 25) | medium | low | low — belum ada snippet siap pakai di rubric, ini bagian baru yang perlu ditulis hati-hati |
| 2 | Implementasi Kategori 1b (`scoreNoFiller`) — copy persis dari rubric | high | low | low |
| 3 | Implementasi Kategori 2a (`scoreCitationRatio`) — copy persis, perlu `paragraphs[]` dari step 1 | high | low | low |
| 4 | Implementasi Kategori 3 (`scoreStructure`, 3a-3c) — copy persis, perlu `md` mentah + `sections[]` | high | low | low |
| 5 | Implementasi Kategori 4b (`scoreEntityClarity`) — copy persis, perlu `paragraphs[]` | high | low | low |
| 6 | Implementasi Kategori 5c (`scoreKeywordDensity`) — copy persis, perlu `md` + `targetKeyword` (dari run context, resolved dari Outline kalau form kosong) | high | low | low |
| 7 | Implementasi Kategori 7a (freshness rule) — perlu `paragraphs[]` + regex `YEAR_NEAR` (rubric baru kasih regex, belum kasih fungsi lengkap — tulis fungsi pembungkusnya) | medium | low | low |
| 8 | Gabung semua ke satu return object: `{ ruleScores: { answerFirst, citation, structure, selfContained, language, freshness }, diagnostics: {...} }` — field `ruleScores` inilah yang dikonsumsi node 30 nanti | high | low | low |
| 9 | Tes manual dengan **3 draft uji** (bikin draft contoh: 1 draft "bagus" harusnya skor rule tinggi, 1 draft "basa-basi tanpa data" harusnya rendah, 1 draft "keyword stuffing" harusnya kena penalti 5c) — validasi skor masuk akal, bukan cuma "tidak error" | medium | low | low — tidak ada AI call, aman diulang berkali-kali |
| 10 | Tulis log Worker `005_geo-rule-checker_log.md`, update `index.md`, update `todos.md` item 13 | high | low | low |

## Catatan teknis

- **Input:** node 26 idealnya bisa dites berdiri sendiri (tidak wajib nunggu node 24 Writer
  selesai di CP 004) — untuk step 9, draft uji bisa ditulis manual sebagai test input, tidak
  harus hasil AI Writer asli. Ini yang bikin CP 005 bisa dikerjakan paralel/independen dari
  status CP 004.
- **`targetKeyword` untuk step 6:** ambil dari `outline.resolvedTargetKeyword` kalau form
  kosong (lihat CP 004 node 22), atau `ctx.targetKeyword` kalau form diisi. Node 26 nanti
  connect dari node 25 (`Parse Draft`) — tapi `targetKeyword` sudah tidak ada di item node 25
  (sama seperti bug context-hilang yang diperbaiki di CP 004) — **ambil lewat referensi
  bernama** `$('Code: Build AI Writer Body').first().json` atau `$('Code: Build Run Context')`,
  jangan asumsikan ada di `$input`.
- **Kategori 6 TIDAK diimplementasikan di sini** — lihat "Keputusan arsitektur" di atas.
  `scoreMetadata` dari rubric baru dipakai nanti di node 39b (CP terpisah, setelah Schema).
- **Output node 26 BUKAN skor final** — cuma `ruleScores` per kategori. Skor gabungan (rule+AI,
  dinormalisasi 0-100) dihitung di node 30 (`Compute GEO Score`), CP terpisah setelah Critic
  (node 27-29) juga jadi.
- **Tidak menyentuh:** node 27-31 (Critic + Compute Score + Gate), node 32-36 (Revise),
  node 37-39+39b (Schema + validasi Kategori 6), node 40 dst — tetap CP berikutnya.

## Definition of done

- [ ] Node 26 (`Code: GEO Rule Checker`) ada di workflow `rwIbdIkIhoVE8nkG`, terhubung dari
      node 25 (`Parse Draft`)
- [ ] Parsing markdown → `sections[]`/`paragraphs[]` benar untuk draft dengan heading `##`
      normal DAN draft tanpa heading sama sekali (tidak boleh error, `sections=[]` valid)
- [ ] 6 fungsi rule (1b, 2a, 3a-c, 4b, 5c, 7a) menghasilkan skor dalam rentang bobot masing-masing
      (tidak negatif, tidak melebihi max)
- [ ] Tes 3 draft uji manual: draft bagus > draft basa-basi > (atau setara) draft keyword-stuffing
      di skor total rule, sesuai ekspektasi kualitatif
- [ ] `005_geo-rule-checker_log.md` tertulis, `index.md` baris CP 005 ditambahkan, `todos.md`
      item 13 dicoret `[x]` + `-> 005`

## Next mode

Worker — brief dulu sebelum eksekusi (§3 rules.md). Karena tidak ada AI call, boleh dikerjakan
dalam 1 run penuh tanpa risiko kena limit token.
