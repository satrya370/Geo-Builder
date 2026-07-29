# Laporan CP 012 — Audit KasUS TEPING (Reviewer)

**Mode:** Reviewer · **Metode:** `get_workflow_details` langsung + analisis `execution 936` + inspeksi struktur koneksi

## Verdict: **TIDAK LULUS** — 5 temuan kritis, 4 sedang, 2 rendah

Pipeline jalur utama (Form→Guard→Research→Outline→Writer→Score→Schema→WordPress) terbukti
berfungsi (execution 936, geoScore 85, PASS, wpPostId 7). Tapi **3 cabang keluar**
(REJECT, blocked-topic, retry-exceeded) semua **menggantung diam-diam** — ini berarti
skenario error-common di produksi akan **menghilang tanpa notifikasi**.

---

## Temuan (urut prioritas)

### 1. KRITIS — Cabang REJECT Switch hilang sepenuhnya
**Skenario:** Verdict <60 (REJECT).
**Bukti:** Struktur koneksi `Switch - GEO Score Gate`:
```json
"main": [
  [{"node": "Code: Build AI Schema Body"}],     // main[0] = PASS
  [{"node": "Code: Check Retry Limit"}]          // main[1] = REVISE
  // main[2] = REJECT → TIDAK ADA
]
```
Hanya 2 dari 3 output. Switch node hanya menghasilkan 2 cabang — REJECT (main[2]) tidak
tersambung. **Item dengan verdict REJECT di-drop diam-diam** tanpa error, tanpa notifikasi.
**Rekomendasi:** Tambahkan output REJECT ke Switch (atau gunakan fallback output), sambungkan
ke node 51+ (error path) saat dibangun. Sementara itu, tambahkan NoOp dengan note
"log entry reject" supaya tidak hilang diam-diam.

### 2. KRITIS — Topik yang ditolak Safety Guard menghilang tanpa response
**Skenario:** User submit topik yang ditolak Safety Guard (mis. "obat apa untuk kecemasan saya").
**Bukti:** Koneksi `If - Topic Allowed?`:
```json
"main": [
  null,                                          // main[0] = FALSE → TIDAK ADA
  [{"node": "Code: Build AI Research Body"}]     // main[1] = TRUE
]
```
Port 0 (false branch) = `null`. Node 15 (`Respond - Topic Blocked`) **tidak ada di workflow**
(bukan bagian dari 39 node). User yang submit topik yang ditolak akan mendapat **form response
kosong atau hang** — tidak ada pesan "topik Anda ditolak karena...".
**Rekomendasi:** Bangun node 15 (`Respond - Topic Blocked`) dengan response JSON yang menjelaskan
alasan penolakan dari `guard_reason`. Sambungkan `If - Topic Allowed?` port 0 ke node 15.

### 3. KRITIS — Iterasi revisi yang melebihi batas (reviseCount > 2) menghilang
**Skenario:** Draft gagal scoring 2x berturut-turut, reviseCount mencapai 3.
**Bukti:** Koneksi `If - Retry Available?`:
```json
"main": [
  [{"node": "Code: Build AI Reviser Body"}]  // main[0] = TRUE (retry available)
  // main[1] = FALSE (exceeded) → TIDAK ADA
]
```
Port 1 (false branch = exceeded retry limit) tidak tersambung. Item di-drop diam-diam.
**Rekomendasi:** Sambungkan main[1] ke node error path (51+) saat dibangun. Sementara itu,
tambahkan NoOp dengan note "retry exceeded — needs error handling".

### 4. SEDANG — Jalur error (node 51-54) tidak dibangun sama sekali
**Skenario:** Kegagalan teknis di mana pun sepanjang pipeline (API timeout, Ollama mati,
parse error yang tidak tertangkap).
**Bukti:** Node 51-54 tidak ada di workflow (39 node total, tidak ada `Code: Build Failure
Context`, `Telegram - Operational Alert`, dll).
**Rekomendasi:** Bangun error path sebagai CP terpisah (per IMPLEMENTATION_PLAN.md Fase 8).
Sementara itu, semua error HTTP node menggunakan `continueErrorOutput` — error item mengalir
ke node Parse yang mungkin tidak handle error dengan baik (Parse Research menerima error item,
parse akan gagal, lalu throw).

### 5. SEDANG — Biaya token nyata JAUH melebihi estimasi `IMPLEMENTATION_PLAN.md` §5
**Bukti:** Actual token usage dari execution 936 vs estimasi lama:

| Role | Estimasi (lama) | Actual (exec 936) | Selisih |
|------|----------------|-------------------|---------|
| Safety Guard | ~0.3-0.1k | 1,163 | 4× lebih |
| Researcher | ~2.6-3.3k | 8,400+ | 2.5× |
| Writing Planner | (tidak ada di estimasi) | 5,000+ | baru |
| Writer | ~4.5-5.5k | 3,500+ | dalam range |
| Critic | ~3.4-4.1k | 5,000+ | 1.2× |
| **Total per artikel** | ~15-18.4k | **23,065+** | **1.3-1.5× lebih** |

