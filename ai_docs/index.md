# Index Checkpoint — GEO Article Builder

> Lihat `rules.md` §5 untuk aturan pengisian. Overflow >500 baris → lanjut di `index-2.md`.
> Setiap mode (Planner/Worker/Reviewer) wajib update baris relevan setelah selesai kerja.

| ID | Subject | Status | Risk | Confidence | Last mode | Link plan | Link log |
|---|---|---|---|---|---|---|---|
| 001 | verifikasi-medium-token | done | low | high | worker | `001_verifikasi-medium-token_plan.md` | `001_verifikasi-medium-token_log.md` |
| 002 | content-context-dan-sheets-setup | done | low | medium | worker | `002_content-context-dan-sheets-setup_plan.md` | `002_content-context-dan-sheets-setup_log.md` |
| 003 | form-trigger-safety-guard | done (diverifikasi lewat tes end-to-end nyata, lihat 011) | medium | medium | reviewer | `003_form-trigger-safety-guard_plan.md` | `003_form-trigger-safety-guard_log.md`, `003_form-trigger-safety-guard_report.md` |
| 004 | researcher-outline-writer | done (diverifikasi lewat tes end-to-end nyata, lihat 011) | medium-high | medium | reviewer | `004_researcher-outline-writer_plan.md` | `004_researcher-outline-writer_log.md`, `004_researcher-outline-writer_report.md` |
| 005 | geo-rule-checker | done (diverifikasi lewat tes end-to-end nyata, lihat 011) | low | high | reviewer | `005_geo-rule-checker_plan.md` | `005_geo-rule-checker_log.md` |
| 006 | ai-writing-planner | done (diverifikasi lewat tes end-to-end nyata, lihat 011) | low-medium | high | reviewer | `006_ai-writing-planner_plan.md` | `006_ai-writing-planner_log.md` |
| 007 | critic-compute-score-gate | done (diverifikasi lewat tes end-to-end nyata, lihat 011) | medium-high | medium | reviewer | `007_critic-compute-score-gate_plan.md` | `007_critic-compute-score-gate_log.md` |
| 008 | revise-loop | in-progress (build selesai, TAPI jalur REVISE belum teruji nyata — draft tes langsung PASS, lihat 011) | medium-high | medium | reviewer | `008_revise-loop_plan.md` | `008_revise-loop_log.md` |
| 009 | schema-dan-wordpress-publish | done (diverifikasi lewat tes end-to-end nyata — WordPress draft berhasil dibuat, lihat 011) | medium-high | medium | reviewer | `009_schema-dan-wordpress-publish_plan.md` | `009_schema-dan-wordpress-publish_log.md` |
| 010 | cp005-cp007-review | done | low | high | reviewer | — (revision, tidak ada plan terpisah) | `010_cp005-cp007-review_revision.md` |
| 011 | e2e-live-test-fixes | done | high | high | reviewer | — (revision, tidak ada plan terpisah) | `011_e2e-live-test-fixes_revision.md` |
| 012 | edge-case-audit | done | high | high | reviewer | — (audit, tidak ada plan) | `012_edge-case-audit_report.md` |
| 013 | model-role-realignment | done | low | high | worker | `013_model-role-realignment_plan.md` | `013_model-role-realignment_log.md` |
| 014 | form-source-language-field | revised (disederhanakan di CP017 — dari "kontrol sumber riset" jadi "kontrol bahasa artikel", scope riset tetap bahasa-agnostik) | low-medium | high | worker | `014_form-source-language-field_plan.md` | `014_form-source-language-field_log.md` |
| 015 | concept-brief-architecture | done (3 bug kritis ditemukan & diperbaiki reviewer setelah klaim "done" pertama — lihat log) | medium | high | reviewer | `015_concept-brief-architecture_plan.md` | `015_concept-brief-architecture_log.md` |
| 016 | citation-quote-verification | done (build selesai; E2E butuh "Execute" di n8n UI + verifikasi $env) | low-medium | high | worker | `016_citation-quote-verification_plan.md` | `016_citation-quote-verification_log.md` |
| 017 | run-mode-and-export | in-progress (Batch 1 inspection complete; resume at Run Mode routing) | high | high | worker | `017_run-mode-and-export_plan.md` | `017_run-mode-and-export_log.md` |
| 018 | google-docs-export | in-progress (Google Doc berhasil dibuat, tetapi response upload masih stream/buffer; perlu normalisasi JSON + verifikasi format) | medium | medium | worker | `018_google-docs-export_plan.md` | `018_google-docs-export_log.md` |
| 019 | geo-authority-freshness-revision | blocked-partial (§1 TERJAWAB di CP020: WordPress.com terbukti menghapus tag `<script>` — schema JSON-LD tumpah jadi teks. Sisa item authority/freshness ditunda; user minta fokus kualitas artikel dulu) | high | high | reviewer | `019_geo-authority-freshness-revision_plan.md` | `019_geo-authority-freshness-revision_log.md` |
| 020 | article-quality-overhaul | planned (6 kritik user terverifikasi dari post live #9 → 2 root cause: markdown tidak dikonversi ke HTML, & pipeline dirancang sebagai answer-engine bukan article-writer) | high | high | reviewer | `020_article-quality-overhaul_plan.md` | — (belum ada log) |
| 021 | satryapudja-api-integration | in-progress (Step 1-5 selesai; Step 6 wiring terpasang; BLOCKED di Step 7 karena error path existing tidak ditemukan dan uji API belum dilakukan) | high | medium | worker | `021_satryapudja-api-integration_plan.md` | `021_satryapudja-api-integration_log.md` |
