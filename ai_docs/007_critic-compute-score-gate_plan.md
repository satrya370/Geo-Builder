# CP 007 — AI Critic, Compute GEO Score & Score Gate (Node 27–31)

**Mode:** Planner · **Asal:** `ai_docs/todos.md` — "Bangun node 27–31: Critic + Compute GEO
Score, validasi skor gabungan stabil antar run"
**Dokumen rujukan wajib dibaca sebelum eksekusi:**
`docs/GEO_SCORING_RUBRIC.md` — terutama §"Agregasi kriteria AI → `aiScores` per kategori"
(BARU, ditulis khusus untuk CP ini), §"Formula skor akhir (gate, node 30 — TANPA Kategori 6)",
§Threshold · `docs/PROMPT_GUIDE.md` §4 (prompt Critic, copy PERSIS) ·
`docs/IMPLEMENTATION_PLAN.md` §3 Fase 3 & §3.5 (pola credential) ·
`docs/ARCHITECTURE_NOTES.md` §"Human-in-the-loop approval" (kenapa node 31 kritis) ·
output node 26 di workflow `rwIbdIkIhoVE8nkG` (`ruleScores` + `diagnostics`)

## Celah desain yang ditutup sebelum plan ini ditulis

Rubric mendefinisikan kriteria AI 5b (quotable) dan 7b (freshness) secara naratif **tanpa
formula skor** — padahal node 29 harus mengubah temuan mentah Critic jadi angka. Sudah
dilengkapi di `GEO_SCORING_RUBRIC.md`:
- **5b:** `(n_section_dengan_count≥1 / max(1, n_section)) × 3` — yang dinilai cakupan, bukan
  total kalimat.
- **7b:** `max(0, 1 − n_flag / max(1, n_section)) × 3` — proporsional, konsisten dengan 2b.
- **Tabel agregasi lengkap** (6 field Critic → 5 kategori) + aturan fallback saat Critic gagal
  parse: `aiScores` diisi **netral 60% bobot**, `verdict` dipaksa `REVISE` — **bukan** 0
  (yang akan menjatuhkan draft bagus ke REJECT karena kegagalan teknis, bukan kualitas).

## Ringkasan CP (gambaran besar)

- **Confidence:** medium — prompt Critic sudah lengkap di `PROMPT_GUIDE.md` §4 dan formula
  agregasi baru dilengkapi, tapi logic konversi temuan→skor belum pernah dijalankan, dan
  stabilitas skor antar-run belum terbukti (itu justru yang harus diuji di CP ini).
- **Risk:** medium-high — node 31 adalah **satu-satunya penjaga kualitas** sebelum artikel
  tayang publik (tidak ada approval manusia, lihat `ARCHITECTURE_NOTES.md`). Salah kalibrasi
  di sini berarti artikel jelek lolos publish, atau artikel bagus tertahan selamanya di
  revise loop.
- **Token/limit usage estimate:** medium saat build, **high saat uji stabilitas** — 1 AI call
  per run (~3.4–4.1k token, Critic membaca draft penuh), dan step uji stabilitas menjalankan
  draft yang sama 3× → ~12k token untuk pengujian saja.
- **Kesimpulan:** risk medium-high tapi confidence medium (bukan rendah) → **tidak** memenuhi
  syarat split sub-CP A/B (`rules.md` §2 butuh risk tinggi **DAN** confidence rendah). Tapi
  karena token uji tergolong tinggi, step **di-chunk jadi 2 blok** dengan titik stop eksplisit
  (lihat tabel) — Worker boleh berhenti di antara blok dan menunggu instruksi lanjut.

## Langkah

### Blok A — Bangun (token rendah, tidak ada AI call sampai step 6)

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Node 27 `Code: Build AI Critic Body` — prompt **copy PERSIS** dari `PROMPT_GUIDE.md` §4 (termasuk kriteria A–F dan skema JSON output). Ambil `draft` dari `$('Code: Parse Draft')`, `targetKeyword`+`entities` dari `$('Code: Parse Outline')`. **Kirim hanya kriteria AI** — kriteria RULE sudah dihitung gratis di node 26, mengirimnya lagi cuma membakar token | high | low | low |
| 2 | Node 28 `HTTP: AI GEO Critic` — POST `https://openrouter.ai/api/v1/chat/completions`, model `openai/gpt-oss-120b`, `temperature: 0.2` (sesuai `content-context.json` → `models.roles.critic`), `response_format: json_object`, timeout 120s. Credential **WAJIB** pola §3.5: `genericCredentialType` + `httpHeaderAuth` → `setNodeCredential` ke `oSM2yzrS4VtlyIcj` ("OpenRouter Authorization - GPT OSS 120B") | high | medium | low |
| 3 | Node 29 `Code: Parse Critique` — implementasi tabel agregasi di `GEO_SCORING_RUBRIC.md` §"Agregasi kriteria AI". `n_section`/`n_paragraf` **WAJIB** dibaca dari `$('Code: GEO Rule Checker').first().json.diagnostics`, bukan dihitung ulang. Sertakan fallback `criticFailed` (netral 60% + paksa REVISE) | medium | medium | low |
| 4 | Node 30 `Code: Compute GEO Score` — copy formula `GATE_WEIGHTS` + `NORM = 100/90` persis dari rubric §"Formula skor akhir (gate)". Gabung `ruleScores` (node 26) + `aiScores` (node 29). Output **wajib** membawa: `geoScore`, `verdict`, `breakdown`, `failedCategories`, dan `critique.topFixes` (dibutuhkan Reviser di CP 008) | high | low | low |
| 5 | Node 31 `Switch - GEO Score Gate` — 3 output berdasarkan `verdict`: `PASS` (≥80) / `REVISE` (60–79) / `REJECT` (<60). **Ketiga output SENGAJA dibiarkan tidak tersambung** — targetnya (node 37 Schema, node 32 Reviser, node 51 error path) belum dibangun | high | low | low |

