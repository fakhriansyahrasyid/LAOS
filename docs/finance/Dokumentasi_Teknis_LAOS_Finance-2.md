# Dokumentasi Teknis — LAOS Finance (n8n Workflow)

Dokumen ini untuk keperluan maintenance dan debugging workflow n8n LAOS Finance. Ditulis untuk pembaca yang sudah familiar dengan n8n, jadi fokus ke struktur, keputusan desain, dan jebakan (gotcha) yang sudah ditemukan selama development — bukan tutorial dasar n8n.

**Status terakhir:** seluruh 163 node sudah diaudit menyeluruh (lihat Bagian 13); semua bug yang ditemukan sudah diperbaiki per Agustus 2026.

---

## 1. Arsitektur Umum

```
Telegram Trigger (updates: message, callback_query)
        │
        ▼
   If Akses (whitelist — cek message.from.id ATAU callback_query.from.id)
        │
        ▼
 Cek Callback Query (pisah klik tombol vs pesan biasa)
    ├─ TRUE  → Parse Callback Data → routing approval (WF-007/008)
    └─ FALSE → Switch1 (routing berdasar regex/startsWith kata kunci pesan)
                 ├─ /dashboard                       → Notion sync (Budget vs Realisasi)
                 ├─ /pl                               → Notion sync (P&L Bulanan)
                 ├─ /tutupbulan                       → WF-008
                 ├─ /aging                            → Notion sync (Aging Piutang & Hutang)
                 ├─ /help                             → Pesan Help
                 ├─ foto (photo[3] exists)            → WF-001/002 (foto struk)
                 ├─ invoice|tagih                     → WF-003
                 ├─ purchase|beli kredit               → WF-004
                 ├─ terima bayar|lunas                 → WF-005 (piutang)
                 ├─ bayar hutang                      → WF-005 (hutang)
                 ├─ jurnal|koreksi                    → WF-006
                 └─ (catch-all, notRegex semua di atas) → WF-001/002 (teks manual)
```

**Prinsip desain kunci:** semua panggilan ke ERPNext pakai node **HTTP Request** manual, BUKAN node native `n8n-nodes-base.erpNext`. Node native gagal (`TypeError: Cannot read properties of undefined (reading 'length')`) saat menangani child table bersarang (`accounts[]`, `items[]`). Ini berlaku di seluruh workflow.

**Base URL ERPNext:** `http://erp.literocoffee.my.id/api/resource/{Doctype}` (spasi pada nama doctype dua kata ditulis apa adanya, di-encode otomatis jadi `%20`).

**Kredensial yang dipakai:**
- `ERPNext API` (Predefined Credential Type) — auth utama ke semua endpoint ERPNext
- `Telegram account` — semua node Telegram
- `Google Gemini(PaLM) Api account` — semua LangChain Agent parser + node HTTP Request langsung ke Gemini (analisa gambar struk)
- `Notion account` — sync ketiga dashboard

---

## 2. Komponen Inti / Reusable

### If Akses (Whitelist)
```
{{ $json.message ? $json.message.from.id : $json.callback_query.from.id }}
```
equals `963246076` (Fakhri). **Wajib diperbarui** di titik ini kalau nanti ada penambahan jenis update baru — lupa update ini pernah menyebabkan klik tombol sendiri ditolak sebagai "tidak punya akses".

### Cek Callback Query
`{{ $json.callback_query !== undefined }}` is true — pemisah jalur approval vs pesan normal, diletakkan sebelum Switch1.

### Parse Callback Data (generik, dipakai WF-007 & WF-008)
Format `callback_data` tombol: `action|doc_type|doc_name` (contoh: `approve|Purchase Invoice|ACC-PINV-2026-00006`). Node ini split string jadi `action`, `doc_type`, `doc_name`, plus `chat_id` dan `message_id` dari `callback_query.message`. **Setiap tombol baru di workflow manapun WAJIB ikut format 3 bagian ini.**

