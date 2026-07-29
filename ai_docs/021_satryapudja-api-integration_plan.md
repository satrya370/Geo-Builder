# CP 021 — Integrasi Publish ke satryapudja.site API (menggantikan WordPress)

## Sumber
Diangkat dari `docs/SATRYAPUDJA_API_INTEGRATION_GUIDE.md` (ditulis dari sisi project
`satryapudja-blog`, dibaca penuh sebelum plan ini disusun — bukan tebakan). Dokumen itu
sudah sangat lengkap (mapping field, error handling, checklist) — CP ini memformalkannya
jadi checkpoint kerja sesuai `rules.md` project ini, PLUS satu penyesuaian penting: **backend
situs baru jalan di localhost, belum di-deploy ke Cloudflare** (dikonfirmasi: `wrangler.toml`
tidak override port → default `http://localhost:8787`, dan proses `wrangler dev` belum
berjalan saat plan ini ditulis).

## ⚠️ Revisi (n8n-mcp sudah tersambung lagi, workflow live di-cek ulang)
`n8n-mcp` sudah tersambung kembali. Workflow live dicek langsung (`get_workflow_details`) —
ditemukan **2 hal yang mengubah step 3 dan step 6 di bawah**, karena CP020 sudah mulai
dieksekusi worker lain sejak plan ini pertama ditulis:

1. **Step 3 diganti sumbernya.** Rencana asli: reuse `Code: Convert Draft to HTML` (CP018,
   dokumen Google Docs). Ternyata sekarang sudah ada node baru `Code: Markdown to WordPress
   HTML` (dibangun CP020) yang outputnya (`_wpHtml`) **sudah persis** body-only HTML yang
   dibutuhkan (H2/H3/blockquote/list, tabel otomatis dikonversi ke teks + warning, H1
   sengaja dibuang) — TIDAK perlu ekstraksi tambahan seperti direncanakan semula. **Reuse
   node ini, bukan `Code: Convert Draft to HTML`.**
2. **Step 6 (wiring) lebih kompleks dari asumsi.** Wiring nyata sekarang:
   ```
   Code: Validate Metadata Rules → If - Is Test Mode?
                                     ├─ port 0 → Code: Convert Draft to HTML (jalur Google Docs)
                                     └─ port 1 → Code: Markdown to WordPress HTML → Code: Build WordPress Payload
   ```
   Bukan pola sederhana "Test→skip total, Live Run→WordPress" seperti CP017 lama. **Port
   mana yang benar-benar Test vs Live Run BELUM diverifikasi lewat eksekusi nyata** — jangan
   asumsi (peringatan berulang di project ini soal port If-node). Node Article API baru
   harus disisipkan pas di titik yang benar dalam percabangan ini, kemungkinan besar
   ditambahkan **sejajar/menggantikan** `Code: Build WordPress Payload` di port yang sama
   (port 1, mengikuti asumsi awal tapi WAJIB dikonfirmasi lewat 1x eksekusi nyata dulu
   sebelum dipakai untuk keputusan wiring).

Kategori (`content-context.json`) sudah dicek ulang — **masih identik**, tidak ada drift
dari asumsi guide. Node `Code: Build Article API Payload`/`HTTP: Create Article` **belum
ada** — CP ini belum pernah dieksekusi siapa pun, aman dilanjutkan.

## Penyesuaian localhost (berlaku HANYA sampai `satryapudja-blog` Chapter 6 selesai)

| Item | Guide asli (asumsi produksi) | Penyesuaian sekarang |
|---|---|---|
| Base URL | `https://api.satryapudja.site` | `http://localhost:8787` (default port `wrangler dev`, **verifikasi ulang** port aktual saat backend dijalankan — cek output terminal `npm run dev` di `satryapudja-blog/backend/`) |
| Syarat sebelum uji nyata | Situs live | **Backend Worker harus dijalankan manual dulu** (`cd satryapudja-blog/backend && npm run dev`) di mesin yang sama dengan n8n — n8n (`localhost:5678`) dan Worker (`localhost:8787`) sama-sama lokal, jadi bisa saling panggil selama SATU-DUANYA aktif bersamaan saat workflow dieksekusi |
| D1 database | Remote (Cloudflare) | Lokal (`--local` flag, sudah default untuk `wrangler dev` tanpa `--remote`) — data tersimpan di file SQLite lokal proyek situs, BUKAN cloud sungguhan |
| Artikel yang tersimpan | Permanen di production | **Sementara/uji coba** — kalau nanti backend di-reset/redeploy ke Cloudflare (Chapter 6), data lokal ini TIDAK otomatis ikut pindah. Jangan anggap hasil tes sekarang sebagai artikel final yang aman selamanya. |

**Konsekuensi desain**: field `articleUrl` yang dibangun `Code: Extract Article API Result`
(§5 guide) harus ikut disesuaikan — untuk fase localhost, base URL publik
`https://satryapudja.site/artikel/...` **belum benar-benar hidup** (situs belum dideploy),
jadi URL itu murni referensi masa depan, bukan link yang bisa diklik sekarang. Pesan Telegram
harus eksplisit bilang "(mode uji lokal — situs publik belum live)" untuk semua hasil selama
fase ini, bukan cuma untuk draft.

