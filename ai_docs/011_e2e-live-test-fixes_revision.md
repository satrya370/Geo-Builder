# CP 011 — Revision: Perbaikan dari Tes End-to-End Nyata (Reviewer)

**Mode:** Reviewer · **Konteks:** User minta review menyeluruh "semua CP sudah selesai" —
karena tracking dokumen tidak sinkron dengan kondisi n8n sebenarnya, dilakukan **tes eksekusi
nyata** (bukan review kode statis) untuk verifikasi tuntas. Ditemukan 6 bug yang **tidak
mungkin ketahuan dari review kode saja** — semuanya baru terlihat saat pipeline benar-benar
dijalankan.

## Ringkasan: dari 0% jalan → pipeline penuh sukses (33/33 node)

Sebelum sesi ini, **belum pernah ada satu eksekusi pun yang lolos melewati node pertama**
(Safety Guard) — semua CP sebelumnya diverifikasi lewat pembacaan kode atau "simulasi" manual,
bukan eksekusi nyata. Setelah 6 bug di bawah diperbaiki satu-satu (tiap perbaikan diverifikasi
ulang dengan menjalankan execution lagi), hasil akhir: **geoScore 85/100, verdict PASS**,
draft berhasil dibuat di WordPress (`wpPostId: 7`, status `draft`).

## Bug yang ditemukan & diperbaiki (urut kronologis penemuan)

### 1. KRITIS — `max_tokens` terlalu kecil untuk model reasoning
**Gejala:** Safety Guard SELALU menolak topik (termasuk topik yang jelas aman), `finish_reason:
"length"`, `content` kosong.
**Akar masalah:** `glm-4.5-flash`, `openai/gpt-oss-120b`, dan `tencent/hy3-preview` semuanya
model **reasoning** — mereka menulis chain-of-thought panjang di field terpisah
(`reasoning_content`/`reasoning`) SEBELUM menulis jawaban final di `content`. `max_tokens`
kecil (512-2048) habis untuk reasoning saja, jawaban final tidak pernah tertulis.
**Fix:** naikkan `max_tokens` di semua node yang pakai model reasoning:
- Safety Guard: 512 → 3000
- Critic: 2048 → 5000
- Writing Planner: 4000 → 7000
- Reviser: `wc*2.2` → `wc*2.2 + 3000` (buffer tetap)
- Researcher: 2048 → 6000 → **16000** (butuh 2x naik, riset+8-12 fakta ternyata makan jauh
  lebih banyak reasoning daripada perkiraan awal)
- Writer: `tw*2.2` → `tw*2.2 + 2500` (defensif, status reasoning `deepseek-v4-flash` tidak pasti)

