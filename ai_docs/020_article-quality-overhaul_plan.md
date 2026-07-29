# CP 020 — Overhaul Kualitas Artikel: dari Mesin Jawaban jadi Tulisan yang Layak Dibaca

## Latar belakang
Enam kritik diajukan user setelah membaca artikel live "Akrasia Aristoteles" (post ID 9, published 2026-07-28). User meminta asumsinya jangan dipercaya begitu saja. Seluruh audit dilakukan terhadap **konten mentah yang benar-benar tersimpan di WordPress** (REST API `posts/9`), bukan draft internal.

**Keenam kritik terbukti benar.** Semuanya bermuara ke dua akar masalah:

- **Akar 1 (teknis, §A)** — Markdown tidak pernah dikonversi ke HTML. Menjelaskan kritik 2, 3, 4, 5.
- **Akar 2 (craft, §B)** — Pipeline ini menulis *jawaban*, bukan *artikel*. Menjelaskan kritik 1, dan ini bagian terbesar dari CP ini.

Arahan scope dari user: **abaikan dulu target lolos GEO checker eksternal, fokus penuh ke kualitas artikel.** Konsekuensi langsung: beberapa aturan yang selama ini ada demi skor GEO justru terbukti merusak prosa dan harus dicabut, bukan sekadar dilonggarkan.

> **Aturan keras yang berlaku di seluruh CP ini: TABEL DILARANG SEPENUHNYA.** Tidak ada tabel markdown, tidak ada `<table>`, tidak ada "tabel perbandingan" dalam bentuk apa pun di artikel. Bukan dilonggarkan, bukan opsional — dihapus dari prompt, dari rubrik, dan dari renderer. Detail penegakan di §A.3, §B.6, dan §C.

---

## Urutan Batch CP020

Setiap batch adalah unit eksekusi terpisah. Hanya satu batch boleh dikerjakan per instruksi user; setelah validasi dan log batch selesai, pekerjaan harus berhenti dan menunggu instruksi berikutnya.

1. **Batch 1 — Rendering WordPress (§A).** Tambahkan converter Markdown → HTML di jalur publish, larang tabel di renderer, dan hapus penyisipan `schemaTag` dari konten post. Validasi draft workflow dan koneksi; belum publish atau menjalankan workflow.
2. **Batch 2 — Contract Concept Brief dan Outline (§B.4/§B.6).** Tambahkan `argumentPattern`, `expectedInsight`, `lede`, `thesis`, `closing`, `closingDelta`, `sectionClaim`, `logicalMove`, dan kontrak keberatan; perbarui parser terkait.
3. **Batch 3 — Writing Planner dan Writer (§B.4/§B.7/§B.8).** Terapkan persona, craft, scene-to-claim, source impact, transisi argumentatif, variasi pola, dan aturan kedalaman ke prompt Planner/Writer.
4. **Batch 4 — Enforcement dan quality gates (§B.6/§B.8).** Perbarui GEO Rule Checker serta validasi deterministik untuk schema baru, tabel, emoji, label, repetisi, dan sinyal kedalaman yang aman diautomasi.
5. **Batch 5 — Verifikasi end-to-end dan editorial review (§C/Definition of Done).** Jalankan test terisolasi, verifikasi REST API WordPress, baca satu artikel utuh dengan rubrik §B.8, dokumentasikan hasil, dan hanya setelah persetujuan terpisah menangani status post publik.

**Status eksekusi saat ini:** Batch 1.

---

# §A — Akar 1: Markdown mentah masuk WordPress (kritik 2, 3, 4, 5)

`Code: Build WordPress Payload` mengirim draft apa adanya:
```js
const schemaTag = '<script type="application/ld+json">' + JSON.stringify(jsonLd) + '</script>';
const content = (draft || '') + '\n\n' + schemaTag;
```
`draft` adalah **Markdown**. WordPress.com tidak mem-parsing Markdown — yang jalan cuma `wpautop` (baris baru → `<p>`/`<br />`) dan `wptexturize` (tanda baca → karakter tipografis).

**Bukti dari konten tersimpan post 9** (15.768 karakter):

- `<p>`: 45
- `<h1>`–`<h6>`: **0**
- `<ul>` / `<ol>` / `<li>`: **0**
- `<blockquote>`: **0**
- `<table>`: **0**
- `##` mentah di badan artikel: 10
- pipe `|` mentah: 111

Seluruh artikel di WordPress saat ini adalah 45 paragraf teks polos tanpa struktur apa pun.

**Kritik 3 (noise JSON)** — mekanismenya: WordPress.com **menghapus tag `<script>` tapi mempertahankan teks di dalamnya**, jadi JSON-LD tumpah jadi paragraf terakhir. `wptexturize` lalu mengubah semua `"` jadi `&#8220;`/`&#8221;`, sehingga JSON-nya rusak juga sebagai data.
→ **Ini menjawab blocker CP019 §1: YA, WordPress.com menghapus tag `<script>`.** Terverifikasi.

**Kritik 4 (FAQ jadi paragraf)** — perlu diluruskan: struktur markdown yang dihasilkan AI **sudah benar** (`### Pertanyaan` + paragraf jawaban). Yang gagal murni rendering. Perbaikan §A menyelesaikan ini; struktur FAQ tidak perlu dirancang ulang.

### Fix §A
1. **Node baru `Code: Markdown to WordPress HTML`**, disisipkan sebelum `Code: Build WordPress Payload`:
   - `## ` → `<h2>`, `### ` → `<h3>`
   - `- item` berurutan → `<ul><li>…</li></ul>`; `1. item` → `<ol>`
   - `> kutipan` → `<blockquote><p>…</p></blockquote>`
   - `**tebal**` → `<strong>`; `[teks](url)` → `<a href="url">`
   - paragraf biasa → `<p>`
   - **`<h1>` DILARANG** — judul post WordPress sudah jadi H1, kalau ditambah lagi jadi H1 ganda.
   - Reuse logika dari `Code: Convert Draft to HTML` (CP018) yang sudah terbukti jalan — jangan tulis dari nol.
2. **`Code: Build WordPress Payload`** — pakai HTML hasil konversi, dan **HAPUS penyisipan `schemaTag`**. Terbukti `<script>` tidak bisa bertahan di WordPress.com. JSON-LD tetap boleh di-generate untuk keperluan lain, tapi jangan ditempel ke konten post.
3. **Penegakan larangan tabel di renderer** — kalau converter menemukan sisa markdown tabel (`| … |` + `|---|`), **jangan render jadi `<table>`**. Buang barisnya atau ubah jadi kalimat biasa, lalu catat peringatan di log. Renderer adalah jaring pengaman terakhir; larangan utamanya ada di §B.6.

---

# §B — Akar 2: Ini mesin jawaban, bukan penulis artikel (kritik 1)

Hipotesis user benar. Tapi masalahnya lebih dalam daripada "ada rule yang salah" — **seluruh rantai prompt memang dirancang untuk menghasilkan jawaban, dan berhasil melakukannya dengan sangat baik.** Yang tidak pernah ada di sistem ini adalah konsep "artikel yang enak dibaca".

## B.1 — Bukti dari kode: perintah eksplisit jadi chatbot

**`Code: Build AI Outline Body` aturan 2:**
> *"Tulis heading seperti cara orang bertanya ke chatbot, bukan seperti judul bab."*

Ini bukan efek samping — ini instruksi harfiah untuk berformat chatbot dan larangan eksplisit terhadap struktur artikel. Diperkuat aturan 1 (minimal 4 dari 5–7 H2 wajib pertanyaan), dan tidak adanya konsep pembuka sama sekali (grep `intro|pembuka|lede|hook` di seluruh node Outline = **0 hasil**).

**`Code: Build AI Writer Body` aturan 1–2:**
> *"Buka SETIAP H2 dengan satu paragraf 40–80 kata yang MENJAWAB LANGSUNG heading… JANGAN buka section dengan kalimat transisi, latar belakang, atau basa-basi."*

Larangan kalimat transisi menghapus jaringan penghubung antar-bagian — hal yang justru membuat tulisan terasa satu kesatuan, bukan tumpukan entri.

## B.2 — Bukti dari prosanya sendiri: apa yang sebenarnya salah dengan tulisannya

Ini bagian yang kemarin terlewat. Berikut pembukaan artikel live, apa adanya:

> "Akrasia Aristoteles adalah kondisi ketika kamu bertindak melawan apa yang secara rasional kamu ketahui sebagai hal yang baik. Aristoteles membedakan fenomena ini dari sekadar ketidaktahuan, karena akrasia terjadi ketika pengetahuan gagal menjadi penuntun tindakan saat dorongan kuat mendominasi. Akrasia Aristoteles juga dapat dipahami sebagai konflik internal: penilaian rasional 'tersedia', tetapi tidak efektif dalam mendorong pilihan praktis."

Tujuh cacat craft, semuanya terukur:

