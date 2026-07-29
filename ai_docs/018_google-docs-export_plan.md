# CP 018 — Ganti Export TXT → Export ke Google Docs (format H/bold/list/table dijaga)

## Latar belakang
Export saat ini (CP017 §7) menulis file `.txt` polos ke disk lokal (`exports/`) lewat Code node + `fs.writeFileSync` — tidak ada salinan yang bisa dibaca dengan format rapi (heading, bold, tabel, blockquote semua jadi teks mentah markdown). Permintaan: ganti tujuan export dari file `.txt` lokal jadi dokumen Google Docs, dengan syarat heading (H2 section utama, H3 sub-pertanyaan FAQ) dan elemen format lain (bold, list, tabel, blockquote) **benar-benar dijaga**, bukan cuma teks markdown mentah yang ditaruh di satu paragraf.

## Feasibility & pilihan pendekatan
Dua opsi teknis dipertimbangkan:

1. **Google Drive API — upload HTML, auto-convert ke Google Docs** (dipilih). Convert markdown → HTML di satu Code node, upload lewat `files.create` (`mimeType: application/vnd.google-apps.document`, source `text/html`). Drive importer memetakan tag HTML ke gaya paragraf native Google Docs (Heading 1/2/3, bold, bullet/numbered list, tabel) secara otomatis dan reliable.
2. **Google Docs API native (`documents.batchUpdate`)** — kontrol presisi penuh lewat request `insertText`/`updateParagraphStyle`/`createParagraphBullets`/`insertTable`, tapi index-nya posisi karakter global yang bergeser tiap insert → butuh logic tracking offset kompleks, paling rawan bug di bagian tabel (draft ini pakai banyak tabel pipe markdown per section).

**Keputusan: pakai Opsi 1.** Kompleksitas jauh lebih rendah, gaya implementasi konsisten dengan pola project ini (Code node regex-based, tanpa library markdown eksternal), dan hasil convert HTML→Docs sudah cukup akurat untuk seluruh elemen yang dipakai draft (H2/H3, bold, blockquote, bullet list, tabel).

## Keputusan desain

### 1. Node baru: `Code: Convert Draft to HTML`
Ditaruh menggantikan alur lama sebelum node write-file. Ambil `draft` (markdown) + `meta` + `score` + WP info dari node yang sama seperti `Code: Build Export Payload` sekarang (lihat CP017 §5 — pola ambil data via referensi nama node eksplisit, JANGAN andalkan `$input`/`$json` langsung karena ada 2 kemungkinan node sumber tergantung Test vs Live Run).

Mapping markdown → HTML yang wajib didukung (semua elemen yang benar-benar dipakai di draft nyata, lihat contoh eksekusi 992):
- `## Heading` → `<h2>Heading</h2>` (section utama)
- `### Heading` → `<h3>Heading</h3>` (sub-pertanyaan di dalam `## FAQ`)
- `**text**` → `<strong>text</strong>`
- `> "quote"` → `<blockquote>quote</blockquote>`
- `- item` (baris berurutan) → dikelompokkan jadi satu `<ul><li>item</li>...</ul>`
- Tabel pipe markdown (`| a | b |` + baris separator `|---|---:|`) → `<table><tr><th>/<td>` sungguhan, bukan teks apa adanya. Alignment kanan (`---:`) boleh disederhanakan jadi rata kiri biasa di HTML — bukan blocker, catat sebagai simplifikasi yang disengaja.
- Paragraf biasa → `<p>`.

Struktur dokumen HTML lengkap yang dihasilkan:
```
<h1>{{meta.title}}</h1>
<p><em>{{ctx.topic}} — Run ID {{ctx.runId}} — Run Mode {{ctx.publishMode}}</em></p>
<p>Meta Description: {{meta.metaDescription}}</p>
<p>Slug: {{meta.slug}} | Kategori: {{meta.categories.join(', ')}} | Tags: {{meta.tags.join(', ')}}</p>
<p>GEO Score: {{score.geoScore}}/100 — Verdict: {{score.verdict}}</p>
<p>{{wpSection — WordPress Post ID/URL/Status, atau catatan Test/REJECT sesuai pola existing}}</p>
<hr>
{{HTML hasil convert draft markdown}}
<hr>
<h3>FAQ Schema (raw JSON, untuk referensi)</h3>
<pre>{{JSON.stringify(meta.faqSchema, null, 2) — di-escape HTML entities}}</pre>
```
FAQ Schema JSON tetap disertakan (dalam `<pre>`, bukan dirender jadi paragraf) supaya tidak ada informasi yang hilang dibanding export TXT lama — ini metadata teknis untuk schema markup WordPress, bukan konten yang perlu format rapi.

**Escape wajib**: semua teks yang disisipkan ke HTML (title, meta description, isi paragraf, dll) harus di-escape entity (`&`, `<`, `>`, `"`) SEBELUM dibungkus tag — draft artikel sering mengandung karakter `<`/`>` dari perbandingan filosofis atau tanda kurung yang bisa dianggap tag HTML kalau tidak di-escape.

### 2. Credential Google Drive OAuth2 — perlu di-setup lewat CLI lain
Sama seperti pola CP016 (Semantic Scholar/Google Books API key) — **hanya CLI lain yang bisa pasang credential baru di n8n**. Yang dibutuhkan:
- Buat OAuth2 Client ID di Google Cloud Console dengan scope minimal `https://www.googleapis.com/auth/drive.file` (scope ini cukup — hanya file yang dibuat lewat aplikasi ini yang bisa diakses, tidak perlu akses penuh ke seluruh Drive user).
- Tambahkan sebagai credential type "Google Drive OAuth2 API" di n8n, lalu attach ke node HTTP Request baru (lihat §3).
- **Jangan** pakai API key biasa — endpoint upload file Drive butuh OAuth2/service account, API key saja tidak cukup untuk operasi tulis.

