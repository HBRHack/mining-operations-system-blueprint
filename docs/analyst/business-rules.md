# Business Rules — Simulasi Sistem Operasional Tambang

43 business rules lintas 3 modul (BAB-1: 5 · BAB-2: 5 · BAB-3: 8 · BAB-4: 12 · BAB-5: 8 · BAB-6: 5). Reserved range per chapter: lihat `NUMBERING-LOG.md`.

## BAB-1 — Auth & Peran (BR-001–010)

### BR-001: Login wajib sebelum akses
| Field | Value |
|-------|-------|
| **ID** | BR-001 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek C (2026-09-25) |

**Rule:** Semua halaman (baca & tulis) hanya dapat diakses pengguna yang sudah login.
**When:** Request masuk tanpa sesi valid.
**Then:** Redirect ke halaman login.
**Else:** — (tidak ada pengecualian).
**Related:** UC-001 · FR-001 · users

---

### BR-002: Hak akses per role
| Field | Value |
|-------|-------|
| **ID** | BR-002 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek C |

**Rule:** Admin = penuh + act-as; Foreman = kelola WO + baca semua; Operator = P2H, log shift, lapor breakdown; Storekeeper = transaksi stok + ambil part.
**When:** User mencoba aksi di luar hak role-nya.
**Then:** Tolak (403), tidak ada perubahan data.
**Else:** — 
**Related:** UC-001, UC-002 · FR-002

---

### BR-003: Act-as hanya Admin
| Field | Value |
|-------|-------|
| **ID** | BR-003 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek C |

**Rule:** Hanya role `admin` boleh mengisi `act_as`; selama `act_as` aktif, seluruh hak akses mengikuti peran yang ditiru (bukan admin).
**When:** Admin memilih "act as" role lain untuk demo.
**Then:** Session memakai hak peran target sampai kembali ke peran asli.
**Else:** User non-admin mengirim `act_as` → ditolak.
**Related:** UC-002 · FR-003

---

### BR-004: Role hanya 4 nilai
| Field | Value |
|-------|-------|
| **ID** | BR-004 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek C (Supervisor digabung Admin) |

**Rule:** `users.role` hanya menerima `admin`, `foreman`, `operator`, `storekeeper`.
**When:** Insert/update user.
**Then:** Validasi enum; nilai lain ditolak.
**Else:** —
**Related:** FR-004 · Value List Role

---

### BR-005: Password minimal 8 karakter
| Field | Value |
|-------|-------|
| **ID** | BR-005 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | NFR-SEC-001 (keamanan minimal) |

**Rule:** Password ≥ 8 karakter dan disimpan ter-hash (bcrypt/argon2), tidak pernah plain text.
**When:** Registrasi/ubah password.
**Then:** Validasi + hash sebelum persist.
**Else:** Validasi gagal → pesan error, data tidak disimpan.
**Related:** UC-001, UC-003 · FR-005

---

## BAB-2 — Master Data (BR-011–020)

### BR-011: Unit code unik & format tetap
| Field | Value |
|-------|-------|
| **ID** | BR-011 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B |

**Rule:** `equipment.unit_code` unik dan cocok pola `(EX|DT|DZ|WT)-NN`.
**When:** Tambah/ubah alat.
**Then:** Cek unique + regex; gagal → pesan error.
**Else:** —
**Related:** UC-004 · FR-006

---

### BR-012: Alat baru selalu idle
| Field | Value |
|-------|-------|
| **ID** | BR-012 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | State machine (koreksi 2026-09-25) |

**Rule:** Alat baru dibuat selalu berstatus `idle`; status lain hanya bisa dicapai lewat alur P2H/WO.
**When:** Insert equipment.
**Then:** `status = 'idle'`.
**Else:** Form menyediakan field status → hapus/diabaikan.
**Related:** UC-004 · FR-007

---

### BR-013: Hapus alat yang punya riwayat = nonaktif
| Field | Value |
|-------|-------|
| **ID** | BR-013 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Integritas referensial (asumsi disepakati) |