**(1) Keyword stuffing merusak prosa.** Frasa "Akrasia Aristoteles" muncul **20 kali dalam 1.604 kata**. Di paragraf pembuka saja dua kali sebagai subjek. Penulis manusia menyebut "akrasia" sekali lalu lanjut dengan "fenomena ini", "kondisi ini", atau tidak menyebut ulang sama sekali.
→ Penyebabnya sistemik: Writer aturan 12 mewajibkan densitas keyword 0,5–2%, dan `scoreKeywordDensity()` di rubrik memberi nilai penuh untuk rentang itu. 20 kemunculan **masuk rentang yang diberi nilai sempurna**. Jadi sistem secara aktif memberi hadiah untuk prosa yang kaku.

**(2) Register ensiklopedia, bukan esai.** "X adalah kondisi ketika…" adalah pembuka kamus. Tidak ada ketegangan, tidak ada taruhan, tidak ada alasan bagi pembaca untuk peduli di kalimat pertama.

**(3) Nol pijakan konkret.** 1.604 kata abstraksi penuh — tanpa satu pun contoh, skenario, atau gambaran situasi nyata. Padahal topiknya *akrasia*: kegagalan melakukan yang kita tahu benar — hal paling relatable yang bisa dibayangkan. Ini kegagalan craft terbesar di artikel itu.

**(4) Sisipan berlabel yang janggal.** Ditemukan **3 kemunculan**: "Kalimat yang bisa kamu jadikan pegangan:", "Kalimat yang bisa kamu kutip:", "Kalimat penentu:". Ini artefak dari aturan 8 (wajib ada kalimat quotable) — dan ironisnya aturan 8 sendiri **melarang** pelabelan semacam itu. Aturannya dilanggar, dan hasilnya membuat artikel terbaca seperti ringkasan belajar.

**(5) Tidak ada benang merah.** Tiap section memulai ulang dari nol ("Akrasia Aristoteles adalah…", "Kamu bertindak melawan pengetahuan karena…"). Tidak ada argumen yang berkembang dari awal ke akhir. Enam entri paralel, bukan satu tulisan.

**(6) Sudut pandang kedua yang mekanis.** Kata "kamu" muncul, tapi dalam konstruksi seperti "apa yang secara rasional kamu ketahui" — secara gramatikal orang kedua, tapi tidak benar-benar menyapa pembaca. Bandingkan dengan menyapa pengalaman nyata: *"Kamu tahu harus tidur jam sebelas. Jam satu kamu masih scrolling."*

**(7) Artikel tidak punya penutup.** Setelah section terakhir langsung FAQ, lalu tumpahan JSON. Tidak ada paragraf yang mendaratkan argumen. Pembaca ditinggal menggantung.

Tambahan: emoji 📌 dan 😶‍🌫️ muncul di badan prosa, dan bahasa hedging bertumpuk ("dapat dipahami sebagai", "mengarah pada gagasan bahwa", "bersifat potensial") yang menjauhkan pembaca.

## B.3 — Bug: sinyal gaya yang ada pun tidak sampai

**Bug A — `tone.stance` diam-diam hilang.** `config/content-context.json` punya field `stance`:
> *"Taking an explicit stance/opinion is allowed and ENCOURAGED — BUT every stance MUST reference a specific thinker/theory/study…"*

Tapi kode penyusun tone hanya mengambil empat field:
```js
const toneText = ['voice: ' + …, 'person: ' + …, 'avoid: ' + …, 'prefer: ' + …].join('\n');
```
`stance` tidak ada di daftar itu. Instruksi "opini didorong" **tidak pernah terkirim**. Ini penjelasan langsung kenapa artikelnya tidak punya pandangan.

**Bug B — persona penulis tidak pernah dikirim.** `author.bio` di config isinya kaya (penulis independen yang mengeksplorasi makna dan kondisi manusia lewat filsafat & psikologi, termasuk pengalaman langsung dengan ADHD/anxiety). Field ini **hanya dipakai untuk JSON-LD**, tidak pernah masuk prompt Writer. Writer memang tidak punya persona untuk diperankan — system prompt-nya malah berkata *"Kamu adalah penulis artikel **teknis**"*.

**Bug C — contoh di prompt salah domain.** Satu-satunya "CONTOH PEMBUKA SECTION" adalah soal **migrasi database dengan statistik Percona**. Contoh adalah sinyal gaya terkuat bagi LLM, dan contoh ini mengajarkan suara penyampai fakta teknis — persis yang dikeluhkan.

**Bug D — bukti empiris bahwa jalur tone tidak efektif.** `tone.avoid` **sudah memuat** `"emoji within article body"`, dan field `avoid` **memang terkirim**. Tapi emoji tetap muncul di artikel live. Jadi masalahnya bukan aturannya belum ada — aturannya ada dan diabaikan. Penyebabnya struktural: tone dikirim sebagai daftar lunak di *system message*, sementara di *user message* ada 18 "ATURAN WAJIB" dengan ancaman eksplisit ("setiap pelanggaran akan ditolak auditor"). Model memprioritaskan yang diancam sanksi.

→ **Implikasi desain:** menambah kalimat ke blok tone tidak akan menyelesaikan apa pun. Aturan craft harus **naik pangkat jadi aturan bernomor setara**, dengan bobot dan penegakan yang sama.

---

## B.4 — SPESIFIKASI CRAFT: seperti apa artikel yang baik untuk situs ini

Ini inti CP 020. Bagian di bawah ditulis untuk dipindahkan langsung jadi isi prompt (Outline & Writer), bukan sekadar catatan desain.

### Persona penulis (menggantikan "penulis artikel teknis")
> Kamu menulis sebagai penulis independen yang berpikir di depan publik — bukan dosen yang menerangkan, bukan ensiklopedia yang mendefinisikan. Kamu mengeksplorasi filsafat dan psikologi karena persoalannya benar-benar mengganggumu, bukan karena harus menjelaskannya ke orang lain. Kamu punya pengalaman langsung bergulat dengan kondisi psikologis, jadi kamu menulis dari dalam persoalan, bukan dari kejauhan. Kamu berani mengambil posisi, tapi kamu menguji posisi itu terhadap keberatan yang paling kuat sebelum mempertahankannya. Kamu menghormati pembaca sebagai orang yang bisa berpikir — jangan menggurui, jangan menyederhanakan berlebihan.

### Arsitektur artikel (menggantikan struktur 6 jawaban paralel)
1. **Pembuka (tanpa heading, 2–4 paragraf pendek).** Mulai dari pengenalan situasi atau ketegangan yang pembaca kenali dari hidupnya sendiri — bukan definisi. Baru setelah pembaca merasa "ini aku", perkenalkan istilah/konsepnya. Pembuka harus membuat orang ingin lanjut, bukan menjawab pertanyaan.
2. **Tesis.** Artikel punya **satu** klaim utama yang dinyatakan di akhir pembuka. Contoh: *"Akrasia bukan kegagalan tahu. Ini kegagalan pengetahuan untuk hadir tepat pada saat dibutuhkan."*
3. **Bagian isi (4–6 H2).** Tiap bagian **memajukan tesis satu langkah**, bukan menjawab pertanyaan terpisah. Setiap bagian wajib melakukan satu gerakan yang eksplisit: `BUILD`, `DEEPEN`, `TEST`, `NARROW`, atau `TURN`, serta menyatakan apa yang berubah dari pemahaman bagian sebelumnya. Kalau dua bagian bisa ditukar tanpa ada yang berubah, berarti belum ada perkembangan argumen.
4. **Bagian keberatan.** Minimal satu bagian menghadapkan tesis pada bantahan terbaik yang benar-benar punya dasar di concept brief, sumber, atau posisi pemikir — bukan straw-man buatan. Artikel wajib mengakui bagian keberatan yang benar dan menunjukkan apakah tesis bertahan, menyempit, atau berubah.
5. **Penutup (wajib, saat ini tidak ada).** Mendaratkan *earned delta*: kerangka, konsekuensi, atau pertanyaan yang baru masuk akal setelah seluruh argumen dibaca dan tidak bisa ditebak hanya dari judul. Bukan ringkasan poin, pengulangan tesis, ajakan generik, atau kewajiban selalu berakhir dengan pertanyaan terbuka.
6. **FAQ** diletakkan **setelah** penutup sebagai pelengkap, bukan sebagai akhir artikel.

### Aturan kalimat & paragraf
- **Variasikan panjang kalimat dengan sengaja.** Setelah dua-tiga kalimat panjang, taruh satu kalimat pendek. Kalimat pendek adalah alat penekanan. Prosa saat ini rata-rata 19,7 kata per kalimat dengan struktur deklaratif seragam — itu yang membuatnya terdengar seperti mesin.
- **Paragraf boleh satu kalimat** kalau kalimat itu memang perlu berdiri sendiri.
- **Kalimat pertama tiap bagian tidak harus jawaban.** Boleh berupa pengamatan, ketegangan, atau jembatan dari bagian sebelumnya.
- **Jembatan antar-bagian didorong**, bukan dilarang — inilah yang membuat artikel terasa utuh.

