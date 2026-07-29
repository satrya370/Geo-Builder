# CP 020 — Article Quality Overhaul Log

## 2026-07-28 — Riset referensi gaya

Status: PASS — fase riset/analisis selesai; implementasi belum dimulai.

Plan ditambah dengan §B.7 yang membandingkan Oliver Burkeman, Heather Havrilesky, Alain de Botton, Maria Popova, dan Mark Manson. Rekomendasi utama adalah Burkeman dengan pengaruh terbatas Havrilesky, disertai instruksi prompt-ready, contoh kalibrasi, batas anti-imitasi, dan koreksi terhadap risiko formulaik di §B.4. Tidak ada node n8n, config, atau eksekusi yang diubah. Implementasi menyusul pada batch berikutnya setelah hasil riset ini direview.

## 2026-07-28 — Review kedalaman argumen

Status: PASS — review Claude dievaluasi dan plan disempurnakan; implementasi belum dimulai.

Ditambahkan §B.8 untuk menutup gap antara suara yang baik dan argumen yang berkembang: tiga pola struktur, enum `logicalMove`, fungsi adegan/sumber, kontrak parser, larangan silent fallback Planner, aturan Writer untuk kedalaman, contoh pembanding, dan rubrik editorial manual PASS/GAGAL. Usul penutup-berupa-pertanyaan dan urutan adegan-sebelum-klaim dilonggarkan agar tidak menjadi formula baru. Workflow diperiksa via n8n MCP tetapi tidak diubah; tidak ada eksekusi dijalankan.

## 2026-07-28 — Batch 1: rendering WordPress

Status: PASS_WITH_WARNING — batch 1 selesai; batch berikutnya menunggu instruksi.

Perubahan workflow `rwIbdIkIhoVE8nkG`: ditambahkan node `Code: Markdown to WordPress HTML` pada false branch `If - Is Test Mode?`, sebelum `Code: Build WordPress Payload`. Renderer menghasilkan `<h2>`, `<h3>`, `<ul>`, `<ol>`, `<blockquote>`, `<p>`, link, dan bold/italic; Markdown table diubah menjadi teks biasa dengan warning; H1 Markdown dihilangkan; metadata setelah `---METADATA---` tidak dirender. Payload WordPress sekarang memakai `_wpHtml` dan tidak lagi menyisipkan JSON-LD `<script>`.

Validasi n8n: PASS, 0 error, 6 warning lama tentang error output HTTP yang belum tersambung. Workflow tetap inactive. Tidak ada execution, publish WordPress, Google Docs upload, atau perubahan config yang dilakukan. Batch 2 belum dikerjakan.

## 2026-07-28 — Batch 2: Concept Brief dan Outline argument contract

Status: PASS_WITH_WARNING — batch 2 selesai; batch berikutnya menunggu instruksi.

Perubahan workflow `rwIbdIkIhoVE8nkG`: Concept Brief sekarang menghasilkan dan parser memvalidasi `argumentPattern` (`REVERSAL`/`ESCALATION`/`DIALECTIC`), `patternRationale`, dan `expectedInsight`. Outline sekarang menghasilkan `lede`, `thesis`, `closing`, `closingDelta`, `sectionClaim`, `logicalMove`, dan kontrak keberatan (`objectionBasis`, `concession`, `thesisEffect`). Parser Outline memvalidasi enum, field wajib, jumlah section 4–6, FAQ minimal 3, dan duplicate `sectionClaim` identik.

Validasi n8n: PASS, 0 error, 6 warning lama tentang error output HTTP yang belum tersambung. Workflow tetap inactive. Tidak ada execution, publish, upload, perubahan Writer/Planner, atau perubahan config. Batch 3 belum dikerjakan.

## 2026-07-28 — Batch 3: Writing Planner dan Writer

Status: PASS_WITH_WARNING — batch 3 selesai; batch berikutnya menunggu instruksi.

Perubahan workflow `rwIbdIkIhoVE8nkG`: Writing Planner sekarang menerima thesis, closingDelta, argument pattern, sectionClaim, dan logicalMove; outputnya wajib membawa `scenePlan`, `sourceUses`, `transitionClaim`, `stanceNote`, dan `quotableDraft`. Parser Planner tidak lagi melakukan silent fallback ke outline mentah dan memvalidasi heading/claim, logicalMove, fungsi adegan, fungsi sumber, serta transisi. Writer sekarang menerima seluruh kontrak argumen dan aturan kedalaman: scene-to-claim, source impact, keberatan, transisi, closingDelta, integritas sumber, larangan tabel/emoji/label, dan larangan memoir klinis yang dibuat-buat.

Validasi n8n: PASS, 0 error, 6 warning lama tentang error output HTTP yang belum tersambung. Workflow tetap inactive. Tidak ada execution, publish, upload Google Docs, atau perubahan config. Batch 4 belum dikerjakan.

## 2026-07-28 — Batch 4: deterministic quality enforcement

Status: PASS_WITH_WARNING — batch 4 selesai; batch berikutnya menunggu instruksi.

