# CP 009 — Schema/Metadata + WordPress Publish (Node 37–39, 39b, 40–42)

**Mode:** Planner · **Asal:** `ai_docs/todos.md` — item "Bangun node 37–39: Schema/Metadata"
dan item "Bangun node 40–42: WordPress publish", **digabung jadi satu CP atas instruksi user**
(Schema → Publish adalah alur linear tanpa keputusan cabang di antaranya, jadi masuk akal
dikerjakan sekaligus)

**Dokumen rujukan wajib dibaca sebelum eksekusi:**
`docs/PROMPT_GUIDE.md` §6 (prompt Schema Generator + kode JSON-LD deterministik, copy PERSIS) ·
`docs/GEO_SCORING_RUBRIC.md` §Kategori 6 (`scoreMetadata`) dan §"Kategori 6 — timing khusus"
(keputusan CP 005 — kenapa node 39b ada) · `docs/IMPLEMENTATION_PLAN.md` §3 Fase 5, Fase 6,
§1.1 (mekanisme WordPress.com — OAuth2, bukan Basic Auth), §3.5 (pola credential httpRequest) ·
`config/content-context.json` blok `wordpress` (categories/tags dengan ID asli) dan `author` ·
workflow `rwIbdIkIhoVE8nkG` (state setelah CP 007 — cek `$('Code: Compute GEO Score')` /
`Switch - GEO Score Gate` benar-benar ada dan cabang `PASS` yang jadi titik sambung node 37)

## Kenapa digabung aman (bukan sekadar hemat waktu)

Schema (37–39b) dan Publish (40–42) adalah **alur linear tanpa percabangan keputusan** — beda
dari mis. Critic+Gate (CP 007) yang punya 3 cabang, atau Revise (CP 008 — belum dibangun,
lihat dependency di bawah) yang punya loop. Menggabungkan keduanya tidak menambah risiko
arsitektural, cuma menambah jumlah node yang dibangun dalam satu sesi.

## Dependency yang perlu disadari SEBELUM mulai

- **CP 008 (Revise loop, node 32–36) belum dibangun.** Node 37 di CP ini nyambung dari cabang
  `PASS` (`Switch - GEO Score Gate`, node 31) yang **sudah ada** (dibangun CP 007, dibiarkan
  menggantung). CP ini **tidak perlu menunggu CP 008** — cabang `PASS` berdiri sendiri, tidak
  bergantung pada Reviser. Cabang `REVISE`/`REJECT` tetap dibiarkan menggantung sampai CP 008
  dan CP error-path (node 51+) ada.
- **Draft final harus diambil via referensi bernama**, bukan asumsi pass-through dari node 31 —
  pola context-hilang yang sudah 2× jadi bug (CP 004, dan berpotensi lagi di sini). Ambil
  `$('Code: Parse Draft').first().json.draft` untuk teks artikel, dan
  `$('Code: Build Run Context').first().json` untuk `contentContext`/`targetKeyword`.

## Ringkasan CP (gambaran besar)

- **Confidence:** medium — prompt Schema & kode JSON-LD sudah lengkap persis di
  `PROMPT_GUIDE.md` §6, tapi node 39b (`scoreMetadata`) dan integrasi WordPress.com OAuth2
  (endpoint non-standar, lihat §1.1) belum pernah dites end-to-end.
- **Risk:** medium-high — node 41 adalah **publish nyata ke situs live** (WordPress.com),
  bukan simulasi. Kesalahan payload bisa menghasilkan post publik yang salah kategori/rusak
  formatnya. Mitigasi: uji dulu dengan `status: 'draft'` sebelum ubah ke `'publish'` (lihat
  step 9).
- **Token/limit usage estimate:** low untuk build (1 AI call lokal/gratis di node 38), tapi
  **medium saat uji end-to-end** (jalan lewat draft asli hasil pipeline penuh).