### Aturan pijakan konkret (baru, wajib)
- Artikel wajib memiliki minimal dua adegan/contoh konkret yang benar-benar bekerja sebagai alat berpikir; bagian lain tetap harus terhubung ke konsekuensi, observasi, atau kasus spesifik tanpa dipaksa membuka vignette baru.
- Adegan harus melakukan salah satu fungsi: `GENERATE` (melahirkan klaim dari friksi), `TEST` (menguji klaim), atau `LIMIT` (menunjukkan batas klaim). Kalau adegan dihapus dan argumen tidak berubah sama sekali, adegan itu dekorasi dan harus ditulis ulang atau dibuang.
- Abstraksi murni dilarang lebih dari dua paragraf berturut-turut. Ini bukan kuota variasi gaya; tujuannya memberi pembaca pijakan kognitif sebelum lapisan abstraksi berikutnya.
- Untuk topik filsafat/psikologi, pijakan konkret **tidak perlu data statistik** — cukup situasi manusia yang nyata. Ini penting supaya aturan ini tidak bentrok dengan larangan mengarang angka.

### Aturan suara
- **Orang kedua yang sungguh menyapa.** Bukan "apa yang kamu ketahui secara rasional", tapi menyapa pengalaman: "Kamu sudah menutup laptop. Lima menit kemudian kamu membukanya lagi."
- **Ambil posisi, lalu pertanggungjawabkan.** Nyatakan pendirian secara langsung, sebutkan pemikir/teori yang mendasarinya, dan akui batas-batasnya.
- **Buang hedging bertumpuk.** "dapat dipahami sebagai" → "adalah". "mengarah pada gagasan bahwa" → "berpendapat". Satu kualifikasi per klaim cukup.

### Larangan keras (dinaikkan jadi aturan bernomor, bukan preferensi)
- **DILARANG tabel** dalam bentuk apa pun.
- **DILARANG emoji** di badan artikel.
- **DILARANG sisipan berlabel** seperti "Kalimat yang bisa kamu jadikan pegangan:", "Kalimat penentu:", "Poin penting:". Kalimat yang layak dikutip harus menyatu sebagai kalimat biasa dalam prosa.
- **DILARANG membuka bagian dengan pola definisi kamus** ("X adalah kondisi ketika…") lebih dari sekali per artikel.
- **DILARANG mengulang frasa keyword sebagai subjek kalimat** lebih dari sekali per bagian.
- Larangan lama tetap berlaku: basa-basi pembuka ("di era digital…"), hype/clickbait, atribusi kabur.

### Yang TIDAK boleh diubah (jangan sampai rusak saat perombakan)
Aturan integritas Writer nomor 5, 7, 10, 11 tetap utuh: dilarang mengarang angka, dilarang mengarang nama tokoh/studi/organisasi, tanda kutip hanya untuk verbatim yang ada di FAKTA, dan closed-world untuk nama. Aturan-aturan ini menjaga kredibilitas dan tidak ada hubungannya dengan masalah gaya.

---

## B.5 — Contoh before/after (untuk dipasang di prompt sebagai teladan)

Contoh domain teknis (Percona/database) di prompt sekarang **harus diganti** dengan pasangan ini:

**❌ BURUK — pembuka artikel gaya sekarang:**
> "Akrasia Aristoteles adalah kondisi ketika kamu bertindak melawan apa yang secara rasional kamu ketahui sebagai hal yang baik. Aristoteles membedakan fenomena ini dari sekadar ketidaktahuan, karena akrasia terjadi ketika pengetahuan gagal menjadi penuntun tindakan saat dorongan kuat mendominasi."

Salah karena: definisi kamus di kalimat pertama, tidak ada alasan untuk peduli, frasa keyword diulang sebagai subjek, abstraksi murni tanpa pijakan, pembaca tidak disapa.

**✅ BAIK:**
> "Jam sebelas malam kamu menutup laptop. Kamu tahu persis kenapa harus tidur — besok ada presentasi, dan kamu sudah membuktikan berkali-kali bahwa kurang tidur membuatmu kacau. Jam satu kamu masih scrolling.
>
> Yang menarik bukan kenapa kamu gagal. Yang menarik: kamu tidak sedang bingung. Kamu tahu. Pengetahuan itu ada, lengkap, dan tidak berguna sama sekali.
>
> Aristoteles menyebut kondisi ini akrasia, dan penjelasannya masih lebih tajam daripada kebanyakan nasihat produktivitas hari ini: masalahnya bukan kamu kurang tahu, tapi pengetahuanmu tidak hadir tepat pada saat dibutuhkan."

Benar karena: masuk lewat situasi yang dikenali pembaca, ketegangan dibangun sebelum istilah diperkenalkan, panjang kalimat bervariasi (termasuk yang pendek untuk penekanan), orang kedua benar-benar menyapa, tesis dinyatakan di akhir, dan istilah teknis muncul saat pembaca sudah siap menerimanya.

---

## B.6 — Fix konkret per node

**`Code: Build Concept Brief Body`**
- Tambahkan `argumentPattern` dengan salah satu nilai `REVERSAL`, `ESCALATION`, atau `DIALECTIC`, plus `patternRationale` yang menjelaskan kenapa pola itu cocok untuk topik tersebut.
- Tambahkan `expectedInsight`: satu kalimat tentang temuan akhir yang tidak dapat ditebak hanya dari judul. Ini hipotesis kerja, bukan kesimpulan yang harus dipertahankan kalau argumen mengoreksinya.
- Pola tidak boleh dipilih sebagai default tetap. `REVERSAL` cocok untuk membongkar intuisi awal; `ESCALATION` bergerak dari kasus sederhana ke kasus yang makin sulit; `DIALECTIC` mempertentangkan dua penjelasan kuat lalu mencari sintesis atau batas.

**`Code: Parse Concept Brief`**
- Validasi dan pertahankan `argumentPattern`, `patternRationale`, serta `expectedInsight`; nilai pola di luar enum harus ditolak. Parser sekarang hanya memeriksa `coreQuestion` dan `sectionSeeds`, sehingga kontrak baru bisa hilang atau kosong tanpa terdeteksi.

**`Code: Build AI Outline Body`**
- **Hapus total aturan 2** ("seperti orang bertanya ke chatbot").
- Turunkan aturan 1: dari "minimal 4 dari 5–7 H2 wajib pertanyaan" jadi **maksimal 2** yang berbentuk pertanyaan; sisanya heading tematik yang menandai perkembangan argumen.
- **Tambahkan ke skema output**: `lede`, `thesis`, `closing`, dan `closingDelta`. Keempatnya wajib; `closingDelta` menjelaskan apa yang hanya dapat diketahui setelah seluruh argumen berjalan.
- Setiap item `sections[]` wajib punya `sectionClaim` dan `logicalMove: {type, from, to}`. `type` hanya boleh `BUILD`, `DEEPEN`, `TEST`, `NARROW`, atau `TURN`; `from` merangkum pemahaman sebelumnya; `to` menyatakan perubahan yang dihasilkan bagian ini.
- Tolak outline jika dua `sectionClaim` secara semantik sama, `logicalMove` hanya berkata "selanjutnya/juga membahas", atau dua bagian bisa ditukar tanpa merusak logika.
- Wajibkan minimal satu keberatan yang punya `objectionBasis`, `concession`, dan `thesisEffect` (`SURVIVES`, `NARROWS`, atau `CHANGES`). Keberatan boleh menjadi bagian tersendiri atau titik balik organik.

**`Code: Parse Outline`**
- Tambahkan validasi struktural untuk `lede`, `thesis`, `closing`, `closingDelta`, `sectionClaim`, dan `logicalMove`; parser sekarang hanya memeriksa judul, jumlah section, dan FAQ.
- Tolak enum yang tidak valid, `from`/`to` kosong, serta duplikasi `sectionClaim` yang identik setelah normalisasi. Kemiripan semantik tetap dinilai oleh rubrik manual §B.8, bukan dipaksakan menjadi regex.

**`Code: Build AI Writer Body`**
- **Ganti system prompt** dengan persona di §B.4, disusun dari `author.bio` + `tone.voice` + `tone.person` + **`tone.stance`**.
- **Perbaiki Bug A**: masukkan `stance` ke `toneText`.
- **Perbaiki Bug B**: kirim `author.bio` ke prompt.
- **Perbaiki Bug C**: ganti contoh Percona dengan pasangan before/after di §B.5.
- Longgarkan aturan 1–2 sesuai "Aturan kalimat & paragraf" di §B.4 (jembatan antar-bagian didorong; kalimat pertama tidak harus jawaban). Pertahankan larangan basa-basi kosong.
- **Hapus kewajiban tabel di aturan 14**, ganti jadi larangan tegas. List boleh dipakai bila cocok, tidak diwajibkan per artikel.
- **Turunkan drastis aturan 12 (densitas keyword)** — ini penyebab langsung pengulangan 20×. Ganti jadi: sebut keyword utama secara natural, maksimal sekali per bagian sebagai subjek.
- **Tambahkan aturan bernomor baru** untuk: adegan/contoh konkret yang fungsional pada level artikel, penutup wajib, larangan emoji, larangan sisipan berlabel, dan larangan pola definisi kamus berulang.
- Perbaiki aturan 8 (kalimat quotable): tegaskan larangan label, dan jangan wajibkan satu per bagian kalau memaksa — lebih baik dua kalimat kuat yang alami daripada enam yang ditempel.
- Tambahkan aturan kedalaman §B.8: tiap bagian harus menjalankan `logicalMove`; adegan harus `GENERATE`/`TEST`/`LIMIT`; sumber harus mengubah argumen; transisi harus menjelaskan kenapa gerakan berikutnya diperlukan; penutup harus menghasilkan `closingDelta` yang tidak tertebak dari judul.

