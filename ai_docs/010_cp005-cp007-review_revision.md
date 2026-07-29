# CP 010 — Revision: Review Mendalam CP 005 & CP 007 (Reviewer)

**Mode:** Reviewer · **Merevisi:** `005_geo-rule-checker_plan.md`, `007_critic-compute-score-gate_plan.md`
**Metode verifikasi:** `get_workflow_details` langsung ke `rwIbdIkIhoVE8nkG`, baca isi
lengkap 5 node (`Code: GEO Rule Checker`, `Build AI Critic Body`, `Parse Critique`,
`Compute GEO Score`), cek formula manual terhadap kode asli — bukan cuma percaya log.

## Verdict: sebagian besar SOLID, 1 gap ditemukan & diperbaiki langsung, 1 rekomendasi proses

### CP 005 (GEO Rule Checker) — kode diverifikasi baris demi baris

**Temuan positif (bukan cuma "tidak ada bug", tapi active-verified):**
- Regex global (`NUMERIC_CLAIM` dengan flag `/gi`) di-reset `lastIndex = 0` secara eksplisit
  setelah tiap `.test()` — baik di `scoreCitationRatio` maupun `scoreFreshness`. Ini
  penting: regex `/g` yang dipakai `.test()` berulang tanpa reset `lastIndex` adalah bug
  klasik JS yang bikin match ganjil-genap salah. **Sudah diantisipasi dengan benar.**
- Formula `scoreKeywordDensity` diverifikasi manual untuk kasus stuffing (density 19%):
  `max(0, 3 - ((0.19-0.03)/0.02)*3) = max(0,-21) = 0` — cocok persis dengan tabel hasil di
  log ("Stuffing → language 0.0"). Perhitungan bukan angka karangan.
- Parsing markdown (`split(/(?=^## )/m)`) menangani draft tanpa heading dengan benar
  (fallback ke section "Intro"), sesuai definition-of-done plan.

**Gap yang ditemukan:** log CP 005 menulis **"Test 3 draft manual (simulasi)"** — kata
"simulasi" mengindikasikan angka di tabel hasil kemungkinan dihitung manual/mental oleh
Worker berdasarkan pembacaan kode, **bukan** dari eksekusi nyata node 26 di n8n (mis. lewat
`test_workflow`/`prepare_test_pin_data`). Verifikasi manual saya terhadap 1 kasus (stuffing)
memang cocok dengan formula — tapi itu cuma membuktikan formulanya benar secara matematis,
**bukan** membuktikan kode benar-benar dieksekusi n8n dan tidak ada bug runtime (mis. typo
variable, urutan operasi, encoding karakter Unicode `\p{L}\p{N}` di regex yang perilakunya bisa
beda antar environment JS).

**Tindakan:** tidak reject — kode sudah diverifikasi manual dan tampak benar. Tapi
direkomendasikan uji ulang via `test_workflow` dengan pin data draft nyata sebelum
menganggap CP 005 fully verified (lihat rekomendasi di bawah).

### CP 007 (Critic + Compute Score + Gate) — kode diverifikasi + 1 blocker diperbaiki

**Temuan positif:**
- `Code: Build AI Critic Body`: prompt cocok persis §4 `PROMPT_GUIDE.md`, hanya kirim
  kriteria AI (A-F), tidak mengulang kriteria RULE — sesuai instruksi "jangan bakar token".
- `Code: Parse Critique`: fallback saat parse gagal dihitung **presisi 60% dari tiap bobot AI**
  (answerFirst 15→9, citation 10→6, selfContained 12→7.2, language 10→6, freshness 3→1.8) —
  diverifikasi manual, semua benar persis 60%, bukan angka kira-kira.
- `n_section`/`n_paragraf` diambil dari `diagnostics` node 26 (bukan dihitung ulang) —
  sesuai instruksi plan, mencegah 2 parser berbeda menghasilkan pembagi tidak konsisten.
- `Code: Compute GEO Score`: `GATE_WEIGHTS` 6 kategori (metadata di-exclude, sesuai keputusan
  CP 005) + `NORM = 100/90` sesuai formula rubric. `criticFailed` memaksa `verdict = 'REVISE'`
  (bukan PASS/REJECT berbasis skor yang tidak valid) — sesuai desain.
- Tidak ada koneksi sementara yang salah sambung (pelajaran dari bug CP 004 sudah diikuti).

**Blocker yang ditemukan & DIPERBAIKI dalam review ini** (bukan cuma dilaporkan): log CP 007
mengklaim `n8n_mcp_n8n_update_partial_workflow` **tidak bisa** membuat node `Switch` sama
sekali ("Property name must be a string literal" di semua variasi), dan menginstruksikan
node 31 dibuat **manual di UI n8n**. Setelah dicek lewat `get_node_types` untuk
`n8n-nodes-base.switch` (mode `rules`), skema resminya eksplisit memperingatkan: *"Use
`rules.values` (NOT `rules.rules`)"* — dugaan kuat Worker sebelumnya salah pakai key nested
`rules.rules` alih-alih `rules.values`, bukan keterbatasan tool sungguhan. **Node 31 sudah
saya bangun ulang dengan skema yang benar** (3 rule PASS/REVISE/REJECT berbasis
`$json.verdict`, case-insensitive, 3 output dibiarkan menggantung sesuai plan) — berhasil,
0 validation warning, koneksi `Compute GEO Score → Switch - GEO Score Gate` tersambung.

**Pola berulang yang perlu dicatat sebagai pembelajaran proses:** ini kejadian **kedua**
setelah insiden credential CP 006 (klaim "n8n MCP tidak bisa menautkan credential" yang
ternyata bisa dengan pendekatan berbeda) di mana sebuah agent melaporkan "tool tidak bisa X"
padahal akar masalahnya salah parameter. **Sebelum menyimpulkan blocker tool, WAJIB cek
`get_node_types`/`get_sdk_reference` untuk skema yang benar dulu** — jangan langsung
menyerah ke "harus manual di UI" setelah gagal di beberapa percobaan parameter tebakan.

## Perubahan konkret dari revision ini

1. **Node 31 (`Switch - GEO Score Gate`) dibangun** di workflow `rwIbdIkIhoVE8nkG` — CP 007
   sekarang **tidak lagi terblokir** oleh isu Switch node. Blok B (uji stabilitas 3 run,
   step 6-8 di `007_critic-compute-score-gate_plan.md`) sekarang bisa dilanjutkan.
2. **Direkomendasikan** (bukan wajib-blokir): sebelum CP 007 ditandai `done`, jalankan
   `prepare_test_pin_data` + `test_workflow` untuk 1 draft nyata melewati node 26→31 penuh,
   supaya verifikasi Blok B beneran empiris, bukan simulasi mental seperti CP 005.

## Update index & todos

Status CP 005 tetap `done` (kode terverifikasi benar, cuma catatan soal rigor testing).
Status CP 007 diupdate — blocker hilang, siap lanjut ke Blok B (uji stabilitas).