Perubahan workflow `rwIbdIkIhoVE8nkG`: `Code: GEO Rule Checker` sekarang menghapus insentif skor untuk heading pertanyaan dan tabel, melonggarkan keyword density agar kepadatan rendah tidak dihukum, serta menghasilkan `qualityGuards` untuk tabel Markdown, emoji, sisipan berlabel, pembuka yang hilang, dan penutup yang hilang. Tabel/emoji/label menjadi hard-fail yang memaksa `REVISE` melalui `Code: Compute GEO Score`; pembuka/penutup tetap berupa diagnostics karena tidak aman dinilai penuh dengan regex.

Validasi n8n: PASS, 0 error, 6 warning lama tentang error output HTTP yang belum tersambung. Workflow tetap inactive. Tidak ada execution, publish, upload Google Docs, atau perubahan config. Batch 5 belum dikerjakan.

## 2026-07-28 — Batch 5: E2E verification attempt

Status: BLOCKED — Batch 5 belum lulus; menunggu perbaikan/validasi berikutnya.

Workflow diaktifkan sementara dan test form `Concept Only` dijalankan tanpa mode publish. Execution `1018` mencapai Concept Brief dan Outline Planner, lalu berhenti di `Code: Parse Outline` karena model menghasilkan 3 section dan 2 FAQ, sedangkan kontrak Batch 2 mensyaratkan 4–6 section dan minimal 3 FAQ. Parser menolak output sesuai desain; ini membuktikan gate bekerja, tetapi pipeline belum lulus E2E.

Percobaan kedua dibuat dengan Topic Detail dan FAQ Seed yang lebih eksplisit (`1019` form-waiting), tetapi browser/form waiting backend timeout dan tidak menghasilkan execution yang dapat diverifikasi. Workflow dikembalikan inactive. Tidak ada WordPress publish, Google Docs upload, atau perubahan config. Batch 5 tetap terbuka; batch tidak boleh dinyatakan selesai sebelum form benar-benar disubmit dan execution mencapai Writer, GEO Checker, renderer, serta output akhir.

## 2026-07-28 — Batch 5 blocker fix attempt

Status: BLOCKED — contract fixes pass static validation; E2E still not verified.

Perbaikan: prompt Outline diperketat menjadi tepat 5 section isi dan minimal 4 FAQ dengan self-check sebelum JSON selesai. Prompt/field contract tetap ketat; tidak ada padding section palsu. Parser Planner diubah agar Outline menjadi sumber kebenaran untuk `heading/sectionClaim` berdasarkan index, sementara `logicalMove`, scene, source, dan transition tetap divalidasi.

Validasi workflow: PASS, 0 error, 6 warning lama. Test baru dibuat (`1021` form-waiting), tetapi form waiting/backend tidak menghasilkan execution setelah timeout; browser backend juga tidak dapat membuka halaman waiting. Workflow dikembalikan inactive. Tidak ada publish atau upload. Batch 5 masih terbuka.

## 2026-07-28 — Batch 5 audit dan structural guard

Status: BLOCKED — execution mencapai output akhir, tetapi artikel gagal quality gate dan renderer WordPress belum terverifikasi.

Execution `1021` berhasil melewati Outline, Planner, Writer, GEO Checker, dan output Google Docs. Namun hasil artikel mendapat `geoScore 38/100` dengan verdict `REJECT`. Root cause struktural: Writer menghasilkan prose tanpa heading section `##`, sehingga GEO Checker hanya melihat `Intro` dan `FAQ`; jalur Test juga memakai `Code: Convert Draft to HTML`, bukan renderer WordPress. Google Docs upload berhasil, tetapi itu bukan bukti publish atau verifikasi renderer WordPress.

Perbaikan diterapkan: prompt Writer sekarang mewajibkan satu heading `##` untuk setiap section Outline dan `## FAQ`; `Code: Parse Draft` menolak draft dengan kurang dari tiga heading section sebelum scoring. Validasi n8n: PASS, 0 error, 6 warning lama. Workflow dikembalikan inactive. Batch 5 masih terbuka sampai execution berikutnya menghasilkan struktur valid, skor lulus, dan renderer WordPress diverifikasi tanpa publish Live.

Percobaan ulang setelah guard dibuat (`1022` dan `1023`) hanya menghasilkan form-waiting URL; halaman waiting lokal timeout sebelum submit sehingga tidak ada execution baru. Workflow kembali inactive. Batch 5 tetap BLOCKED karena belum ada verifikasi E2E pasca-guard dan renderer WordPress belum dijalankan.

Percobaan verifikasi alternatif menambahkan webhook sementara `TEMP CP020 Test Webhook`. Endpoint mengembalikan HTTP 200, tetapi tidak menghasilkan execution baru setelah 20 detik. Node sementara dihapus dan workflow kembali inactive. Validasi akhir tetap PASS, 0 error, 6 warning lama; Batch 5 tetap BLOCKED hanya pada verifikasi runtime.