**`Code: Build AI Writing Planner Body`**
- Hapus kontrak "draft jawaban langsung" sebagai pusat brief. Selaraskan dengan arsitektur baru: setiap brief memuat `sectionClaim`, salinan `logicalMove`, `scenePlan` opsional (`detail`, `friction`, `function`), `sourceUses[]` (`source`, `function`, `impact`), dan `transitionClaim`.
- `sourceUses[].function` hanya boleh `SUPPORT`, `TEST`, `NARROW`, atau `CONTRAST`; `impact` wajib menjelaskan apa yang berubah pada argumen setelah sumber hadir.
- Tolak brief jika adegan hanya mengilustrasikan kesimpulan yang sudah jadi, sumber hanya diringkas tanpa dampak, atau `transitionClaim` bisa dihapus tanpa membuat hubungan antarbagian kehilangan logika.

**`Code: Parse Planner Result`**
- Validasi enum dan field kualitas baru, bukan hanya jumlah `sectionBriefs`. Parser saat ini menganggap brief valid selama jumlahnya cocok.
- Hapus *silent fallback* ke outline mentah untuk run CP020. Jika kontrak argumen Planner tidak valid, tandai run gagal sebelum Writer; prosa yang tetap dilanjutkan tanpa brief akan menghidupkan kembali enam bagian paralel yang sedang diperbaiki.

**`Code: GEO Rule Checker`**
- **Hapus `hasTable`** dari `scoreStructure()` — jangan lagi memberi insentif tabel.
- **Turunkan bobot `s3a`** (rasio heading pertanyaan) supaya heading tematik tidak dihukum.
- **Longgarkan `scoreKeywordDensity()`** — rentang sekarang memberi nilai sempurna untuk pengulangan yang merusak prosa.
- **Tambah pemeriksaan deterministik baru** (jangan andalkan kepatuhan model, lihat Bug D): deteksi emoji, deteksi markdown tabel, deteksi sisipan berlabel (`/Kalimat (yang bisa kamu |penentu|kunci)/i`), deteksi ketiadaan paragraf pembuka, deteksi ketiadaan penutup.

---

## B.7 — Referensi gaya: Oliver Burkeman + Heather Havrilesky

### Status dan batas penggunaan

Referensi utama yang direkomendasikan adalah **Oliver Burkeman** untuk arsitektur argumen, kerendahan epistemik, dan humor yang menurunkan jarak; dipadukan secara terbatas dengan **Heather Havrilesky** untuk kedekatan emosional dan penggunaan orang kedua yang terasa benar-benar menyapa. Kombinasi ini paling dekat dengan persona situs: penulis independen yang berpikir dari dalam persoalan psikologis, berani berpendapat, tetapi tidak memakai jubah otoritas akademik atau menjual resep hidup universal.

**Ini acuan gaya, suara, ritme, dan perangkat craft saja.** Sistem tidak boleh meniru frasa khas, metafora, susunan paragraf, atau persona kedua penulis; tidak boleh mengambil atau mengklaim ide mereka sebagai ide orisinal; dan tidak boleh menyatakan atau menyiratkan bahwa penulis blog ini adalah, menulis sebagai, atau setara dengan Burkeman maupun Havrilesky. Nama penulis dipakai di dokumen desain untuk menjelaskan kalibrasi. Prompt produksi sebaiknya memuat ciri craft hasil abstraksi di bawah, bukan perintah "tulis seperti [nama penulis hidup]".

### Kandidat yang dipertimbangkan

#### 1. Oliver Burkeman — rekomendasi utama

**Hook.** Burkeman sering membuka dari pengakuan kecil yang agak memalukan atau kebiasaan sehari-hari: merapikan inbox, mengejar sistem produktivitas, atau mencoba mengendalikan kecemasan. Detail itu segera dibelokkan menjadi paradoks yang lebih besar. Pembaca masuk melalui pengalaman, bukan definisi.

**Mengambil posisi.** Ia menyatakan tesis yang tegas, lalu merusak kenyamanan tesis itu sendiri dengan kalimat seperti "tetapi saya belakangan melihat keterbatasannya". Pengalaman pribadi menjadi bukti posisi penulis, bukan bukti bahwa semua orang pasti sama. Otoritasnya datang dari keterbukaan mengoreksi diri.

**Ritme.** Paragraf argumentatifnya memakai kalimat menengah-panjang, tanda kurung untuk koreksi diri atau humor, lalu kalimat pendek yang membalik arah. Ia dapat bergerak dari "nasihat yang masuk akal" ke klaim yang lebih mengganggu tanpa terasa seperti daftar poin.

**Penutup.** Penutupnya kembali ke dilema awal dengan pemahaman yang berubah. Ia biasanya tidak merangkum semua bagian atau memberi daftar tindakan; ia mendarat pada satu konsekuensi kecil yang bisa dijalani sekarang, sambil membiarkan ketegangan eksistensial tetap ada.

**Perbedaan dari artikel "Akrasia Aristoteles".** Burkeman tidak mengulang istilah target sebagai subjek, tidak memulai dengan entri kamus, dan tidak menyusun enam jawaban paralel. Argumen berkembang lewat pembalikan: keyakinan awal → kegunaan keyakinan itu → keterbatasannya → posisi yang lebih jujur. Suara "saya juga terjebak di sini" menggantikan suara ensiklopedia yang berdiri di luar persoalan.

**Kecocokan dan risiko.** Sangat cocok untuk tesis, bagian keberatan, serta penutup. Risikonya adalah model meniru formula paradoks "masalahnya bukan X, tetapi Y" di setiap artikel atau memenuhi prosa dengan tanda kurung. Perangkat itu harus dipakai hanya ketika benar-benar memperjelas pemikiran.

#### 2. Heather Havrilesky — rekomendasi pendamping terbatas

**Hook.** Havrilesky lazim memulai dari rasa sakit atau kontradiksi emosional yang sudah terbuka, lalu menamai mekanisme batin di baliknya. Dalam format advice essay, ia membuat pembaca merasa dilihat sebelum mengajukan interpretasi.

**Mengambil posisi.** Ia berani memakai "kamu", tetapi sering memasukkan dirinya ke dalam medan masalah yang sama. Posisi terasa sebagai dorongan dari seseorang yang ikut bergulat, bukan diagnosis dari atas. Ia juga memperlihatkan bahwa nasihat yang ia berikan harus diuji terhadap hidupnya sendiri.

**Ritme.** Energinya dibangun lewat akumulasi: beberapa kalimat panjang yang mengikuti gerak pikiran, pengulangan retoris yang disengaja, lalu satu deklarasi pendek. Ritme ini memberi kehangatan dan desakan yang tidak ada pada prosa ensiklopedis.

**Penutup.** Ia cenderung menutup dengan izin atau komitmen emosional: bukan "lakukan lima langkah ini", melainkan cara baru memperlakukan diri atau melihat konflik. Penutup terasa seperti perubahan sikap, bukan rangkuman materi.

**Perbedaan dari artikel "Akrasia Aristoteles".** Orang kedua tidak dipakai sebagai pengganti subjek gramatikal, melainkan untuk mengakui pengalaman yang spesifik. Prosa memiliki temperatur emosional, risiko pribadi, dan momentum. Hal itu berlawanan dengan "apa yang secara rasional kamu ketahui" yang benar secara tata bahasa tetapi tidak terasa ditujukan kepada manusia tertentu.

**Kecocokan dan risiko.** Berguna untuk keintiman, pengalaman anxiety, dan bagian yang perlu menyapa rasa malu atau konflik batin. Namun intensitas, imperatif, hiperbola, dan pengulangan khas advice column mudah menjadi menggurui atau melodramatis. Karena itu Havrilesky hanya menjadi pengaruh sekunder; jangan mengadopsi format surat nasihat, sumpah emosional, profanity, atau klaim bahwa penulis tahu persis apa yang pembaca rasakan.

#### 3. Alain de Botton — ambil teknik adegan, bukan suara utuh

**Hook.** Kekuatan utamanya adalah mendramatisasi gagasan abstrak lewat adegan sehari-hari yang sangat spesifik, seperti seseorang di kereta yang membangun seluruh fantasi dari interaksi kecil. Filsafat baru masuk setelah pengalaman manusia menjadi nyata.

**Mengambil posisi.** De Botton memakai adegan sebagai bukti bahwa persoalan yang dianggap remeh layak dipikirkan secara filosofis. Ia tidak selalu memerintah pembaca, tetapi generalisasinya tentang motif manusia kadang terlalu luas dan membuatnya kembali terdengar sebagai otoritas.

