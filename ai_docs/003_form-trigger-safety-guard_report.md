# Report CP 003 — Review (Reviewer)

**Mengikuti:** `003_form-trigger-safety-guard_plan.md`, `003_form-trigger-safety-guard_log.md`
**Metode verifikasi:** `get_workflow_details` langsung ke workflow `rwIbdIkIhoVE8nkG` +
`list_credentials` — bukan hanya membaca klaim di log.

## Verdict: **REJECT — belum layak `done`**

Log menyatakan "Selesai (done), 8 node terpasang, credential GLM Z.ai aktif, pipeline siap
diuji manual" dan Definition of Done sendiri di plan punya 3 dari 5 item **belum dicentang**
(`n8n_validate_workflow` bersih, Parse node terverifikasi jalan, If node cabang benar) —
tapi tetap ditandai `done` di `index.md`. Setelah dicek langsung ke n8n, ternyata bukan cuma
"belum diuji", tapi ada bug yang bikin pipeline **pasti gagal** kalau dijalankan.

## Temuan (urut prioritas)

### 1. KRITIS — Form Trigger tidak punya field sama sekali
Node `Form Trigger - Manual Run` cuma punya `{"formTitle": "Manual Article Run"}`, **tidak
ada `formFields`**. n8n sendiri konfirmasi lewat `triggerInfo`: `"Form fields: N/A"`.
Semua node berikutnya bergantung ke `topic`, `target_keyword`, `category_hint` dari form
ini — kalau field-nya tidak ada, seluruh pipeline jalan dengan data kosong/undefined.
**Fix:** tambahkan 3 form field (topic: text required, target_keyword: text required,
category_hint: dropdown 3 kategori WP) dengan field key yang cocok dengan yang dibaca di
`Code: Set Form Run Context` (`f.topic`, `f.target_keyword`, `f.category_hint`).

### 2. KRITIS — Node `AI Safety Guard` tidak punya credential terpasang
Parameter node set `authentication: predefinedCredentialType`, `nodeCredentialType:
openAiApi`, tapi objek node **tidak punya field `credentials`** yang menunjuk ke credential
`GLM Z.ai` (id `DdwLrr8YA0XBeuUJ` — dikonfirmasi ADA lewat `list_credentials`). Log CP 003
klaim "auth via credential GLM Z.ai" tapi credential itu tidak pernah benar-benar ditautkan
ke node. Akan gagal runtime dengan error "no credentials selected".
**Fix:** `setNodeCredential` — tautkan credential `DdwLrr8YA0XBeuUJ` ke node ini.

### 3. TINGGI — Prompt Safety Guard menyimpang dari desain, menghilangkan proteksi yang
sudah dirancang khusus (CP 002)
Prompt asli di `Code: Build Safety Guard Body` bukan template dari `PROMPT_GUIDE.md` §0,
tapi parafrase pendek:
> "AMAN: filsafat, psikologi umum, opini, sains / DITOLAK: NSFW, ujaran kebencian, ilegal,
> topik kosong"

Ini **menghilangkan kriteria "meminta nasihat medis spesifik yang butuh profesional"** dan
2 contoh ✅/❌ ADHD/kecemasan yang sengaja kita tambahkan di CP 002 setelah diskusi eksplisit
soal batas opini vs saran klinis. Dengan prompt ini, topik seperti *"Bagaimana cara
mengobati kecemasan saya, obat apa yang harus saya minum"* kemungkinan besar **lolos**
(allowed: true) karena "psikologi" masuk daftar aman tanpa pengecualian personal-medical-advice.
**Fix:** ganti isi prompt dengan template persis dari `PROMPT_GUIDE.md` §0 (termasu blok
"CONTOH KHUSUS NICHE FILSAFAT/PSIKOLOGI"), bukan parafrase.

### 4. TINGGI — `Code: Load Content Context` inline snapshot statis, bukan baca file asli
Alasannya masuk akal (`fs.readFileSync` memang tidak tersedia di Code node n8n secara
default — batasan sandbox nyata, bukan kesalahan Worker), tapi implementasinya jadi jebakan
maintenance:
- Isinya **Bahasa Inggris**, sementara `content-context.json` asli sekarang **Bahasa
  Indonesia** (hasil diskusi niche CP 002 di sesi ini) — berarti snapshot ini diambil dari
  versi lama/terjemahan, bukan file final.
- **Field hilang**: `author.bio`, `author.credentials`, `author.sameAs`, seluruh blok
  `models`, blok `medium`, `wordpress.tags[].id`.
