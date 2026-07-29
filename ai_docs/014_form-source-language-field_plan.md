# CP 014 — Field "Source Language" di Form Trigger

## Latar belakang
Dari serangkaian test terminal (di luar n8n) membandingkan model Researcher, ditemukan pola konsisten: ketika prompt riset ditulis dalam Bahasa Indonesia, hampir semua model `:online` (deepseek, gpt-5-nano, mimo-v2.5, gpt-5.4-nano, hy3 non-preview) jatuh ke cluster sumber yang sama — blog pop-media Indonesia berkualitas rendah (RRI.co.id, Kompasiana, Liputan6, BCA Karir) dengan sedikit fakta bernomor. Hanya `tencent/hy3-preview:online` (model Researcher saat ini) yang konsisten mendapat sumber akademik internasional (Stanford, PMC, Nature, BMC).

Untuk niche filsafat/psikologi ini, sumber internasional/akademik seringkali lebih kredibel dan lebih kaya data untuk sebagian topik, TAPI untuk topik yang secara inheren lokal (mis. isu budaya/masyarakat Indonesia spesifik) sumber Indonesia justru lebih relevan/wajib. Solusinya: beri kontrol eksplisit ke user di Form Trigger, bukan bergantung pada perilaku implisit model.

## Tujuan
Tambah 1 field dropdown baru di `Form Trigger - Manual Run`: **"Source Language"** dengan 4 opsi:
- **Auto** (default) — tidak ada instruksi bahasa sumber tambahan; biarkan Researcher model memilih sumber terbaik apa adanya (perilaku saat ini, tidak berubah).
- **International** — paksa instruksi eksplisit ke Researcher: WAJIB memprioritaskan sumber berbahasa Inggris/internasional (jurnal akademik, situs institusi `.edu`/`.gov`, publikasi riset), hindari blog SEO/media populer.
- **Force Indonesia** — paksa instruksi eksplisit: WAJIB mencari dan memakai sumber berbahasa Indonesia (media, institusi, riset lokal).
- **Bilingual** — paksa instruksi eksplisit: WAJIB mengumpulkan campuran sumber internasional/akademik DAN sumber Indonesia (targetkan proporsi seimbang, mis. minimal 2-3 fakta dari masing-masing kelompok), supaya artikel dapat kredibilitas akademik sekaligus relevansi lokal.

## Perubahan yang dibutuhkan

### 1. `Form Trigger - Manual Run` (formTrigger node)
Tambah field baru di `formFields.values`, ditempatkan setelah "Target Keyword" (sebelum "Category Hint", supaya mengelompok dengan field-field yang memengaruhi arah riset):
```json
{
  "fieldLabel": "Source Language",
  "fieldType": "dropdown",
  "requiredField": false,
  "fieldName": "source_language",
  "fieldOptions": {
    "values": [
      { "option": "Auto", "value": "Auto" },
      { "option": "International", "value": "International" },
      { "option": "Force Indonesia", "value": "Force Indonesia" },
      { "option": "Bilingual", "value": "Bilingual" }
    ]
  }
}
```

### 2. `Code: Set Form Run Context`
Tambah 1 baris memakai helper `dropAuto` yang sudah ada (pola sama seperti `categoryHint`/`wordCountTarget` — value "Auto" di-drop jadi `null`):
```js
sourceLanguage: dropAuto(f.source_language),
```
Tidak perlu ubah `Code: Build Run Context` — field ini otomatis ikut lewat spread `...a`.

### 3. `Code: Build AI Research Body`
Baca `d.sourceLanguage` dan sisipkan instruksi tambahan ke prompt SEBELUM baris "Gunakan web search. Aturan wajib:". Logika:
```js
const sourceLanguage = d.sourceLanguage || null;
let sourceLanguageInstruction = '';
if (sourceLanguage === 'International') {
  sourceLanguageInstruction = '\n\nINSTRUKSI BAHASA SUMBER: WAJIB memprioritaskan sumber berbahasa Inggris/internasional (jurnal akademik, situs institusi .edu/.gov, publikasi riset seperti PMC/Nature/BMC/APA). JANGAN memakai blog SEO atau media populer sebagai sumber utama kecuali tidak ada alternatif akademik.';
} else if (sourceLanguage === 'Force Indonesia') {
  sourceLanguageInstruction = '\n\nINSTRUKSI BAHASA SUMBER: WAJIB mencari dan memakai sumber berbahasa Indonesia (media, institusi, riset lokal Indonesia).';
} else if (sourceLanguage === 'Bilingual') {
  sourceLanguageInstruction = '\n\nINSTRUKSI BAHASA SUMBER: WAJIB mengumpulkan CAMPURAN sumber internasional/akademik (jurnal, situs institusi .edu/.gov) DAN sumber berbahasa Indonesia (media, institusi lokal) — targetkan proporsi seimbang, minimal 2-3 fakta dari masing-masing kelompok bahasa.';
}
```
Sisipkan `${sourceLanguageInstruction}` ke template `prompt` tepat setelah baris `TANGGAL HARI INI: ${today}`.

Tidak perlu ubah `Code: Parse Research`, tidak ada perubahan skema output JSON — field ini murni memengaruhi instruksi prompt, bukan struktur data.

## Yang HARUS diverifikasi setelah build (jangan asumsi)
1. **Test empiris "International"**: jalankan Researcher dengan instruksi ini pada topik yang sama yang dipakai di test-test sebelumnya (prokrastinasi/Piers Steel/Aristoteles), bandingkan sumber yang dihasilkan vs baseline "Auto" — pastikan instruksi baru benar-benar mengubah perilaku pencarian model (hy3-preview:online belum pernah dites dengan instruksi eksplisit semacam ini; ada kemungkinan instruksi tidak berpengaruh kalau model tetap bias ke bahasa prompt).
2. **Test empiris "Force Indonesia"**: pastikan tidak menurunkan kualitas fakta bernomor terlalu jauh dibanding Auto.
3. **Test empiris "Bilingual"**: pastikan hasil benar-benar campuran (bukan tetap 100% satu bahasa) dan proporsi numerik-fact tetap lolos `minNumericFactsRequired`.
4. Pastikan dropdown baru tidak merusak form yang sudah ada (test submit form kosong/default = Auto, harus identik dengan perilaku sebelum perubahan ini).
5. Update `ai_docs/index.md` (baris CP 014) dan tulis log build + hasil ke-3 test di atas ke `ai_docs/014_form-source-language-field_log.md`.

## Definition of done
- [ ] Field "Source Language" muncul di form dengan 4 opsi, default kosong = Auto
- [ ] `Code: Set Form Run Context` meneruskan `sourceLanguage` (null kalau Auto)
- [ ] `Code: Build AI Research Body` menyisipkan instruksi sesuai pilihan (termasuk Bilingual)
- [ ] Minimal 1 eksekusi live test untuk "International", 1 untuk "Force Indonesia", dan 1 untuk "Bilingual", dicatat hasil sumber & jumlah fakta bernomor di log
- [ ] `ai_docs/index.md` diperbarui
