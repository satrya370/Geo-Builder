# CP 019 — Revisi GEO: Authority, Freshness, Citation untuk Topik Konseptual

## Latar belakang
Artikel hasil eksekusi 1003 (topik "Man's Search for Meaning", `Language = English`) lolos gate internal kita dengan **83/100 PASS**, tapi saat diuji dengan GEO checker eksternal (pihak ketiga, berbasis AI) hanya dapat **50/100 — "Needs work"**. Selisih 33 poin ini bukan sekadar perbedaan kalibrasi; setelah diaudit, sebagian temuan checker terbukti NYATA dan menunjuk celah sistemik di pipeline, sebagian lain terbukti FALSE POSITIVE karena yang diuji adalah file TXT/docx polos, bukan halaman WordPress ter-render.

Rincian skor checker: Citability 75, Authority 40, Structure 70, Freshness 20, AI Elements 30. Per platform: ChatGPT 60, Perplexity 55, Google AI 40.

Audit ini memisahkan keduanya secara eksplisit supaya kita tidak "memperbaiki" hal yang sudah benar, dan tidak melewatkan hal yang benar-benar rusak.

---

## A. Temuan checker yang TERVERIFIKASI FALSE POSITIVE
Keempat hal ini ditandai ✗ oleh checker, tapi setelah dicek langsung ke node workflow + `config/content-context.json`, semuanya SUDAH ADA di pipeline. Checker tidak bisa melihatnya karena input yang diuji adalah dokumen polos tanpa lapisan HTML/JSON-LD.

1. **"Missing Author"** — `config/content-context.json` sudah terisi lengkap (`name: "Satrya Pudja"`, `jobTitle`, `url`, `bio`, `sameAs`, `wpAuthorId`). `Code: Parse and Validate Schema` menyuntikkannya ke JSON-LD sebagai `author: { @type: 'Person', name, jobTitle, url }`.
2. **"Publication/update date visible ✗"** — `datePublished` dan `dateModified` sudah di-set (`today`) di node yang sama.
3. **"No Schema Markup"** — `Code: Parse and Validate Schema` menghasilkan `@graph` berisi `BlogPosting` + `FAQPage` (kalau `faqSchema.length >= 3`), lalu `Code: Build WordPress Payload` menempelkannya: `'<script type="application/ld+json">' + JSON.stringify(jsonLd) + '</script>'`.
4. **"FAQ section present ✗"** — artikelnya JELAS punya section FAQ dengan 3 pasang Q&A. Checker gagal mendeteksi karena masalah format di file yang diuji, bukan karena FAQ-nya tidak ada (lihat Bug E).

**Konsekuensi penting**: karena empat hal ini sebenarnya sudah ada, skor 50 dari checker BUKAN cerminan jujur kualitas artikel yang ter-publish di WordPress. Tapi ini juga TIDAK berarti aman — lihat §B.

---

## B. Risiko kritis yang BELUM terverifikasi (prioritas verifikasi #1)

**Dugaan: WordPress.com menghapus tag `<script>` dari konten post.**

WordPress.com hosted (bukan self-hosted, bukan plan Business) diketahui melakukan sanitasi tag `<script>` di dalam konten post untuk alasan keamanan. Kalau dugaan ini benar untuk `satryapudja.wordpress.com`, maka **SELURUH schema markup yang workflow hasilkan hilang begitu artikel dipublish** — dan penilaian checker "No Schema Markup" berubah dari false positive menjadi **BENAR untuk halaman nyata**, sekaligus menjelaskan kenapa Google AI cuma dapat 40 (schema adalah sinyal terkuat untuk rich result).

**Status: BELUM TERVERIFIKASI.** Saya sudah coba memverifikasi lewat post ID 10 (hasil eksekusi 992) tapi gagal karena post itu masih berstatus `draft`:
- `GET public-api.wordpress.com/rest/v1.1/sites/satryapudja.wordpress.com/posts/10` → `{"error":"unauthorized","message":"User cannot view post"}`
- Fetch `https://satryapudja.wordpress.com/?p=10` → dapat 46KB HTML (bukan halaman post itu), grep `application/ld+json` / `BlogPosting` / `FAQPage` / `Satrya Pudja` → semuanya tidak ditemukan. Tidak konklusif karena yang ter-fetch bukan halaman draft-nya.

