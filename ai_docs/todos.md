# Todos — Backlog GEO Article Builder

> Backlog mentah, lihat `rules.md` §7. Item dicoret `[x]` + `-> NNN` saat Planner
> mengangkatnya jadi CP (`ai_docs/NNN_subject_plan.md`). Tidak dihapus. Overflow >500
> baris → lanjut di `todos-2.md`.

Diseed dari `docs/IMPLEMENTATION_PLAN.md` §6 (urutan implementasi, tahap 0–10).

- [x] Verifikasi Medium API token tersedia atau tidak (§1.1 IMPLEMENTATION_PLAN.md) — blocker untuk Fase 6 -> 001
- [x] Buat `config/content-context.json` terisi data real (bukan placeholder) + sheet `topic_queue`/`article_log` terisi ≥3 topik uji -> 002
- [x] Bangun node 1–15: trigger (cron+form), run context, locking, safety guard -> 003 (partial: node 2,4,6,10-14 jalur form + safety guard GLM Z.ai. Sisanya cron trigger, locking, node 15 lanjut CP lain)
- [x] Bangun node 16–25: Researcher → Outline → Writer, hasilkan draft 1500 kata dengan fakta bersumber -> 004
- [x] Bangun node 26 (`GEO Rule Checker`) saja, validasi skor rule-based di 3 draft uji sebelum lanjut ke Critic -> 005
- [x] Tambah ROLE 2.5 AI Writing Planner (node 22b-22d, antara Outline & Writer) — kompensasi model Writer yang tier ringan (`gpt-5-nano`), OpenRouter `openai/gpt-oss-120b` (pola dari "AI Planner (MiMo)" `ig Content Builder`). Lihat `IMPLEMENTATION_PLAN.md` §3 Fase 2 & `PROMPT_GUIDE.md` §2.5 -> 006
- [x] Bangun node 27–31: Critic + Compute GEO Score, validasi skor gabungan stabil antar run -> 007
- [x] Bangun node 32–36: revise loop, pastikan berhenti tepat di 2x dan `reviseCount` naik benar -> 008
- [x] Bangun node 37–39: Schema/Metadata (model lokal), validasi JSON-LD + fallback saat model keliru format. **Termasuk node 39b baru** (`Code: Validate Metadata Rules` — cek Kategori 6 rubric yang di-exclude dari node 26/30, keputusan CP 005, lihat `GEO_SCORING_RUBRIC.md` §"Kategori 6 — timing khusus") -> 009 (digabung dengan WordPress publish atas instruksi user)
- [x] Bangun node 40–42: WordPress publish, validasi artikel live + kategori benar + schema tertanam -> 009
- [x] ~~Bangun node 43–45: Medium publish~~ — **DIHAPUS dari scope**, keputusan final di CP 001 (Medium token tidak tersedia) -> 001
- [ ] Bangun node 46–54: logging Sheets, notifikasi Telegram, error path — uji error path dengan sengaja digagalkan
- [ ] (Opsional, prioritas kedua — setelah semua di atas selesai) Refactor node 26–31 jadi sub-workflow `GEO Scoring Core`, lihat `docs/OPTIONAL_SCORING_WORKFLOW.md` R1–R3
- [ ] (Operasional berulang, bukan one-time build — jalankan setelah ≥10 artikel published) Review kolom `score_breakdown` di sheet `article_log`, kalibrasi ulang threshold/prompt sesuai `docs/GEO_SCORING_RUBRIC.md` §Kalibrasi awal (naikkan threshold kalau hampir semua langsung PASS, perbaiki prompt role penyebab kalau satu kategori selalu rendah — bukan menurunkan threshold)
