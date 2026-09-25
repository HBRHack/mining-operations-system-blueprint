# GLOSSARY — Simulasi Sistem Operasional Tambang

Satu sentral untuk seluruh istilah lintas modul. Owner: pemilik proyek. Definisi disetujui via question-framework (2026-09-25).

| Istilah | Definisi | Jangan Dikira |
|---------|----------|---------------|
| **P2H** | Penunjukan, Pemeriksaan Harian — checklist inspeksi yang wajib diisi operator sebelum alat dipakai hari itu | Bukan maintenance berkala, bukan service besar |
| **Safety-critical (item P2H)** | Item P2H yang kalau gagal → alat ditolak jalan: oli mesin, rem, hidrolik | Bukan seluruh item — 7 item lain cukup warning |
| **Alat / Equipment** | Unit alat berat (excavator, dump truck, dozer, water truck) | Bukan kendaraan operasional kantor |
| **Unit ID** | Kode unik alat: `EX-01`, `DT-01`, `DZ-01`, `WT-01` | Bukan nomor rangka / nomor lambung asli |
| **Hour Meter (HM)** | Akumulasi jam operasi mesin alat, bertambah tiap log shift sesuai jam running | Bukan odometer kilometer |
| **Shift** | Pembagian waktu kerja: Day 06:00–18:00, Night 18:00–06:00 (2 shift) | Bukan 3 shift 8 jam |
| **Log Shift** | Catatan jam operasional alat per shift: jam mulai/selesai, material (OB/coal/ore), tonase, alasan idle | Bukan absensi karyawan |
| **OB** | Overburden — lapisan tanah/batuan di atas material bernilai yang dibuang | Bukan batubara |
| **Breakdown** | Kerusakan alat yang ditemukan saat alat sedang operasi (running) | Bukan P2H gagal (temuan sebelum jalan) |
| **Rejected** | Status transisi: alat ditolak jalan karena P2H gagal item safety-critical | Bukan status permanen — langsung dispar ke WO |
| **Work Order (WO)** | Tiket perbaikan resmi, selalu terhubung ke 1 alat; nomor `WO-YYYY-NNNN` | Bukan surat perintah kerja kertas |
| **trigger_type** | Kolom pembeda asal WO: `p2h_gagal` atau `breakdown_lapangan` — dasar 2 KPI terpisah | Bukan status WO |
| **Menunggu Part** | Status WO: rencana part belum bisa diambil seluruhnya karena stok kurang (backorder) | Bukan stok minus |
| **Ambil Part** | Aksi sekali-klik dari halaman WO yang memvalidasi stok lalu membuat transaksi keluar atomik | Bukan edit biasa form WO |
| **Part / Spare Part** | Komponen pengganti: kode `ENG-OIL-15W40` dst., kategori engine/hidrolik/electrical/undercarriage/tire | Bukan jasa/perbaikan |
| **Downtime** | Durasi alat tidak produktif: dari `detected_at` WO sampai `closed_at` WO | Bukan idle menunggu material |
| **Foreman** | Pemilik proses Work Order: klaim, prioritas, tutup WO | Bukan Supervisor (digabung ke Admin) |
| **Storekeeper** | Penjaga gudang part: satu-satunya yang mencatat transaksi stok | Bukan pembeli/procurement |
| **Act as** | Fitur Admin masuk meniru peran lain untuk keperluan demo | Bukan user ganda |
