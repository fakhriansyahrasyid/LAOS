# LAOS — Konsep Agentic AI & Roadmap Pengembangan

> Dokumen ini disusun untuk dimasukkan ke Business Operating System (Notion) sebagai bagian dari Knowledge Base / Project Management LAOS.

---

## 1. Apa itu Agentic AI

Agentic AI adalah sistem AI yang tidak hanya menjawab pertanyaan, tetapi mampu **mengambil keputusan dan mengeksekusi tugas secara bertahap** untuk mencapai tujuan tertentu — tanpa perlu diarahkan langkah demi langkah oleh manusia.

### Perbedaan dengan Chatbot Biasa

| Chatbot Biasa | Agentic AI |
|---|---|
| Tanya → jawab | Tujuan → rencana → eksekusi |
| Satu kali interaksi | Rangkaian langkah otomatis |
| Tidak melakukan aksi nyata | Akses tools & sistem lain, eksekusi aksi |
| Tanpa memori kerja | Menyimpan konteks antar langkah |

### 5 Komponen Inti Agentic AI

1. **Tujuan/Instruksi** — apa yang harus dicapai
2. **Reasoning** — memecah tujuan menjadi langkah-langkah
3. **Tool Use** — akses ke sistem eksternal (database, API, aplikasi)
4. **Memory** — mengingat konteks dari langkah sebelumnya
5. **Approval/Guardrail** — kapan perlu izin manusia sebelum eksekusi

### Bagaimana Ini Terwujud di LAOS

LAOS Finance sudah mengimplementasikan pola agentic AI secara penuh:

```
Telegram Trigger → Identity Service → Memory Retrieval → Intent Detection (OpenAI) 
→ ERP Connector → AI Analysis → Approval Engine (branch) → Audit Log → Format Response → Telegram
```

**Approval Engine** adalah pembeda utama dari "AI generik" — memastikan sistem aman dipakai untuk keputusan bisnis, bukan sekadar auto-eksekusi tanpa kontrol.

---

## 2. Prinsip Desain Lintas Modul

Setiap modul baru LAOS mengikuti kerangka node yang sama seperti Finance, sehingga pengembangan lebih cepat karena reuse pattern, bukan membangun dari nol:

`Trigger → Identity → Memory → Intent Detection → Connector → AI Analysis → Approval Engine → Audit Log → Response`

**Aturan gating:** modul berikutnya baru mulai dikembangkan setelah modul sebelumnya lolos VTDS (Validation, Testing & Deployment Spec) dan berjalan stabil minimal 2–4 minggu tanpa insiden besar.

---

## 3. Roadmap Pengembangan Modul

### Fase 1 — LAOS Finance *(sedang berjalan)*
| Aspek | Detail |
|---|---|
| Status | Tahap TDD |
| Tahapan tersisa | TDD → PES → Workflow Blueprint → FIDS → VTDS |
| Fokus | Deploy 8 workflow (WF-001–WF-008), uji dengan transaksi riil |
| Target keluaran | Transaksi Telegram → ERPNext → Notion berjalan stabil, minim intervensi manual |
| Estimasi durasi | 2–3 bulan |

### Fase 2 — LAOS Inventory & Production
| Aspek | Detail |
|---|---|
| Alasan prioritas | Berdampak langsung ke cash flow; datanya dibutuhkan Finance untuk laporan COGS akurat |
| Fokus | Sinkronisasi stok green bean, jadwal roasting, threshold auto-reorder |
| Approval Engine | Pembelian di bawah threshold → auto-eksekusi; di atas threshold → perlu approval |
| Estimasi durasi | 1,5–2 bulan |

### Fase 3 — LAOS Sales & CRM
| Aspek | Detail |
|---|---|
| Alasan digabung | Data pelanggan adalah dasar dari proses sales |
| Fokus | Draft penawaran HORECA otomatis, deteksi lead follow-up, churn detection |
| Approval Engine | Kritis — agent tidak boleh kirim penawaran/diskon tanpa review manusia |
| Estimasi durasi | 2 bulan |

### Fase 4 — LAOS HR
| Aspek | Detail |
|---|---|
| Prioritas | Lebih rendah — dampak langsung ke revenue lebih kecil |
| Fokus | Tracking KPI, draft SOP onboarding |
| Batasan | Keputusan people (promosi, teguran) tetap 100% manual |
| Estimasi durasi | 1 bulan |

### Fase 5 — LAOS CEO Intelligence
| Aspek | Detail |
|---|---|
| Prasyarat | Minimal Finance + Inventory + Sales sudah berjalan |
| Fokus | Ringkasan eksekutif otomatis, deteksi anomali lintas divisi |
| Fungsi strategis | Menutup loop "Vision → OKR → KPI → Project → Task → Review" |
| Estimasi durasi | 1,5 bulan |

**Total timeline kasar: ±8–10 bulan** dari Finance selesai hingga CEO Intelligence live (dapat berubah tergantung ketersediaan resource development).

---

## 4. Ringkasan Visual Urutan Modul

```
Finance ──▶ Inventory/Production ──▶ Sales/CRM ──▶ HR ──▶ CEO Intelligence
(2-3 bln)      (1.5-2 bln)            (2 bln)      (1 bln)    (1.5 bln)
```

---

## 5. Catatan untuk Review Berkala

- [ ] Finance: WF-001–008 lolos VTDS
- [ ] Finance: stabil ≥2–4 minggu tanpa insiden besar
- [ ] Inventory: threshold auto-reorder ditentukan & divalidasi
- [ ] Sales/CRM: aturan approval untuk diskon/penawaran disepakati
- [ ] HR: scope dibatasi hanya data pendukung, bukan keputusan people
- [ ] CEO Intelligence: sumber data dari ≥3 modul sudah bersih & konsisten
