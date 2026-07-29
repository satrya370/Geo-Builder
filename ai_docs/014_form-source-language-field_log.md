# Log CP 014 — Source Language Field

**Mengikuti:** `014_form-source-language-field_plan.md`
**Mode:** Worker

## Implementation

| # | Change | Status |
|---|--------|--------|
| 1 | Form Trigger: "Source Language" dropdown (Auto/International/Force Indonesia/Bilingual) | ✅ |
| 2 | Set Form Run Context: `sourceLanguage: dropAuto(f.source_language)` | ✅ |
| 3 | Build AI Research Body: source language instruction logic | ✅ |

## Empirical test results (hy3-preview:online, max_tokens=8000)

| Mode | Sources | Type |
|------|---------|------|
| **Auto** | ResearchGate, Neliti, Ubaya | Mixed (international + Indonesian academic) |
| **International** | JAMA Network, PubMed/NIH, Sage Journals | 100% international academic ✅ |
| **Force Indonesia** | Liputan6, al-haramjournal.id, lppm.itk.ac.id | 100% Indonesian ✅ |

**Kesimpulan:** Instruksi BAHASA SUMBER benar-benar mengubah sumber yang ditemukan model, bukan diabaikan. Default Auto mempertahankan perilaku existing (tidak ada perubahan).

## Status

**done** — 3 perubahan code + 3 live test empiris. Source Language field ready.