**Cara verifikasi yang benar (WAJIB dilakukan sebelum apa pun di §C dikerjakan):**
1. Jalankan 1x `Run Mode = Live Run - Publish` (atau ubah manual 1 draft yang sudah ada jadi publish lewat WP admin).
2. Fetch halaman publiknya, lalu grep `application/ld+json` dan `BlogPosting`.
3. Kalau HILANG → ini jadi bug prioritas tertinggi di seluruh proyek, dan solusinya bukan di workflow (lihat opsi di bawah).
4. Setelah verifikasi, unpublish/hapus artikel ujinya kalau memang cuma untuk tes.

**Opsi solusi kalau terkonfirmasi ke-strip** (jangan dieksekusi sebelum verifikasi):
- **Opsi 1**: Pakai plugin SEO di WordPress (butuh plan Business) — di luar kendali workflow.
- **Opsi 2**: Ganti pendekatan schema dari `<script>` ke **microdata/RDFa inline** di HTML konten (atribut `itemscope`/`itemprop`), yang tidak disanitasi karena bukan script. Lebih tahan sanitasi, tapi lebih verbose dan hanya mendukung subset schema.org.
- **Opsi 3**: Terima bahwa schema tidak bisa ditanam di WordPress.com plan sekarang, dan alihkan investasi ke sinyal authority yang TIDAK butuh schema (§C Bug A) — sinyal on-page seperti sitasi eksternal, byline terlihat, dan tanggal terlihat di badan artikel.

---

## C. Temuan yang TERVERIFIKASI NYATA (ini yang harus diperbaiki)

### Bug A — Nol sitasi eksternal untuk topik konseptual (root cause sistemik, prioritas tertinggi)

**Bukti dua sistem independen sepakat**: rubrik internal kita memberi `citation` 3.5/8.9 = **39%**, checker eksternal memberi `Authority` **40**. Angka yang hampir identik dari dua metode berbeda — ini temuan paling solid di audit ini.

**Root cause (dilacak sampai ke kode, bukan dugaan):**
`Code: Build AI Writer Body` hanya mewajibkan sitasi untuk ANGKA:
- Rule 5: *"Kamu HANYA boleh menyebut angka/statistik yang ada di FAKTA + SUMBER. DILARANG mengarang angka, tahun, persentase, atau nama studi."*
- Rule 6: *"Setiap angka yang kamu tulis WAJIB disertai sumbernya di kalimat/paragraf yang sama, format markdown link: `[Nama Sumber](url)`."*

Rantai sebabnya untuk topik konseptual:
```
Topik filsafat/psikologi konseptual (= SELURUH niche situs ini)
  → Concept Brief menilai topik tidak butuh empiricalClaims
  → Research tidak jalan / facts kosong
  → Writer tidak boleh menulis angka apa pun (rule 5)
  → Writer tidak menulis angka
  → aturan sitasi (rule 6) TIDAK PERNAH AKTIF
  → artikel selesai dengan NOL sitasi eksternal
```

Ini bukan edge case — ini jalur NORMAL untuk niche situs ini. Artinya arsitektur CP015 (targeted research) secara tidak sengaja membuat kategori artikel terbesar kita structurally tidak bisa punya sitasi.

**Yang hilang**: tidak ada satu pun aturan yang mewajibkan sitasi **karya primer** — padahal untuk artikel tentang Frankl, sitasi paling relevan dan paling mudah diverifikasi justru: `Frankl, V. E. (1946). Man's Search for Meaning`. Artikel ini menyebut Frankl belasan kali, Nietzsche, Auschwitz, amor fati — nol tautan/sitasi untuk semuanya.

**Fix yang diusulkan** — tambahkan kategori sitasi baru yang tidak bergantung pada keberadaan angka:

1. **Di `Code: Build Concept Brief Body`**: minta Concept Brief mengeluarkan field baru `primarySources[]` — daftar karya primer/kanonik yang relevan dengan topik (judul, penulis, tahun terbit). Ini murni pengetahuan kanonik, tidak butuh web search, jadi tidak melanggar prinsip "Concept Brief bahasa-agnostik & tanpa riset" dari CP015.
   ```
   "primarySources": [{ "title": "Man's Search for Meaning", "author": "Viktor E. Frankl", "year": 1946, "relevance": "sumber utama teori logoterapi & kisah kamp konsentrasi" }]
   ```
2. **Di `Code: Build AI Writer Body`**: tambahkan rule baru (JANGAN mengubah rule 5-6 yang sudah ada, itu tetap valid untuk angka):
   ```
   7b. WAJIB menyebut karya primer secara eksplisit minimal 1x per artikel dengan format
       penulis + judul + tahun (mis. "Viktor Frankl dalam Man's Search for Meaning (1946)
       berargumen bahwa ..."). Daftar karya primer ada di [SUMBER PRIMER]. JANGAN mengarang
       judul/tahun yang tidak ada di daftar itu.
   7c. Setiap teori/konsep yang kamu atribusikan ke seorang tokoh WAJIB menyebut nama tokohnya
       secara eksplisit di kalimat yang sama (bukan "para ahli berpendapat"). Ini berlaku untuk
       klaim konseptual, bukan hanya klaim numerik.
   ```
3. **Di `Code: GEO Rule Checker` — `scoreCitationRatio()`**: rubrik sekarang hanya menghitung paragraf yang mengandung `NUMERIC_CLAIM`. Kalau artikel tidak punya angka sama sekali, `withClaim === 0` → skor 0 (nol mutlak, tidak bisa diperbaiki dengan cara apa pun). Ubah supaya kategori `citation` juga menghargai sitasi karya primer:
   ```js
   // Tambahkan di samping perhitungan numerik yang sudah ada:
   const PRIMARY_WORK_CITATION = /\b(\d{4})\)|\(\d{4}\)/;           // pola tahun terbit dalam kurung
   const NAMED_WORK = /\b[A-Z][\w']+(?:\s+[A-Z][\w']+){1,6}\s*\((\d{4})\)/; // Judul Karya (tahun)
   // kalau withClaim === 0, JANGAN langsung return 0 —
   // beri skor berbasis keberadaan sitasi karya primer + atribusi bernama, mis.:
   //   ada >=1 karya primer dengan tahun  -> 6/10
   //   ada >=1 karya primer + >=3 atribusi bernama eksplisit -> 8/10
   //   tidak ada apa pun -> 0/10 (seperti sekarang)
   ```
   Angka persisnya boleh disesuaikan worker, tapi PRINSIPNYA: artikel konseptual yang mengutip karya primer dengan benar TIDAK BOLEH dapat nol di kategori citation, karena itu justru bentuk sitasi yang paling tepat untuk jenis artikel tersebut.

### Bug B — Rubrik freshness kita false-pass (rubrik internal salah, checker benar)

`Code: GEO Rule Checker` → `scoreFreshness()`:
```js
if (numericPara === 0) return { score: 2, reason: 'no numeric paragraphs' };
```
Artikel **tanpa klaim numerik sama sekali dapat nilai PENUH 2/2 (=100%)**. Ini kebalikan dari `scoreCitationRatio()` yang memberi 0 untuk kondisi yang sama — dua fungsi memperlakukan kasus identik dengan cara berlawanan. Salah satu pasti salah, dan yang salah adalah freshness.

Bukti: eksekusi 1003 dapat `freshness` 2.2/2.2 = **100%** dari rubrik kita, sementara checker eksternal memberi Freshness **20**. Untuk artikel yang benar-benar tidak punya satu pun penanda waktu, 100% jelas tidak masuk akal.

Akar konseptualnya: rubrik kita hanya mengukur satu hal sangat sempit — *"apakah klaim numerik punya tahun di dekatnya"*. Rubrik kita TIDAK PERNAH mengukur apa yang sebenarnya dinilai sebagai freshness oleh mesin/AI crawler:
- adakah tanggal publish/update yang terlihat di halaman,
- apakah sumber yang dirujuk relatif baru,
- adakah penanda temporal apa pun yang bisa dibaca crawler.