**Ritme.** Kalimat panjang membawa detail visual dan gerak pikiran; kalimat pendek atau kutipan filsuf menghentikan adegan dan membuka lapisan konsep. Ironi muncul dari jarak antara fantasi besar dan tindakan sehari-hari yang kecil.

**Penutup.** Ia cenderung mengembalikan konsep filosofis ke fungsi praktisnya sebagai cara membaca cinta, kerja, status, atau kecemasan. Penutup memberi konsolasi intelektual, tetapi kadang terlalu rapi seolah filsafat telah menjelaskan seluruh pengalaman.

**Perbedaan dari artikel "Akrasia Aristoteles".** De Botton membuat pembaca mengalami konflik sebelum memberinya nama, sementara artikel live memberi nama dan definisi sebelum pembaca punya alasan untuk peduli. Detail visualnya juga memecah abstraksi yang di artikel live berlangsung berparagraf-paragraf.

**Nilai dan risiko untuk situs.** Teknik konkret → konsep sangat berguna dan langsung memperbaiki abstraksi penuh di §B.2. Tetapi jika suaranya diambil utuh, artikel bisa bergeser menjadi "philosophy lite": terlalu rapi, terlalu universal, dan kembali terdengar seperti otoritas yang menjelaskan manusia. Ambil ketepatan adegannya; jangan ambil kebiasaan membuat generalisasi psikologis besar dari satu ilustrasi.

#### 4. Maria Popova — referensi kurasi, bukan persona

**Hook.** Popova sering membuka dengan tesis liris yang sudah menyatukan fakta, metafora, dan pertanyaan eksistensial. Ia tidak masuk melalui skenario orang kedua, melainkan melalui satu cara melihat yang ingin ia buktikan lewat bacaan.

**Mengambil posisi.** Ia jarang memerintah pembaca. Posisi muncul dari apa yang ia pilih untuk sandingkan dan dari interpretasinya setelah kutipan, sehingga otoritas terasa kuratorial. Risikonya, kepadatan nama dan pujian dapat membuat sumber tampak kebal terhadap keberatan.

**Ritme.** Kalimatnya panjang, bertingkat, dan kaya metafora, lalu diimbangi aforisme pendek. Transisi antarsumber membentuk alur asosiasi, bukan progresi tesis yang seketat Burkeman.

**Penutup.** Popova kerap menghubungkan bacaan utama ke karya lain, sehingga tulisan terasa sebagai jaringan intelektual yang terus terbuka. Ini memberi resonansi, tetapi tidak selalu memberi pendaratan personal yang dibutuhkan situs.

**Perbedaan dari artikel "Akrasia Aristoteles".** Sumber pada tulisan Popova berinteraksi dan mengubah pembacaan satu sama lain; pada artikel live, nama dan konsep lebih sering berfungsi sebagai isi yang dijelaskan. Namun keduanya dapat sama-sama menjauh dari pengalaman pembaca jika kepadatan abstraksi tidak dikendalikan.

**Nilai dan risiko untuk situs.** Ia menunjukkan cara sumber primer dapat menjadi percakapan, bukan tempelan sitasi. Namun suara ini terlalu ornamental, terlalu bergantung pada kutipan, dan kurang cocok dengan persona pengalaman langsung ADHD/anxiety. Jika ditiru, model berisiko mengganti register ensiklopedia dengan register puitis-generik serta mengarang koneksi antartokoh. Ambil prinsip bahwa sumber harus memajukan pemikiran; jangan ambil kepadatan metafora atau struktur kolase kutipan.

#### 5. Mark Manson — kandidat pembanding yang ditolak sebagai acuan suara

**Hook.** Manson efektif membuka dengan anekdot personal yang memalukan, fakta ganjil, atau klaim yang sengaja berlawanan dengan intuisi. Pembukanya bergerak cepat dan segera menjanjikan kesimpulan yang mudah diingat.

**Mengambil posisi.** Ia memasukkan kegagalannya sendiri, tetapi kemudian sering berpindah ke klaim universal dan imperatif langsung. Kerentanan personal mengurangi jarak, sedangkan kepastian retoris membuat pembaca sulit melihat batas argumennya.

**Ritme.** Ia memakai paragraf pendek, repetisi, pertanyaan retoris, profanity, dan kalimat satu baris sebagai punchline. Ritmenya sangat terbaca dan energik, tetapi mudah berubah menjadi rangkaian slogan.

**Penutup.** Penutup biasanya mengulang tesis dalam bentuk tantangan atau tindakan langsung. Pembaca mendapat dorongan yang jelas, tetapi kompleksitas yang dibangun sebelumnya sering dipadatkan menjadi pesan motivasional.

**Perbedaan dari artikel "Akrasia Aristoteles".** Manson memiliki adegan, risiko personal, variasi ritme, dan tesis yang menonjol—semuanya hilang dari artikel live. Namun mengganti repetisi keyword dengan repetisi slogan hanya menukar satu suara AI-generik dengan formula self-help yang sama mudah dikenali.

**Alasan ditolak.** Suaranya sengaja konfrontatif, sangat yakin, penuh imperatif, repetisi slogan, dan identitas self-help yang kuat. Itu dapat mengoreksi prosa yang dingin, tetapi terlalu mudah berubah menjadi clickbait, simplifikasi psikologi, atau penulis yang terdengar lebih yakin daripada dasar faktanya. Situs ini membutuhkan keberanian Burkeman tanpa kepastian performatif Manson.

### Rekomendasi sintesis

Gunakan proporsi desain berikut sebagai arah editorial, **bukan rumus yang dihitung di output**:

- **Burkeman sebagai tulang punggung:** mulai dari keterlibatan atau keterbatasan penulis; bangun satu argumen melalui pola yang dipilih sesuai topik dan keberatan yang jujur; akhiri dengan konsekuensi yang diperoleh, bukan resep final. Pembalikan hanya salah satu pilihan, bukan default.
- **Havrilesky sebagai temperatur:** ketika topik menyentuh rasa malu, anxiety, perhatian yang pecah, atau konflik diri, gunakan sapaan orang kedua yang spesifik dan penuh belas kasih; turunkan intensitas sebelum menjadi nasihat yang memerintah.
- **De Botton sebagai teknik visual:** gunakan adegan yang memiliki benda, tindakan, tempat, dan pilihan nyata sebelum memberi nama konsep abstrak.
- **Popova sebagai disiplin sumber:** hadirkan pemikir karena gagasannya mengubah arah argumen, bukan untuk memamerkan nama atau memenuhi kuota sitasi.
- **Hindari pola Manson:** jangan mengandalkan shock, profanity, hiperbola, slogan, atau kepastian universal untuk menciptakan suara.

### Terjemahan langsung menjadi instruksi prompt

Blok berikut dapat dipindahkan ke prompt Writer pada batch implementasi setelah direview:

> **SUARA PENULIS**
>
> 1. Tulis sebagai orang yang sedang menguji persoalan yang juga menyentuh hidupnya, bukan sebagai pakar yang sudah berdiri di luar persoalan. Gunakan pengalaman atau keterbatasan penulis untuk menunjukkan posisi, tetapi jangan menjadikannya bukti universal.
> 2. Masuk lewat gesekan yang konkret: satu tindakan kecil, benda, keputusan, percakapan, atau momen ketika niat dan perilaku berpisah. Jangan membuka dengan definisi istilah, ringkasan topik, atau klaim besar tentang "manusia".
> 3. Nyatakan tesis dengan tegas, lalu uji dengan keberatan terkuat. Kalau keberatan itu benar sebagian, akui bagian yang benar dan biarkan tesis berubah menjadi lebih presisi. Jangan mempertahankan posisi hanya agar penulis tampak konsisten.
> 4. Pakai "saya" untuk pengalaman yang memang diberikan dalam konteks; "kamu" hanya untuk situasi yang cukup konkret dan umum untuk dikenali; "kita" untuk keterbatasan yang sungguh dibagi. Jangan menebak isi kepala, diagnosis, trauma, atau perasaan pembaca.
> 5. Variasikan ritme berdasarkan fungsi, bukan kuota: kalimat panjang untuk mengikuti pemikiran atau menunjukkan komplikasi; kalimat pendek untuk pembalikan atau konsekuensi. Jangan membuat setiap paragraf berakhir dengan aforisme.
> 6. Humor boleh muncul sebagai pengakuan diri atau ironi kecil. Jangan menertawakan penderitaan pembaca, jangan memakai sarkasme sebagai pengganti argumen, dan jangan mengejar punchline.
> 7. Masukkan pemikir atau sumber pada saat gagasannya mengubah, menantang, atau memperjelas argumen. Setelah menyebut sumber, jelaskan apa yang berubah dalam cara kita melihat masalah; jangan menumpuk nama dan kutipan sebagai dekorasi otoritas.
> 8. Tutup dengan kembali ke situasi atau ketegangan pembuka dalam bentuk yang sudah berubah. Berikan satu implikasi yang dapat dibawa pembaca, bukan rangkuman section, daftar langkah, motivasi generik, atau janji bahwa masalah selesai.
> 9. Pertahankan sedikit ketidaknyamanan yang jujur. Artikel tidak harus menyelesaikan semua paradoks; artikel harus membuat persoalan lebih jelas dan posisi penulis lebih dapat dipertanggungjawabkan.
> 10. Jangan meniru ungkapan, metafora, profanity, pola repetisi, atau persona penulis mana pun. Hasil harus terdengar sebagai suara situs ini sendiri.

