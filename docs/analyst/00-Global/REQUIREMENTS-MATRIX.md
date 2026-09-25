# Requirements Matrix — Master

50 FR (38 Must + 10 Should + 2 Could; nomor unik FR-001…FR-048 + FR-049/FR-050) + 8 NFR + 43 BR tercatat, terpetakan ke 6 chapter. Semua status **Confirmed** (sumber: question-framework + koreksi state machine, 2026-09-25). **Count diverifikasi otomatis** via script dari baris matrix (bukan hitung manual).

## Requirements by Priority

### Must Have

| ID | Requirement | Source | Rationale | Chapter | Use Case | Business Rules | Status | Owner |
|----|-------------|--------|-----------|---------|----------|----------------|--------|-------|
| FR-001 | Sistem SHALL membatasi seluruh halaman ke pengguna ter-login | QF Aspek C | Keamanan dasar aplikasi multi-peran | BAB-1-Auth-Peran | UC-001 | BR-001 | Confirmed | Pemilik proyek |
| FR-002 | Sistem SHALL menerapkan hak akses berbeda per role (admin/foreman/operator/storekeeper) | QF Aspek C | Meniru pembagian kerja nyata tambang | BAB-1-Auth-Peran | UC-001 | BR-002 | Confirmed | Pemilik proyek |
| FR-003 | Admin SHALL bisa "act as" role lain untuk demo | QF Aspek C | Demo cepat tanpa ganti akun | BAB-1-Auth-Peran | UC-002 | BR-003 | Confirmed | Pemilik proyek |
| FR-004 | Sistem SHALL membatasi role hanya 4 nilai enum | QF Aspek C | Supervisor digabung Admin (keputusan) | BAB-1-Auth-Peran | UC-003 | BR-004 | Confirmed | Pemilik proyek |
| FR-005 | Sistem SHALL memvalidasi password ≥ 8 dan menyimpan hash | NFR-SEC-002 | Praktik keamanan portofolio | BAB-1-Auth-Peran | UC-003 | BR-005 | Confirmed | Pemilik proyek |
| FR-006 | Sistem SHALL memvalidasi unit_code unik & pola (EX\|DT\|DZ\|WT)-NN | QF Aspek B | Format ID konsisten untuk laporan | BAB-2-Master-Data | UC-004 | BR-011 | Confirmed | Pemilik proyek |
| FR-007 | Sistem SHALL membuat alat baru selalu berstatus idle | QF Aspek B + koreksi | State awal deterministik | BAB-2-Master-Data | UC-004 | BR-012 | Confirmed | Pemilik proyek |
| FR-008 | Sistem SHALL menolak hapus keras alat ber-riwayat (nonaktifkan saja) | Integritas FK | Jaga riwayat P2H/log/WO | BAB-2-Master-Data | UC-004 | BR-013 | Confirmed | Pemilik proyek |
| FR-009 | Sistem SHALL memvalidasi kode part unik + enum kategori/satuan | QF Aspek B | Katalog rapi | BAB-2-Master-Data | UC-005 | BR-014 | Confirmed | Pemilik proyek |
| FR-010 | Sistem SHALL membuat part baru dengan stok 0 | QF Aspek B | Stok hanya lewat transaksi | BAB-2-Master-Data | UC-005 | BR-015 | Confirmed | Pemilik proyek |
| FR-011 | Admin/storekeeper SHALL bisa koreksi stok manual sebagai transaksi berketerangan | BR-058 | Stock opname | BAB-2-Master-Data | UC-006 | BR-058 | Confirmed | Pemilik proyek |
| FR-013 | Sistem SHALL mengizinkan idle → running hanya bila P2H hari ini pass | State machine koreksi | Gate keselamatan | BAB-3-Monitoring-Harian | UC-007, UC-008 | BR-021 | Confirmed | Pemilik proyek |
| FR-014 | Sistem SHALL membatasi 1 P2H per alat per hari | ASM-002 | Checklist harian unik | BAB-3-Monitoring-Harian | UC-007 | BR-022 | Confirmed | Pemilik proyek |
| FR-015 | Sistem SHALL menolak jalan & auto-buat WO saat item safety-critical gagal | **Koreksi 2026-09-25** | Kunci desain: rejected transisi singkat | BAB-3-Monitoring-Harian | UC-007, UC-009 | BR-023, BR-036 | Confirmed | Pemilik proyek |
| FR-016 | Sistem SHALL meneruskan P2H pass dengan warning bila hanya item non-kritis gagal | QF Aspek B | 7 item non-kritis bukan penghalang | BAB-3-Monitoring-Harian | UC-007 | BR-024 | Confirmed | Pemilik proyek |
| FR-017 | Sistem SHALL menghitung jam_operasi dari jam mulai–selesai (lintas hari) | QF Aspek B | Dasar HM & laporan | BAB-3-Monitoring-Harian | UC-010 | BR-025 | Confirmed | Pemilik proyek |
| FR-018 | Sistem SHALL menambah hour_meter otomatis = jam_operasi | QF Aspek B | HM realistis tanpa input manual | BAB-3-Monitoring-Harian | UC-010 | BR-026 | Confirmed | Pemilik proyek |
| FR-019 | Sistem SHALL mewajibkan alasan_idle bila jam_operasi = 0 | QF Aspek B | Akuntabilitas downtime kecil | BAB-3-Monitoring-Harian | UC-010 | BR-027 | Confirmed | Pemilik proyek |
| FR-020 | Sistem SHALL menolak log shift duplikat per (alat, shift, tanggal) | ASM-004 | Cegah dobel input | BAB-3-Monitoring-Harian | UC-010 | BR-028 | Confirmed | Pemilik proyek |
| FR-021 | Sistem SHALL membuat WO otomatis saat P2H gagal kritis, tanpa approval | **Koreksi 2026-09-25** | Temuan wajib ditindaklanjuti hari itu | BAB-4-Work-Order | UC-009 | BR-036, BR-023 | Confirmed | Pemilik proyek |
| FR-022 | Sistem SHALL membuat WO otomatis saat operator lapor breakdown | Task description | Alur breakdown digital | BAB-4-Work-Order | UC-009 | BR-036, BR-039 | Confirmed | Pemilik proyek |
| FR-023 | Sistem SHALL menegakkan transisi status WO tertutup (open→menunggu_part→in_progress→closed) | Task description | State machine sesuai spesifikasi | BAB-4-Work-Order | UC-011 | BR-037 | Confirmed | Pemilik proyek |
| FR-024 | Sistem SHALL men-set equipment=maintenance selama seluruh fase WO | **Koreksi 2026-09-25** | Mapping fase WO ke status alat | BAB-4-Work-Order | UC-011 | BR-038 | Confirmed | Pemilik proyek |
| FR-025 | Sistem SHALL menyimpan asal WO hanya di trigger_type (p2h_gagal/breakdown_lapangan) | **Koreksi 2026-09-25** | Dua KPI dari satu tabel | BAB-4-Work-Order | UC-009 | BR-039 | Confirmed | Pemilik proyek |
| FR-026 | Sistem SHALL membatasi maksimal 1 WO terbuka per alat | Keputusan 2026-09-25 | Cegah konflik status alat | BAB-4-Work-Order | UC-009 | BR-040 | Confirmed | Pemilik proyek |
| FR-027 | Sistem SHALL memisahkan rencana part (work_order_part) dari transaksi stok | QF Aspek F | Anti stok-dobel saat edit WO | BAB-4-Work-Order | UC-013 | BR-041 | Confirmed | Pemilik proyek |
| FR-028 | Sistem SHALL mengeksekusi Ambil Part atomik: validasi→transaksi→kurangi stok→tandai taken | QF Aspek D (6W) | Integritas stok | BAB-4-Work-Order | UC-013 | BR-042, BR-062 | Confirmed | Pemilik proyek |
| FR-029 | Sistem SHALL menolak pengambilan bila qty > stok dan me-rollback seluruhnya | QF Aspek D | Stok tak boleh minus | BAB-4-Work-Order | UC-013 | BR-042, BR-056 | Confirmed | Pemilik proyek |
| FR-030 | Sistem SHALL mengubah WO ke menunggu_part saat ada rencana part yang stoknya kurang | QF Aspek D | Backorder state | BAB-4-Work-Order | UC-011 | BR-043 | Confirmed | Pemilik proyek |
| FR-031 | Sistem SHALL menolak close WO bila masih ada part status planned | QF Aspek D | Data stok akurat saat close | BAB-4-Work-Order | UC-014 | BR-044 | Confirmed | Pemilik proyek |
| FR-032 | Sistem SHALL menentukan status alat setelah close (running/idle) otomatis dari jadwal shift | QF Aspek D (keputusan 3) | Tanpa pilihan manual | BAB-4-Work-Order | UC-014 | BR-045 | Confirmed | Pemilik proyek |
| FR-033 | Sistem SHALL mengizinkan klaim/ubah prioritas/close WO hanya foreman/admin | QF Aspek C | Pemilik proses WO jelas | BAB-4-Work-Order | UC-011, UC-014 | BR-046 | Confirmed | Pemilik proyek |
| FR-035 | Sistem SHALL menjamin stok ≥ 0 pada setiap transaksi out (check + rollback) | QF Aspek B | Hard constraint | BAB-5-Inventory | UC-013, UC-015 | BR-056 | Confirmed | Pemilik proyek |
| FR-036 | Sistem SHALL mengubah stok hanya lewat tabel stock_transaction (append-only) | Desain audit | Semua perubahan stok terjelaskan | BAB-5-Inventory | UC-016 | BR-057, BR-063 | Confirmed | Pemilik proyek |
| FR-038 | Sistem SHALL menyediakan form stok masuk (qty>0, keterangan) role storekeeper/admin | Task description | Transaksi keluar-masuk | BAB-5-Inventory | UC-015 | BR-059 | Confirmed | Pemilik proyek |
| FR-042 | Dashboard SHALL menampilkan jumlah alat per status (query live) | Task description | Kartu status real-time | BAB-6-Dashboard | UC-017 | BR-076, BR-079 | Confirmed | Pemilik proyek |
| FR-043 | Dashboard SHALL menampilkan total downtime per rentang tanggal | Task description | KPI ketersediaan alat | BAB-6-Dashboard | UC-018 | BR-077, BR-047 | Confirmed | Pemilik proyek |
| FR-044 | Dashboard SHALL menampilkan 5 part paling sering dipakai (out + WO) | Task description | Perencanaan stok | BAB-6-Dashboard | UC-019 | BR-078 | Confirmed | Pemilik proyek |