**Fix yang diusulkan:**
1. Hapus jalan pintas `numericPara === 0 → skor penuh`. Untuk artikel tanpa klaim numerik, nilai freshness dari sinyal lain, bukan diberi gratis.
2. Tambahkan komponen "penanda temporal eksplisit": cek keberadaan tahun terbit karya yang dirujuk (`(1946)`), rujukan periode ("Hellenistik", "pasca-Perang Dunia II"), atau tanggal update. Beri skor proporsional.
3. **Yang lebih penting dari rubrik**: pastikan tanggal publish benar-benar TERLIHAT di halaman WordPress (bukan hanya di JSON-LD yang mungkin ke-strip, lihat §B). Ini setting tema WordPress, di luar workflow — perlu dicek manual apakah tema situs menampilkan tanggal post.

### Bug C — Draft tidak punya H1

Draft dari `Code: Parse Draft` selalu dimulai dari `## `, tidak pernah ada `# ` (H1). Untuk WordPress ini BENAR (judul post otomatis jadi H1, kalau ditambah `#` di konten malah jadi dua H1 = masalah SEO). Tapi untuk export mandiri (docx/Google Docs/TXT), dokumen jadi tanpa H1 sama sekali.

**Fix**: jangan ubah draft-nya. Tambahkan H1 hanya di jalur EXPORT (`Code: Convert Draft to HTML` dari CP018 sudah melakukan ini dengan `<h1>${meta.title}</h1>` — pastikan tetap begitu dan tidak diduplikasi ke jalur WordPress).

### Bug D — Kutipan sintetis disajikan sebagai kutipan asli (masalah integritas, kesalahan saya)

Draft asli dari n8n menandai **6 dari 6 kutipan** dengan `⚠️ **[PERLU CEK MANUAL]** Kutipan/sitasi di bawah ini tidak ditemukan lewat verifikasi otomatis` — sistem verifikasi CP016 bekerja dengan benar, mendeteksi bahwa kutipan-kutipan itu tidak bisa diverifikasi ke sumber mana pun.

Saat saya merapikan artikel jadi `Logotherapy_Meaning_of_Life_FINAL_CLEAN.txt`, **saya menghapus semua penanda peringatan itu** dan membiarkan kutipannya tetap dalam tanda kutip. Akibatnya 5 pull-quote di versi bersih (dan di docx) sekarang TAMPAK seperti kutipan otoritatif Frankl, padahal sebenarnya parafrase hasil AI. Contoh:

> "Logotherapy, as developed by Viktor Frankl, posits that the primary human drive is the search for meaning, not pleasure or power."

Ini bukan kalimat Frankl — ini rangkuman AI yang diberi tanda kutip. Kalau pembaca mencoba melacak kutipan ini, tidak akan ketemu. Ini justru **menurunkan** authority, bukan menaikkan, dan berisiko merusak kredibilitas situs — persis risiko yang sudah diperingatkan sendiri di `content-context.json` (*"False credential claims risk damaging site credibility if checked by readers"*).

**Fix, pilih salah satu (jangan dibiarkan seperti sekarang):**
- **Opsi 1 (disarankan)**: ubah semua pull-quote jadi kalimat rangkuman TANPA tanda kutip (mis. sebagai kalimat penegas ber-bold atau callout box), sehingga jujur sebagai rangkuman penulis, bukan kutipan.
- **Opsi 2**: ganti dengan kutipan Frankl yang BENAR-BENAR verbatim dari *Man's Search for Meaning*, lengkap dengan nomor halaman/edisi — butuh verifikasi manual ke sumber asli.
- **Opsi 3**: hapus pull-quote sepenuhnya.

**Aturan proses yang harus dipegang ke depan**: penanda `⚠️ PERLU CEK MANUAL` dari CP016 TIDAK BOLEH dihapus tanpa menyelesaikan masalah yang ditandainya. Menghapus penanda ≠ memperbaiki kutipan.

### Bug E — Defect format di docx hasil ChatGPT

