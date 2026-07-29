# CP 016 — Citation & Quote Verification (versi minimal, bukan RAG penuh)

## Latar belakang

Setelah riset mendalam ke 4 model AI berbeda soal cara mencegah halusinasi di pipeline ini (lihat file `Chatgpt_research`, `Glm_Research`, `deepseek_Research` di root proyek, plus 1 hasil dari Claude), 3 dari 4 sumber konvergen merekomendasikan arsitektur verifikasi berat: pecah draft jadi atomic claims, klasifikasi tipe klaim, route ke 5+ API berbeda, post-draft re-audit, reject/revise loop otomatis.

Sumber ke-4, diminta secara eksplisit menantang premis itu, menunjukkan cacatnya: proyek ini **sudah punya gerbang manusia** (artikel publish sebagai WordPress **draft**, bukan langsung live — selalu ada satu orang yang baca sebelum benar-benar terbit). Literatur yang dirujuk 3 sumber lain (FActScore, SelfCheckGPT, CoVe, FacTool, RARR) semuanya mengevaluasi efektivitas otomasi dengan asumsi **tidak ada** manusia yang membaca output sebelum publikasi — target use case mereka adalah produksi skala besar tanpa review (chatbot, search engine). Menerapkan bukti itu ke situasi kita, yang sudah punya human-in-the-loop, adalah category error.

**Bukti dari data kita sendiri mengonfirmasi diagnosis ini**: semua kegagalan fabrikasi nyata yang ditemukan selama testing ada di kategori kutipan/atribusi — kutipan Aristoteles yang tidak pernah ada (test `gpt-5-nano`), penyisipan "DSM-5" dan "ADHD" yang tidak ada di data manapun yang diberikan (test `deepseek-v4-flash`, `big-pickle`). **Nol** kegagalan ditemukan di kategori "salah paham konsep filsafat" — kategori itu memang lebih aman diserahkan ke pengetahuan parametrik model + manusia yang baca draft (kesalahan konseptual filsafat cenderung "terasa salah" saat dibaca orang yang familiar dengan topiknya; kutipan/sitasi yang dikarang secara plausible — nama benar, tahun masuk akal — tidak bisa dibedakan dari asli tanpa cek sumber, berapa pun telitinya orang yang baca).

## Kenapa BUKAN arsitektur penuh yang direkomendasikan 3 sumber lain

- **Effort vs manfaat timpang**: arsitektur penuh (claim extractor + retrieval router 5+ API + 3-way critic split + post-draft audit) realistis 3-6 minggu kerja + maintenance berkelanjutan (API berubah endpoint, rate limit, format drift) untuk satu operator. Kalau kutipan palsu cuma muncul di 1 dari 15-20 artikel, itu rasio effort:benefit yang buruk.
- **Setiap node tambahan = titik gagal baru** yang harus di-debug sendiri tanpa tim, di platform low-code (n8n) yang tidak didesain untuk orkestrasi kompleks semacam ini.
- **Blind spot konteks Indonesia** (ditemukan lewat riset ini, tidak disentuh 3 sumber lain): API yang direkomendasikan (Wikidata, Semantic Scholar, OpenLibrary, PubMed, SEP) semuanya berbasis indeks Anglo-internasional. Kalau logic-nya "reject hanya jika terkontradiksi", klaim palsu tentang tokoh/data Indonesia akan lolos sambil memberi rasa aman palsu ("sudah lewat pipeline verifikasi").

## Desain: verifikasi 2 kelas saja, tanpa reject/revise otomatis

```
Code: Build AI Writer Body (Writer diminta output quotes[]/citations[] terstruktur)
  → HTTP: AI Writer
  → Code: Parse Draft (extend: ambil juga quotes[]/citations[] dari JSON, bukan cuma markdown)
  → Code: Verify Quotes And Citations (SATU Code node, HTTP call internal via this.helpers.httpRequest)
  → Code: Annotate Draft With Warnings
  → (lanjut ke GEO Rule Checker seperti biasa — TIDAK ada gate/reject baru di sini)
```

**Keputusan desain penting**: verifikasi dilakukan lewat **1 Code node** yang memanggil API dari dalam kode (`this.helpers.httpRequest`), BUKAN lewat node `HTTP Request` terpisah per API. Alasan: `quotes[]`/`citations[]` panjangnya berubah-ubah per artikel (bisa 0, bisa 5) — kalau dipecah ke node `HTTP Request` biasa, n8n otomatis menjalankan node itu SEKALI PER ITEM yang masuk, yang berarti butuh langkah split-jadi-N-item lalu merge-balik-jadi-1-item, dan draft (yang cuma 1 item) harus disatukan lagi dengan N hasil verifikasi — kompleksitas percabangan/merge ini sendiri sumber bug baru. Dengan 1 Code node yang loop di dalam kode, draft tetap 1 item dari awal sampai akhir, tidak ada percabangan sama sekali.