### Should Have

| ID | Requirement | Source | Rationale | Chapter | Use Case | Business Rules | Status | Owner |
|----|-------------|--------|-----------|---------|----------|----------------|--------|-------|
| FR-012 | Admin SHALL CRUD master alat (tipe, model, kapasitas) | Task description | Modul 1 | BAB-2-Master-Data | UC-004 | BR-011 | Confirmed | Pemilik proyek |
| FR-049 | Admin/storekeeper SHALL CRUD master part | Task description | Modul 2 | BAB-2-Master-Data | UC-005 | BR-014 | Confirmed | Pemilik proyek |
| FR-050 | Foreman SHALL mengelola prioritas & klaim WO (assigned_to) | QF Aspek B | Alur kerja foreman | BAB-4-Work-Order | UC-011 | BR-046 | Confirmed | Pemilik proyek |
| FR-034 | Sistem SHALL menghitung downtime = closed_at − detected_at (berjalan bila belum closed) | ASM-005 | Dasar KPI | BAB-4-Work-Order | UC-012 | BR-047 | Confirmed | Pemilik proyek |
| FR-037 | Sistem SHALL menampilkan badge "Stok Menipis" bila stok ≤ min_stok | QF (min_stok) | Peringatan dini | BAB-5-Inventory | UC-016, UC-020 | BR-060 | Confirmed | Pemilik proyek |
| FR-039 | Sistem SHALL menyediakan kartu stok / riwayat transaksi per part | Task description | Traceability | BAB-5-Inventory | UC-016 | BR-063 | Confirmed | Pemilik proyek |
| FR-040 | Sistem SHALL memvalidasi qty transaksi selalu > 0 | DATA-DICT | Anti data rusak | BAB-5-Inventory | UC-015 | BR-061 | Confirmed | Pemilik proyek |
| FR-041 | Sistem SHALL mencatat laporan breakdown < 30 menit sebagai idle biasa (tanpa WO) | QF Aspek A | Batas downtime penting | BAB-4-Work-Order | UC-009 | BR-047 | Confirmed | Pemilik proyek |
| FR-045 | Laporan safety SHALL menghitung KPI terpisah per trigger_type | **Koreksi 2026-09-25** | KPI safety vs keandalan | BAB-6-Dashboard | UC-021 | BR-079 | Confirmed | Pemilik proyek |
| FR-046 | Sistem SHALL membatasi dashboard penuh ke foreman/admin (operator: ringkasan) | QF Aspek C | Hak laporan | BAB-6-Dashboard | UC-017 | BR-080 | Confirmed | Pemilik proyek |