### Contoh kalibrasi suara

**❌ BURUK — orang kedua mekanis dan posisi tanpa risiko:**
> "Kamu mengalami akrasia ketika pengetahuan rasional kamu tidak mampu mengendalikan dorongan. Oleh karena itu, kamu perlu memperkuat kebajikan agar keputusan menjadi lebih baik."

Masalah: "kamu" hanya mengganti subjek ensiklopedia; tidak ada situasi, keraguan, atau alasan untuk mempercayai posisi penulis.

**❌ BURUK — koreksi yang terlalu meniru self-help konfrontatif:**
> "Masalahmu bukan kurang disiplin. Masalahmu adalah kamu terus membohongi diri sendiri. Berhenti mencari alasan dan lakukan yang benar."

Masalah: ritmenya kuat tetapi diagnosisnya sembrono, menggurui, dan tidak memberi ruang bagi kompleksitas psikologis.

**✅ BAIK — konkret, terlibat, argumentatif, tetapi terbuka dikoreksi:**
> "Kamu sudah meletakkan ponsel di luar kamar. Lima menit kemudian kamu berdiri untuk mengambilnya lagi, sambil merangkai alasan yang bahkan tidak kamu percaya. Sulit menyebut momen itu sebagai kekurangan pengetahuan: kamu tahu apa yang ingin dilakukan, tahu kenapa, dan tetap bergerak ke arah sebaliknya.
>
> Saya tergoda menyebutnya kegagalan disiplin. Penjelasan itu rapi, tetapi terlalu rapi. Ia tidak menjelaskan kenapa pengetahuan yang terasa meyakinkan pada sore hari bisa kehilangan seluruh bobotnya menjelang tengah malam. Di titik inilah akrasia menjadi lebih dari nama kuno untuk kebiasaan buruk: konsep itu memaksa kita bertanya kapan sebuah pengetahuan benar-benar hadir dalam tindakan."

Benar karena: adegan mendahului istilah, orang kedua tidak mengklaim emosi pembaca, penulis memperlihatkan hipotesisnya sendiri lalu mengoreksinya, dan tesis lahir dari masalah alih-alih ditempel sebagai definisi.

**✅ BAIK — penutup yang mendarat tanpa menjanjikan penyelesaian:**
> "Malam berikutnya ponsel itu mungkin tetap kembali ke tanganmu. Tetapi pertanyaannya sudah berubah. Bukan lagi apakah kamu tahu mana yang baik, melainkan apa yang membuat pengetahuan itu cukup nyata untuk ikut menentukan gerakan berikutnya. Aristoteles tidak memberi kita jalan keluar yang bersih. Ia memberi sesuatu yang lebih berguna: gambaran yang lebih jujur tentang tempat kegagalan itu terjadi."

### Koreksi dan tambahan terhadap §B.4

Riset ini **memperkuat** struktur utama §B.4: pembuka konkret, tesis tunggal, progresi argumen, keberatan, jembatan antarbagian, posisi yang dipertanggungjawabkan, dan penutup wajib. Namun ada empat koreksi agar aturan baru tidak berubah menjadi formula AI yang baru:

1. **Koreksi "satu pijakan konkret per bagian".** Tujuannya benar, tetapi kuota per bagian dapat menghasilkan enam vignette tipis yang sama mekanisnya dengan enam jawaban paralel. Wajibkan artikel memiliki minimal dua adegan konkret yang benar-benar dikembangkan; bagian lain tetap harus terhubung ke contoh, konsekuensi, atau observasi spesifik, tetapi tidak harus selalu membuka vignette baru.
2. **Batasi orang kedua.** "Kamu" yang hidup lebih baik daripada "kamu" gramatikal, tetapi pemakaian terus-menerus mudah menjadi presumptif. Tambahkan aturan perspektif pada prompt: jangan menyatakan apa yang pembaca pasti rasakan atau alami; gunakan "saya" untuk pengalaman penulis dan "kita" hanya untuk kondisi yang dapat dipertanggungjawabkan sebagai umum.
3. **Larang fabrikasi pengalaman pribadi.** Persona dengan pengalaman ADHD/anxiety tidak memberi izin kepada model untuk mengarang kejadian autobiografis, diagnosis, pengobatan, gejala, atau riwayat hidup. Detail orang pertama hanya boleh berasal dari fakta persona atau input yang benar-benar tersedia. Jika tidak ada detail, gunakan pengamatan terbatas atau skenario transparan, bukan memoir palsu.
4. **Keberatan harus organik, bukan slot.** Setiap artikel wajib menghadapi komplikasi terkuat, tetapi tidak selalu perlu heading berlabel "Keberatan". Keberatan boleh menjadi satu bagian tematik atau titik balik di bagian lain, selama benar-benar mengubah atau membatasi tesis.

### Referensi yang dibaca untuk kalibrasi

- Oliver Burkeman, "There's No Such Thing as a Fresh Start": https://www.oliverburkeman.com/freshstart
- Oliver Burkeman, "Anything Could Happen, at Any Moment": https://www.oliverburkeman.com/anythingcouldhappen
- Heather Havrilesky, "How Status Anxiety Erodes Joy" dan wawancara tentang proses *Ask Polly*: https://www.ask-polly.com/p/how-status-anxiety-erodes-joy dan https://www.bkmag.com/2016/07/27/heather-havrilesky-ask-polly-new-york-magazine-advice-writing-writing-advice/
- Alain de Botton, cuplikan *The Consolations of Philosophy*: https://www.alaindebotton.com/philosophy/extract/
- Maria Popova, "A Glow in the Consciousness": https://www.themarginalian.org/2024/06/11/projection-perception/
- Mark Manson, "Why I'm Wrong About Everything (And So Are You)": https://markmanson.net/wrong-about-everything

---

## B.8 — Kedalaman isi: dari prosa yang meyakinkan menjadi argumen yang benar-benar bergerak

### Evaluasi terhadap review Claude

Review Claude menemukan gap terbesar CP020 dengan tepat: §B.4 memperbaiki arsitektur secara konseptual dan §B.7 mematangkan suara, tetapi belum ada kontrak data yang membuktikan bahwa pemahaman pembaca berubah dari satu bagian ke bagian berikutnya. Tanpa perbaikan ini, pipeline dapat menghasilkan enam bagian paralel yang jauh lebih enak dibaca tetapi tetap kosong secara struktural — masalah lama yang tersamarkan oleh prosa lebih baik.

Temuan yang **diterima**:

- progresi argumen harus menjadi field wajib, bukan hanya larangan pasif;
- repetisi gagasan harus dinilai di level makna, bukan keyword;
- adegan dan sumber harus punya fungsi terhadap argumen;
- transisi harus membawa hubungan logis;
- penutup harus menghasilkan wawasan yang diperoleh sepanjang artikel;
- kedalaman harus dinilai lewat rubrik editorial manual, terpisah dari skor GEO.

Tiga usul diterima dengan modifikasi agar tidak melahirkan formula baru:

1. **`logicalMove` memakai enum gerakan, bukan kata sambung.** Kata "maka/tapi/selanjutnya" mudah digame. Kontrak memakai `BUILD`, `DEEPEN`, `TEST`, `NARROW`, atau `TURN`, disertai `from` dan `to` yang wajib konkret.
2. **Penutup tidak wajib selalu mengandung pertanyaan baru.** Ia wajib menghasilkan `closingDelta` yang tidak tertebak dari judul; bentuknya boleh pertanyaan, konsekuensi, keputusan, batas, atau kerangka baru. Mewajibkan pertanyaan akan membuat semua artikel berakhir dengan nada terbuka yang sama.
3. **Adegan tidak wajib selalu mendahului klaim.** Pola utama memang adegan → friksi → klaim, tetapi adegan juga boleh hadir setelah klaim untuk menguji atau membatasinya. Yang wajib adalah fungsi `GENERATE`, `TEST`, atau `LIMIT`, bukan posisi yang seragam.

### Taksonomi gerakan argumen

- **`BUILD`** — membangun klaim pertama dari tesis atau pijakan yang sudah tersedia.
- **`DEEPEN`** — mempertahankan arah tesis tetapi menunjukkan sebab, mekanisme, atau konsekuensi yang lebih rumit.
- **`TEST`** — menghadapkan tesis pada kasus atau keberatan yang tampak dapat mematahkannya.
- **`NARROW`** — menyatakan kondisi, ruang lingkup, atau pengecualian yang membuat tesis lebih presisi.
- **`TURN`** — mengganti kerangka awal ketika argumen menunjukkan bahwa pertanyaan atau asumsi semula keliru.