> **STOP POINT** — sampai sini belum ada AI call yang dijalankan. Worker boleh berhenti dan
> lapor sebelum masuk Blok B kalau estimasi token mepet.

### Blok B — Uji (di sinilah token terpakai)

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 6 | Uji fungsional 1×: jalankan 1 draft uji lewat node 26→31, verifikasi `aiScores` masuk rentang bobot (tidak negatif/melebihi max), `geoScore` 0–100, `verdict` konsisten dengan threshold | medium | medium | medium |
| 7 | **Uji stabilitas**: jalankan **draft yang sama 3×**, catat `geoScore` tiap run. Target: sebaran ≤ **±3 poin**. Kalau lebih lebar → turunkan temperature ke 0.1 atau pertajam kriteria di prompt; **jangan** naikkan threshold untuk menutupi ketidakstabilan | medium | medium | **high** (3 AI call) |
| 8 | Uji fallback: paksa Critic gagal (mis. sementara arahkan URL ke endpoint invalid), pastikan node 29 **tidak** menghentikan run dan menghasilkan `verdict: REVISE` + `criticFailed: true` — bukan REJECT | high | low | low |
| 9 | Verifikasi via `get_workflow_details` langsung ke n8n: credential node 28 benar-benar tertaut, koneksi 26→27→28→29→30→31 utuh, output node 31 memang 3 cabang | high | low | low |
| 10 | Tulis `007_critic-compute-score-gate_log.md` (catat skor 3 run uji stabilitas — ini jadi baseline kalibrasi awal), update `index.md`, coret item `todos.md` | high | low | low |

## Catatan teknis

- **Pembagi harus sama antara rule dan AI.** `n_section`/`n_paragraf` diambil dari
  `diagnostics` node 26. Kalau node 29 menghitung ulang dengan parser sendiri, dua parser bisa
  menghasilkan angka berbeda dan skornya jadi tidak konsisten tanpa ketahuan — bug yang sangat
  sulit dilacak karena tidak melempar error.
- **JANGAN buat koneksi sementara** dari output node 31 ke node yang belum relevan. Ini
  pengulangan bug CP 004 (`Form Trigger → HTTP: AI Writer` "sementara" yang duduk di workflow
  live). Cabang menggantung lebih aman daripada cabang yang salah sambung.
- **Kategori 6 tetap TIDAK dihitung di node 30** — keputusan CP 005, dicek terpisah di node 39b
  setelah Schema. `GATE_WEIGHTS` di rubric sudah cuma 6 kategori; jangan tambahkan `metadata`.
- **Kontrak untuk CP 008 (revise loop):** node 36 (`Parse Revised Draft`) nanti harus output
  field bernama **`draft`** dan menyambung balik ke node 26 — karena node 26 membaca
  `$input.first().json.draft`. Kalau node 36 memakai nama field lain, loop re-scoring akan
  membaca `undefined` dan skor iterasi kedua tidak valid.
- **Kenapa Critic dipisah dari Writer** (jangan digabung "sekalian" demi hemat 1 call): model
  yang menilai tulisannya sendiri bias optimistis. Ini alasan arsitektural, bukan preferensi —
  lihat `IMPLEMENTATION_PLAN.md` §3 Fase 3.
- **Baseline kalibrasi:** skor 3 run di step 7 wajib dicatat di log. Setelah ~10 artikel nanti
  (`todos.md` item kalibrasi), angka ini jadi pembanding untuk menilai apakah threshold 80
  terlalu ketat/longgar — tanpa baseline, kalibrasi cuma tebakan.
- **Tidak menyentuh:** node 32–36 (Revise, CP 008), node 37–39+39b (Schema), node 40–42
  (WordPress), node 46–54 (logging/notif/error path).

## Definition of done

- [ ] Node 27–31 ada di workflow `rwIbdIkIhoVE8nkG`, koneksi 26→27→28→29→30→31 utuh
- [ ] Credential node 28 **terverifikasi tertaut** lewat `get_workflow_details` (bukan cuma
      `nodeCredentialType` di parameters — pelajaran CP 003/004)
- [ ] Node 29 membaca `n_section`/`n_paragraf` dari `diagnostics` node 26, bukan hitung ulang
- [ ] `aiScores` tiap kategori dalam rentang bobotnya (answerFirst≤15, citation≤10,
      selfContained≤12, language≤10, freshness≤3, structure=0)
- [ ] `geoScore` 0–100 dan `verdict` cocok threshold (≥80 PASS / 60–79 REVISE / <60 REJECT)
- [ ] Uji stabilitas 3 run tercatat di log, sebaran ≤ ±3 poin (atau kalau lebih lebar:
      tercatat tindakan yang diambil dan hasilnya)
- [ ] Fallback Critic-gagal menghasilkan REVISE + `criticFailed: true`, **tidak** REJECT dan
      tidak menghentikan run
- [ ] 3 cabang node 31 dibiarkan tidak tersambung (tidak ada koneksi sementara)
- [ ] Log ditulis, `index.md` baris CP 007 ditambahkan, `todos.md` dicoret `-> 007`

## Next mode

Worker — brief dulu sebelum eksekusi (§3 `rules.md`). Boleh berhenti di STOP POINT antara
Blok A dan Blok B kalau estimasi token mepet; lanjutkan di run berikutnya dengan resume point
dicatat di log (§3 `rules.md`).