- **Kesimpulan:** risk medium-high + confidence medium → **tidak** memenuhi syarat split
  sub-CP A/B (butuh risk tinggi DAN confidence rendah bersamaan). Tapi karena ini publish ke
  situs live pertama kali, step **di-chunk 3 blok** dengan 2 STOP POINT eksplisit.

## Langkah

### Blok A — Schema (node 37–39, 39b)

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Node 37 `Code: Build AI Schema Body` — prompt copy PERSIS dari `PROMPT_GUIDE.md` §6. `finalDraft` dari `$('Code: Parse Draft')`, `targetKeyword` dari `$('Code: Parse Outline').resolvedTargetKeyword` atau run context, `categories`/`tags` dari `$('Code: Build Run Context').contentContext.wordpress` (ID sudah asli, lihat CP 002), `author` dari `contentContext.author` | high | low | low |
| 2 | Node 38 `HTTP: AI Schema Generator` — POST `http://localhost:11434/api/chat`, model `qwen2.5:7b-instruct-q4_K_M`, `format: "json"`, `stream: false` (pola identik node 21 Outline — **tidak** butuh credential, localhost) | high | low | low |
| 3 | Node 39 `Code: Parse & Validate Schema` — parse JSON meta, bangun JSON-LD deterministik persis kode di `PROMPT_GUIDE.md` §6. **Fallback wajib**: kalau parse gagal atau `metaDescription` di luar 120–160 char → generate versi deterministik (judul dari H1 draft, meta description dari paragraf pertama dipotong batas kata). **Jangan throw** — artikel yang lolos scoring tidak boleh gagal publish karena model 7B keliru format | medium | medium | low |
| 4 | Node 39b `Code: Validate Metadata Rules` — implementasi `scoreMetadata()` dari `GEO_SCORING_RUBRIC.md` §Kategori 6 PERSIS. Kalau `pct < 70` → panggil ulang logic fallback deterministik yang sama dengan step 3 (bukan logic baru, reuse), replace `meta`/`jsonLd` dengan versi fallback. **Bukan gate** — selalu lanjut ke node 40 apa pun hasilnya | medium | low | low |

> **STOP POINT 1** — verifikasi Blok A dulu (via `get_workflow_details` + tes manual dengan
> 1 draft) sebelum lanjut ke Blok B yang menyentuh publish live.

### Blok B — WordPress Payload & Publish (node 40–42)

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 5 | Node 40 `Code: Build WordPress Payload` — map `meta.categories`/`meta.tags` (nama) ke ID numerik dari `contentContext.wordpress.categories`/`.tags` (kalau nama tidak cocok persis → pakai `defaultCategoryId`, JANGAN biarkan `undefined` terkirim ke API). Sisipkan JSON-LD sebagai `<script type="application/ld+json">...</script>` di **akhir** `content` (bukan field terpisah — WordPress.com REST API tidak punya slot schema, lihat `IMPLEMENTATION_PLAN.md` §1.1). `status`: **`'draft'` dulu untuk uji** (lihat step 9), baru `'publish'` setelah lolos verifikasi manual | medium | medium | low |
| 6 | Node 41 `HTTP: WordPress Create Post` — POST `https://public-api.wordpress.com/rest/v1.1/sites/satryapudja.wordpress.com/posts/new`. Auth: `genericCredentialType` + `genericAuthType: "oAuth2Api"`, credential **`WordpressApi`** (id `q0ZOvF43b2FlIsLj`, sudah ada dan sudah connect — jangan buat baru). **WAJIB** ikuti pola §3.5 — sematkan `credentials` langsung di `addNode` (bukan `setNodeCredential` — riwayat CP 006 menunjukkan itu kadang ditolak MCP untuk `httpRequest`, walau agent lain berhasil pakai `updateNode`; coba `setNodeCredential` dulu, kalau ditolak fallback ke inline `credentials` di `addNode`) | medium | **high** | low |
| 7 | Node 42 `Code: Extract WP Result` — ambil `link`/`URL` (WordPress.com REST API balas field `URL`, bukan `link` seperti self-hosted — **cek response asli**, jangan asumsi nama field dari dokumentasi self-hosted) + `ID` untuk log & notifikasi Telegram nanti | high | low | low |