Tidak ada node classifier terpisah, tidak ada retrieval router, tidak ada reject/revise loop baru. Verifikasi ini murni **cosmetic annotation** — hasilnya cuma menyuntik warning ke draft, tidak menghentikan pipeline atau memicu revisi otomatis.

### 1. `Code: Build AI Writer Body` — tambahan output contract
Selain markdown draft seperti sekarang, prompt Writer diminta tambahkan section akhir berisi JSON (dipisah delimiter jelas supaya gampang diekstrak):
```
---METADATA---
{
  "quotes": [
    {"text": "<kutipan verbatim persis seperti ditulis di draft>", "attributedTo": "<nama tokoh>"}
  ],
  "citations": [
    {"type": "book|paper", "title": "...", "author": "..."}
  ]
}
```
Instruksi tambahan ke prompt: "Kalau TIDAK ADA kutipan bertanda kutip di draft, `quotes` HARUS array kosong `[]` — JANGAN mengarang entry supaya field ini terisi. Sama untuk `citations`."

### 2. `Code: Parse Draft` — extend
Split response jadi 2 bagian di delimiter `---METADATA---`: bagian atas tetap markdown draft (behavior lama tidak berubah), bagian bawah di-parse JSON (pakai pola robust `{`...`}` extraction yang sudah ada di node Parse lain). Kalau parsing metadata gagal → treat sebagai `{quotes: [], citations: []}`, JANGAN sampai gagal parse metadata menghentikan draft yang sudah berhasil ditulis.

### 3. Node baru: `Code: Verify Quotes And Citations`
Kode lengkap ada di prompt untuk worker (lihat jawaban chat) — ringkasannya:
- Ambil `quotes[]`/`citations[]` dari output `Code: Parse Draft`.
- Kalau keduanya array kosong → langsung `return` tanpa panggilan API apa pun (jalur tercepat, harus jadi kasus paling umum untuk artikel opini filsafat).
- Untuk tiap quote: `this.helpers.httpRequest` ke **Google Books API**, fallback ke **Wikiquote API** kalau nihil.
- Untuk tiap citation: `type==="book"` → **OpenLibrary**, `type==="paper"` → **Semantic Scholar** (WAJIB sertakan `fields=title,authors,year` di query — tanpa itu backend mereka kadang balas `500`, sudah dikonfirmasi via test langsung 2026-07-27).
- API key dibaca dari **environment variable n8n** (`$env.GOOGLE_BOOKS_API_KEY`, `$env.SEMANTIC_SCHOLAR_API_KEY`) — BUKAN n8n Credential object, supaya tidak kena masalah `setNodeCredential`/inline `credentials` yang sudah berkali-kali gagal lewat MCP untuk node `httpRequest` (lihat CP 008, CP 011). Wikiquote & OpenLibrary tidak butuh key sama sekali.
- Semua hasil (`verified: true/false` per item) ditempel balik ke json, draft tetap 1 item, diteruskan ke node berikutnya.

### 6. Node baru: `Code: Annotate Draft With Warnings`
Untuk tiap quote/citation dengan `verified: false`, cari teks itu persis di draft markdown, sisipkan penanda tepat sebelum kemunculannya:
```
> ⚠️ **[PERLU CEK MANUAL]** Kutipan/sitasi di bawah ini tidak ditemukan lewat verifikasi otomatis — mohon cek keasliannya sebelum publish.
```
Draft yang sudah dianotasi ini yang diteruskan ke node berikutnya (GEO Rule Checker dst. — **tidak ada perubahan** di node-node itu, mereka tetap menilai draft seperti biasa; anotasi warning ini murni untuk mata manusia yang nanti buka WordPress draft).

**Kebijakan Indonesia**: kalau `citations[]`/`quotes[]` menyebut entitas yang tampak lokal Indonesia (heuristik sederhana: bukan huruf Latin standar internasional, atau nama domain/organisasi yang dikenali Indonesia) dan tidak ketemu di API manapun, JANGAN treat beda dari kegagalan verifikasi biasa — tetap kasih warning yang sama. Jangan menambah API Indonesia khusus di tahap ini (over-engineering untuk manfaat yang belum terbukti dibutuhkan) — cukup pastikan kebijakan "tidak ketemu = wajib cek manual" berlaku sama rata, bukan di-skip untuk kasus lokal.

