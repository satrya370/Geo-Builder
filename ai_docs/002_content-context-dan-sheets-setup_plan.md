# CP 002 — Isi Content Context Real + Setup Google Sheets

**Mode:** Planner · **Asal:** `ai_docs/todos.md` item 2
**Dokumen rujukan wajib dibaca sebelum eksekusi:** `docs/IMPLEMENTATION_PLAN.md` §1, §4,
`config/content-context.json` (state saat ini, sudah sebagian diperbarui di CP 001)

## Ringkasan CP (gambaran besar)

- **Confidence:** medium — mekanisme sudah jelas (isi JSON + buat sheet), tapi sebagian
  nilai (niche, audience, author bio, kategori/tag WP asli, Telegram chat ID) **belum
  diketahui** dan harus digali dari user; tidak bisa diasumsikan/dikarang.
- **Risk:** low — murni konfigurasi data, tidak ada node/logic yang dieksekusi ke publik.
  Salah isi gampang dikoreksi sebelum CP mana pun yang benar-benar publish jalan.
- **Token/limit usage estimate:** low — tidak ada AI call, hanya tanya-jawab + tulis file
  + panggil 1-2 HTTP GET (WordPress categories/tags) untuk verifikasi ID.
- **Kesimpulan:** tidak perlu dipecah sub-CP A/B (risk rendah, syarat pemecahan di
  `rules.md` §2 tidak terpenuhi) — tapi CP ini **akan blocked beberapa kali menunggu user**,
  sama seperti CP 001, karena banyak nilai yang hanya user yang tahu.

## Langkah

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Tanya user: niche situs, audience, author (nama/jobTitle/bio/credentials/sameAs), Telegram chat ID | medium | low | low — murni Q&A |
| 2 | Panggil `GET https://public-api.wordpress.com/rest/v1.1/sites/satryapudja.wordpress.com/categories` dan `/tags` (pakai access token dari credential OAuth2 yang sudah connect di CP 001) untuk dapat ID kategori/tag asli — bukan dikarang | medium | low | low — 2 HTTP GET manual/test, bukan lewat node produksi |
| 3 | Isi `config/content-context.json`: `site`, `tone` (pakai default template atau sesuaikan kalau user mau), `author`, `wordpress.categories`/`tags` dengan ID asli dari step 2, `telegram.chatId` | high | low | low |
| 4 | Buat Google Sheet baru (atau pakai yang sudah ada punya user) dengan 2 tab: `topic_queue` dan `article_log`, kolom persis sesuai `IMPLEMENTATION_PLAN.md` §4 | medium | low | low — perlu user share/buat sheet, AI tidak punya akses Google Sheets langsung tanpa credential terpasang di n8n |
| 5 | Isi `sheets.spreadsheetId` di `content-context.json` dengan ID sheet dari step 4 | high | low | low |
| 6 | Isi `topic_queue` dengan **≥3 topik uji** (topik nyata dari niche situs, bukan dummy "test123") supaya CP build berikutnya (node 1–15) punya data yang realistis untuk ditest | medium | low | low |
| 7 | Update `ai_docs/index.md` (baris CP 002) dan `ai_docs/todos.md` (coret item asal, `-> 002`) | high | low | low |

## Keterbatasan yang harus disadari

- Step 1, 4, 6 **bergantung pada input/aksi manual user** (data bisnis yang cuma user tahu,
  dan pembuatan/sharing Google Sheet — AI tidak punya kredensial Google Sheets terpasang di
  n8n MCP untuk membuatnya otomatis). CP ini akan **stop dan menunggu** di titik-titik itu,
  dicatat di log sebagai `blocked: waiting-on-user`, bukan `stopped: token-limit`.
- Step 2 butuh dipastikan dulu credential OAuth2 dari CP 001 masih valid (token belum
  revoked) — kalau gagal di step ini, itu regresi ke CP 001, bukan bug baru di CP 002.

## Definition of done

- `config/content-context.json` tidak ada lagi field `<...>` placeholder yang relevan untuk
  fase build berikutnya (niche, audience, author, kategori/tag ID, chatId, spreadsheetId).
- Google Sheet punya 2 tab (`topic_queue`, `article_log`) dengan kolom sesuai §4, dan
  `topic_queue` terisi ≥3 baris topik uji nyata dengan status `pending`.
- Baris CP 002 di `index.md` berstatus `done`.
- Item asal di `todos.md` tercoret dengan `-> 002`.

## Next mode

Worker mengeksekusi CP ini — brief singkat dulu (§3 rules.md), lalu mulai step 1 dengan
menanyakan data yang dibutuhkan ke user.
