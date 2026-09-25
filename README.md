# PROJEK-TAMBANG

## Analysis & System Design Portfolio — Simulasi Operasional Tambang

> **Status proyek:** Analysis-first / design portfolio  
> **Bentuk:** Dokumentasi analisis, requirement, data model, business rules, process flow, dan rekomendasi arsitektur  
> **Implementasi aplikasi:** Belum menjadi fokus utama repository ini

---

## 1. Apa sebenarnya proyek ini?

`PROJEK-TAMBANG` adalah **simulasi analisis sistem operasional tambang terbuka (open-pit)**, dengan fokus pada alur **loading–hauling equipment**, **monitoring alat**, **maintenance work order**, dan **inventory spare part**.

Repository ini sengaja dibuat **analysis-first**.

Saya tidak menjadikan pembuatan aplikasi sebagai tujuan utama. Yang ingin ditunjukkan adalah bagaimana sebuah masalah operasional diterjemahkan menjadi:

- kebutuhan bisnis dan requirement;
- aktor, hak akses, dan tanggung jawab;
- master data dan relasi data;
- business rules;
- process flow dan state machine;
- hubungan antar-modul;
- kebutuhan non-fungsional;
- asumsi dan constraint;
- serta rekomendasi teknologi untuk implementasi.

Dengan pendekatan ini, interviewer dapat membaca **cara saya berpikir sebagai analis/sistem designer**, tanpa harus terlebih dahulu menjalankan program.

---

## 2. Repository ini ditujukan kepada siapa?

Dokumen ini terutama ditujukan kepada pembaca yang menilai kemampuan **analisis sistem dan system thinking**, khususnya:

**Untuk interviewer / hiring team**
- IT Business Analyst
- System Analyst
- Solution Architect
- System / Software Architecture track
- posisi IT yang berhubungan dengan transformasi proses operasional

**Untuk technical interviewer**

Bagian yang paling relevan adalah model data, state machine, business rules, integrasi inventory–maintenance, serta alasan pemilihan teknologi.

**Untuk pembaca non-teknis**

Tidak perlu memahami Laravel untuk mengikuti proyek ini. Mulai dari bagian **Project Overview**, **Operational Flow**, dan **Business Rules**.

> **Catatan penting:** ini adalah **proyek latihan dan portofolio**, bukan representasi sistem produksi tambang dan bukan klaim bahwa seluruh prosedur lapangan telah dimodelkan secara lengkap.

---

## 3. Kenapa tidak langsung membuat program?

Karena pada proyek ini **program bukan titik utama yang ingin saya demonstrasikan**.

Sebuah aplikasi dapat dibuat dari requirement yang sudah tersedia, tetapi nilai analitisnya justru terlihat sebelum kode ditulis: apa masalahnya, siapa yang bertanggung jawab, data apa yang diperlukan, kapan status berubah, apa yang tidak boleh terjadi, dan bagaimana modul saling memengaruhi.

Jadi repository ini diposisikan sebagai:

> **"Bukti bahwa saya mampu memetakan masalah operasional menjadi rancangan sistem yang dapat diimplementasikan."**

Implementasi tetap dipikirkan sejak awal, tetapi diperlakukan sebagai **jalur implementasi yang direkomendasikan**, bukan sebagai pusat portofolio.

---

# 4. Project Overview

## Domain

Simulasi operasional alat berat pada tambang terbuka, dengan tiga modul utama:

1. **Equipment Monitoring**  
   P2H, status alat, hour meter, dan log shift.
2. **Maintenance Work Order**  
   Breakdown, pembuatan WO, maintenance, penggunaan part, dan penutupan WO.
3. **Inventory Spare Part**  
   master part, stok, transaksi pemakaian, dan histori penggunaan.

## Tujuan

Proyek memiliki dua tujuan yang berjalan bersama:

- **Latihan:** memahami desain relasi data, state machine, business rules, dan integrasi antarmodul.
- **Portofolio:** menyediakan artefak analisis yang dapat dibedah saat interview.

---

# 5. Operational Flow / State Machine

Alur utama yang dianalisis dalam proyek ini:

```text
                         ┌─ lolos ───────────────► Running
                         │
Idle ──(P2H)─────────────┤
                         │
                         └─ gagal safety-critical ─► Rejected
                                                        │
Running ──(breakdown di lapangan)──────────────────────┤
                                                        ▼
                                                   WO auto-dibuat
                                                        │
                                                        ▼
                                                   Maintenance
                                                        │
                                                 Part diambil
                                                        │
                                                        ▼
                                                   WO Closed
                                                        │
                                                        ▼
                                           ┌────────────┴────────────┐
                                           │                         │
                                  shift masih berjalan       shift sudah habis
                                           │                         │
                                           ▼                         ▼
                                        Running                    Idle
```

### Representasi ringkas

```text
idle
  └── P2H
       ├── lolos ─────────────► running
       │
       └── safety-critical fail ► rejected ──┐
                                             │
running ── breakdown ───────────────────────┤
                                             ▼
                                      WO auto-dibuat
                                             ▼
                                        maintenance
                                             ▼
                                        ambil part
                                             ▼
                                        WO closed
                                             ▼
                              running / idle berdasarkan shift
```

### Prinsip state machine

Status alat tidak diperlakukan sebagai label bebas yang dapat diganti sembarang role. Perubahan status mengikuti **event dan business rule**.

Contoh:

- P2H lolos → alat dapat masuk `Running`.
- P2H gagal pada item **safety-critical** → alat menjadi `Rejected` / ditolak jalan.
- Breakdown → sistem membuat WO dan alat masuk alur maintenance.
- WO selesai → sistem menentukan `Running` atau `Idle` berdasarkan sisa waktu shift.

Aturan ini menjadi salah satu inti desain karena status alat harus konsisten dengan proses maintenance dan kondisi operasionalnya.

---

# 6. Business Rules Utama

## Equipment / P2H

Terdapat 10 item P2H pada model latihan:

- oli mesin
- coolant
- tekanan ban
- rem
- hidrolik
- lampu
- klakson
- APAR
- sabuk pengaman
- kaca spion

Item yang ditetapkan sebagai **safety-critical** dalam simulasi adalah oli, rem, dan hidrolik.

Apabila pemeriksaan safety-critical gagal, alat tidak boleh langsung berstatus `Running`.

## Work Order

Format WO yang digunakan:

`WO-2026-0001`

Prioritas WO: `low`, `medium`, `high`.

Foreman menangani pembukaan/penutupan WO dalam model role yang digunakan.

## Inventory

Stok part **tidak boleh menjadi minus**.

Jika quantity yang diminta melebihi stok tersedia:

`WO → Menunggu Part`

Sistem tidak mengizinkan pengambilan sebagian pada skenario ini.

## Integrasi penggunaan part

Pengurangan stok terjadi **saat aksi `Ambil Part` dilakukan**, bukan setiap kali form WO disimpan atau diedit.

Tujuannya mencegah pemakaian part tercatat lebih dari satu kali akibat edit/re-save WO.

## Validasi penutupan WO

WO tidak dapat ditutup apabila part yang memang direncanakan untuk digunakan belum memiliki catatan pemakaian.

## Setelah WO Closed

Sistem menentukan kondisi alat secara otomatis:

- masih ada sisa waktu shift → `Running`
- shift sudah selesai → `Idle`

---

# 7. Roles & Responsibilities

Empat role utama pada simulasi:

| Role | Fokus utama |
|---|---|
| **Admin** | administrasi dan kebutuhan demo; dapat bertindak sebagai role lain |
| **Foreman** | pengelolaan WO, termasuk prioritas serta buka/tutup WO |
| **Operator** | P2H dan aktivitas operasional/log shift |
| **Storekeeper** | transaksi spare part dan histori pemakaian |

Login menggunakan email + password. Role-based access digunakan sebagai kontrol dasar.

---

# 8. Data yang Dimodelkan

Contoh data dummy untuk simulasi:

- **5 unit alat**
- **15 jenis spare part**
- **2 shift**: Day 06:00–18:00 dan Night 18:00–06:00
- **30 hari log shift**
- sekitar **10–15 WO histori**

### Contoh equipment ID

| Jenis | Format |
|---|---|
| Excavator | `EX-01` |
| Dump Truck | `DT-01` |
| Dozer | `DZ-01` |
| Water Truck | `WT-01` |

### Contoh part code

- `ENG-OIL-15W40`
- `HDL-HOSE-001`

Kategori part pada model:

`engine / hidrolik / electrical / undercarriage / tire`

---

# 9. Modul dan Hubungannya