**Rule:** Alat dengan riwayat P2H/log/WO tidak boleh dihapus keras (FK constraint) — hanya bisa dinonaktifkan.
**When:** Admin mencoba hapus alat ber-riwayat.
**Then:** Tolak dengan pesan "gunakan nonaktifkan".
**Else:** Alat tanpa riwayat boleh dihapus.
**Related:** UC-004 · FR-008

---

### BR-014: Kode part unik
| Field | Value |
|-------|-------|
| **ID** | BR-014 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B |

**Rule:** `part.kode` unik; kategori & satuan hanya dari Value List.
**When:** Tambah/ubah part.
**Then:** Validasi unique + enum.
**Else:** —
**Related:** UC-005 · FR-009

---

### BR-015: Stok awal part = 0
| Field | Value |
|-------|-------|
| **ID** | BR-015 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B |

**Rule:** Part baru dibuat dengan `stok = 0`; stok hanya bertambah lewat transaksi `in` (BR-057).
**When:** Insert part baru.
**Then:** `stok = 0` (field stok tidak diisi manual di form).
**Else:** koreksi manual lewat transaksi (BR-058).
**Related:** UC-005 · FR-010

---

## BAB-3 — Monitoring Harian (BR-021–035)

### BR-021: P2H wajib sebelum running
| Field | Value |
|-------|-------|
| **ID** | BR-021 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | State machine (koreksi 2026-09-25) |

**Rule:** Alat hanya bisa berpindah `idle → running` jika P2H hari itu berstatus `pass`.
**When:** Operator mencoba mulai operasi tanpa P2H pass hari ini.
**Then:** Tolak, arahkan ke form P2H.
**Else:** Izinkan, `status = running`.
**Related:** UC-007, UC-008 · FR-013

---

### BR-022: P2H maksimal 1× per alat per hari
| Field | Value |
|-------|-------|
| **ID** | BR-022 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | ASM-002 |

**Rule:** Unik `(equipment_id, tanggal)` — checklist kedua di hari yang sama ditolak (revisi: bukan edit, tapi isi ulang gagal).
**When:** Submit P2H untuk alat+tanggal yang sudah ada.
**Then:** Tolak / arahkan ke checklist eksisting.
**Else:** —
**Related:** UC-007 · FR-014

---

### BR-023: Gagal safety-critical → rejected + auto-WO
| Field | Value |
|-------|-------|
| **ID** | BR-023 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | **Koreksi user 2026-09-25** (state machine 5 state) |

**Rule:** P2H gagal pada item safety-critical (oli mesin, rem, hidrolik) → `equipment.status = rejected` → **WO otomatis dibuat** (trigger_type=`p2h_gagal`, prioritas=`high`, assignee=null) → status langsung `maintenance`. `rejected` hanya hidup dalam satu transaksi DB (atau stuck bila pembuatan WO gagal = fail-safe).
**When:** Submit P2H dengan ≥1 item safety-critical berstatus `fail`.
**Then:** Satu DB transaction: insert checklist → set rejected → insert WO auto → set maintenance. Bila insert WO gagal: rollback status ke `rejected` + tampilkan error ke Foreman (jangan pernah kembali `idle`/`running`).
**Else:** Semua item ok → `hasil = pass`.
**Related:** UC-007, UC-009 · FR-015, FR-021 · BR-036

---

### BR-024: Gagal non-kritis = warning saja
| Field | Value |
|-------|-------|
| **ID** | BR-024 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B (10 item, 3 kritis) |

**Rule:** Kegagalan item non-safety-critical (lampu, klakson, APAR, sabuk, spion, coolant, tekanan ban) → checklist tetap `pass` dengan catatan warning; alat tetap boleh jalan.
**When:** Hanya item non-kritis yang fail.
**Then:** `hasil = pass`, item tetap tercatat `fail` + catatan.
**Else:** —
**Related:** UC-007 · FR-016

---

### BR-025: jam_operasi = selisih jam
| Field | Value |
|-------|-------|
| **ID** | BR-025 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B |

**Rule:** `jam_operasi = jam_selesai − jam_mulai` (menit→desimal jam), dihitung sistem, bukan input manual; lintas hari (Night shift) dihitung +24 jam.
**When:** Simpan log shift.
**Then:** Hitung & simpan otomatis; tolak jika < 0.
**Else:** —
**Related:** UC-010 · FR-017

