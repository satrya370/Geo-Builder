# Architecture Notes — Pola yang Direuse dari `ig Content Builder`

Diambil dari inspeksi workflow `ig Content Builder For Demo` (n8n ID `V8VbQm1KWpe7jj7Y`,
**82 node**, status active) via n8n MCP. Yang diinspeksi adalah **daftar dan tipe node**
untuk memahami arsitekturnya — bukan isi prompt atau kredensialnya.

> Workflow produksi `ig Content Builder` (ID `ubTXPJPsMYWwmgDI`) **tidak disentuh** sama
> sekali, sesuai aturan yang berlaku di workspace ini.

> **Workflow target GEO Builder** (tempat node-node di plan ini dibangun):
> `http://localhost:5678/workflow/rwIbdIkIhoVE8nkG` (ID `rwIbdIkIhoVE8nkG`) — bukan
> workflow inspirasi di atas.

---

## 8 pola yang diadopsi

### 1. Dual-trigger + Sheets locking

**Di ig-content-builder:** `Form Trigger - Manual Run` + jalur cron
(`Code: Set Cron Run Context` → `If - Cron Run?`) bertemu di
`Google Sheets: Claim Content Plan Slot`.

**Diadopsi jadi:** node 1–10, dengan `Google Sheets: Claim Topic Slot`.

Tanpa locking, cron dan form-trigger yang jalan bersamaan bisa memproses topik yang sama →
dobel biaya token dan dua artikel identik ter-publish. Ini bukan edge case teoretis; ini
konsekuensi langsung dari memilih dual-trigger.

### 2. Safety Guard sebelum generation, bukan sesudah

**Di ig-content-builder:** `Build Safety Guard Body` → `AI Safety Guard` →
`Parse Guard Result` → `If Topic Allowed?` — ditempatkan sebelum tahap riset.

**Diadopsi jadi:** node 11–15.

Menolak topik setelah riset berarti sudah membakar ~3k token untuk hasil yang dibuang.

### 3. Retry limit sebagai node eksplisit

**Di ig-content-builder:** `Check Retry Limit` sebagai Code node tersendiri.

**Diadopsi jadi:** node 32–33 sebagai penjaga Reviser loop (max 2x).

Counter dibawa **di dalam item data** (`reviseCount`), bukan diandalkan ke struktur
koneksi — n8n tidak punya loop-back native yang aman tanpa state, dan loop tanpa counter
akan membakar token sampai kamu sadar.

### 4. Parse node terpisah setelah setiap AI call

**Di ig-content-builder:** konsisten di semua tahap — `AI Research` → `Parse Research`,
`AI Brief Generator` → `Parse Brief`, `AI Planner` → `Parse Plan`,
`AI Caption Generator` → `Parse Caption`, plus `Validate Output` / `Code: Validate Image
Candidates` sebagai lapisan tambahan.

**Diadopsi jadi:** setiap `HTTP: AI *` diikuti `Code: Parse *`.

Ini paling kritis di role Outline & Schema kita, karena keduanya pakai model lokal 7B yang
lebih rawan salah format JSON daripada model frontier. Parse node harus bisa gagal dengan
anggun (fallback), bukan melempar error yang membatalkan seluruh run.

### 5. Google Sheets sebagai state + audit trail, bukan hanya sumber input

**Di ig-content-builder:** `Load Content History` (cek duplikat),
`Log to Content Log`, `Mark Content Plan Success`, `Mark Content Plan Failed`.

**Diadopsi jadi:** sheet `topic_queue` (state + lock) dan `article_log` (audit).

Tambahan kita: kolom `score_breakdown` per-kategori. Ini yang mengubah log dari sekadar
catatan menjadi alat diagnosis — kalau setelah 20 artikel kategori Citation selalu terendah,
masalahnya di prompt Researcher, bukan di Writer.

### 6. Jalur error terpisah dari happy path

**Di ig-content-builder:** `Code: Build Run Alert` / `Code: Build Failure Context` →
`Telegram - Send Operational Alert`, terpisah dari jalur sukses.

**Diadopsi jadi:** node 51–54.

Wajib karena cron jalan tanpa ada manusia di depan layar — kegagalan yang tidak
memberitahu siapa pun akan terdeteksi berhari-hari kemudian.

### 7. Restore context setelah node yang memutus aliran data

**Di ig-content-builder:** `Code: Restore Run Context`, `Code: Restore Final Output`,
`Code: Restore Demo Access Context`.

