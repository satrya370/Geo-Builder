# Rules — Proses Kerja AI untuk Membangun GEO Article Builder

> Ini aturan **proses development** (bagaimana AI CLI bekerja membangun workflow n8n ini),
> bukan bagian dari workflow n8n itu sendiri. Dokumen desain workflow tetap di
> `docs/IMPLEMENTATION_PLAN.md`, `docs/GEO_SCORING_RUBRIC.md`, `docs/PROMPT_GUIDE.md`,
> `docs/ARCHITECTURE_NOTES.md`, `docs/OPTIONAL_SCORING_WORKFLOW.md`.

## 0. Aturan umum

- **Workflow n8n target:** `http://localhost:5678/workflow/rwIbdIkIhoVE8nkG`
  (ID `rwIbdIkIhoVE8nkG`). Semua operasi n8n MCP (buat/update/validasi node) untuk CP apa
  pun di project ini mengarah ke workflow ID tersebut — bukan workflow inspirasi
  (`ig Content Builder For Demo`) atau workflow produksi (`ig Content Builder`).
- **Wajib pakai n8n MCP** untuk sebagian besar tugas yang menyentuh workflow n8n (inspeksi,
  buat node, validasi, update) — jangan menebak struktur/parameter node tanpa `get_node_types`
  / `get_sdk_reference` dari n8n MCP.
- Tiga mode AI CLI bekerja bergiliran di project ini: **Planner**, **Worker**, **Reviewer**.
  Semua mode menulis ke folder `ai_docs/`.
- **⚠️ BRIEF WAJIB — sebelum tindakan apa pun (membaca file, memeriksa workflow, membuat/mengubah node, memvalidasi, menulis log), setiap mode (Planner/Worker/Reviewer) wajib memberi brief singkat berisi: (1) pemahaman terhadap tugas, (2) langkah yang akan diambil, (3) estimasi token. JANGAN langsung bertindak tanpa brief — ini aturan absolut, bukan opsional. Brief datang SEBELUM tool call pertama dalam satu run.**

---

## 1. Penamaan file di `ai_docs/`

| Jenis | Format nama file |
|---|---|
| Checkpoint plan (dari Planner) | `NNN_subject_plan.md` |
| Revision plan (dari Reviewer) | `NNN_subject_revision.md` — `NNN` **baru**, bukan menimpa `NNN` asal |
| Review report | `NNN_subject_report.md` |
| Log kerja (Worker/Reviewer) | `NNN_subject_log.md` — `NNN` **sama** dengan plan/revision yang diikuti |
| Index tracker | `index.md`, overflow → `index-2.md`, `index-3.md`, ... |

`NNN` = ID sekuensial 3 digit (001, 002, ...), naik terus lintas hari — bukan reset per hari.
`subject` = slug singkat kebab-case yang menjelaskan isi CP.

**Aturan overflow (berlaku ke `index.md` DAN ke setiap file log individual):**
Kalau sebuah file (`index.md` atau `NNN_subject_log.md`) melebihi **500 baris**, buat file
lanjutan dengan suffix `-2` (mis. `index-2.md`, `005_writer-prompt_log-2.md`). AI yang menulis
selanjutnya **wajib** mengikuti/menulis ke versi suffix tertinggi yang ada, bukan file aslinya.

---

## 2. Mode: Planner

- Membuat implementation plan yang dipecah jadi beberapa **checkpoint (CP)**.
- Tiap **step** di dalam CP wajib punya 3 label kualitatif (penilaian AI, bukan angka baku):
  `confidence` (low/medium/high), `risk` (low/medium/high), `token/limit usage estimate`
  (low/medium/high) — dinilai dari kompleksitas step (jumlah node yang disentuh, jumlah AI
  call, panjang draft yang harus dibaca ulang, dsb).
- CP itu sendiri juga punya label ringkasan (gambaran besar dari total semua step di
  dalamnya) — ini yang dipakai untuk keputusan mode-switching di bawah.
- **Kalau CP terdeteksi risk tinggi ATAU token usage tinggi** → Planner wajib memecah step-step
  di CP itu jadi sub-langkah yang lebih kecil (chunking) sebelum diserahkan ke Worker, supaya
  satu run Worker tidak mengerjakan potongan yang terlalu besar.