---

### BR-026: Hour meter bertambah otomatis
| Field | Value |
|-------|-------|
| **ID** | BR-026 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B (HM ada) |

**Rule:** `equipment.hour_meter` bertambah sebesar `jam_operasi` saat log shift disimpan; `hour_meter_akhir` dicatat di log. HM tidak bisa diedit manual.
**When:** Log shift tersimpan.
**Then:** Transaksi update HM + log.
**Else:** —
**Related:** UC-010 · FR-018

---

### BR-027: Alasan idle wajib bila jam_operasi = 0
| Field | Value |
|-------|-------|
| **ID** | BR-027 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B (alasan_idle kolom) |

**Rule:** Log dengan `jam_operasi = 0` wajib mengisi `alasan_idle` (mis. "menunggu excavator", "hujan").
**When:** Simpan log shift dengan durasi 0.
**Then:** Validasi `alasan_idle` terisi.
**Else:** —
**Related:** UC-010 · FR-019

---

### BR-028: Log shift tidak boleh duplikat
| Field | Value |
|-------|-------|
| **ID** | BR-028 |
| **Category** | Constraint |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Asumsi (ASM-004) |

**Rule:** Satu alat maksimal 1 log per shift per tanggal (revisi: boleh lebih dari 1 blok jam dalam sehari selain overlap — default: 1 log per (equipment, shift, tanggal)).
**When:** Insert log dengan (equipment, shift, tanggal) sudah ada.
**Then:** Tolak / tawarkan edit log eksisting.
**Else:** —
**Related:** UC-010 · FR-020

---

## BAB-4 — Work Order (BR-036–055)

### BR-036: WO selalu auto-created, tanpa approval
| Field | Value |
|-------|-------|
| **ID** | BR-036 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | **Koreksi user 2026-09-25** — rejected/breakdown = transisi singkat |

**Rule:** WO dibuat **otomatis oleh sistem** saat (a) P2H gagal safety-critical atau (b) operator melapor breakdown — tanpa menunggu persetujuan manual. Default: `status=open`, `prioritas=high`, `assigned_to=null`, `detected_at=now()`.
**When:** P2H fail kritis / laporan breakdown diterima.
**Then:** Insert WO dalam DB transaction yang sama dengan perubahan status alat.
**Else:** Pembuatan gagal → alat tetap `rejected`/`breakdown` (fail-safe) + error ke Foreman.
**Related:** UC-007, UC-009 · FR-021, FR-022 · BR-023

---

### BR-037: Transisi status WO tertutup
| Field | Value |
|-------|-------|
| **ID** | BR-037 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Task description (open → menunggu_part → in_progress → closed) |

**Rule:** `open → menunggu_part` (saat ada part planned tapi stok kurang) · `menunggu_part → in_progress` (semua part taken) · `open → in_progress` (tanpa part / semua langsung tersedia) · `→ closed` hanya dari `in_progress` (BR-044). Tidak ada lompatan lain; `closed` final.
**When:** Update status WO.
**Then:** Validasi transisi; langkah tak sah → tolak.
**Else:** —
**Related:** UC-011 · FR-023

---

### BR-038: Status alat mengikuti fase WO
| Field | Value |
|-------|-------|
| **ID** | BR-038 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | **Koreksi user 2026-09-25** — mapping maintenance |

**Rule:** Selama WO berstatus `open`/`menunggu_part`/`in_progress`, `equipment.status = maintenance`. Perubahan status WO selalu menulis status alat dalam transaksi yang sama.
**When:** Status WO berubah.
**Then:** Sinkronkan equipment.status = maintenance.
**Else:** WO closed → BR-045.
**Related:** UC-011 · FR-024

---

### BR-039: Asal WO ada di trigger_type
| Field | Value |
|-------|-------|
| **ID** | BR-039 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | **Koreksi user 2026-09-25** — KPI safety |

**Rule:** Asal kerusakan TIDAK disimpan di status alat — hanya di `work_orders.trigger_type` ∈ {`p2h_gagal`, `breakdown_lapangan`}. KPI safety dihitung dari kolom ini, dua-duanya dari 1 tabel WO.
**When:** Pembuatan WO.
**Then:** Wajib isi trigger_type; laporan kelompokkan per trigger_type.
**Else:** —
**Related:** UC-009, UC-021 · FR-025, FR-042

