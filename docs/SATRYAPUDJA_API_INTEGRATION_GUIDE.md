# Guide — Menyambungkan Workflow ke API satryapudja.site (Chapter 3 sisi n8n)

> Dokumen ini ditulis dari sisi project **satryapudja-blog** (situs), setelah membaca
> `docs/IMPLEMENTATION_PLAN.md`, `docs/PROMPT_GUIDE.md` §6, `ai_docs/017_run-mode-and-export_plan.md`,
> `ai_docs/018_google-docs-export_plan.md`, dan `config/content-context.json` project ini —
> bukan tebakan generik. Konteks kenapa mekanismenya begini ada di
> `D:\Agent_workspace\satryapudja-blog\Context.md`, wajib dibaca dulu sebelum eksekusi.
>
> **Ini panduan referensi, bukan CP.** Kalau item ini diangkat jadi checkpoint kerja
> nyata, ikuti proses `rules.md` project ini seperti biasa (Planner buat plan formal dari
> isi dokumen ini, bukan main eksekusi dari sini langsung).

## 1. Yang diganti

| Lama (Fase 6, node 40-42) | Baru |
|---|---|
| `Code: Build WordPress Payload` | `Code: Build Article API Payload` |
| `HTTP: WordPress Create Post` (OAuth2, `public-api.wordpress.com`) | `HTTP: Create Article (satryapudja.site API)` (header `X-API-Key`, bukan OAuth2) |
| `Code: Extract WP Result` (baca `link`/`ID`) | `Code: Extract Article API Result` (baca `slug`/`id`, bangun URL sendiri) |

Node 37-39b (Schema/Metadata) **tidak berubah** — output `meta`/`jsonLd` dari node itu tetap
dipakai apa adanya sebagai sumber data payload baru. `Code: Convert Draft to HTML` (dibangun
CP018 untuk Google Docs export) **direuse**, bukan ditulis ulang — lihat §3.

## 2. Auth — beda total dari WordPress

Endpoint ini pakai **API key statis di header**, bukan OAuth2:

```
POST https://api.satryapudja.site/api/articles
Header: X-API-Key: <ARTICLE_API_KEY>
Content-Type: application/json
```

Nilai `ARTICLE_API_KEY` **sudah digenerate** di sisi situs (CP013,
`satryapudja-blog/ai_docs/013_article-api-key-setup_log.md`) dan **ditempel manual ke
`Credential_information.md` di project ini oleh user** — ambil nilainya dari sana, jangan
minta AI menebak/generate ulang. Simpan sebagai n8n Credential HTTP Header Auth (nama
header `X-API-Key`), **bukan** dimasukkan ke body/URL.

⚠️ Base URL di atas (`api.satryapudja.site`) **asumsi** — situs belum di-deploy ke
Cloudflare (Chapter 6 project situs belum berjalan). Sebelum node ini benar-benar dieksekusi
live, konfirmasi dulu subdomain/URL final Worker yang dipakai (bisa juga
`satryapudja-api.<akun>.workers.dev` kalau belum attach custom domain).

## 3. Mapping field lengkap

Sumber data (pola "ambil via referensi nama node eksplisit" — **konsisten dengan pola yang
sudah dipakai `Code: Build Export Payload` CP017**, jangan andalkan `$json` langsung karena
ada 2 kemungkinan node sumber tergantung Test vs Live Run):

```js
// Code: Build Article API Payload
const validated = $('Code: Validate Metadata Rules').first().json; // { meta, jsonLd, ... }
const draft = $('Code: Parse Draft').first().json.draft || '';
const ctx = $('Code: Build Run Context').first().json;
const score = $('Code: Compute GEO Score').first().json; // { geoScore, verdict, breakdown }
const meta = validated.meta || {};
```

| Field API situs | Sumber n8n | Catatan |
|---|---|---|
| `slug` | `meta.slug`, **di-slugify ulang deterministik** (lihat §3.1 — JANGAN kirim mentah) | AI-generated, tidak selalu bersih |
| `title` | `meta.title` | Langsung, tanpa modifikasi |
| `content` | HTML hasil convert `draft` markdown (reuse logic `Code: Convert Draft to HTML`, **hanya body artikel** — lihat §3.2) | **Bukan** dokumen export lengkap CP018 (yang ada header run info dll) |
| `meta_description` | `meta.metaDescription` | Prompt sudah membatasi 120-160 char — aman di bawah limit 300 char backend (lihat `satryapudja-blog` CP012 D4) |
| `category` | `meta.categories[0]` | **Temuan penting** — lihat §3.3, tidak perlu tabel mapping |
| `tags` | `meta.tags` | Array string, pass-through langsung |
| `faq_schema` | `meta.faqSchema` | Format **sudah identik** `{question, answer}[]` — pass-through langsung, nol konversi |
| `status` | Turunan `ctx.publishMode` — lihat §3.4 | `'published'` atau `'draft'`, **bukan** dikirim untuk mode `Test` |
| `geo_score` | `score.geoScore` | Number, dikirim apa adanya (field ini "hanya referensi admin", tidak tampil publik) |

