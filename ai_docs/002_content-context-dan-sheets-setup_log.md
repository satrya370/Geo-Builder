# Log CP 002 — Isi Content Context Real + Setup Google Sheets

**Mengikuti:** `002_content-context-dan-sheets-setup_plan.md`
**Mode:** Worker (dieksekusi interaktif bersama user, masih berjalan)

## Progress

### Step 1 (niche, audience, author, tone) — SEBAGIAN SELESAI

Diskusi niche berlangsung lebih dalam dari perkiraan awal (bukan cuma isi field, tapi
keputusan scope): dari 4 kandidat pillar (filsafat/psikologi, tech/AI, game, science)
dipersempit jadi **satu niche: Filsafat + opini/perspektif psikologis niche**.

Keputusan yang sudah dikunci lewat AskUserQuestion:
- **Tidak perlu disclaimer wajib** di tiap artikel gangguan psikologis spesifik — cukup
  dari framing tone (opini/perspektif, bukan gaya klinis).
- **Rubric 5a (Tone) disesuaikan** — opini/stance diperbolehkan bahkan dianjurkan, yang
  dinilai adalah apakah dirujuk ke pemikir/teori spesifik, bukan "netral atau tidak".
- **3 kategori WordPress**: Filsafat Klasik & Modern / Filsafat Sains (naturalisme, evolusi)
  / Opini & Perspektif Psikologis.

Field yang **sudah** diisi di `content-context.json`: `site.niche`, `site.audience`,
`site.contentStyle`, seluruh `tone` block (termasuk field baru `tone.stance`), `author`
(framing: penulis/peneliti independen, eksplisit TIDAK klaim psikolog klinis/akademisi
formal kecuali benar), `wordpress.categories` (nama + `useWhen`, 3 kategori final).

Field yang **masih placeholder**, menunggu user: `site.name`, `author.name`, `author.bio`
(butuh 1-2 kalimat asli dari user), `author.credentials` (opsional), `author.sameAs`.

### Dokumen desain lain yang ikut diperbarui (scope CP ini melebar sedikit, dicatat karena
langsung konsekuensi dari keputusan niche, bukan topik baru terpisah)

- `docs/GEO_SCORING_RUBRIC.md` §5a — kriteria Tone diganti jadi "Reasoned/argued vs
  unsupported hype", skala 1-5 baru.
- `docs/PROMPT_GUIDE.md` — Critic prompt kriteria D (TONE), Writer prompt aturan 9, dan
  Safety Guard prompt (tambah 2 contoh ✅/❌ khusus niche ini) diperbarui agar konsisten
  dengan keputusan rubric di atas.

### Step 2 (fetch WordPress categories/tags asli) — BELUM DIMULAI

`wordpress.categories[].id` masih `null` — menunggu GET
`{restApiBase}/categories` pakai access token dari credential OAuth2 (CP 001) untuk dapat
ID numerik asli sebelum bisa dipetakan.

### Step 3-6 (author real values, Google Sheets, topik uji) — BELUM DIMULAI

## Status

**Selesai (done).** Definition of done tercapai:
- [x] `content-context.json` tidak ada placeholder — semua field terisi real (English)
- [x] Google Sheet ` topic_queue` terisi 5 topik uji status `pending`
- [x] Baris CP 002 di `index.md` → `done`
- [x] Item asal di `todos.md` sudah tercoret `-> 002`

### Ringkasan hasil akhir

| Field | Value |
|-------|-------|
| `site.name` | `satryapudja.wordpress.com` |
| `author.name` | `Satrya Pudja` |
| `author.bio` | English, 2 kalimat — framing penulis independen + pengalaman ADHD/kecemasan |
| `wordpress.categories[].id` | `790703683`, `62245`, `790703690` (dibuat via OAuth2 API) |
| `sheets.spreadsheetId` | `1TuezG43j45nXBrL-KRq_n-srMZyw_832YBvszmLveuo` |
| `telegram.chatId` | `8675308460` (dari `Telegram - Send Result` di ig Content Builder Demo) |
| `telegram.credential` | `Telegram Bot - IG Content Builder` (`CQbwRmBqoj3j86Jf`) |
| `sheets.credential` | `Google Sheets OAuth2 - IG Content Builder` (`OvZoOWeznFSWBvuV`) |