---

### BR-040: Maksimal 1 WO terbuka per alat
| Field | Value |
|-------|-------|
| **ID** | BR-040 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Keputusan pasca-koreksi 2026-09-25 |

**Rule:** Satu equipment hanya boleh punya 1 WO berstatus ≠ `closed` (partial unique index). Selama `maintenance`, alat tidak bisa running → tidak bisa breakdown kedua.
**When:** Auto-buat WO untuk alat yang sudah punya WO terbuka.
**Then:** Tolak + notifikasi "WO-XXX masih terbuka".
**Else:** —
**Related:** UC-009 · FR-026

---

### BR-041: Rencana part ≠ transaksi stok
| Field | Value |
|-------|-------|
| **ID** | BR-041 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek F (relasi paling riskan) |

**Rule:** Menambah part ke daftar rencana WO **tidak** mengubah stok. Stok hanya berkurang saat tombol "Ambil Part" ditekan — transaksi dicatat **sekali per klik**, bukan otomatis tiap form WO disimpan (anti stok-dobel).
**When:** Save form WO dengan baris part rencana.
**Then:** Insert/update `work_order_part` status=planned; `part.stok` tidak disentuh.
**Else:** —
**Related:** UC-011, UC-013 · FR-027 · BR-042

---

### BR-042: Ambil part atomik & sekali jalan
| Field | Value |
|-------|-------|
| **ID** | BR-042 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek D (Interface 6W) |

**Rule:** Klik "Ambil Part" dalam SATU DB transaction: validasi `stok ≥ qty` → insert `stock_transaction` (out, work_order_id) → kurangi `part.stok` → tandai `work_order_part.status=taken` + isi `stock_transaction_id` → evaluasi status WO (BR-037). Gagal di satu langkah = rollback semua.
**When:** Storekeeper/Foreman menekan Ambil Part dengan qty melebihi stok.
**Then:** Seluruh transaksi di-rollback; pesan "stok tidak cukup — WO masuk Menunggu Part".
**Else:** Berhasil → stok berkurang tepat 1×.
**Related:** UC-013 · FR-028, FR-029 · NFR-REL-001

---

### BR-043: Stok kurang → menunggu_part
| Field | Value |
|-------|-------|
| **ID** | BR-043 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek D (backorder) |

**Rule:** Jika ada `work_order_part` planned dengan qty > stok tersedia, WO berstatus `menunggu_part`. Pengambilan sebagian (parsial) tidak diizinkan v1 — semua atau tidak sama sekali.
**When:** Rencana part ditambah / stok berubah.
**Then:** Evaluasi & set status WO.
**Else:** Semua part tersedia → `in_progress`.
**Related:** UC-011 · FR-030

---

### BR-044: Close WO wajib semua part taken
| Field | Value |
|-------|-------|
| **ID** | BR-044 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek D (validasi close) |

**Rule:** WO tidak bisa `closed` selama masih ada `work_order_part.status = planned`. Foreman melihat daftar part yang belum diambil; sistem menolak close.
**When:** Foreman mencoba tutup WO dengan part planned tersisa.
**Then:** Tolak + daftar part belum terpakai.
**Else:** Semua taken → izinkan close, isi `closed_at`.
**Related:** UC-014 · FR-031

---

### BR-045: Setelah close, alat kembali running/idle otomatis
| Field | Value |
|-------|-------|
| **ID** | BR-045 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek D (keputusan 3) |

**Rule:** Saat WO closed, sistem membandingkan `now()` dengan jadwal shift hari ini (WITA): masih dalam shift → `equipment.status = running`; di luar shift → `idle`. Ditentukan sistem, bukan pilihan manual.
**When:** WO berstatus closed.
**Then:** Hitung berdasar jam & jadwal shift → set status alat.
**Else:** —
**Related:** UC-014 · FR-032

---

### BR-046: Hanya Foreman/Admin yang boleh tutup & klaim WO
| Field | Value |
|-------|-------|
| **ID** | BR-046 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek C |