### Could Have

| ID | Requirement | Source | Rationale | Chapter | Use Case | Business Rules | Status | Owner |
|----|-------------|--------|-----------|---------|----------|----------------|--------|-------|
| FR-047 | Sistem SHALL menampilkan daftar WO aktif (board) untuk foreman | QF Aspek C | Klaim kerja | BAB-4-Work-Order | UC-011 | BR-046 | Confirmed | Pemilik proyek |
| FR-048 | Sistem SHALL menyediakan halaman daftar alat + filter status | QF Aspek C | Monitoring | BAB-3-Monitoring-Harian | UC-008 | BR-021 | Confirmed | Pemilik proyek |

## Full Traceability Matrix (NFR)

| ID | Requirement | Source | Rationale | Chapter | Use Case | Status |
|----|-------------|--------|-----------|---------|----------|--------|
| NFR-PERF-001 | Render halaman < 2 dtk p95 skala 1–5 user | QF Aspek E | Responsiveness form-based | 00-Global | UC-017 | Confirmed |
| NFR-SEC-001 | 100% route tulis terproteksi auth; salah role → 403 | QF Aspek E | Keamanan minimal | 00-Global | UC-001 | Confirmed |
| NFR-SEC-002 | 0 password plain text (hash) | Best practice | Portofolio & keamanan | 00-Global | UC-003 | Confirmed |
| NFR-REL-001 | Operasi stok/WO/alat atomik — tanpa kondisi parsial | QF Aspek F | Anti korupsi data | 00-Global | UC-013 | Confirmed |
| NFR-TEST-001 | ≥ 10 feature test logika kritis, suite < 30 dtk | QF Aspek G | Bahan portofolio + safety net | 00-Global | — | Confirmed |
| NFR-USAB-001 | P2H ≤ 3 mnt tanpa training; UI 100% Bahasa Indonesia | QF Aspek G (konvensi) | Demo mulus | 00-Global | UC-007 | Confirmed |
| NFR-PORT-001 | migrate:fresh --seed jalan < 10 mnt di environment baru | QF Aspek E/G | Reproduksi demo | 00-Global | — | Confirmed |
| NFR-MAINT-001 | Tambah CRUD baru ≤ 1 jam; logika di Service class | QF Aspek G | Kode rapi untuk reviewer | 00-Global | — | Confirmed |