> **STOP POINT 2** — setelah node 40–42 jadi, **JANGAN langsung ubah `status` ke `'publish'`
> dan trigger run sungguhan** tanpa konfirmasi user (ini publish ke situs live pribadi user,
> di luar wewenang Worker untuk memutuskan sendiri kapan "siap publish beneran").

### Blok C — Uji

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 8 | Uji node 37–39b dengan 1 draft uji manual (boleh reuse draft dari CP 005/007) — verifikasi JSON-LD valid (field wajib ada), fallback jalan saat sengaja dirusak (mis. paksa `metaDescription` kosong) | medium | low | low |
| 9 | Uji node 40–42 dengan `status: 'draft'` dulu — **konfirmasi ke user** post draft-nya muncul benar di wp-admin (kategori benar, JSON-LD ada di source, tidak ada artikel published tak sengaja) SEBELUM mengubah default ke `'publish'` | medium | high | medium |
| 10 | Verifikasi via `get_workflow_details`: koneksi `31(PASS)→37→38→39→39b→40→41→42` utuh, credential node 41 tertaut, tidak ada koneksi sementara ke node yang belum dibangun (46+) | high | low | low |
| 11 | Tulis log, update `index.md`, coret 2 item `todos.md` (37–39 dan 40–42) sekaligus, arahkan keduanya `-> 009` | high | low | low |

## Catatan teknis

- **WordPress.com REST API field names beda dari self-hosted** (`URL` bukan `link`,
  kemungkinan struktur response lain juga beda) — jangan asumsikan dari training data soal
  wp-json self-hosted, cek response asli dari 1 test call dulu.
- **Kategori/tag mapping (node 40) harus match-by-name dari `meta.categories`/`meta.tags` (yang
  dipilih AI Schema Generator) ke ID di `content-context.json`** — kalau AI Schema pilih nama
  yang tidak persis sama (typo/beda kapitalisasi), payload akan kirim ID yang salah/kosong.
  Pertimbangkan normalisasi case-insensitive saat matching.
- **`status: 'draft'` di step 5/9 adalah pagar keamanan, bukan preferensi** — publish pertama
  kali ke situs live milik user harus diverifikasi manual dulu.
- **Tidak menyentuh:** node 32–36 (Revise, CP 008 — terpisah), node 46–54 (logging/notif/error,
  CP terpisah setelah ini).

## Definition of done

- [ ] Node 37–39, 39b, 40–42 ada di workflow `rwIbdIkIhoVE8nkG`, koneksi utuh dari cabang
      `PASS` node 31
- [ ] Fallback deterministik (step 3) teruji jalan tanpa throw saat model lokal gagal format
- [ ] Node 39b (`scoreMetadata`) menghasilkan skor 0–10 sesuai formula rubric, trigger fallback
      dengan benar saat `pct < 70`
- [ ] Kategori/tag ter-mapping ke ID numerik yang benar (bukan `undefined`/`defaultCategoryId`
      keliru)
- [ ] JSON-LD tersemat di `content` sebagai `<script>` tag, tervalidasi field wajib ada
- [ ] Credential node 41 (`WordpressApi`) terverifikasi tertaut via `get_workflow_details`
- [ ] Uji publish `status: 'draft'` dikonfirmasi user SEBELUM default diubah ke `'publish'`
- [ ] Log ditulis, `index.md` baris CP 009 ditambahkan, `todos.md` (2 item) dicoret `-> 009`

## Next mode

Worker — brief dulu sebelum eksekusi (§3 `rules.md`). Berhenti di STOP POINT 1 (antar Blok A/B)
kalau token mepet; **wajib** berhenti di STOP POINT 2 untuk konfirmasi user sebelum publish
live sungguhan, ini bukan opsional.
