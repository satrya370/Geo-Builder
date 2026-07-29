# Log CP 009 — Schema & WordPress Publish (Node 37–39, 39b, 40–42)

**Mengikuti:** `009_schema-dan-wordpress-publish_plan.md`
**Mode:** Worker

## Blok A — Schema (node 37–39, 39b)

### Node built

| Node | Name | Status |
|------|------|--------|
| 37 | `Code: Build AI Schema Body` | ✅ prompt = `PROMPT_GUIDE.md` §6 persis, inject draft + context + categories/tags |
| 38 | `HTTP: AI Schema Generator` | ✅ Ollama `qwen2.5:7b-instruct-q4_K_M`, `format: json`, no credential |
| 39 | `Code: Parse and Validate Schema` | ✅ parse JSON + JSON-LD deterministik (PROMPT_GUIDE.md §6 code), fallback ke metadata dari draft saat parse gagal |
| 39b | `Code: Validate Metadata Rules` | ✅ `scoreMetadata()` dari rubric Kategori 6, Kategori 6 cek terpisah (keputusan CP 005) |

### Koneksi
`Switch (31) PASS → Build Schema Body (37) → HTTP Schema (38 main+error) → Parse Schema (39) → Validate Metadata (39b)`

## Blok B — WordPress Publish (node 40–42)

| Node | Name | Status |
|------|------|--------|
| 40 | `Code: Build WordPress Payload` | ✅ map categories/tags to numeric IDs (case-insensitive), inject JSON-LD as `<script>`, status: **`draft`** |
| 41 | `HTTP: WordPress Create Post` | ✅ `POST .../sites/satryapudja.wordpress.com/posts/new`, credential `q0ZOvF43b2FlIsLj` (WordpressApi, oAuth2Api) |
| 42 | `Code: Extract WP Result` | ✅ ambil `URL`/`ID` dari response WordPress.com REST API |

### Koneksi
`Validate Metadata (39b) → Build WP Payload (40) → WP Create Post (41) → Extract Result (42)`

## Compliance

| Aturan | Status |
|--------|--------|
| Schema prompt = PROMPT_GUIDE.md §6 persis | ✅ |
| JSON-LD deterministik (bukan dari model) | ✅ |
| Fallback tidak throw (artikel lolos ≠ gagal publish) | ✅ |
| Node 39b = scoreMetadata() dari rubric | ✅ |
| Kategori/tag map ke ID case-insensitive | ✅ |
| JSON-LD semat di content sebagai `<script>` | ✅ |
| status: `draft` (belum `publish`) | ⚠️ STOP POINT 2 |
| Credential node 41 (WordpressApi) via genericCredentialType | ✅ |
| Koneksi 31(PASS)→37→38→39→39b→40→41→42 utuh | ✅ |

## STOP POINT 2 — Butuh konfirmasi user

- `status: 'draft'` di node 40 — post TIDAK akan published otomatis
- User harus verifikasi post draft muncul benar di wp-admin (kategori, tag, JSON-LD)
- Setelah lolos verifikasi manual, ubah `'draft'` → `'publish'` di node 40

## Status

**done** (Blok A+B) — 7 node + 8 koneksi built. Blok C (uji end-to-end) menunggu konfirmasi user untuk test publish.