### Submit Draft / Hapus Draft (generik multi-doctype)
```
URL: {{ 'http://erp.literocoffee.my.id/api/resource/' + encodeURIComponent($json.doc_type) + '/' + encodeURIComponent($json.doc_name) }}
Submit: PUT, body { docstatus: 1 }
Hapus:  DELETE
```
Generik untuk Journal Entry (WF-002/006), Purchase Invoice (WF-004), dan Period Closing Voucher (WF-008) — tidak perlu duplikasi per doctype.

### Switch1 — Daftar Lengkap Rule (urutan penting)
| # | Kondisi | Tujuan |
|---|---|---|
| 0 | startsWith `/dashboard` | Notion sync Budget |
| 1 | startsWith `/pl` | Notion sync P&L |
| 2 | startsWith `/tutupbulan` | WF-008 |
| 3 | startsWith `/aging` | Notion sync Aging |
| 4 | startsWith `/help` | Pesan Help |
| 5 | exists `message.photo[3]` | Jalur foto struk |
| 6 | regex `/^(invoice\|tagih)\s/i` | WF-003 |
| 7 | regex `/^(purchase\|beli kredit)\s/i` | WF-004 |
| 8 | regex `/^(terima bayar\|lunas)\s/i` | WF-005 piutang |
| 9 | regex `/^(bayar hutang)\s/i` | WF-005 hutang |
| 10 | regex `/^(jurnal\|koreksi)\s/i` | WF-006 |
| 11 | notRegex (semua kata kunci di atas + `\/`) | Catch-all → WF-001/002 teks manual |

**Aturan wajib saat menambah command baru:** (1) rule regex/startsWith harus pakai flag `/i` untuk yang regex, (2) tidak boleh ada tanda `=` nyempil di depan `rightValue` kecuali memang bermaksud expression, (3) update rule catch-all (nomor terakhir) untuk ikut mengecualikan kata kunci baru — kalau lupa, command baru akan terkirim dobel (sekali ke rule-nya sendiri, sekali lagi ke catch-all).

---

## 3. WF-001 / WF-002 — Cash In / Cash Out

Dua jalur input konvergen di satu titik:
- Teks manual → `Parser Manual Input` (LangChain Agent, output TYPE/NAME/TOTAL/ITEMS/TANGGAL)
- Foto struk → `Extract from File` → `Analisa Gambar Struk` (HTTP Request langsung ke Gemini API) → `If Error Struk` → `Edit Fields` (normalisasi ke field `output` supaya format sama dengan jalur teks)

Keduanya → `Cek Pemasukan/Pengeluaran` → `Code Cash In/Out - COA Mapping` (keyword → akun) → `If Validasi` (`total > 0`) → **gate budget WF-007** → `Cash In/Out - ERPNext` (POST Journal Entry, `docstatus: 1`).

**Peta akun kunci:** `5610.000` HPP umum (Cherry/bahan mentah), `5611.000` HPP Green Bean (sudah diproses), `5612.000` HPP Bahan Baku Tambahan. Awalnya `5611`/`5612` salah ditaruh sebagai anak `4000 Penjualan` (Root Type ikut jadi Income) — sudah dikoreksi ke cabang `5000 Beban`.

---

## 4. WF-003 — Sales Invoice

`Parser Invoice` (CUSTOMER/TANGGAL/TERMIN/ITEMS) → `Item Mapping + Susun Body` → `If Validasi` → `Cek Customer Ada` (GET, `onError: continueRegularOutput`) → `Cek Apakah Customer Baru` (`error !== undefined`) → kalau baru: `Buat Customer Baru` (POST, child table `accounts: [{ company, account: "1131.0010 - Piutang Dagang - LCC" }]` eksplisit) → `Sales Invoice (WF-003)` (POST `items[]`, `due_date` dari termin) → PDF.

Status hasil selalu **Unpaid** (piutang) — disengaja, beda dari WF-001 yang kas langsung.

---