**Rule:** Klaim WO (`assigned_to`), ubah prioritas, dan close WO hanya role `foreman`/`admin`. Operator hanya boleh melapor & melihat; Storekeeper hanya mengurus bagian part.
**When:** Aksi di halaman WO.
**Then:** Cek role (BR-002).
**Else:** 403.
**Related:** UC-011, UC-014 · FR-033

---

### BR-047: Downtime = detected_at → closed_at
| Field | Value |
|-------|-------|
| **ID** | BR-047 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | ASM-005 + Question-framework Aspek A (>30 mnt wajib WO) |

**Rule:** Downtime satu WO = `closed_at − detected_at`. WO belum closed dihitung sebagai downtime berjalan. Aturan >30 menit: laporan breakdown di bawah 30 menit dicatat sebagai idle biasa (log shift), tidak memicu WO.
**When:** Query laporan downtime.
**Then:** Hitung selisih timestamp; bila belum closed pakai `now()`.
**Else:** —
**Related:** UC-012, UC-018 · FR-034, FR-041

---

## BAB-5 — Inventory (BR-056–075)

### BR-056: Stok tidak boleh minus
| Field | Value |
|-------|-------|
| **ID** | BR-056 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B & D |

**Rule:** `part.stok ≥ 0` selalu — divalidasi dalam transaksi DB (check constraint + application lock) pada setiap transaksi `out`.
**When:** Transaksi out yang membuat stok < 0.
**Then:** Rollback seluruh transaksi.
**Else:** —
**Related:** UC-013, UC-015 · FR-035 · NFR-REL-001

---

### BR-057: Stok hanya berubah lewat stock_transaction
| Field | Value |
|-------|-------|
| **ID** | BR-057 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Integritas desain (konsisten dengan BR-042) |

**Rule:** Kolom `part.stok` tidak pernah diedit langsung — perubahan hanya melalui insert `stock_transaction` in/out dalam transaksi yang sama.
**When:** Kode mana pun ingin mengubah stok.
**Then:** Wajib membuat baris stock_transaction (audit trail).
**Else:** — (tidak ada jalur lain).
**Related:** UC-013, UC-015, UC-016 · FR-036

---

### BR-058: Koreksi stok manual = transaksi + keterangan
| Field | Value |
|-------|-------|
| **ID** | BR-058 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Question-framework (Admin/storekeeper) |

**Rule:** Stock opname/koreksi manual dibuat sebagai `stock_transaction` (in/out) tanpa `work_order_id`, wajib berketerangan, hanya role `storekeeper`/`admin`.
**When:** Selisih fisik stok ditemukan.
**Then:** Catat transaksi koreksi + alasan.
**Else:** —
**Related:** UC-006 · FR-011, FR-037

---

### BR-059: Transaksi stok masuk dari form terpisah
| Field | Value |
|-------|-------|
| **ID** | BR-059 |
| **Category** | Workflow |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Task description (transaksi keluar-masuk) |

**Rule:** Stok masuk (penerimaan part) dicatat via form "Stok Masuk": pilih part, qty > 0, keterangan → insert transaksi `in` → `stok bertambah`. Role `storekeeper`/`admin`.
**When:** Submit form stok masuk.
**Then:** Validasi qty > 0 → transaksi atomik.
**Else:** qty ≤ 0 → tolak.
**Related:** UC-015 · FR-038

---

### BR-060: Alert stok ≤ minimum
| Field | Value |
|-------|-------|
| **ID** | BR-060 |
| **Category** | Workflow |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Question-framework (min_stok) |

**Rule:** Part dengan `stok ≤ min_stok` ditandai di dashboard & daftar part (badge "Stok Menipis").
**When:** Query daftar part / dashboard.
**Then:** Tampilkan penanda bila kondisi terpenuhi.
**Else:** —
**Related:** UC-016, UC-020 · FR-037, FR-039

---

### BR-061: qty transaksi selalu positif
| Field | Value |
|-------|-------|
| **ID** | BR-061 |
| **Category** | Validation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | DATA-DICT stock_transaction |

**Rule:** `stock_transaction.qty > 0`; arah ditentukan kolom `tipe`, bukan nilai negatif.
**When:** Insert transaksi.
**Then:** Validasi > 0.
**Else:** Tolak.
**Related:** UC-013, UC-015 · FR-040

