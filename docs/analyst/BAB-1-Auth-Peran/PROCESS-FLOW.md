# Process Flows — BAB-1 Auth & Peran

Proses inti chapter ini: masuk ke sistem dengan kredensial, dan Admin meniru peran lain untuk demo. Sisanya (CRUD pengguna) mengikuti pola CRUD umum — detail di UC-003.

## PF-006: Alur Login, Sesi & Act-As

Proses autentikasi pengguna dan pengalihan peran sementara oleh Admin.

**Trigger:** Pengguna membuka halaman mana pun dalam aplikasi.

**Goal:** Sesi aktif dengan peran yang benar — atau Admin berjalan sebagai role lain untuk demo.

**Actors:**
- Pengguna: login dengan email+password
- Admin: mengaktifkan act-as role lain

### Flow

```
START
  |
  v
[1] Pengguna buka halaman
  |
  v
[2] System cek sesi aktif?
  |
  +--- TIDAK ---> [3] Redirect /login (+ ?redirect= tujuan)
  |                    |
  |                    v
  |                 [4] Pengguna isi email + password
  |                    |
  |                    v
  |                 [5] System validasi (BR-005: password >= 8,
  |                    |                hash bcrypt/argon2 cocok?)
  |                    |
  |                    +--- SALAH --> [6] Pesan generik "Email atau
  |                    |                 password salah" (jangan
  |                    |                 bocorkan email ada/tidak)
  |                    |                    |
  |                    |                    +-----> kembali [4]
  |                    |
  |                    +--- BENAR --> [7] Buat sesi; users.act_as = NULL
  |
  +--- YA ----+
              |
              v
[8] Pilih halaman berdasarkan role (BR-002)
  |
  v
[9] Ada route terproteksi?
  |
  +--- YA, role cocok -------> [10] Render halaman ----> END
  |
  +--- YA, role tidak cocok -> [11] 403 / redirect halaman role ----> END
  |
  v
=== CABANG ACT-AS (Admin) ===
[12] Admin klik "Act As" -> pilih role target
  |
  v
[13] System simpan users.act_as = role_target
  |
  v
[14] Semua request berikutnya DIPERLUKAN pakai role_target
  |
  v
[15] Admin klik "Kembali ke Peran Asli"
  |
  v
[16] users.act_as = NULL ----> END
```

### Catatan Implementasi

| Aspek | Nilai |
|-------|-------|
| State | Sesi login + kolom `users.act_as` (bukan sesi terpisah) |
| Pesan error | Generik — anti user enumeration (BR-005) |
| Route protection | Middleware global: tanpa sesi → `/login` |
| BR terkait | BR-001 (semua halaman wajib login), BR-002 (akses per role), BR-005 (password) |
| UC terkait | UC-001, UC-002 |

---

## PF-007: Alur Kelola Data Pengguna (Admin)

Proses CRUD user & peran — mengikuti pola CRUD umum chapter lain, dicatat ringkas di sini agar chapter punya alur utuh.

**Trigger:** Admin membuka menu "Pengguna".

**Goal:** User baru siap login; peran & status akun terkini.

**Actors:**
- Admin: pelaku CRUD

### Flow

```
START
  |
  v
[1] Admin buka daftar pengguna
  |
  v
[2] Aksi?
  |
  +--- TAMBAH ---> [3] Form (nama, email, password >= 8, role)
  |                    |
  |                    v
  |                 [4] Validasi: email unik? role valid?
  |                    |
  |                    +--- GAGAL --> tampilkan error ----> kembali [3]
  |                    |
  |                    +--- OK ----> [5] Insert users (hash password)
  |                                     |
  |                                     v
  |                                  [6] Tampilkan baris baru ----> END
  |
  +--- UBAH ----> [3b] Form edit (role / nama / reset password)
  |                    |
  |                    v
  |                 [5b] Update users ----> END
  |
  +--- NONAKTIFKAN ---> [7] Validasi: bukan akun aktif sendiri?
  |                        |
  |                        +--- YA, diri sendiri --> tolak ----> END
  |                        |
  |                        +--- BUKAN --> [8] Update status aktif ----> END
  |
  +--- HAPUS ---> (TIDAK DIADAKAN — nonaktifkan saja, demi jejak audit)
```

### Catatan Implementasi

| Aspek | Nilai |
|-------|-------|
| Tidak ada hard delete | User dinonaktifkan — log shift/WO miliknya harus tetap terbaca |
| BR terkait | BR-003, BR-004 (siapa boleh kelola user), BR-005 |
| UC terkait | UC-003 |