### 3.1 Slug — WAJIB di-slugify ulang, jangan percaya output AI mentah

Backend situs (`satryapudja-blog`) **menolak keras** (400) slug yang tidak persis format
`lowercase-dengan-tanda-hubung-tunggal` — ditambahkan justru untuk mengantisipasi kasus ini
(lihat `satryapudja-blog/ai_docs/012_db-be-fe-contract-hardening_plan.md` §K3). Prompt
Schema Generator (`PROMPT_GUIDE.md` §6) sudah instruksikan "huruf kecil, dipisah tanda
hubung", tapi ini keluaran model 7B — tidak dijamin bersih (bisa ada spasi tersisa, tanda
baca, atau karakter non-ASCII). **Jangan kirim `meta.slug` mentah.** Tambahkan slugify
deterministik di `Code: Build Article API Payload`:

```js
function slugify(value) {
  return value
    .normalize('NFKD')
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '');
}
const slug = slugify(meta.slug || meta.title);
```

Ini persis fungsi `slugify()` yang dipakai backend sendiri untuk endpoint admin — menjamin
format yang dikirim n8n akan selalu diterima.

### 3.2 Content — HTML body artikel saja, bukan dokumen export CP018

`Code: Convert Draft to HTML` (CP018) menghasilkan **satu dokumen lengkap** dengan header
run info (topik, run ID, meta description, dst) dibungkus di sekitar body artikel — itu
untuk kebutuhan Google Docs export, **bukan** untuk field `content` API ini. Ambil/reuse
**hanya bagian mapping markdown→HTML-nya** (H2/H3/bold/blockquote/list/tabel — daftar
lengkap di `ai_docs/018_google-docs-export_plan.md` §1), terapkan ke `draft` saja, tanpa
bagian `<h1>{{meta.title}}</h1>`, info run, dan `<pre>` FAQ JSON yang ada di dokumen export.

Escape HTML tetap wajib (poin yang sama persis disebutkan di CP018) — draft sering
mengandung `<`/`>` dari perbandingan filosofis yang bisa dianggap tag kalau tidak di-escape.

### 3.3 Category — TIDAK perlu tabel mapping (temuan, bukan asumsi)

Dicek langsung ke `config/content-context.json` blok `wordpress.categories` — 3 nama
kategori yang sudah dipakai project ini:

```
"Classical & Modern Philosophy"
"Philosophy of Science"
"Psychological Opinion & Perspective"
```

**Persis sama, karakter demi karakter**, dengan `VALID_CATEGORIES` di backend
`satryapudja-blog` (`backend/src/index.ts`). Ini menjawab tuntas open question yang
ditandai di `Context.md` §5 ("field yang tidak ada padanan langsung... category") — untuk
category, **tidak ada** padanan yang perlu diputuskan, tinggal pass-through
`meta.categories[0]`. Prompt Schema Generator sudah diinstruksikan pilih dari daftar ini
(`contentContext.wordpress.categories`), jadi selama prompt itu tidak diubah, nilainya akan
selalu valid.

⚠️ Kalau `content-context.json` di project ini pernah diubah (nama kategori diedit/ditambah)
**setelah** dokumen ini ditulis, cek ulang 3 nama itu masih identik dengan backend situs
sebelum wiring — kalau tidak identik, `POST /api/articles` akan menolak 400 (`category must
be one of: ...`).

### 3.4 Status — turunan Run Mode, dan Test **tidak memanggil endpoint sama sekali**

Sama persis pola yang sudah ada untuk WordPress (CP017 §6 — `status` dinamis dari
`ctx.publishMode`), plus satu aturan tambahan khusus API baru ini:

| `ctx.publishMode` | `status` yang dikirim | Panggil endpoint? |
|---|---|---|
| `Live Run - Draft` | `'draft'` | Ya |
| `Live Run - Publish` | `'published'` | Ya |
| `Test` | — | **Tidak — skip node ini sepenuhnya** |

```js
if (ctx.publishMode === 'Test') {
  // Node ini di-skip lewat wiring IF/Switch yang sama dengan cabang
  // "Test skip WordPress" (CP017 §5) — bukan logic baru, cabang yang sudah ada
  // tinggal diarahkan ke node baru ini juga.
}
const status = ctx.publishMode === 'Live Run - Publish' ? 'published' : 'draft';
```

Wiring percabangan **reuse pola CP017 §5** (`Test` → langsung ke Export, skip WordPress) —
tinggal ganti tujuan cabang `Live Run` dari node WordPress lama ke `Code: Build Article API
Payload` → `HTTP: Create Article` baru.