Diverifikasi langsung dari `word/document.xml` di `Logotherapy_and_the_Meaning_of_Life.docx`:
- Style terpakai: `Heading1` ×1, `Heading2` ×7, `PullQuote` ×5, **`Heading3` ×0**
- 8 gambar tertanam (`word/media/image1-8.png`, 118–179 KB) ✓ sesuai permintaan

Dua defect nyata:
1. **Pertanyaan FAQ tidak jadi Heading 3, dan pertanyaan menempel ke jawaban tanpa pemisah apa pun.** Contoh persis dari paragraf 40: `"Does logotherapy require you to find meaning even when things are unbearably hard?Logotherapy holds that meaning can be found..."` — tidak ada spasi/baris baru antara `?` dan kata berikutnya. Ini kemungkinan besar penyebab checker menandai "FAQ section present ✗": secara struktural FAQ-nya tidak terbaca sebagai FAQ.
2. **`Meta Description` dan `Slug` ikut masuk badan artikel** (paragraf 2–5) sebagai teks biasa. Ini metadata internal yang bocor ke konten — harus dibuang sebelum dipaste ke WordPress.

**Fix**: perbaiki prompt ChatGPT-nya (tegaskan pertanyaan FAQ HARUS Heading 3, dan metadata title/meta description/slug JANGAN dimasukkan ke badan dokumen — cukup dipakai untuk judul dokumen saja), atau rapikan manual 2 hal itu di docx-nya.

---

## D. Yang HARUS diverifikasi (jangan asumsi)
1. **`<script>` JSON-LD bertahan atau tidak di WordPress.com** — 1x Live Publish + fetch halaman publik + grep `application/ld+json`. **Ini prioritas #1**, karena hasilnya menentukan apakah §B Opsi 1/2/3 perlu dikerjakan sama sekali.
2. Apakah tema WordPress situs menampilkan **tanggal publish** dan **byline penulis** secara terlihat di halaman artikel (bukan hanya di JSON-LD).
3. Setelah fix Bug A: jalankan ulang topik konseptual (mis. topik Frankl yang sama) dan pastikan artikel hasilnya mengandung minimal 1 sitasi karya primer dengan tahun, dan `citation` di rubrik internal naik dari 39%.
4. Setelah fix Bug B: pastikan artikel tanpa klaim numerik TIDAK lagi dapat freshness 100%.
5. Setelah semua fix: uji ulang di GEO checker eksternal yang sama, dengan input **halaman WordPress ter-publish** (bukan TXT/docx), supaya perbandingannya adil. Catat skornya untuk dibandingkan dengan baseline 50.
6. Jangan mengejar skor checker secara buta — checker itu juga alat berbasis AI dan sudah terbukti salah di 4 poin (§A). Perlakukan sebagai sinyal, bukan sebagai kebenaran.

## E. Definition of done
- [ ] Verifikasi survival `<script>` JSON-LD di WordPress.com terpublish (§D.1) — hasilnya dicatat di log, apa pun hasilnya
- [ ] `Code: Build Concept Brief Body` mengeluarkan `primarySources[]` (judul, penulis, tahun, relevance)
- [ ] `Code: Build AI Writer Body` punya rule 7b (wajib sebut karya primer + tahun) dan 7c (atribusi konseptual wajib bernama)
- [ ] `Code: GEO Rule Checker` → `scoreCitationRatio()` tidak lagi memberi 0 mutlak untuk artikel konseptual yang mengutip karya primer dengan benar
- [ ] `Code: GEO Rule Checker` → `scoreFreshness()` tidak lagi memberi skor penuh untuk artikel tanpa klaim numerik
- [ ] Pull-quote sintetis di artikel Frankl ditangani (pilih Opsi 1/2/3 di Bug D) — tidak ada lagi parafrase AI yang tampil sebagai kutipan verbatim
- [ ] Docx: pertanyaan FAQ jadi Heading 3 dengan pemisah yang benar; metadata (meta description/slug) dibuang dari badan dokumen
- [ ] 1 eksekusi topik konseptual pasca-fix, dibandingkan skornya vs baseline (internal 83 / eksternal 50)
- [ ] `ai_docs/index.md` diperbarui (baris 019 baru)