## 5. WF-004 — Purchase Invoice

Cermin WF-003 untuk Supplier (`accounts: [{ company, account: "2111.001 - Hutang Dagang Dalam Negeri - LCC" }]`). Perbedaan kunci:
- Satu Purchase Invoice bisa multi-akun (Item Master beda kategori) → `Kumpulkan Total per Akun` mengelompokkan per `expense_account` sebelum masuk rangkaian cek budget (di-reuse dari WF-002); hasil digabung via `Cek Ada yang Melebihi` (`.some(r => r.json.exceeds)`).
- Urutan wajib: auto-create Supplier **harus selesai dulu** sebelum cek budget — baik jalur Draft maupun submit langsung sama-sama butuh Supplier valid.

---

## 6. WF-005 — Payment

Dua cabang paralel (arsitektur duplikasi, field beda):

| | Terima Bayar (piutang) | Bayar Hutang |
|---|---|---|
| `payment_type` | `Receive` | `Pay` |
| `party_type` | `Customer` | `Supplier` |
| Field akun kas | `paid_to` + `received_amount` | `paid_from` |
| `reference_doctype` | `Sales Invoice` | `Purchase Invoice` |

FIFO: `order_by=due_date%20asc&limit_page_length=1` — ambil satu invoice tertua. `allocated_amount = Math.min(total, outstanding_amount)`; field `sisa_outstanding_invoice` (info cicilan) dan `sisa_belum_teralokasi` (uang lebih — **belum otomatis dialokasikan ke invoice berikutnya**, keterbatasan diketahui).

---

## 7. WF-006 — Jurnal Umum

User wajib sebut kode akun eksplisit (bukan nama bebas). `Parser Jurnal Umum` (KETERANGAN/TANGGAL/LINES, baris `DEBIT|CREDIT | kode_akun | jumlah`) → `Cek Balance` → `Cari Nama Akun` (GET Account by `account_number in [...]`) → `Susun Body Journal Entry` (deteksi `missing_accounts`) → `Cek Akun Ditemukan` → submit.

---

## 8. WF-007 — Budget Approval

Gate yang disisipkan di tengah WF-002 dan WF-004, bukan workflow terpisah.

`Cek Budget Akun` (GET Budget by `account`; kosong = tidak dipantau, lanjut normal) → `Ambil Detail Budget` (GET by name, child table `budget_distribution`) → `Ambil Realisasi Bulan Ini` (GET GL Entry, filter bulan berjalan + `is_cancelled=0`) → `Hitung Persentase Budget` (cari distribusi bulan berjalan, fallback `budget_amount/12`) → IF `exceeds`.

- FALSE → submit langsung (`docstatus: 1`).
- TRUE → Draft (`docstatus: 0`) → Telegram Inline Keyboard (`callback_data: action|doc_type|doc_name`) → alur approval generik (Bagian 2).

**Struktur Budget di instance ini:** satu dokumen Budget = satu akun. `budget_against: Cost Center`, `distribution_frequency: Monthly`, child table `budget_distribution` (12 baris per bulan: `start_date`/`end_date`/`amount`/`percent`).

**Akun yang dipantau saat ini:** BBM, Tol, Parkir, Utilitas Roastery, Maintenance Mesin, Produksi Konten, PLN Kantor, Biaya Penjualan Lain-lain, Green Bean (`5611.000`).

---

## 9. WF-008 — Month-End Closing

Command `/tutupbulan` → `Cek Journal Entry Draft` + `Cek Purchase Invoice Draft` (GET `docstatus=0`) → `Gabungkan Daftar Draft` → IF `has_drafts` (kalau ada, tolak & minta selesaikan approval dulu) → `Minta Konfirmasi Tutup Buku` (Inline Keyboard, `callback_data: confirm_close|Period Closing Voucher|{tanggal}`, tombol batal pakai `cancel_close|-|-`) → lewat jalur callback yang sama → `Buat Period Closing Voucher` (POST) → PDF.

