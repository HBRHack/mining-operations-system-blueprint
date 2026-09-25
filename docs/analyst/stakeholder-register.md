# Stakeholder Register: SIMTAS — Sistem Informasi Tambang (Monitoring, Inventori & Work Order)

Sistem latihan simulasi operasional tambang loading-hauling yang dipakai 4 role internal (Admin, Foreman, Operator, Storekeeper) serta menjadi portofolio pemilik proyek. Tidak ada stakeholder eksternal (customer, regulator, vendor, supplier) — simulasi internal dengan data 100% fiktif.

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 1.0 |
| **Last Updated** | 2026-09-25 |
| **Owner** | Pemilik proyek (analis + implementer + tester) |

## Register

| ID | Nama / Peran | Organisasi | Tipe | Influence | Interest | Kebutuhan dari Sistem | Cara Libatkan |
|----|--------------|------------|------|-----------|----------|----------------------|---------------|
| SH-001 | Pemilik proyek (analis + implementer + tester) | Internal | Sponsor / Tech Owner | High | High | Aplikasi jadi, lolos ≥10 test, dokumentasi SA lengkap sebagai portofolio | Sign-off dokumen SA & rilis; self-interview untuk keputusan scope |
| SH-002 | Admin (termasuk fungsi Supervisor) | Internal | User / Administrator | High | High | Kelola master data (alat, part, user), koreksi stok, act-as role lain untuk demo | Interview Aspek C; UAT modul Master Data |
| SH-003 | Foreman | Internal | User | Med | High | Board WO, prioritas & klaim WO, approve/tutup WO, gate P2H safety | Interview Aspek C; UAT alur BAB-4 |
| SH-004 | Operator | Internal | User (end user utama) | Med | High | P2H checklist cepat, input log shift & hour meter, lapor breakdown | Interview Aspek C; UAT alur BAB-3 |
| SH-005 | Storekeeper | Internal | User | Med | High | Stok masuk/koreksi, ambil part untuk WO, kartu stok & alert stok menipis | Interview Aspek C; UAT alur BAB-5 |

**Non-stakeholder (sudah dicek, tidak relevan):** customer, regulator, sponsor eksternal, supplier/vendor, tim infrastruktur luar — tidak ada transaksi eksternal, keputusan E sudah menghapus kelimanya. Bila muncul di kemudian hari (mis. dosen/penguji sebagai reviewer), tambahkan baris baru.

## Influence × Interest Map

```
              LOW Interest        HIGH Interest
            ┌──────────────────┬──────────────────┐
HIGH        │ MONITOR          │  MANAGE CLOSELY  │
Influence   │ (info berkala)   │  (libatkan aktif)│
            │ SH-002 (Admin    │  SH-001          │
            │   bila bukan      │  SH-003 Foreman  │
            │   pemilik proyek) │  SH-004 Operator │
            ├──────────────────┼──────────────────┤
LOW         │ MONITOR          │  KEEP INFORMED   │
Influence   │ (minim effort)   │  (update rutin)  │
            │ —                │  SH-005          │
            │                  │  Storekeeper     │
            └──────────────────┴──────────────────┘
```

## Key Decisions per Stakeholder

| Stakeholder | Keputusan yang Butuh Mereka | Kapan Dibutuhkan |
|-------------|----------------------------|------------------|
| SH-001 | Approve scope, sign-off dokumen SA (SRS/ERD/BR), keputusan go/no-go coding, penerima hasil rilis & test | Awal proyek; setiap Validation Gate; akhir tiap tahap coding |
| SH-002 | Validasi alur master data & koreksi stok; UAT modul Master Data | Tahap 3 (CRUD) & tahap 4 (logika stok) |
| SH-003 | Validasi state machine WO, prioritas, klaim, tombol tutup WO; UAT modul BAB-4 | Tahap 4 (state machine) |
| SH-004 | Validasi urutan langkah P2H & input log shift agar ringkas; UAT modul BAB-3 | Tahap 3–4 |
| SH-005 | Validasi aturan ambil part atomik & alert stok menipis; UAT modul BAB-5 | Tahap 4 (logika integrasi stok) |

## Peran Ganda (One-Person Project)

Karena proyek dikerjakan satu orang, SH-001 sekaligus memegang peran SA, Dev Lead, QA Lead, dan PM. Peran-peran tersebut tidak didaftarkan sebagai stakeholder terpisah — lihat `raci.md` untuk pembagian tanggung jawab lintas peran.
