# Log CP 006 — AI Writing Planner (Node 22b–22d)

**Mengikuti:** `006_ai-writing-planner_plan.md`
**Mode:** Worker

## Evaluasi ulang draft sebelum publish (diminta user)

Draft di plan ditulis **sebelum** CP 004/005 selesai. Saat dievaluasi ulang terhadap kondisi
workflow yang sebenarnya lewat `get_workflow_details`, **3 asumsi draft ternyata salah** dan
kalau di-publish apa adanya akan menghasilkan `undefined` diam-diam (bukan error yang
kelihatan):

| # | Asumsi draft | Kenyataan di workflow | Dampak kalau tidak dikoreksi |
|---|---|---|---|
| 1 | Node 22 output `{outline: {...}}` (nested) | Output **spread di top-level** (`workingTitle`, `sections`, `faq`, ...) + `_outlineParsed` | `outlineData.outline` = `undefined` → prompt planner kosong |
| 2 | Node 22 membawa `facts` | Tidak — node 23 ambil sendiri dari `$('Code: Parse Research')` | `carried._facts` = `undefined` → planner tanpa fakta |
| 3 | Node 23 punya baris `const outlineData = $input.first().json;` | Tidak ada — kodenya `const outline = $input.first().json;` dengan struktur berbeda total | Instruksi patch di draft tidak bisa diterapkan |

**Koreksi desain yang diambil:** node 22d dibuat **meneruskan bentuk output node 22 apa adanya**
(`...outline`) lalu menambahkan `sectionBriefs`. Dengan begitu node 23 (`const outline =
$input.first().json;`) tetap jalan tanpa perubahan struktural — cuma perlu tambahan kecil
untuk membaca `outline.sectionBriefs`. Ini lebih aman daripada mengubah kontrak antar-node.

## Yang dibangun

| Node | Nama | Status |
|---|---|---|
| 22b | `Code: Build AI Writing Planner Body` | ✅ |
| 22c | `HTTP: AI Writing Planner` | ⚠️ terpasang, **credential belum tertaut** (lihat blocker) |
| 22d | `Code: Parse Planner Result` | ✅ |
| 23 | `Code: Build AI Writer Body` | ✅ diupdate — menyerap `sectionBriefs`, fallback ke outline mentah kalau `null` |

**Koneksi:** `Code: Parse Outline` → `22b` → `22c` → (main+error) `22d` → `Code: Build AI Writer
Body`. Koneksi lama `Parse Outline → Build AI Writer Body` sudah diputus. Node 23/24/25/26
digeser ke kanan (x +900) supaya alur canvas tetap terbaca kiri→kanan.

**Model:** `openai/gpt-oss-120b` via OpenRouter — endpoint/model direplikasi dari node
"AI Planner (MiMo)" di `ig Content Builder` sesuai instruksi user. Skema JSON brief dirancang
khusus GEO Builder, **tidak** mengikuti format ig-content-builder.

## Blocker: n8n MCP tidak bisa menautkan credential ke node `httpRequest`

Dicoba **3 cara**, semuanya ditolak dengan error yang sama
(`node type 'n8n-nodes-base.httpRequest' does not accept credential '<X>'`):

1. `setNodeCredential` dengan `credentialKey: "openAiApi"` → ditolak
2. `setNodeCredential` dengan `credentialKey: "httpHeaderAuth"` → ditolak
3. `credentials` inline di `addNode` (typeVersion 4 maupun 4.4) → ditolak

n8n sendiri mengonfirmasi di response: *"HTTP Request nodes were skipped during credential
auto-assignment. Their credentials must be configured manually."*

**Ini menjelaskan temuan lama yang sempat saya salah-diagnosis** di review CP 003/004: node
`HTTP: AI Researcher`, `HTTP: AI Writer`, dan `AI Safety Guard` tidak punya credential tertaut
**bukan** karena Worker lalai, tapi karena tool MCP-nya memang tidak mampu. Koreksi ini juga
berlaku untuk temuan #2/#4 di `004_researcher-outline-writer_report.md` — penyebabnya batasan
tool, bukan kelalaian.

**Konsekuensi:** pipeline **belum bisa dijalankan sama sekali** sampai credential 4 node HTTP
dipasang manual di UI n8n (atau oleh agent CLI lain yang punya akses lebih).

## Status

**in-progress** — node & koneksi selesai, logic terpasang. Menunggu credential dipasang
(diserahkan ke agent CLI lain per instruksi user).

---

## Re-run — Pasang Credential (Worker, sambungan CP 007 Task A)

**Mode:** Worker · **Verifikasi:** `get_workflow_details` langsung

### 4 credential terpasang & diverifikasi (kontra klaim "n8n MCP tidak bisa")

| Node | Credential | ID | Diverifikasi |
|------|-----------|-----|-------------|
| `AI Safety Guard` | Z.ai API (httpHeaderAuth) | `7XGXMnIGgDLfu6oV` | ✅ `credentials.httpHeaderAuth` muncul di node |
| `HTTP: AI Researcher` | OpenRouter Authorization - GPT OSS 120B | `oSM2yzrS4VtlyIcj` | ✅ |
| `HTTP: AI Writing Planner` | OpenRouter Authorization - GPT OSS 120B | `oSM2yzrS4VtlyIcj` | ✅ |
| `HTTP: AI Writer` | OpenRouter Authorization - GPT OSS 120B | `oSM2yzrS4VtlyIcj` | ✅ |

### Perubahan tambahan

- **AI Safety Guard:** pola credential diubah dari `predefinedCredentialType`+`openAiApi` (INVALID) ke `genericCredentialType`+`httpHeaderAuth` (BENAR per §3.5). Credential Z.ai baru (`7XGXMnIGgDLfu6oV`, `httpHeaderAuth` header `Authorization: Bearer ...`) dibuat menggantikan credential lama `openAiApi`.
- Klaim di log sebelumnya ($41–56) bahwa "n8n MCP tidak bisa menautkan credential" **sudah tidak berlaku** — MCP sekarang berhasil menautkan via `updateNode` dengan field `credentials`.

## Status (re-run)

**done** — 4 credential terpasang & terverifikasi via `get_workflow_details`. Pipeline siap jalan.