- **Kalau CP terdeteksi risk tinggi DAN confidence rendah** → Planner wajib memecah jadi dua
  sub-CP: **sub-CP A** (rencana eksekusi untuk Worker) dan **sub-CP B** (rencana khusus untuk
  Reviewer — apa yang harus diperiksa/divalidasi sebelum/sesudah A dieksekusi).
- Planner menulis hasil ke `NNN_subject_plan.md` dan **wajib** menambahkan/update baris CP ini
  di `index.md` (lihat §5).

---

## 3. Mode: Worker

**⚠️ BRIEF WAJIB SEBELUM EKSEKUSI.** Sebelum melakukan satu tindakan pun — bahkan membaca file atau memeriksa workflow — Worker wajib memberi brief: (1) pemahaman CP yang akan dikerjakan, (2) apa yang akan dibuat/diubah, (3) estimasi token/limit yang akan dipakai. Eksekusi tanpa brief tidak sah.

- **Satu run = maksimal satu checkpoint.** Tidak boleh mengerjakan lebih dari 1 CP dalam satu run.
- Punya batas max token per run. Kalau estimasi akan melebihi batas itu di tengah pengerjaan →
  **wajib berhenti** dan menunggu instruksi lanjut dari user (tidak boleh melanjutkan sendiri).
- **Sebelum eksekusi**, wajib memberi brief singkat berisi: pemahaman terhadap CP yang akan
  dikerjakan, apa yang akan dibuat/diubah, dan estimasi token/limit yang akan dipakai.
- Menulis log ke `NNN_subject_log.md` (ID sama dengan plan yang diikuti).
- **Kalau berhenti di tengah checkpoint karena kena max token**: sebelum stop, wajib menulis
  field eksplisit di log — progress yang sudah selesai dan **titik lanjut** (`next action` /
  `resume point`) yang cukup jelas untuk run berikutnya melanjutkan tanpa membaca ulang seluruh
  histori percakapan.
- Wajib update baris index-nya sendiri di `index.md` setelah selesai (atau setelah stop
  paksa karena limit — status jadi `in-progress` dengan catatan resume point).

---

## 4. Mode: Reviewer

- Bisa membuat revision plan dan implementation plan baru dengan aturan yang **sama persis**
  seperti Planner (§2) — termasuk label confidence/risk/token per step, dan aturan pemecahan
  sub-CP A/B kalau relevan.
- Me-review hasil kerja Worker terhadap plan yang diikuti; hasil review ditulis di
  `NNN_subject_report.md`.
- Menulis log ke `NNN_subject_log.md` dengan ID yang sama dengan plan/CP yang sedang di-review.
- **Kalau revision plan mengubah sebuah CP yang sudah ada**: revision itu dapat `NNN` **baru**
  (entry index terpisah), dan wajib mencantumkan link/referensi ke `NNN` CP asal yang direvisi —
  bukan menimpa entry lama, supaya riwayat perubahan tetap terlihat.
- Wajib update index.md untuk entry baru ini + update status entry CP asal (mis. jadi
  `revised`, dengan link ke revision-nya).

---

## 5. Index tracker (`ai_docs/index.md`)

Satu baris per CP/plan. Kolom wajib:

| ID | Subject | Status | Risk | Confidence | Last mode | Link plan | Link log |
|---|---|---|---|---|---|---|---|
| 001 | ... | pending / in-progress / done / revised | low/med/high | low/med/high | planner/worker/reviewer | `001_subject_plan.md` | `001_subject_log.md` |

- **Setiap mode wajib update index.md** untuk baris yang relevan sebagai bagian dari
  definition-of-done mereka masing-masing — bukan tanggung jawab satu mode saja.
- Revision plan → baris baru (lihat §4), bukan overwrite baris CP asal.
- Overflow di 500 baris → lanjut ke `index-2.md` dst. (lihat §1). Saat sudah ada file
  lanjutan, AI wajib menulis entry baru ke situ, dan menyebutkan di baris terakhir file
  sebelumnya bahwa index dilanjutkan di file berikutnya.

---

## 6. Reviewer: reject minor vs revision plan formal

