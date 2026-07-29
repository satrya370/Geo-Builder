# CP 015 — Concept Brief Architecture: Log

> Mulai: 2026-07-27 | Target: replace Researcher-first flow dgn Concept-first flow
> Workflow: `rwIbdIkIhoVE8nkG` | Node count: 42 → 47 (+5 new)

## Ringkasan

Mengganti model "Researcher dulu, baru Outline/Writer" dengan "Concept Brief dulu, baru targeted research (kalau perlu), lalu Outline/Writer dengan kerangka concept brief". Ini menyelesaikan root cause dari 3 kegagalan CP 011:
1. **Kutipan Aristoteles palsu** — Writer kehabisan fakta → mengarang dari ingatan
2. **DSM-5/ADHD diselipkan** — tidak ada closed-world guard di Writer
3. **Label scaffolding bocor** — `*Kalimat quotable:*` muncul di badan artikel

## Batch 1: Pipeline Konsep (5 node baru)

| # | Node | Detail |
|---|------|--------|
| 1 | Form Trigger + Research Mode | Dropdown: Auto / Concept Only / Force Research |
| 2 | Set Form Run Context | +sourceLanguage (CP 014 fix) +researchMode |
| 3 | Code: Build Concept Brief Body | KobiLLM gpt-5.4-nano, max_tokens=4000 |
| 4 | HTTP: AI Concept Brief | Credential `b8p2bAdF3uhQB71A` |
| 5 | Code: Parse Concept Brief | Robust parse + researchMode override logic |
| 6 | If - Need Empirical Data? | Checks `needsResearch` from brief |
| 7 | Rewire If-Allowed → Concept Brief | Was: → Research Body; Now: → Concept Brief Body |

**Parse Concept Brief researchMode logic:**
```js
b.needsResearch = rm === 'Force Research' || (rm !== 'Concept Only' && b.needsResearch === true && empiricalClaims.length > 0);
b.isConceptOnly = rm === 'Concept Only';
```

## Batch 2: Targeted Research + Normalize

| # | Item | Detail |
|---|------|--------|
| 1 | Code: Normalize Research Payload | Unifies both branches: `{facts, commonQuestions, entities, conceptBrief, researchRan}` |
| 2 | Wiring: If-Empirical → Research / Normalize → Outline | Both paths converge at Normalize |
| 3 | Validate Research Facts → mode-aware | Skip numeric threshold if `researchRan===false` |
| 4 | Build AI Outline Body → reads conceptBrief | Injects sectionSeeds + angle into prompt |
| 5 | Build AI Writing Planner → reads conceptBrief | Injects thinkers/keyConcepts/debates |
| 6 | Build AI Research Body → targeted | Uses empiricalClaims[].searchQuery instead of generic 8-12 facts |

## Batch 3: Writer Anti-Fabrikasi + Rubrik Rebalance

### Writer rules (§9)

| Rule | Change |
|------|--------|
| 7 | Enhanced: NO vague attribution. WAJIB "Studi oleh [Nama] menemukan..." | 
| 8 | Fixed: scaffolding ban — `*Kalimat quotable:*` label BANNED from article body |
| 9 | Redirected: stance → conceptBrief.thinkers/keyConcepts (not facts) |
| 10 (new) | Quote vs paraphrase: `"..."` ONLY for verbatim facts. Paraphrase ideas → no quotes |
| 11 (new) | Closed-world names: only mention studies/orgs/figures from FACTS or conceptBrief |
| 16 | claimsSourceMap + contradictoryClaims HTML comment blocks for audit |

### Rubrik rebalance (§10) — verified across ALL 3 sync nodes

| Node | Change | Status |
|------|--------|--------|
| Rule Checker (node-26) | All scores ×0.125 (total rule: 40→5) | ✅ |
| Critic (node-27) | Added dim G (CONCEPT_COHERENCE): checks sectionSeeds coverage, angle, thinkers | ✅ |
| Parse Critique (node-29) | ai_concept = (conceptCoherence/5)×25, passthrough to aiScores | ✅ |
| Compute Score (node-30) | New GATE_WEIGHTS: conceptCoherence:25, rebalanced, NORM=100/80 | ✅ |

## Bug Fix Kritis (pre-Batch 3)

### Bug 1: If - Topic Allowed? wiring rusak
- **Before:** both `Respond - Topic Blocked` AND `Code: Build Concept Brief Body` on port 0; port 1 empty
- **After:** port 0 = Respond only (blocked), port 1 = Concept Brief Body (allowed)
- **Verification:** Structure diff confirmed. E2E test needs n8n UI ("Execute workflow" button)

### Bug 2: Writer Body stale reference
- **Before:** `$('Code: Parse Research')` → crash in concept-only mode
- **After:** `$('Code: Normalize Research Payload')` → works in all modes
- **Impact:** Parse Research hanya jalan di cabang research. Normalize selalu jalan.

## Final State

- **Nodes:** 47 (42 original + 5 new)
- **Connections:** 43
- **Workflow:** inactive (deactivated after verification)

## Definition of Done

| Item | Status |
|------|--------|
| 5 new nodes built + wired | ✅ |
| 3-mode research control (Auto/Concept Only/Force Research) | ✅ |
| Writer anti-fabrikasi 4 rules | ✅ |
| Rubrik rebalance 3 node sync | ✅ |
| If-Allowed wiring fixed | ✅ |
| Writer stale reference fixed | ✅ |
| E2E test via MCP | ⚠️ blocked — needs n8n UI "Execute" click |
| E2E test 3 modes | ⚠️ user must verify via n8n editor form |

## Catatan untuk Pengujian Mandiri

1. Buka workflow di n8n editor → klik "Execute workflow"
2. Submit form via URL yang muncul (`/form-test/<uuid>` atau `/webhook-test/<uuid>`)
3. Test 3 mode:
   - **Concept Only:** topik filsafat abstrak (akrasia, determinisme) → seharusnya TANPA AI Researcher
   - **Auto:** topik netral → pipeline memutuskan sendiri
   - **Force Research:** topik apa pun → AI Researcher SELALU jalan
4. Verifikasi anti-fabrikasi: cek draft untuk kutipan bertanda kutip yang diatribusikan ke tokoh, nama studi di luar brief, label "*Kalimat quotable:*"
5. Pastikan geoScore artikel concept-only mendarat di zona PASS (>=80) dengan rubrik baru