## Coverage Summary

| Area | Total | Confirmed | Pending | N/A |
|------|-------|-----------|---------|-----|
| Functional Requirements | 50 | 50 | 0 | 0 |
| Business Rules | 43 | 43 | 0 | 0 |
| Non-Functional | 8 (+8 kategori N/A) | 8 | 0 | 8 |

## Gaps and Risks

| Gap | Impact | Mitigation |
|-----|--------|------------|
| ~~FR-012b / FR-027b~~ (sebelumnya nomor tempelan) | Sudah diperbaiki → renumber jadi **FR-049** (CRUD part) & **FR-050** (kelola WO) | — | Tidak ada dampak kode — hanya penataan nomor |
| Belum ada UAT scenario | Rilis tanpa skenario uji formal | Lanjut `/analyst-uat-planner` bila perlu |
| Ketersediaan PHP 8.3 di mesin belum diverifikasi (ASM-001) | Laravel 12 butuh PHP ≥ 8.2 | Cek `php -v` saat mulai coding — langkah pertama eksekusi |
| Belum ada post-RS traceability ke test case | Test belum terpetakan ke FR | Saat Tahap 3, tulis test case yang menunjuk FR/BR ID |

## Status Legend

| Status | Meaning |
|--------|---------|
| Confirmed | Disetujui & didokumentasikan |
| Pending | Belum divalidasi |
| Deprecated | Tidak berlaku |