### 2. TINGGI — Parse node gagal kalau model kasih kalimat pembuka sebelum JSON
**Gejala:** `Code: Parse Research` selalu `facts: []` walau Researcher jelas-jelas dapat data
riset asli (terbukti dari isi mentah response).
**Akar masalah:** Fungsi parse cuma menghapus fence ` ```json `, tidak menghapus teks prosa
("I'll conduct web searches...") yang kadang ditulis model SEBELUM blok JSON.
**Fix:** semua 6 node Parse (`Parse Research`, `Parse Guard Result`, `Parse Outline`,
`Parse Critique`, `Parse Planner Result`, `Parse and Validate Schema`) diperbaiki: ekstrak
substring dari `{` pertama sampai `}` terakhir SEBELUM `JSON.parse`, bukan cuma strip fence.

### 3. KRITIS — Koneksi `If - Topic Allowed?` ke port yang salah
**Gejala:** Topik yang **diterima** (`allowed: true`) malah berhenti di node If, tidak lanjut
ke Researcher.
**Akar masalah:** Item dengan kondisi TRUE ternyata mendarat di **output port 1**, sementara
koneksi ke `Build AI Research Body` disambung dari **port 0** — terbalik dari asumsi
konvensi n8n standar (true=0/false=1). Dibuktikan langsung dari data eksekusi nyata, bukan
ditebak dari baca kode.
**Fix:** pindahkan koneksi dari port 0 ke port 1.

### 4. KRITIS — Syntax error JS di `Code: Build AI Outline Body` (bug lama dari CP004)
**Gejala:** Node ini selalu error "Invalid or unexpected token", menghentikan seluruh run.
**Akar masalah:** Ada 4 karakter literal `\n\n` (backslash-n dua kali) yang nyasar **DI LUAR**
template literal — di antara `` `...`; `` dan `return [...]` — bukan newline sungguhan. Bug ini
ada sejak CP004/005, tidak pernah ketahuan karena tidak pernah ada eksekusi nyata sampai sejauh
ini sebelumnya.
**Fix:** tulis ulang jsCode dengan newline asli di posisi yang benar. **Diverifikasi ekstra**:
semua 26 node Code di-cek pakai `vm.Script` (Node.js) untuk memastikan tidak ada syntax error
lain yang tersembunyi — semua bersih.

### 5. TINGGI — Ollama tidak bisa dihubungi via `localhost`
**Gejala:** `HTTP: AI Outline Planner` dan `HTTP: AI Schema Generator` gagal dengan
`ECONNREFUSED ::1:11434`.
**Akar masalah:** Di Windows ini, `localhost` resolve ke IPv6 (`::1`) duluan, tapi Ollama cuma
listen di IPv4. `127.0.0.1` langsung terbukti bisa diakses.
**Fix:** ganti URL kedua node itu (dan `content-context.json` `models.local.endpoint`) dari
`localhost` ke `127.0.0.1`.

### 6. SEDANG — Dua mekanisme web search yang saling konflik
**Gejala:** Researcher intermiten — kadang dapat fakta lengkap, kadang `content` cuma kalimat
pembuka tanpa data ("Let me search for...").
**Akar masalah:** Body request punya **DUA** mekanisme web search sekaligus: suffix `:online`
(OpenRouter, otomatis server-side) DAN parameter `tools: [{type: 'web_search'}]` (client-side
tool-calling yang butuh follow-up call terpisah, tapi node ini tidak punya handler untuk itu).
Model kadang memilih jalur tool-call eksplisit yang tidak pernah selesai.
**Fix:** hapus `tools`/`tool_choice`, andalkan `:online` saja (sudah terbukti bekerja baik
sendirian).

## Temuan penting di luar bug teknis

- **Klaim "credentials: None" di `get_workflow_details` untuk SEMUA node httpRequest ternyata
  bukan bukti kredensial hilang** — itu field yang disembunyikan tool demi keamanan. Eksekusi
  nyata membuktikan kredensial-kredensial itu (Researcher, Writer, Critic, Schema, WordPress)
  semuanya berfungsi. Review sebelumnya (termasuk punya saya sendiri di CP 008) yang
  menyimpulkan "credential belum terpasang" dari field ini **berpotensi salah** — pelajaran:
  field ini tidak bisa dipercaya sebagai bukti, cuma eksekusi nyata yang bisa membuktikan.
- **`tencent/hy3-preview:online` benar-benar melakukan web search asli** — fakta yang
  dihasilkan bersumber dari Stanford Encyclopedia, PMC/NIH, Nature Communications, BMC
  Psychology, Forbes — sumber nyata, bukan karangan. Kekhawatiran soal "tidak ada web search"
  di review CP 004 lama **terbukti tidak berlaku lagi** (mungkin sudah diperbaiki agent lain
  sebelum sesi ini, atau asumsi awal salah).
- **Revise loop (CP 008) tidak sempat teruji** di run sukses ini karena draft langsung PASS
  (skor 85) di percobaan pertama — loop tidak ter-trigger. Verifikasi CP 008 (apakah
  `reviseCount` benar-benar naik dan skor berubah antar iterasi) **masih perlu tes terpisah**
  dengan draft yang sengaja dibuat jelek supaya masuk jalur REVISE.

## Hasil tes akhir (execution 936, workflow `rwIbdIkIhoVE8nkG`)

| Node | Hasil |
|---|---|
| Safety Guard | `allowed: true` |
| Researcher | 12+ fakta bersumber nyata |
| Validate Research Facts | lolos threshold |
| Outline Planner | 5-7 section, FAQ ≥3 |
| Writing Planner | brief per-section terisi |
| Writer | draft lengkap |
| GEO Rule Checker | 6 kategori rule dihitung |
| Critic | temuan AI terisi |
| Compute GEO Score | **85/100, verdict PASS** |
| Switch Gate | → cabang PASS |
| Schema Generator | title+slug valid, **fallback tidak terpakai** |
| WordPress Create Post | **berhasil, `wpPostId: 7`, status `draft`** |

## Rekomendasi tindak lanjut

1. **Cek draft di wp-admin** (`https://satryapudja.wordpress.com/wp-admin`, post ID 7) —
   verifikasi manual kategori benar, JSON-LD tertanam di source, konten terbaca wajar sebelum
   pertimbangkan mengubah default `status` ke `'publish'`.
2. **Uji jalur REVISE secara terpisah** — draft yang lolos langsung PASS tidak menguji CP 008
   (revise loop). Perlu draft uji yang sengaja lemah di 1-2 kategori untuk memicu REVISE.
3. **Kalibrasi**: kategori `freshness` (58%) dan `language`/`answerFirst` (79%) adalah yang
   terendah — kalau pola ini konsisten di beberapa artikel lagi, lihat `GEO_SCORING_RUBRIC.md`
   §Kalibrasi awal untuk arah perbaikan prompt (bukan menurunkan threshold).
4. Update semua entri `index.md` untuk CP 004-009 — status sebenarnya sekarang **lebih baik**
   dari yang tercatat (banyak "in-progress"/"rejected-minor" yang sekarang sudah terbukti
   berfungsi lewat tes nyata ini), tapi juga catat bug-bug baru di atas sebagai bagian dari
   riwayat perbaikan masing-masing CP asal.
