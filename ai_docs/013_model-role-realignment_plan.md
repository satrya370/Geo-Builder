# CP 013 — Model/Role Realignment (verifikasi & pengerasan)

## Konteks
User menetapkan mapping model↔provider final untuk 8 role pipeline:

| Role | Model | Provider |
|---|---|---|
| Writer | openai/gpt-5.4-nano | KobiLLM |
| Reviser | deepseek-v4-flash-free | OpenCode Zen |
| Planner | deepseek-v4-flash-free | OpenCode Zen |
| Critic | openai/gpt-oss-120b | OpenRouter |
| Researcher | tencent/hy3-preview:online | OpenRouter |
| Outline | llama-3.3-70b-versatile | Groq (free) |
| Schema | qwen2.5:7b-instruct-q4_K_M | Ollama (local) |
| Safety Guard | glm-4.5-flash | Z.ai |

**Verifikasi per 2026-07-26** (fetch langsung `get_workflow_details` workflow `rwIbdIkIhoVE8nkG`): perubahan model-string + endpoint URL di 8 node Code/HTTP terkait **SUDAH diterapkan dan cocok 100% dengan tabel di atas**. Bagian ini TIDAK perlu dikerjakan ulang. Yang belum pasti adalah butir-butir di bawah — worker WAJIB mengerjakan ini, bukan mengulang swap model yang sudah selesai.

## Temuan risiko yang harus ditindaklanjuti

### 1. Verifikasi kredensial per node (HIGH)
4 node pindah provider dan sekarang butuh API key yang BERBEDA dari sebelumnya. `get_workflow_details` menampilkan `credentials: None` untuk SEMUA node httpRequest — ini dikenal sebagai artefak redaksi tool (bukan bukti kredensial kosong, lihat CP 011). Jangan percaya status ini secara statis. Verifikasi lewat `execute_workflow` + `get_execution` bahwa tiap node ini benar-benar sukses (bukan 401/403):

- `AI Safety Guard` → harus pakai key Z.ai (lihat `Credential_information.md` baris "GLM Z.ai Credential")
- `HTTP: AI Outline Planner` → harus pakai key Groq (lihat `Credential_information.md` baris "qroq credential")
- `HTTP: AI Writing Planner` → harus pakai key OpenCode Zen (lihat `Credential_information.md` baris "Opencode credential")
- `HTTP: AI Writer` → harus pakai key KobiLLM (lihat `Credential_information.md` baris "credential model koboillm")
- `HTTP: AI Reviser` → harus pakai key OpenCode Zen (sama seperti Writing Planner)
- `HTTP: AI GEO Critic`, `HTTP: AI Researcher` → tetap key OpenRouter (tidak berubah)
- `HTTP: AI Schema Generator` → tanpa auth (local Ollama), sudah benar apa adanya

Kalau ada node pakai kredensial LAMA (misal Writer masih pakai key OpenRouter lama, bukan KobiLLM) — request akan gagal auth. Kalau `setNodeCredential`/`addNode` dengan `credentials` inline ditolak MCP (limitation yang sudah pernah ditemukan di CP 008), lakukan via n8n UI langsung dan laporkan hasilnya di log.

### 2. Risiko `max_tokens` untuk `deepseek-v4-flash-free` @ OpenCode Zen (HIGH)
Test terminal (di luar n8n, endpoint `https://opencode.ai/zen/v1/chat/completions`, model `deepseek-v4-flash-free`) membuktikan model ini reasoning-heavy:
- Di `max_tokens: 5361` → SELURUH budget habis untuk reasoning, `content` kosong, `finish_reason: "length"` (article writer role, ~1300 kata target).
- Baru sukses di `max_tokens: 16000` (reasoning_tokens 2536/4188 completion).

Model ini dipakai untuk **Planner** (`Code: Build AI Writing Planner Body`, saat ini `max_tokens: 7000`) dan **Reviser** (`Code: Build AI Reviser Body`, formula `Math.ceil(Math.max(wc,300)*2.2) + 3000` — untuk draft pendek (wc≈300) ini cuma ≈3660). Keduanya berisiko mengulang bug "empty content karena reasoning menghabiskan budget" yang sudah pernah diperbaiki untuk role lain di CP 011.

**Rekomendasi**: naikkan `max_tokens` kedua node ini ke minimal 16000 flat (atau formula dengan buffer jauh lebih besar), sesuai instruksi user sebelumnya: prioritaskan keberhasilan/kualitas dibanding efisiensi token.

### 3. Full end-to-end live test (MANDATORY)
Eksekusi sukses terakhir (execution 936) terjadi SEBELUM 8 role ini pindah provider. Wajib `execute_workflow` ulang penuh sekarang, lalu untuk tiap node AI cek: `finish_reason`, apakah `content` tidak kosong, tidak ada error auth/network. Termasuk pastikan jalur REVISE (yang sampai sekarang belum pernah teruji nyata — lihat CP 008) tetap jalan dengan Reviser model baru.

### 4. (Low priority, cosmetic) `config/content-context.json`
`models._comment` masih bilang "Outline & schema & guard routed to local Ollama" — sudah tidak akurat (cuma Schema yang tetap Ollama; Outline→Groq, Guard→Z.ai). `models.roles.safetyGuard.tier` dan `models.roles.outline.tier` masih berlabel `"local"`. Perbarui supaya tidak menyesatkan pembaca berikutnya — tidak ada bukti field `tier` ini dipakai untuk branching runtime, jadi ini murni dokumentasi, bukan bug fungsional.

## Definition of done
- [ ] Semua 8 node dikonfirmasi memakai kredensial yang benar (bukti: `get_execution` sukses per node, bukan asumsi statis)
- [ ] `max_tokens` Planner & Reviser dinaikkan dan diverifikasi tidak lagi menghasilkan `content` kosong
- [ ] 1x eksekusi end-to-end penuh sukses dengan role mapping baru, hasil (execution ID, geoScore, verdict) dicatat di log
- [ ] `content-context.json` comment/tier diperbarui
- [ ] `ai_docs/index.md` ditambah baris CP 013