Tidak semua artikel harus memakai kelimanya. Yang wajib: tiap bagian punya satu gerakan yang berbeda fungsinya dari bagian sebelumnya, dan urutannya tidak dapat ditukar tanpa merusak alur.

### Tiga pola struktur yang boleh dipilih Concept Brief

1. **`REVERSAL`** — mulai dari intuisi yang masuk akal, temukan keterbatasannya, lalu bangun kerangka pengganti. Jangan otomatis memakai formula "bukan X, melainkan Y"; pembalikan harus lahir dari bukti atau kasus.
2. **`ESCALATION`** — mulai dari kasus paling jelas, bawa tesis ke kasus yang makin sulit, lalu tentukan batas berlakunya. Cocok ketika kedalaman datang dari perubahan skala atau kondisi.
3. **`DIALECTIC`** — hadirkan dua penjelasan kuat sejak awal, uji kelebihan dan kebuntuan masing-masing, lalu pilih, sintesis, atau biarkan perbedaan yang memang tidak dapat didamaikan.

Pemilihan pola adalah keputusan per topik di Concept Brief. Tidak ada pola default global dan tidak boleh semua artikel memakai `REVERSAL` hanya karena contohnya paling mudah ditiru.

### Aturan Writer siap pakai — kedalaman argumen

Blok ini melengkapi, bukan mengganti, blok **SUARA PENULIS** §B.7:

> **GERAKAN ARGUMEN**
>
> 11. Setiap bagian isi wajib menjalankan `logicalMove` dari outline. Nyatakan apa yang dipahami pembaca sebelum bagian ini dan apa yang berubah setelahnya. "Menambahkan informasi lain tentang topik yang sama" bukan perkembangan argumen.
> 12. Gunakan adegan sebagai alat berpikir. Adegan harus melahirkan friksi (`GENERATE`), menguji klaim (`TEST`), atau menunjukkan batasnya (`LIMIT`). Kalau adegan dapat dihapus tanpa mengubah argumen, buang atau tulis ulang.
> 13. Setelah menghadirkan pemikir atau sumber, lakukan kerja argumentatif dalam paragraf yang sama atau paragraf berikutnya: gunakan sumber untuk `SUPPORT`, `TEST`, `NARROW`, atau `CONTRAST`, lalu jelaskan dampaknya terhadap tesis. Dilarang membiarkan pola "Tokoh X berpendapat Y" berdiri tanpa konsekuensi.
> 14. Transisi harus menjelaskan mengapa gerakan berikutnya diperlukan oleh bagian sebelumnya. Kata "selain itu", "di sisi lain", atau "selanjutnya" tidak cukup kalau hubungan logisnya tetap hilang setelah kata itu dihapus.
> 15. Hadirkan keberatan terbaik yang punya dasar nyata, akui bagian yang benar, lalu nyatakan efeknya terhadap tesis: bertahan, menyempit, atau berubah. Dilarang membuat lawan yang sengaja lemah agar mudah dijatuhkan.
> 16. Jangan mengulang gagasan dengan contoh atau parafrase baru. Setiap `sectionClaim` harus memberi perubahan yang dapat diringkas dalam satu kalimat berbeda dari bagian lain.
> 17. Setelah maksimal dua paragraf abstrak, beri pembaca pijakan kognitif yang benar-benar bekerja: adegan, kasus, konsekuensi, pertanyaan tajam, atau kalimat pendek yang mengubah arah. Jangan menyisipkan kalimat pendek acak hanya untuk memenuhi variasi ritme.
> 18. Penutup wajib menghasilkan `closingDelta`: wawasan, batas, konsekuensi, keputusan, atau pertanyaan yang hanya masuk akal setelah seluruh artikel dibaca. Kalau penutup dapat diparafrasekan sebagai "seperti dijelaskan sejak awal, tesis ini benar", tulis ulang.

### Uji negatif integrasi sumber

Sebuah bagian gagal bila salah satu kondisi ini terjadi:

- sumber disebut, tetapi kalimat/paragraf setelahnya tidak mengubah, menguji, mempersempit, atau mempertentangkan klaim;
- dua pemikir berjejer tanpa perbedaan, hubungan, atau sintesis;
- kutipan dapat dihapus dan argumen tetap identik;
- nama besar dipakai sebagai penutup diskusi, bukan sesuatu yang dapat diuji;
- ringkasan karya mengambil lebih banyak ruang daripada persoalan yang hendak dijelaskan.

### Contoh: penjelasan dipoles vs argumen berkembang

**❌ Penjelasan dipoles:**

> Bagian 1 menunjukkan akrasia ketika seseorang terus menggulir ponsel meski tahu harus tidur. Bagian 2 menunjukkan akrasia ketika seseorang melewatkan olahraga meski tahu manfaatnya. Bagian 3 menjelaskan bahwa bahkan orang berpengetahuan dapat mengalami akrasia menurut Aristoteles.

Tiga bagian memakai contoh berbeda, tetapi `sectionClaim`-nya sama: orang dapat bertindak melawan pengetahuan. Urutannya bisa ditukar dan tidak ada pemahaman yang berubah.

**✅ Argumen berkembang:**

> **BUILD:** Akrasia bukan sekadar kurang tahu; pengetahuan dapat kehilangan daya tepat ketika tindakan harus dipilih.
>
> **TEST:** Namun ancaman yang sangat dekat kadang tetap mengubah tindakan. Ini menunjukkan bahwa masalahnya bukan "pengetahuan hadir atau tidak" secara biner, melainkan bobot yang dimiliki pengetahuan dibanding dorongan pada saat tertentu.
>
> **NARROW:** Karena itu, tesis paling kuat berlaku pada keputusan dengan konsekuensi tertunda. Akrasia bukan kegagalan rasionalitas secara umum, tetapi kegagalan tertentu dalam menjembatani penilaian sekarang dengan akibat yang belum terasa.

Bagian kedua diperlukan untuk mengoreksi bagian pertama; bagian ketiga tidak dapat muncul secara logis sebelum pengujian di bagian kedua.

### Rubrik editorial manual — PASS/GAGAL, bukan skor

Rubrik ini wajib dipakai untuk satu artikel uji utuh setelah implementasi. Setiap kegagalan harus menyertakan kutipan atau ringkasan bagian yang bermasalah.

1. **Uji substitusi konsep.** Ganti konsep/tokoh utama dengan konsep dekat. Jika argumen tetap masuk akal tanpa perubahan berarti, **GAGAL: wawasan generik**.
2. **Uji tebak kesimpulan.** Jika penutup sudah dapat ditebak dari judul atau tesis pembuka, **GAGAL: penjelasan dipoles, bukan pemikiran**.
3. **Uji ringkasan per bagian.** Tulis satu kalimat `sectionClaim` untuk setiap bagian. Jika dua ringkasan menyatakan gagasan yang sama, **GAGAL: repetisi semantik**.
4. **Uji hapus adegan.** Hapus adegan secara mental. Jika argumen tidak kehilangan pertanyaan, bukti, batas, atau perubahan apa pun, **GAGAL: adegan dekoratif**.
5. **Uji tukar posisi.** Tukar dua bagian isi. Jika alur tetap sama kuatnya, **GAGAL: daftar paralel, bukan argumen**.
6. **Uji keberatan jujur.** Pastikan bantahan punya dasar nyata dan penulis mengakui bagian yang benar. Jika lawan mudah dijatuhkan karena dibuat lemah, **GAGAL: straw-man**.
7. **Uji penutup vs pembuka.** Jika esensi penutup hanya mengulang tesis awal dengan kata baru, **GAGAL: tidak ada `closingDelta`**.
8. **Uji sumber.** Untuk setiap sumber, tunjukkan fungsi dan dampaknya pada argumen. Jika sumber hanya menyumbang nama/ringkasan, **GAGAL: rangkuman literatur**.
9. **Uji transisi.** Hapus kalimat transisi. Jika hubungan antarbagian tetap sama jelasnya, transisi kemungkinan kosmetik; jika hubungan memang tidak pernah dijelaskan, **GAGAL: progresi tersirat saja**.
10. **Uji pacing.** Baca sekali tanpa berhenti dan tandai titik perhatian jatuh karena abstraksi menumpuk. Jika tidak ada pijakan fungsional sebelum titik itu, **GAGAL: beban kognitif tidak dikelola**.

Artikel belum boleh dinilai siap publish jika uji substitusi, ringkasan per bagian, tukar posisi, keberatan, penutup, atau sumber gagal. Uji lain wajib diperbaiki bila kegagalannya mengganggu pembacaan utuh. Skor GEO internal tidak dapat menggantikan rubrik ini.

### Batas otomatisasi

- Validasi enum, field kosong, jumlah section, dan duplikasi teks identik boleh dilakukan di Code node.
- Kemiripan semantik, kejujuran keberatan, spesifisitas wawasan, serta kualitas `closingDelta` tidak boleh dipura-purakan sebagai regex deterministik.
- Penilaian utama CP020 tetap pembacaan manusia. AI Critic boleh memberi sinyal tambahan pada batch lanjutan, tetapi bukan bukti kelulusan dan bukan syarat implementasi fase ini.