## Steps

| # | Step | Risk | Confidence | Token usage |
|---|---|---|---|---|
| 1 | Baca ulang `Context.md` (project situs) + verifikasi 3 nama kategori masih identik `VALID_CATEGORIES` backend (§3.3 guide) | low | high | low |
| 2 | Ambil `ARTICLE_API_KEY` dari `Credential_information.md`, buat n8n Credential "HTTP Header Auth" (header `X-API-Key`) | low | high | low |
| 3 | Buat `Code: Build Article API Payload` — baca `_wpHtml` dari `Code: Markdown to WordPress HTML` (BUKAN `Code: Convert Draft to HTML`/CP018) sebagai field `content`, tambah slugify deterministik (§3.1 guide) | low | high | low |
| 4 | Buat `HTTP: Create Article (satryapudja.site API)` — base URL **localhost** (lihat tabel penyesuaian di atas), method POST, credential dari step 2 | medium | medium | low |
| 5 | Buat `Code: Extract Article API Result` (baca `id`/`slug`/`status`, bangun `articleUrl` dengan catatan mode-lokal) | low | high | low |
| 6a | **WAJIB DULU**: jalankan 1x eksekusi form (topik apa saja, Run Mode `Test`) HANYA untuk mengonfirmasi via `get_execution` port mana dari `If - Is Test Mode?` yang benar-benar tereksekusi untuk Test — JANGAN asumsi dari nama port | low | high | low |
| 6b | Wiring: sisipkan node baru (step 3-5) sejajar dengan `Code: Markdown to WordPress HTML` → `Code: Build WordPress Payload` di port yang terkonfirmasi step 6a sebagai Live Run. **JANGAN hapus node WordPress lama** — biarkan tetap ada tapi lepas dari wiring aktif (atau `setNodeDisabled`), supaya bisa rollback kalau API situs belum stabil | high | medium | medium |
| 7 | Error handling — arahkan ke jalur `Code: Build Failure Context` yang sudah ada (§6 guide), TANPA node baru | low | high | low |

Ringkasan CP: **risk medium-high** (step 6 mengubah wiring inti tail pipeline, ada dependency
eksternal — backend lokal harus hidup), **confidence medium** (mapping field sudah sangat
jelas dari guide, tapi belum pernah diuji nyata end-to-end). → Sesuai `rules.md` §3 (n8n
project), CP risk tinggi + confidence tidak penuh tinggi: **worker eksekusi step 1-5 dulu
(low-medium risk), lalu STOP sebelum step 6 (wiring) untuk konfirmasi eksplisit user**
bahwa backend lokal sudah dinyalakan dan siap diuji — jangan lanjut wiring otomatis tanpa
verifikasi itu.

## Yang HARUS diverifikasi sebelum dianggap selesai
1. **Backend Worker benar-benar hidup** sebelum uji apa pun — `curl http://localhost:8787/api/articles` dari terminal n8n/manual dulu, harus dapat response (bukan connection refused), sebelum menyentuh node n8n.
2. 3 nama kategori (`content-context.json` vs `VALID_CATEGORIES` backend) — cek ulang PERSIS sama, bukan asumsi dari dokumen (kategori bisa saja sudah diubah salah satu sisi sejak guide ditulis).
3. Slugify deterministik benar-benar dipasang (§3.1 guide) — uji dengan judul yang sengaja mengandung karakter kotor (spasi ganda, tanda baca) untuk pastikan hasilnya lolos validasi backend.
4. Uji end-to-end PERTAMA KALI wajib `status: 'draft'` dulu (checklist §7 guide poin terakhir) — jangan langsung `Live Run - Publish`.
5. Field `content` yang dikirim benar-benar HANYA body artikel (tanpa header run info/FAQ `<pre>` dari dokumen export CP018) — cek payload mentah sebelum dikirim, bukan asumsi reuse function otomatis benar.
6. Response error (400/401/409) benar-benar masuk jalur alert Telegram existing, tidak diam-diam dianggap sukses.

## Definition of done
- [ ] Credential "HTTP Header Auth" (`X-API-Key`) dibuat di n8n
- [ ] `Code: Build Article API Payload` dibuat, termasuk slugify deterministik
- [ ] `HTTP: Create Article` dibuat, base URL localhost (dicatat eksplisit di node/deskripsi bahwa ini sementara)
- [ ] `Code: Extract Article API Result` dibuat, `articleUrl` mencatat status "mode uji lokal"
- [ ] **STOP checkpoint**: konfirmasi user backend lokal aktif, sebelum lanjut wiring
- [ ] Wiring cabang `Live Run` diarahkan ke node baru, node WordPress lama dinonaktifkan (bukan dihapus)
- [ ] 1x uji end-to-end `status: 'draft'` sukses — artikel muncul benar di `/admin` (sisi situs)
- [ ] `ai_docs/index.md` (project n8n) diperbarui
- [ ] Dicatat eksplisit di log: base URL ini WAJIB diganti ke domain produksi begitu `satryapudja-blog` Chapter 6 (Deploy) selesai — jangan sampai tertinggal localhost setelah situsnya live