### 3. Node baru: `HTTP: Upload to Google Docs`
**WAJIB panggil `get_suggested_nodes`/`get_node_types` dulu sebelum membangun node ini** — jangan menebak parameter multipart upload (proyek ini sudah pernah rugi waktu karena menebak skema node, lihat `rules.md` §3.5). Yang perlu dipastikan lewat tool tersebut, bukan diasumsikan:
- Apakah n8n HTTP Request node punya dukungan native multipart/related upload (dua bagian: metadata JSON + konten HTML), atau perlu disusun manual sebagai raw body dengan `Content-Type: multipart/related; boundary=...`.
- Apakah "Google Drive OAuth2 API" tersedia sebagai **Predefined Credential Type** di HTTP Request node (opsi ini jauh lebih sederhana daripada generic OAuth2 manual) — cek lewat `get_node_types` untuk `n8n-nodes-base.httpRequest`.
- Endpoint: `POST https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart`, metadata part: `{"name": "{{ctx.runId}}_{{ctx.publishMode}} - {{meta.title}}", "mimeType": "application/vnd.google-apps.document"}`, media part: `Content-Type: text/html`, body = HTML dari §1.

Output yang perlu ditangkap: `id` (file ID Google Docs hasil) dan `webViewLink` (kalau field ini diminta lewat `fields=id,webViewLink` di query) — supaya link dokumen bisa dicatat/ditampilkan.

### 4. Ganti node lama, bukan tambah paralel
Sesuai permintaan ("ganti aja export txt nya") — **hapus** `Code: Write Export to File` (fs write lokal) dari wiring, gantikan dengan dua node baru di atas (`Code: Convert Draft to HTML` → `HTTP: Upload to Google Docs`). `Code: Build Export Payload` tetap ada tapi disederhanakan/direname jadi sumber data untuk converter HTML (field `exportText` gaya TXT lama tidak dipakai lagi, ganti concern-nya jadi menyiapkan data terstruktur untuk §1).

Tidak ada perubahan pada wiring `If - Is Test Mode?` — export tetap jalan di ketiga Run Mode (Test/Live Draft/Live Publish), cuma tujuannya yang berubah dari file lokal ke Google Docs.

### 5. Folder tujuan di Drive (opsional, perlu keputusan user)
Kalau user punya folder khusus di Google Drive untuk menyimpan hasil generate, tambahkan `"parents": ["{{folderId}}"]` di metadata upload — folder ID perlu didapat manual dari URL folder Drive (bagian setelah `/folders/` di address bar). Kalau tidak ada preferensi, dokumen dibuat di root Drive akun yang credential-nya dipakai.

## Yang HARUS diverifikasi (jangan asumsi)
1. Dokumen yang dihasilkan benar-benar **Google Docs native** (bisa diedit langsung di docs.google.com), bukan file HTML mentah yang di-attach — cek dengan membuka `webViewLink` hasil dari respons API.
2. Heading `## ` di artikel jadi gaya paragraf **Heading 2** asli di Google Docs (cek dari Format > Paragraph styles di UI Google Docs, bukan cuma tebal/besar visual).
3. Heading `### ` di dalam FAQ jadi **Heading 3**, berbeda level dari section utama.
4. Tabel pipe markdown jadi tabel Google Docs sungguhan (baris/kolom native, bisa di-resize), bukan teks dengan simbol `|` apa adanya.
5. Blockquote (`>`) tampil sebagai gaya kutipan (indented/italic sesuai default Docs), bukan paragraf biasa.
6. Karakter `<`/`>`/`&` di dalam isi artikel (kalau ada) ter-escape dengan benar, tidak merusak struktur HTML atau hilang dari hasil akhir.
7. Credential Google Drive OAuth2 benar-benar aktif dan attached ke node yang benar (redaction artifact `get_workflow_details` membuat field `credentials` selalu terlihat null — verifikasi HANYA lewat eksekusi nyata yang berhasil, bukan lewat pembacaan statis).
8. Export tetap jalan tanpa error di ketiga Run Mode, termasuk jalur REJECT/RETRY EXCEEDED (draft tidak lengkap/kosong) — pastikan converter HTML tidak crash kalau `draft` kosong atau `meta` sebagian null.

## Definition of done
- [ ] Credential Google Drive OAuth2 (scope `drive.file`) terpasang di n8n (dikerjakan CLI lain)
- [ ] `Code: Convert Draft to HTML` — mapping H2/H3/bold/blockquote/list/tabel lengkap, escape HTML aman
- [ ] `HTTP: Upload to Google Docs` — skema dikonfirmasi lewat `get_node_types`/`get_suggested_nodes`, bukan tebakan
- [ ] `Code: Write Export to File` (fs lokal) dihapus dari wiring
- [ ] Minimal 1 eksekusi live per Run Mode (Test/Live Draft/Live Publish) menghasilkan Google Docs yang bisa dibuka & formatnya benar (poin 2-5 di atas)
- [ ] 1 eksekusi dengan jalur REJECT/RETRY EXCEEDED dites, tidak crash
- [ ] `ai_docs/index.md` diperbarui (baris 018 baru)