(Writing Planner adalah node baru yang tidak ada di estimasi lama — menambah ~5k token per run.)
**Rekomendasi:** Update §5 `IMPLEMENTATION_PLAN.md` dengan angka aktual. `max_tokens`
Researcher naik ke 16000 (dari 2048), menambah potensi biaya signifikan kalau model reasoning
benar-benar menghabiskannya.

### 6. SEDANG — Locking / concurrency tidak ada
**Skenario:** 2 form submit bersamaan, atau (nanti) cron + form bersamaan.
**Bukti:** Node 1 (cron), 3, 5, 7, 8, 9 (locking) tidak dibangun. Tidak ada Google Sheets
locking di jalur form-only. Tidak ada deteksi duplicate run.
**Rekomendasi:** Risiko rendah saat ini (jalur form-only, kemungkinan bersamaan kecil), tapi
**wajib ditangani sebelum cron ditambahkan**. Diklasifikasi sebagai "ditunda" di CP 003
("Tunda dulu" — keputusan user). Jangan update status ini tanpa konfirmasi user.

### 7. RENDAH — Semua `finish_reason: "stop"` (tidak ada bug max_tokens)
**Bukti:** Dari execution 936, semua 5 HTTP AI node menunjukkan `finish_reason: "stop"`:
- Safety Guard: stop
- Researcher: stop
- Writing Planner: stop
- Writer: stop
- Critic: stop
**Status:** Bug "max_tokens habis untuk reasoning" (CP 011 bug #1) sudah teratasi.
`max_tokens` yang dinaikkan (Safety 3000, Researcher 16000, Planner 7000, Critic 5000,
Writer tw*2.2+2500) cukup untuk reasoning semua model.
**Rekomendasi:** Tidak ada tindakan diperlukan.

### 8. RENDAH — Test REVISE loop belum dilakukan
**Bukti:** Execution 936 langsung PASS (85) — loop tidak ter-trigger. Belum ada execution
lain yang masuk jalur REVISE.
**Rekomendasi:** Perlu tes dengan temporary threshold change (`geoScore >= 95` di node 30)
untuk memaksa REVISE, verifikasi `reviseCount` naik dan `geoScore` berubah antar iterasi.
Tes ini tidak bisa dilakukan via MCP tanpa aktivasi workflow + form submit.

### 9. RENDAH — `word_count_target` override belum diuji
**Skenario:** User pilih "2000" atau "2500" di form.
**Bukti:** Execution 936 menggunakan "Auto" — `totalTargetWords` di-override ke default 1500.
Belum ada test dengan eksplisit "2000"/"2500".
**Rekomendasi:** Tes dengan `word_count_target: "2500"`, verifikasi `max_tokens` Writer
mengikuti (`2500 × 2.2 + 2500 = 8000`).

### 10. RENDAH — `related_article_url` belum diuji
**Skenario:** User isi URL artikel terkait.
**Bukti:** Execution 936 menggunakan "" (kosong) — aturan 15 Writer dilewati.
**Rekomendasi:** Tes dengan URL dummy, verifikasi Writer menyisipkan link di draft.

---

## Yang TIDAK bisa diuji via MCP (butuh aktivasi workflow + form submit manual)

- **Input edge case** (point 1): topic_detail pendek/panjang, category_hint eksplisit,
  key_entities/faq_seed diisi, semua opsional kosong — semua butuh eksekusi nyata
- **Safety Guard rejection** (point 2): butuh form submit dengan topik medis
- **REVISE loop** (point 3): butuh threshold sementara + form submit
- **WordPress GET post 7** (point 5): butuh HTTP GET ke WordPress API

## Yang TERBUKTI berfungsi (dari execution 936)

- ✅ Pipeline utama (Form→Guard→Research→Outline→Planner→Writer→Score→Schema→WordPress)
- ✅ Safety Guard menilai dengan benar (allowed: true untuk topik filsafat)
- ✅ Researcher menghasilkan 10 fakta bersumber nyata (Stanford, NIH, dll)
- ✅ Outline Planner (Ollama local) menghasilkan JSON valid
- ✅ Writing Planner menghasilkan brief per-section
- ✅ Writer menghasilkan draft lengkap
- ✅ GEO Rule Checker menghitung 6 kategori
- ✅ Critic menilai dengan JSON valid
- ✅ Compute GEO Score: 85/100, PASS
- ✅ Schema Generator (Ollama) menghasilkan metadata valid
- ✅ WordPress Create Post berhasil (wpPostId: 7, status draft)
- ✅ Semua finish_reason: "stop" (tidak ada truncation)
- ✅ Semua credential berfungsi (bukti: eksekusi berhasil, bukan asumsi)