### 7. TIDAK ADA perubahan di sini (penting, supaya scope tidak melebar)
- `Code: Compute GEO Score` — tidak berubah karena CP016.
- `Code: Build AI Critic Body` — tidak dipecah jadi 3 (usulan GPT) — di luar scope, terlalu besar untuk manfaat yang belum terbukti perlu.
- Tidak ada reject/revise loop baru — kalau semua quote/citation gagal verifikasi, draft tetap lanjut ke tahap berikutnya seperti biasa, cuma dengan anotasi warning.
- Tidak ada Wikidata/PubMed/SEP untuk verifikasi konsep filsafat — area itu diserahkan ke pengetahuan parametrik Concept Brief (CP015) + manusia yang baca draft.

## API yang perlu di-setup

| API | Untuk apa | Perlu API key? | Rate limit gratis |
|---|---|---|---|
| **Google Books API** | Verifikasi kutipan verbatim (exact phrase search) | Ya, tapi gratis — daftar di Google Cloud Console, aktifkan "Books API" | 1000 request/hari tanpa billing aktif |
| **Wikiquote API** | Fallback verifikasi kutipan terkenal | Tidak perlu key | Tidak ada limit ketat untuk pemakaian wajar |
| **OpenLibrary API** | Verifikasi keberadaan buku | Tidak perlu key | Tidak ada limit resmi, tapi jangan spam (n8n otomatis 1 artikel = beberapa request saja) |
| **Semantic Scholar API** | Verifikasi keberadaan paper akademik | Tidak wajib, tapi disarankan daftar API key gratis (menaikkan rate limit) | 100 request/5 menit tanpa key; jauh lebih tinggi dengan key gratis |

Detail cara setup ada di jawaban chat, bukan di plan doc ini (supaya plan tetap fokus ke arsitektur, bukan tutorial akun).

## Yang HARUS diverifikasi (jangan asumsi)
1. **Test dulu apakah Writer bisa konsisten mengeluarkan `quotes[]`/`citations[]` kosong ketika memang tidak ada kutipan** — risiko: model mungkin "mengarang" entry supaya field terisi (pola fabrikasi yang sama seperti masalah awal). Kalau ini terjadi, prompt perlu diperkuat lagi.
2. **Test parsing delimiter `---METADATA---`** — pastikan tidak bentrok kalau draft markdown kebetulan mengandung teks serupa.
3. **Test dengan draft yang MEMANG punya kutipan asli** (cari topik dengan kutipan terkenal yang benar-benar terverifikasi) untuk pastikan jalur "verified: true" juga bekerja, bukan cuma jalur gagal.
4. **Cek biaya waktu tambahan** — tiap artikel sekarang nambah beberapa HTTP call sekuensial; ukur berapa detik tambahan ke total runtime pipeline (target: tidak lebih dari beberapa detik, karena ini API ringan, bukan AI call).

## Setup environment variable n8n (WAJIB sebelum node baru bisa jalan)
API key TIDAK memakai n8n Credential object — dibaca via `$env` di dalam Code node. Tambahkan ke environment n8n (file `.env` / docker-compose environment, tergantung cara n8n dijalankan) lalu **restart n8n**:
```
GOOGLE_BOOKS_API_KEY=<isi dari Credential_information.md, baris "Google credential">
SEMANTIC_SCHOLAR_API_KEY=<isi dari Credential_information.md, baris "SEMANTIC_SCHOLAR_API_KEY=...">
```
Kalau n8n dikonfigurasi dengan `N8N_BLOCK_ENV_ACCESS_IN_NODE=true`, `$env` tidak akan bisa diakses dari Code node — cek dulu variabel ini tidak di-set (atau di-set `false`) sebelum lanjut.

## Definition of done
- [ ] Environment variable `GOOGLE_BOOKS_API_KEY` + `SEMANTIC_SCHOLAR_API_KEY` sudah ditambahkan ke n8n dan n8n sudah di-restart, dikonfirmasi `$env.GOOGLE_BOOKS_API_KEY` terbaca (test print di Code node sementara, jangan sampai ke-commit)
- [ ] `Code: Build AI Writer Body` mengeluarkan `quotes[]`/`citations[]` (array kosong kalau memang tidak ada)
- [ ] `Code: Parse Draft` extend untuk split markdown + metadata JSON
- [ ] `Code: Verify Quotes And Citations` — skip semua API call kalau kedua array kosong; Google Books+Wikiquote untuk quotes; OpenLibrary+Semantic Scholar (dengan `fields=...`) untuk citations
- [ ] `Code: Annotate Draft With Warnings` menyisip warning inline yang tepat lokasinya
- [ ] Draft yang sudah dianotasi tetap lanjut normal ke GEO Rule Checker (tidak ada reject/revise baru)
- [ ] Minimal 2 eksekusi live diverifikasi: 1 draft tanpa kutipan (jalur cepat, 0 API call), 1 draft dengan kutipan buatan untuk test jalur verifikasi (baik verified maupun tidak)
- [ ] `ai_docs/index.md` diperbarui + log ditulis di `016_citation-quote-verification_log.md`