**Status: kerangka selesai, field `closing_account_head` (diisi `3220.000 - Laba Tahun Berjalan`) belum divalidasi lewat test manual di ERPNext.** Eksekusi sungguhan ditunda sampai data transaksi percobaan dibersihkan dari ERPNext.

---

## 10. Integrasi Notion — 3 Dashboard

Semua dashboard pakai pola **"cari dulu, update kalau ada, create kalau belum ada"** (bukan selalu create — mencegah data duplikat menumpuk):

```
Data dari ERPNext → Cari Row Existing (Notion Get Many, filter by nama)
                            │
                    IF row ditemukan?
                    ├─ TRUE  → Update a database page (pakai Page ID hasil pencarian)
                    └─ FALSE → Create a database page (row baru)
```

| Dashboard | Command | Sumber Data |
|---|---|---|
| Keuangan Litero Coffee (Budget vs Realisasi) | `/dashboard` | Reuse rangkaian cek budget WF-007, loop semua Budget |
| P&L Bulanan | `/pl` | GL Entry (akun `4%` = income, `5%` = expense) bulan berjalan |
| Aging Piutang & Hutang | `/aging` | Sales/Purchase Invoice `outstanding_amount > 0`, agregasi per pihak + status keterlambatan |

**Mapping properti Notion** pakai format `"NamaKolom|tipe"` (contoh: `"Kategori|select"`, `"Budget Bulanan|number"`) — harus persis sama dengan nama kolom asli di Notion (case-sensitive, termasuk spasi).

**Get Many dengan opsi "Simplify" aktif** mengembalikan field dengan prefix `property_` dan huruf kecil semua (contoh: `property_total_outstanding`, bukan `Total Outstanding`) — beda dari field yang dipakai saat **menulis** (`propertiesUi`) yang pakai nama kolom asli. Jangan disamakan saat baca-balik data untuk ringkasan Telegram.

---

## 11. Referensi Kode Akun Kunci

| Kode | Nama | Dipakai di |
|---|---|---|
| `1111.001` | Kas Kecil | Semua Cash In/Out, Payment |
| `1131.0010` | Piutang Dagang | Default Receivable Account (Customer) |
| `2111.001` | Hutang Dagang Dalam Negeri | Default Payable Account (Supplier) |
| `4110.000`–`4114.000` | Penjualan (default/HORECA/Retail/Green Bean/Ongkir) | Income mapping WF-001 |
| `5610.000` | HPP Pembelian (umum) | Cherry Robusta (bahan mentah) |
| `5611.000` | HPP Pembelian Green Bean | Green Bean (sudah diproses), dipantau Budget |
| `5612.000` | HPP Bahan Baku Tambahan | Susu, sirup, dll — WF-002 |
| `5110.001`–`5160.002`, `5230.001` | Berbagai biaya operasional | Expense mapping WF-002, akun ber-Budget |
| `3220.000` | Laba Tahun Berjalan | `closing_account_head` WF-008 (belum divalidasi) |

---

## 12. Referensi: Item Master

| Item Code | Nama | Rate Default | Expense Account | Digunakan di |
|---|---|---|---|---|
| `LKB-R` | Robusta Kersabumi | Rp25.000 | — (Sales, income) | WF-003 |
| `LSH-R` | Robusta Sinagar Ciirohani | Rp27.000 | — (Sales, income) | WF-003 |
| `CH-Rob` | Cherry Robusta | Rp13.000 | `5610.000` | WF-004 |
| `GB-Rob-b` | GB Rousta Bouron | Rp60.000 | `5611.000` | WF-004 |
| `GB-Rob-h` | GB Rousta Huni | Rp62.000 | `5611.000` | WF-004 |

---

## 13. Hasil Audit Menyeluruh (163 Node) — Agustus 2026

Audit sistematis atas seluruh file export workflow. Metode: parsing JSON, cek koneksi kosong, tanda `=` ganda, regex tanpa flag `/i`, referensi `$('Node')` tidak valid, konsistensi `callback_data`, dan struktur node Notion.