```text
                 ┌──────────────────────┐
                 │ Equipment Monitoring │
                 │ P2H + Shift + HM    │
                 └──────────┬───────────┘
                            │
                 breakdown / rejected
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Maintenance / WO     │
                 │ status + priority    │
                 └──────────┬───────────┘
                            │
                        Ambil Part
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Inventory Spare Part │
                 │ stock + transaction │
                 └──────────────────────┘
```

Tiga integrasi inti:

1. **WO menggunakan part → stok inventory berkurang otomatis.**
2. **Status alat mengikuti state machine maintenance.**
3. **WO closed → status alat ditentukan otomatis berdasarkan sisa shift.**

---

# 10. Scope dan Non-Scope

## In Scope

- master data alat
- master data spare part
- user dan role
- P2H checklist
- log shift / hour meter
- breakdown
- work order
- transaksi inventory
- dashboard / laporan dasar
- business rules dan state machine

## Out of Scope

Untuk menjaga fokus analisis, simulasi ini **tidak** memodelkan integrasi eksternal seperti:

- GPS fleet tracking
- fuel management system
- HRD / payroll
- notifikasi WhatsApp / SMS
- procurement eksternal
- supplier transaction
- regulator integration

---

# 11. Struktur Dokumentasi Repository

Struktur utama repository memisahkan **analisis global**, **analisis per domain/modul**, dan **artefak arsitektur**.

```text
docs/
├── agents/
├── analyst/
│   ├── 00-Global/
│   │   ├── DATA-DICT-MASTER.md
│   │   ├── ERD-MASTER.md
│   │   ├── GLOSSARY.md
│   │   ├── REQUIREMENTS-MATRIX.md
│   │   └── SRS-MASTER.md
│   │
│   ├── BAB-1-Auth-Peran/
│   ├── BAB-2-Master-Data/
│   ├── BAB-3-Monitoring-Harian/
│   ├── BAB-4-Work-Order/
│   ├── BAB-5-Inventory/
│   │   ├── DATA-DICT-EXCERPT.md
│   │   ├── ERD-EXCERPT.md
│   │   ├── PROCESS-FLOW.md
│   │   └── USE-CASE.md
│   └── BAB-6-Dashboard/
│       ├── DATA-DICT-EXCERPT.md
│       ├── ERD-EXCERPT.md
│       ├── PROCESS-FLOW.md
│       └── USE-CASE.md
│
├── architect/                         # rekomendasi / jalur implementasi
│   ├── ARCHITECTURE.md
│   ├── TECH-STACK.md
│   ├── INTEGRATION-DESIGN.md
│   └── DEPLOYMENT-VIEW.md
│
├── assumptions-constraints.md
├── business-rules.md
├── change-management.md
├── nfr.md
├── NUMBERING-LOG.md
├── question-framework.md
├── raci.md
└── stakeholder-register.md
```

> `architect/` diposisikan sebagai **lapisan rekomendasi implementasi**. Ia menjawab pertanyaan: *"Kalau rancangan ini benar-benar dibangun, teknologi dan bentuk deployment apa yang masuk akal?"* Bukan berarti seluruh stack tersebut sudah diimplementasikan di repository ini.

---

# 12. Recommended Technology Stack

Rekomendasi implementasi yang digunakan sebagai baseline analisis arsitektur:

| Layer | Recommendation | Catatan |
|---|---|---|
| Backend | **PHP 8.3 + Laravel 12** | framework utama untuk implementasi |
| Database | **MySQL** | MariaDB kompatibel untuk development |
| UI | **Blade** | mulai sederhana; cocok untuk form dan CRUD |
| Auth | **Laravel Breeze** | login + role dasar |
| Dynamic UI | **HTMX (opsional)** | hanya untuk kebutuhan interaksi tertentu, bukan dari awal |
| Testing | **PHPUnit / Feature Test** | fokus state machine dan integrasi stok |
| Local Environment | **Laragon / XAMPP** | development lokal |
| Deployment demo | **VPS murah / shared hosting** | hanya bila diperlukan untuk interview/demo |
| Git | **Git + GitHub** | commit per tahap desain/implementasi |

### Prinsip pemilihan stack

Stack dipilih untuk menjaga implementasi tetap **ringan, mudah dijelaskan, dan tidak mengaburkan inti analisis**.

Karena repository ini analysis-first:

- **Livewire tidak diwajibkan**.
- **Filament tidak digunakan sebagai baseline**.
- CRUD manual berbasis Blade tetap menjadi opsi utama latihan.
- HTMX baru ditambahkan bila ada interaksi yang memang membutuhkan partial update/dynamic behavior.

Detail arsitektur dan keputusan teknologi sebaiknya ditempatkan di:

`docs/architect/TECH-STACK.md`

---

# 13. What I Am Demonstrating

Repository ini tidak dimaksudkan untuk membuktikan bahwa saya bisa membuat sebanyak mungkin halaman atau fitur.

Yang ingin saya tunjukkan adalah kemampuan untuk:

**Problem → Requirement → Data → Rule → Process → State → Integration → Architecture**

Contoh konkretnya:

```text
Breakdown
   ↓
State berubah
   ↓
WO terbentuk
   ↓
Maintenance membutuhkan part
   ↓
Inventory divalidasi
   ↓
Stock berkurang
   ↓
WO dapat ditutup
   ↓
State alat dihitung kembali
   ↓
Running / Idle
```

Pada titik ini, interviewer dapat menguji saya dengan pertanyaan lanjutan seperti:

- Apa yang terjadi jika stok part tidak cukup?
- Kapan stok berkurang?
- Bagaimana mencegah transaksi part tercatat dua kali?
- Siapa yang boleh mengubah status tertentu?
- Mengapa alat tidak boleh langsung `Running` setelah P2H safety-critical gagal?
- Bagaimana sistem menentukan `Running` atau `Idle` ketika WO ditutup?
- Apa konsekuensi jika state machine diubah?
- Entitas mana yang menjadi source of truth untuk status tertentu?

Jawaban terhadap pertanyaan tersebut diturunkan dari artefak yang ada di folder `docs/analyst/`.

---

# 14. Why This Repository Is Interview-Friendly

Repository ini dirancang supaya interviewer dapat **membaca artefaknya terlebih dahulu**, lalu melakukan deep-dive pada reasoning saya.

Urutan baca yang disarankan:

```text
README.md
   ↓
question-framework.md
   ↓
requirements / business-rules
   ↓
ERD + data dictionary
   ↓
process flow + use case
   ↓
NFR / RACI / stakeholder
   ↓
architect/TECH-STACK.md
   ↓
architect/ARCHITECTURE.md
```

Dengan urutan itu, pembaca bisa melihat hubungan antara **keputusan bisnis** dan **keputusan teknis**, bukan hanya melihat diagram yang berdiri sendiri.

---

# 15. Development Path (Bila Nanti Diimplementasikan)

Apabila simulasi ini dilanjutkan menjadi aplikasi, urutan implementasi yang disarankan:

```text
Database Schema
      ↓
Seeder / Dummy Data
      ↓
CRUD 3 Modul
      ↓
State Machine
      ↓
Inventory Integration
      ↓
Validation & Feature Tests
      ↓
Dashboard
      ↓
Optional Deployment Demo
```

Fokus testing pertama adalah dua area yang paling mudah rusak secara logika:

1. `Ambil Part` → stok berkurang tepat satu kali.
2. Perubahan state equipment → mengikuti business rule yang ditetapkan.

---

# 16. Limitations

Model ini sengaja dibatasi agar dapat dikerjakan sebagai solo-project.

Beberapa hal yang belum dimodelkan secara penuh antara lain detail dispatching fleet, telemetry/IoT, geospatial tracking, procurement lifecycle, budgeting, vendor management, dan integrasi enterprise lainnya.

Karena itu, pembaca sebaiknya melihat proyek ini sebagai **analisis dan system design simulation**, bukan blueprint produksi untuk perusahaan tambang.

---

# 17. Closing

> **Saya tidak sedang menunjukkan "aplikasi tambang yang sudah jadi".**
>
> Saya sedang menunjukkan bagaimana saya membedah sebuah domain operasional, menemukan aturan penting, menghubungkan data dan proses, lalu menurunkannya menjadi rancangan sistem yang siap diimplementasikan.

Repository ini adalah **analysis portfolio** dengan implementasi sebagai langkah lanjutan, bukan syarat utama untuk memahami hasil analisis.

---

## Source of Decisions

Keputusan fungsional dan teknis pada README ini mengacu pada `docs/analyst/question-framework.md`, yang mendokumentasikan keputusan tujuan proyek, role, data, state machine, integrasi inventory–WO, skala dummy data, testing, dan baseline tech stack.