---

### BR-062: Pengambilan part hanya dari halaman WO
| Field | Value |
|-------|-------|
| **ID** | BR-062 |
| **Category** | Authorization |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek D (Interface 6W) |

**Rule:** Transaksi `out` dengan referensi WO hanya terjadi lewat tombol "Ambil Part" di detail WO (role storekeeper/foreman/admin). Transaksi `out` tanpa WO (konsumsi/hilang) = koreksi manual (BR-058).
**When:** Sumber transaksi out.
**Then:** Validasi jalur & role.
**Else:** —
**Related:** UC-013 · FR-028

---

### BR-063: Riwayat transaksi tidak bisa dihapus
| Field | Value |
|-------|-------|
| **ID** | BR-063 |
| **Category** | Constraint |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek B (semua permanen) |

**Rule:** Baris `stock_transaction` bersifat append-only — tidak ada fitur edit/hapus di UI v1.
**When:** Aksi hapus/edit transaksi.
**Then:** Tidak tersedia (fitur tidak dibuat).
**Else:** —
**Related:** UC-016 · FR-036

---

## BAB-6 — Dashboard (BR-076–085)

### BR-076: Jumlah alat per status = real-time
| Field | Value |
|-------|-------|
| **ID** | BR-076 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Task description (dashboard) |

**Rule:** Kartu dashboard menghitung `COUNT(equipment) GROUP BY status` saat halaman dimuat — tidak disimpan/di-cache.
**When:** Halaman dashboard dibuka.
**Then:** Query agregat live.
**Else:** —
**Related:** UC-017 · FR-042

---

### BR-077: Total downtime dari WO
| Field | Value |
|-------|-------|
| **ID** | BR-077 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Task description + BR-047 |

**Rule:** Total downtime = SUM(durasi WO) untuk rentang tanggal terpilih, mengikuti definisi BR-047.
**When:** Dashboard/laporan downtime.
**Then:** Agregasi per periode.
**Else:** —
**Related:** UC-018 · FR-043

---

### BR-078: Part paling sering dipakai
| Field | Value |
|-------|-------|
| **ID** | BR-078 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | Task description |

**Rule:** Top part = `stock_transaction` tipe=out dengan `work_order_id NOT NULL`, di-group per part, urut COUNT DESC, limit 5 (tampil qty total juga).
**When:** Dashboard.
**Then:** Query agregat.
**Else:** —
**Related:** UC-019 · FR-044

---

### BR-079: KPI safety terpisah per trigger_type
| Field | Value |
|-------|-------|
| **ID** | BR-079 |
| **Category** | Calculation |
| **Priority** | Must |
| **Status** | [x] Confirmed |
| **Source** | **Koreksi user 2026-09-25** — KPI beda dari 1 tabel |

**Rule:** "Berapa kali alat ditolak jalan karena P2H" = COUNT(WO trigger_type='p2h_gagal'); "berapa kali rusak saat operasi" = COUNT(trigger_type='breakdown_lapangan'). Dua angka, satu tabel WO.
**When:** Laporan safety.
**Then:** GROUP BY trigger_type.
**Else:** —
**Related:** UC-021 · FR-042, FR-045

---

### BR-080: Laporan hanya untuk Foreman/Admin
| Field | Value |
|-------|-------|
| **ID** | BR-080 |
| **Category** | Authorization |
| **Priority** | Should |
| **Status** | [x] Confirmed |
| **Source** | Question-framework Aspek C |

**Rule:** Halaman dashboard penuh (downtime, KPI, top part) hanya `foreman`/`admin`. Operator & storekeeper melihat ringkasan minimal (status alat saja).
**When:** Akses halaman laporan.
**Then:** Filter per role.
**Else:** 403 untuk bagian terlarang.
**Related:** UC-017–UC-021 · FR-046

---

## Rule Categories

| Code | Category | Description |
|------|----------|-------------|
| VAL | Validation | Validasi input |
| CALC | Calculation | Logika hitung |
| AUTH | Authorization | Hak akses |
| WF | Workflow | Transisi status |
| CON | Constraint | Batas keras |
| RET | Retention | Data lifecycle |