**Diadopsi jadi:** `Code: Build Run Context` (node 10) membawa satu objek konteks utuh
sepanjang workflow.

Node seperti Google Sheets dan HTTP Request **menimpa** item data dengan response-nya
sendiri. Tanpa pola restore, konteks run (topik, runId, content context) hilang di
tengah jalan — ini penyebab bug yang paling sering muncul dan paling membingungkan
saat membangun workflow panjang di n8n.

### 8. Kerja berat dilempar ke luar n8n

**Di ig-content-builder:** `Execute Command: Build & Render` memanggil script Node.js
eksternal untuk render Puppeteer, bukan menulis logikanya di Code node.

**Belum diadopsi** — GEO Rule Checker masih cukup nyaman sebagai jsCode inline. Tapi kalau
rubric-nya tumbuh (mis. butuh library NLP untuk deteksi entitas), pindahkan ke
`scripts/geo-rule-checker.mjs` dan panggil via Execute Command. Pola ini sudah terbukti
jalan di workflow sebelahnya.

---

## Pola yang SENGAJA TIDAK diadopsi

### Human-in-the-loop approval (Telegram Wait)

**Di ig-content-builder ada dua gate:** `Telegram - Morning Brief` + `Wait - User Response`
(approve topik), dan `Poll: Slide Review` + `Wait - Slide Review` (approve hasil), lengkap
dengan `Set Wait State` dan `Parse * Reply`.

**Keputusan: di-skip** (permintaan eksplisit). GEO Builder auto-publish live ke WordPress
(Medium di-drop, lihat `IMPLEMENTATION_PLAN.md` §1.1 — CP 001); Telegram hanya mengirim
notifikasi hasil (skor + link) dan alert error.

Konsekuensi yang perlu disadari: **`Switch - GEO Score Gate` (node 31) adalah satu-satunya
penjaga kualitas** sebelum artikel tayang publik. Di ig-content-builder, manusia jadi
jaring pengaman terakhir; di sini tidak ada. Karena itu:

- Threshold 80 sebaiknya tidak diturunkan hanya karena banyak draft tertahan.
- Jalur REJECT (`<60`) harus benar-benar berhenti dan alert, bukan "publish saja dengan
  catatan".
- Validasi fakta di node 19 (`Validate Research Facts`) jadi lebih penting, karena tidak
  ada manusia yang akan menangkap statistik halusinasi sebelum tayang.

Kalau nanti ternyata ada artikel bermasalah yang lolos, pola `Wait` + `Parse Reply` dari
ig-content-builder bisa disisipkan sebelum node 40 tanpa mengubah bagian lain.

### Demo quota / access control

`Normalize Demo Access`, `Hash Demo Email`, `Get Daily Demo Usage`,
`Assess Daily Demo Quota`, `Respond - Daily Limit Reached` — seluruh blok ini spesifik
untuk kebutuhan demo publik ig-content-builder (membatasi pemakaian orang luar).
Tidak relevan untuk workflow internal ini.

### Image pipeline

`Execute Command: Resolve Images`, `Code: Read Image Candidates`,
`Code: Validate Image Candidates` — GEO Builder text-only untuk sekarang
(keputusan eksplisit). Kalau nanti featured image diaktifkan, pola resolve→validate
ini yang dipakai sebagai acuan.

---

## Perbandingan skala

| | ig Content Builder | GEO Builder (rencana) |
|---|---|---|
| Jumlah node | 82 | ~51 (54 dikurangi node 43–45 Medium yang di-drop, CP 001) |
| AI call | 4–5 (Research, Brief, Planner, Caption, Guard) | 7 (Guard, Research, Outline, Writer, Critic, Reviser, Schema) |
| Approval gate | 2 (Telegram Wait) | 0 |
| Output | Gambar carousel IG (Puppeteer) | Artikel markdown → WordPress (WordPress.com, OAuth2) |
| Panjang output | Teks pendek per slide (3–5 slide) | Artikel ~1500 kata |
| Token/run | ~3–6k (estimasi) | ~15–18.4k ditagih (~19.4–23.4k total) |

Node-nya lebih sedikit meski AI call-nya lebih banyak, karena tidak ada blok approval,
demo-quota, dan image pipeline. Yang lebih berat justru **token**: output artikel jauh lebih
panjang, dan tiga role terakhir (Critic, Reviser, Schema) masing-masing membaca ulang draft
penuh yang sama.
