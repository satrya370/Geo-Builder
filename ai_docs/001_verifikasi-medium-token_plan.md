# CP 001 — Verifikasi Medium API Token

**Mode:** Planner · **Asal:** `ai_docs/todos.md` item 1
**Dokumen rujukan wajib dibaca sebelum eksekusi:** `docs/IMPLEMENTATION_PLAN.md` §1.1,
`config/content-context.json` (blok `medium`)

## Ringkasan CP (gambaran besar)

- **Confidence:** high — langkahnya sudah jelas, tinggal cek 1 fakta lalu tentukan cabang.
- **Risk:** low — tidak ada yang dieksekusi ke n8n/produksi di CP ini, murni verifikasi +
  update config. Kalau salah, gampang dikoreksi (revert isi JSON).
- **Token/limit usage estimate:** low — tidak ada AI call besar, tidak ada Execute Command
  berat, hanya baca file + tulis balik config kecil.
- **Kesimpulan:** CP ini **tidak perlu** dipecah jadi sub-CP A/B (risk & confidence tidak
  memenuhi syarat pemecahan di `rules.md` §2).

## Kenapa CP ini duluan

`IMPLEMENTATION_PLAN.md` §1.1 menandai token Medium sebagai **blocker potensial** yang belum
terverifikasi, dan secara eksplisit mewanti-wanti: *"Jangan buang waktu debug node Medium
sebelum ini dicek."* Fase 6 (node 43–45, Medium publish) tidak boleh mulai dibangun sebelum
status ini pasti — makanya ini backlog pertama yang diangkat, sebelum node apa pun disentuh.

## Langkah

| # | Step | Confidence | Risk | Token/limit |
|---|---|---|---|---|
| 1 | Minta user cek **Medium → Settings → Security and apps → Integration tokens** — apakah opsi generate token tersedia/ada token aktif | high | low | low (tidak ada eksekusi AI, murni tanya-jawab) |
| 2a | **Kalau token ADA:** minta user generate/salin token, catat scope-nya. Set `medium.enabled: true` di `config/content-context.json`, isi `authorId` via `GET https://api.medium.com/v1/me` (butuh 1 HTTP call test, bisa manual via curl/Postman dulu — belum perlu node n8n) | medium | low | low |
| 2b | **Kalau token TIDAK ADA:** set `medium.enabled: false` (sudah default). Tambahkan catatan eksplisit di `IMPLEMENTATION_PLAN.md` §1.1 bahwa keputusan final: WordPress-only, Fase 6 (node 43–45) di-skip permanen kecuali kebijakan Medium berubah nanti | high | low | low |
| 3 | Update `ai_docs/index.md` (baris CP 001) dan `ai_docs/todos.md` (coret item asal, `-> 001`) | high | low | low |

## Keterbatasan yang harus disadari

Step 1 dan 2 (bagian generate token) **bergantung pada aksi manual user di akun Medium
mereka** — Worker/Planner tidak bisa melakukan ini sendiri (tidak ada akses browser/akun
Medium). CP ini akan **stop dan menunggu jawaban user** di step 1 sebelum bisa lanjut ke 2a/2b.
Ini bukan "kena max token", tapi dependency eksternal — dicatat di log sebagai
`blocked: waiting-on-user`, bukan `stopped: token-limit`.

## Definition of done

- `config/content-context.json` blok `medium` mencerminkan keputusan final (enabled true+authorId terisi, ATAU tetap false).
- `IMPLEMENTATION_PLAN.md` §1.1 diperbarui dari "belum terverifikasi" jadi keputusan final + tanggal verifikasi.
- Baris CP 001 di `index.md` berstatus `done`.
- Item asal di `todos.md` tercoret dengan `-> 001`.

## Next mode

Worker mengeksekusi CP ini (§3 rules.md: brief singkat dulu sebelum step 1, lalu tunggu
jawaban user di step 1 sebelum lanjut 2a/2b).