---

# §C — Kritik 6: jangan publish dulu
Setuju. Default sistem memang sudah `Live Run - Draft`, tapi ada dua artikel yang terlanjur live.

1. **Unpublish post 9 dan 10 sekarang** — keduanya menampilkan markdown mentah dan tumpahan JSON di halaman publik.
2. Jangan pakai `Live Run - Publish` sampai §A dan §B selesai dan diverifikasi lewat pembacaan manual.

---

# §D — `config/content-context.json` juga harus direvisi, bukan cuma prompt di node

CP ini sebelumnya cuma menyasar kode di node n8n. Itu keliru: `content-context.json` adalah **sumber kebenaran untuk tone/persona/stance** yang dibaca `Code: Load Content Context` lalu diteruskan ke Outline/Writer — kalau file ini tidak ikut direvisi, perbaikan di node hanya menambal gejala sementara isi acuannya tetap usang.

## D.1 — Temuan: sebagian field di config TERNYATA TIDAK PERNAH DIBACA

Diverifikasi langsung ke kode `Code: GEO Rule Checker`:
- `geo.keywordDensityRange: [0.005, 0.02]` — **tidak pernah dibaca**. Nilai 0.005/0.02/0.03 di-hardcode langsung di dalam `scoreKeywordDensity()`. Mengubah angka ini di config **tidak akan berpengaruh apa pun**.
- `geo.minQuestionHeadingRatio: 0.6` — **tidak pernah dibaca**. Rasio heading-pertanyaan juga hardcoded di `scoreStructure()`.

Ini bug tersembunyi yang berbahaya: siapa pun (termasuk worker yang mengerjakan CP019 sebelumnya) bisa mengira mengedit field ini di config sudah cukup, padahal tidak berefek sama sekali. **Wajib diperbaiki sekalian**: kalau field ini memang dimaksudkan untuk jadi threshold yang bisa diatur, `Code: GEO Rule Checker` harus benar-benar membacanya dari `ctx.contentContext.geo`, bukan hardcode. Kalau tidak, hapus field itu dari config supaya tidak menyesatkan.

## D.2 — Bagian `tone` perlu ditulis ulang, bukan cuma ditambah field

Isi `tone` sekarang (`voice`, `person`, `stance`, `avoid`, `prefer`) sudah mengarah ke arah yang benar secara SUBSTANSI (stance didorong, hindari klise), tapi **terbukti kalah bersaing** melawan 18 "ATURAN WAJIB" di prompt Writer (lihat §B.3 Bug D — larangan emoji sudah ada di `avoid` tapi tetap dilanggar). Menambah field baru ke `tone` tanpa mengubah cara node mengonsumsinya tidak akan menyelesaikan masalah. Yang harus terjadi:

1. **Isi `tone` diperkaya** dengan hasil riset "standar emas" (lihat prompt untuk CLI worker di bawah) — kemungkinan perlu field baru seperti `openingStyle`, `closingStyle`, `sentenceRhythm`, atau `concreteGrounding` yang secara eksplisit menuntun ke arsitektur artikel §B.4 (bukan cuma daftar kata yang dihindari/disukai).
2. **`Code: Build AI Outline Body` dan `Code: Build AI Writer Body` harus benar-benar mengonsumsi field baru itu** dan menaikkannya jadi ATURAN BERNOMOR di prompt (pola yang sama seperti fix Bug A/B di §B.6) — supaya tidak terulang pola "field ada di config, tapi tidak sampai/tidak dipatuhi di prompt".

## D.3 — `author.bio` perlu ditinjau untuk dipakai sebagai suara, bukan cuma byline

Saat ini `author.bio` hanya dipakai untuk JSON-LD (E-E-A-T signal). §B.6 sudah menugaskan agar field ini dikirim ke prompt Writer sebagai bahan persona. Tapi isinya sendiri perlu ditinjau ulang: apakah kalimatnya cukup spesifik untuk membentuk suara yang konsisten (pengalaman pribadi dengan ADHD/anxiety, cara berpikir "in public"), atau perlu diperkaya dengan detail suara/perspektif yang lebih tajam supaya benar-benar bisa jadi bahan persona, bukan sekadar bio formal.

## D.4 — Config drift yang ditemukan (bukan bug CP020, tapi harus dicatat)

`models.roles.reviser` di config masih menyebut `deepseek-v4-flash-free` / OpenCode Zen — ini **sudah usang**, node live `Code: Build AI Reviser Body` sekarang memakai `openai/gpt-oss-120b` via OpenRouter (fix darurat pasca-eksekusi 1001/1002/1003). Update field ini supaya config kembali jadi cerminan akurat dari workflow, bukan dokumentasi yang menyesatkan.

---

# Urutan pengerjaan
**§A dulu, baru §B.** Selama markdown masih tumpah mentah, mustahil menilai apakah perbaikan craft berhasil — semua artikel akan tetap terlihat rusak apa pun isinya.

# Yang HARUS diverifikasi (jangan asumsi)
1. Setelah §A: publish 1 artikel uji, ambil ulang via REST API, hitung `<h2>`/`<ul>`/`<blockquote>` (**harus > 0**), dan `##`/`|`/`schema.org`/`<table>` (**harus 0**). Verifikasi dari konten tersimpan, bukan tampilan editor.
2. Tidak ada `<h1>` di dalam konten (judul post sudah H1).
3. **Setelah §B, penilaian utamanya adalah pembacaan manual — bukan skor.** Jalankan seluruh rubrik PASS/GAGAL §B.8 pada satu artikel utuh, termasuk uji substitusi konsep, ringkasan per bagian, hapus adegan, tukar posisi, keberatan jujur, penutup, sumber, transisi, dan pacing.
4. Hitung ulang pengulangan frasa keyword — harus jauh di bawah 20 per 1.600 kata.
5. Pastikan nol tabel, nol emoji, nol sisipan berlabel.
6. Pastikan aturan integritas tidak ikut rusak: tetap tidak ada angka karangan, nama karangan, atau kutipan tanpa verbatim.
7. Jangan menilai keberhasilan dari skor GEO internal — sesuai arahan user, skor bukan tolok ukur di CP ini.

# Definition of done
- [ ] Node konversi Markdown → HTML dibuat (reuse CP018), tersisip sebelum `Code: Build WordPress Payload`
- [ ] `schemaTag` tidak lagi ditempel ke konten post
- [ ] Terverifikasi via REST API: `<h2>`/`<ul>`/`<blockquote>` nyata; nol `##`/`|`/`schema.org`/`<table>`
- [ ] Concept Brief: memilih `argumentPattern` per topik dan menghasilkan `patternRationale`/`expectedInsight`; parser memvalidasi kontraknya
- [ ] Outline: aturan "seperti bertanya ke chatbot" dihapus; heading pertanyaan maks 2; `lede`/`thesis`/`closing`/`closingDelta` masuk skema output
- [ ] Outline: setiap section punya `sectionClaim` dan `logicalMove {type, from, to}`; keberatan punya basis, concession, dan efek terhadap tesis; parser memvalidasi semua field/enum
- [ ] Writer: system prompt memakai persona §B.4; Bug A (`tone.stance`) dan Bug B (`author.bio`) diperbaiki
- [ ] Writer: contoh Percona diganti pasangan before/after §B.5
- [ ] Writer: aturan craft baru (pijakan konkret, penutup wajib, variasi kalimat, larangan emoji/label/definisi-kamus) naik jadi aturan bernomor
- [ ] Writer: kewajiban tabel dihapus dan diganti larangan; densitas keyword diturunkan drastis
- [ ] Writing Planner menghasilkan `scenePlan`, `sourceUses`, dan `transitionClaim`; parser memvalidasi kontrak dan tidak melakukan silent fallback ke outline mentah
- [ ] GEO Rule Checker: `hasTable` dihapus; bobot heading-pertanyaan & densitas keyword diturunkan; pemeriksaan deterministik baru ditambahkan
- [ ] Post 9 & 10 di-unpublish
- [ ] **1 artikel uji dibaca manual utuh dengan seluruh rubrik PASS/GAGAL §B.8; enam uji inti tidak boleh gagal**
- [ ] CP019 §1 ditandai terjawab (WordPress.com terbukti menghapus `<script>`)
- [ ] `config/content-context.json` — `geo.keywordDensityRange`/`minQuestionHeadingRatio` dibuat benar-benar dibaca kode ATAU dihapus dari config (§D.1)
- [ ] `config/content-context.json` — blok `tone` ditulis ulang berdasar riset standar emas (§D.2), dan field barunya benar-benar dikonsumsi di prompt Outline/Writer
- [ ] `config/content-context.json` — `author.bio` ditinjau/diperkaya untuk dipakai sebagai bahan persona (§D.3)
- [ ] `config/content-context.json` — `models.roles.reviser` diperbarui ke `openai/gpt-oss-120b`/OpenRouter (§D.4)
- [ ] `ai_docs/index.md` diperbarui