## 4. Payload akhir (bentuk lengkap)

```json
{
  "slug": "akrasia-aristoteles-bertindak-melawan-pengetahuan",
  "title": "Akrasia Aristoteles: Bertindak Melawan Pengetahuan",
  "content": "<p>...</p><blockquote>...</blockquote>...",
  "meta_description": "Aristoteles menyebut kondisi ini akrasia...",
  "category": "Classical & Modern Philosophy",
  "tags": ["philosophy", "aristotle", "akrasia"],
  "faq_schema": [{ "question": "...", "answer": "..." }],
  "status": "draft",
  "geo_score": 83
}
```

## 5. Baca response — `Code: Extract Article API Result`

Beda dari WordPress (`link`/`ID`), response API ini (lihat kontrak lengkap di
`satryapudja-blog/backend/src/index.ts`):

```json
{
  "id": 15, "slug": "akrasia-aristoteles-bertindak-melawan-pengetahuan",
  "title": "...", "status": "draft", "created_at": "2026-...", "...": "..."
}
```

Node pengganti `Code: Extract WP Result`:

```js
const result = $input.first().json;
const articleUrl = `https://satryapudja.site/artikel/${result.slug}/`;
return [{ json: { articleId: result.id, articleSlug: result.slug, articleUrl, articleStatus: result.status } }];
```

Dipakai untuk `Telegram - Send Result` (link artikel) dan `Google Sheets: Log Article + GEO
Score` — pola sama seperti sebelumnya, cuma sumber field-nya beda.

**Catatan `articleUrl` untuk draft**: kalau `status: 'draft'`, halaman itu **belum ter-generate**
di situs statis (backend hanya query `status='published'` saat build) — URL di atas baru
benar-benar hidup setelah artikel di-publish (manual via admin panel, atau run mode
`Live Run - Publish`). Untuk draft, pertimbangkan Telegram message bilang "tersimpan sebagai
draft, review di `/admin`" alih-alih menyodorkan link yang belum jalan.

## 6. Error handling

Endpoint bisa balas:
- `401` — API key salah/hilang. Cek credential header n8n.
- `400` — validasi gagal (`category`, `slug` format, `tags` kosong, `meta_description` >300
  karakter — lihat §3.1/§3.3). **Jangan retry otomatis** kalau 400 — itu kesalahan payload
  permanen, retry tidak akan mengubah hasil (pola sama dengan yang sudah diterapkan di sisi
  backend sendiri untuk fetch build-time, `satryapudja-blog` CP012 T2).
- `409` — slug sudah dipakai artikel lain. Ini bisa terjadi kalau workflow di-run ulang untuk
  topik yang sama — masuk jalur error existing (`Build Failure Context` → alert Telegram),
  **bukan** dianggap sukses diam-diam.
- `5xx` — masalah sesaat di backend/D1. Retry pendek (2-3x) masuk akal di sini.

Semua masuk jalur error path yang sudah ada (`Code: Build Failure Context` → `Google Sheets:
Mark Topic Failed` → `Telegram - Operational Alert`) — tidak perlu node error-handling baru,
cukup diarahkan node HTTP baru ini ke jalur error yang sama seperti node WordPress lama.

## 7. Checklist sebelum wiring nyata

- [ ] `ARTICLE_API_KEY` sudah ditempel di `Credential_information.md` (user, manual)
- [ ] Base URL API situs dikonfirmasi final (bukan asumsi `api.satryapudja.site`)
- [ ] Backend situs sudah bisa diakses dari environment n8n jalan (local↔local kalau masih
      development, atau live kalau sudah Chapter 6 selesai)
- [ ] 3 nama kategori di `content-context.json` dicek ulang masih identik dengan
      `VALID_CATEGORIES` backend situs (§3.3)
- [ ] Slugify deterministik ditambahkan (§3.1) — jangan kirim `meta.slug` mentah
- [ ] Wiring `Test` mode benar-benar skip node ini total (§3.4)
- [ ] Uji end-to-end dulu dengan `status: 'draft'` (aman, tidak publish live), baru coba
      `Live Run - Publish` setelah draft-nya diverifikasi masuk dengan benar ke `/admin`

## 8. Yang di luar scope dokumen ini

- Eksekusi node baru ke workflow n8n sungguhan — itu kerja Planner/Worker project ini
  sendiri, ikuti `rules.md` seperti biasa.
- Perubahan skema/endpoint API situs — kalau nanti kontraknya berubah, dokumen acuan
  utamanya `satryapudja-blog/IMPLEMENTATION_PLAN.md` §2, bukan dokumen ini.
- Deploy Cloudflare Pages/Worker (Chapter 6, project situs) — base URL final baru pasti
  setelah itu selesai.