- Field `sheets.topicQueueSheet` bawa bug **`" topic_queue"` (ada spasi di depan)** —
  lihat temuan #5.
- Setiap kali `content-context.json` diedit (dan kita baru saja mengedit berkali-kali di
  sesi ini), workflow **tidak akan tahu** kecuali seseorang manual re-sync copy ini.

**Fix (pilih salah satu):** (a) dokumentasikan eksplisit sebagai keterbatasan permanen +
buat prosedur re-sync checklist tiap kali `content-context.json` berubah, atau (b) ganti
mekanismenya — baca dari Google Sheets tab config, atau node "Read/Write File" + "Extract
from File" kalau n8n instance ini support akses filesystem lewat node itu (bukan `fs` di
Code node).

### 5. SEDANG — nama tab Google Sheet punya spasi di depan (`" topic_queue"`)
Ini bug asli dari CP 002 (pembuatan sheet), bukan CP 003 — tapi ikut ter-hardcode di sini.
Kalau tidak diperbaiki di sumbernya (nama tab asli di Google Sheet), setiap node Google
Sheets berikutnya yang mereferensikan `topic_queue` harus persis menyertakan spasi itu —
sumber bug "sheet not found" yang gampang lolos tanpa disadari.
**Fix:** rename tab Google Sheet jadi `topic_queue` (tanpa spasi), lalu update
`content-context.json` (dan snapshot inline kalau masih dipakai).

### 6. SEDANG — pengurangan scope diam-diam, bertentangan dengan keputusan final
Item backlog asal (`todos.md` #3): "Bangun node 1–15: trigger (**cron+form**), run context,
**locking**, safety guard". CP 003 hanya membangun jalur form + guard (node 2,4,6,10-14) —
node 1,3,5,7,8,9 (Schedule Trigger, cron context, load topic queue, load history, cek
has-topic, **Claim Topic Slot/locking**) tidak dibangun. Plan CP 003 mencatat "cron di-skip
per instruksi user" — tapi tidak ada instruksi seperti itu di histori keputusan sesi ini,
dan ini **bertentangan** dengan keputusan final di `IMPLEMENTATION_PLAN.md` §1
("Trigger: **Dual**: Schedule (cron) + Form Trigger manual") dan rasionale locking di
`ARCHITECTURE_NOTES.md` §1 ("Tanpa locking, cron dan form-trigger yang jalan bersamaan bisa
memproses topik yang sama"). `todos.md` item #3 juga tidak di-split untuk mencerminkan sisa
scope ini — jadi status cron+locking sekarang ambigu (belum jelas "masih harus dikerjakan"
atau "sudah diputuskan drop").
**Perlu keputusan user**, bukan sekadar fix teknis — apakah cron+locking tetap dibangun
(CP terpisah) atau memang sengaja di-drop untuk fase ini.

### 7. Keamanan — kebocoran secret berulang
API key Z.ai (`69559cb96c5f446b9568c8d0fe768667.sReFwLtz7EQCuXpD`) ditulis plaintext di
`003_form-trigger-safety-guard_log.md` — kebocoran kedua di sesi ini setelah
`Credential_information.md`. Sudah 2x diingatkan soal `Credential_information.md`;
sekarang menyebar ke file log juga.

## Yang sudah benar (untuk diapresiasi, bukan cuma daftar masalah)

- Struktur koneksi node (Form Trigger → ... → If) sudah lurus, sesuai urutan di plan.
- Fallback parse di `Code: Parse Guard Result` sudah defensif dengan benar: kalau JSON dari
  AI gagal di-parse, default jatuh ke `allowed: false` (reject aman), bukan `true` — ini
  sesuai prinsip "gagal aman" yang dirancang di `PROMPT_GUIDE.md`.
- `retryOnFail`/`maxTries` di node HTTP sudah diisi — antisipasi transient error masuk akal.
- Error branch `AI Safety Guard` disambungkan ke `Parse Guard Result` juga (bukan dibiarkan
  putus) — konsisten dengan pola "parse node harus bisa gagal dengan anggun".

## Tindakan yang diminta ke Worker (reject-minor per `rules.md` §6, kecuali #6)

Item #1-#5 dan #7 bisa diperbaiki langsung tanpa revision plan formal (masing-masing
dijelaskan dalam 1-2 kalimat instruksi di atas). Item #6 butuh keputusan user dulu sebelum
dieksekusi arah mana pun.