- Reviewer boleh langsung **menolak hasil Worker dan minta re-run CP yang sama** tanpa bikin
  revision plan formal, khusus untuk masalah kecil (typo, salah parameter node, format output
  tidak sesuai, dsb yang tidak mengubah scope/arsitektur CP). Cukup dicatat di log Reviewer:
  `status: rejected-minor`, alasan singkat, dan instruksi perbaikan. Status CP di index.md
  kembali ke `in-progress` (bukan `revised`), ID tetap sama.
- **Revision plan formal (§4) wajib** kalau masalahnya mengubah scope, pendekatan, atau
  arsitektur CP — bukan sekadar salah eksekusi kecil. Aturan praktis: kalau perbaikannya bisa
  dijelaskan dalam 1-2 kalimat instruksi ke Worker → reject-minor. Kalau perlu menjelaskan
  ulang "kenapa" dan "bagaimana" pendekatannya berubah → revision plan.

## 7. `ai_docs/todos.md` — backlog mentah

- `todos.md` adalah **backlog mentah**: checklist sederhana (`- [ ] deskripsi singkat`),
  **tanpa** label risk/confidence/token — itu baru ditentukan Planner saat item diangkat
  jadi CP.
- Boleh ditambah oleh **user maupun Planner** (mis. Planner menemukan sub-task/dependency
  baru saat merencanakan — boleh langsung ditambahkan ke todos.md, bukan cuma disebut di CP).
  Worker dan Reviewer **tidak** menambah item baru ke todos.md — cukup catatan di log/report
  mereka; kalau catatan itu berarti ada kerjaan baru, teruskan ke Planner untuk ditambahkan.
- **Alur:** Planner ambil satu item dari `todos.md` → riset & tulis jadi `NNN_subject_plan.md`
  lengkap dengan label → item asal di `todos.md` dicoret `[x]` dan diberi anotasi
  `-> NNN` (link ke CP yang dihasilkan). Item **tidak dihapus** — riwayat asal-usul CP
  tetap terlihat.
- Sama seperti file log/index, `todos.md` juga overflow di 500 baris → `todos-2.md` dst,
  dan AI wajib menulis ke versi suffix tertinggi yang ada.

## 8. Semua mode wajib baca dokumen desain sebelum kerja

- Sebelum Planner membuat CP, atau Worker/Reviewer mengeksekusi/mereview CP yang menyentuh
  bagian tertentu dari workflow, **wajib baca dulu** dokumen desain yang relevan di `docs/`:
  `IMPLEMENTATION_PLAN.md` (arsitektur & node mapping), `GEO_SCORING_RUBRIC.md` (rubric),
  `PROMPT_GUIDE.md` (prompt tiap role), `ARCHITECTURE_NOTES.md` (pola yang direuse/tidak),
  `OPTIONAL_SCORING_WORKFLOW.md` (bagian opsional), dan `config/content-context.json`.
- Jangan menebak parameter/keputusan desain yang sudah tertulis di dokumen-dokumen itu —
  kalau CP menyentuh sesuatu yang sudah didefinisikan di sana (mis. nomor node, threshold
  skor, nama kolom sheet), rujuk langsung ke dokumennya, jangan reproduksi dari ingatan.
- `IMPLEMENTATION_PLAN.md` §7 ("Dokumen terkait") jadi pintu masuk daftar dokumen ini —
  jaga agar tetap sinkron kalau ada dokumen desain baru ditambahkan.

## 9. Mode switching

- **Otomatis berdasarkan label yang ditetapkan Planner di CP** — tidak perlu user memicu
  manual setiap kali:
  - Risk low/medium + confidence cukup → lanjut ke Worker.
  - Risk tinggi + confidence rendah → split sub-CP A/B (§2); sub-CP B otomatis berarti giliran
    Reviewer merencanakan sebelum/paralel dengan Worker mengerjakan sub-CP A.
  - Worker yang stop karena max token → run berikutnya otomatis tetap di mode Worker,
    melanjutkan dari resume point di log (§3) — bukan kembali ke Planner.
- User tetap bisa override kapan saja (mis. minta Reviewer turun tangan lebih awal), aturan
  otomatis ini hanya default behavior tanpa instruksi eksplisit.