**Ditemukan & sudah diperbaiki:**
1. Regex `purchase|beli kredit` di Switch1 punya backslash dobel (`\\s` bukan `\s`) — pattern tidak pernah match, command `purchase ...` mati total (jatuh ke jalur kosong karena catch-all juga mengecualikannya). **Ini bug paling berdampak yang ditemukan.**
2. Rule `/aging` masih ada tanda `=` nyempil di depan `rightValue` (`=/aging` bukan `/aging`).
3. Rule `invoice|tagih` juga ada tanda `=` nyempil, tidak konsisten dengan rule sejenis lainnya.
4. Node Notion "Buat Aging" — property `Jatuh Tempo Terdekat|date` tidak punya value expression, selalu terkirim kosong.

**Dikonfirmasi aman (tidak ada masalah):** seluruh koneksi node (tidak ada jalan buntu), seluruh referensi `$('Nama Node')` valid, kredensial ERPNext konsisten di semua HTTP Request, format `callback_data` approval konsisten di WF-002/004/008.

---

## 14. Daftar Bug Berulang & Cara Diagnosa

Pola-pola bug yang paling sering muncul selama development — cek ini duluan sebelum menelusuri lebih jauh kalau ada node yang "tidak jalan":

1. **Koneksi node kosong (`"main": [[]]`)** — node terlihat benar isinya tapi tidak tersambung ke node berikutnya. Workflow selesai diam-diam tanpa error, gejalanya seperti "macet" padahal sudah selesai di titik itu. **Bug paling sering terjadi**, terutama setelah copy-paste/duplikasi node.
2. **Tanda `=` dobel atau nyempil tidak semestinya** di field expression — terjadi kalau field sudah otomatis mode Expression lalu template yang di-paste ikut bawa tanda `=`. Bisa menyebabkan `JSON parameter needs to be valid JSON`, Markdown Telegram gagal parse, atau rule Switch/IF tidak pernah match (paling berbahaya karena tidak selalu memunculkan error eksplisit).
3. **Regex case-sensitive by default & backslash ganda** — semua rule harus pakai format `/pattern/i` (satu backslash untuk `\s`, dst). Cek dengan test regex langsung (mis. `node -e`) kalau ragu, jangan cuma baca sekilas.
4. **Referensi `$('Nama Node')` tidak persis sama** dengan nama node di kanvas (termasuk suffix angka otomatis n8n) — gagal ambil data atau ambil dari konteks yang salah.
5. **Node native `n8n-nodes-base.erpNext` gagal untuk child table nested** — selalu pakai HTTP Request manual untuk doctype dengan child table.
6. **Default Receivable/Payable Account salah** di level Company/Customer/Supplier — selalu cek field `credit_to`/`debit_to` di dokumen hasil test pertama sebelum lanjut otomasi.
7. **PDF gagal generate karena broken image link (`wkhtmltopdf ContentOperationNotPermittedError`)** — masalah infrastruktur server, bukan n8n. Solusi sementara: `&no_letterhead=1`. Sudah diperbaiki di sisi server untuk instance ini.
8. **Referensi node di jalur callback berbeda dari jalur normal** — node yang dipanggil dari mekanisme approval TIDAK bisa merujuk node dari eksekusi workflow yang berbeda. Harus ambil ulang data dari `Parse Callback Data` atau lewat GET request baru.
9. **Data Notion menumpuk duplikat** — kalau operation selalu "Create" tanpa cek existing dulu, tiap kali command dijalankan ulang row baru selalu dibuat. Solusi: pola Get Many → IF → Update/Create (lihat Bagian 10).
10. **Field Notion tanpa value expression** — property di `propertiesUi` yang cuma punya `"key"` tanpa pasangan value (`"date"`, `"numberValue"`, dst) akan selalu terkirim kosong tanpa error apapun — mudah terlewat saat review sekilas.
