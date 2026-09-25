# Use Cases — BAB-1 Auth & Peran

## UC-001: Login dan Akses Halaman Terproteksi

Pengguna masuk ke aplikasi dengan email+password; seluruh halaman wajib login.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Pengguna (semua role) | admin/foreman/operator/storekeeper | Masuk & mengakses fitur sesuai perannya |

## Preconditions

- User sudah ada di tabel `users` (dibuat Admin / diseed)
- Belum ada sesi aktif

## Postconditions

- Sesi aktif; `users.act_as` null (peran asli)
- Halaman sesuai role terbuka

## Main Flow

1. **Pengguna** membuka halaman mana pun
2. **System** cek sesi → tidak ada → redirect ke `/login`
3. **Pengguna** isi email + password, tekan "Masuk"
4. **System** validasi kredensial (BR-005: password ≥ 8, hash match)
5. **System** buat sesi, arahkan ke dashboard sesuai role

## Alternative Flows

### AF-1: Password salah
- **Trigger:** Email ada, password tidak cocok
- **Step 4a:** System tampilkan "Email atau password salah" (pesan generik — jangan bocorkan email ada/tidak)
- **Step 4b:** Form ditampilkan ulang, percobaan tetap boleh

## Exception Flows

### EF-1: Akses route tanpa login
- **Trigger:** Request langsung ke halaman proteksi tanpa sesi
- **Step X:** System redirect ke `/login` dengan `?redirect=` kembali setelah sukses

## Business Rules

- **BR-001:** Semua halaman wajib login
- **BR-002:** Hak akses mengikuti role
- **BR-005:** Password ≥ 8, hash bcrypt/argon2

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| users | Read | Cek email + password hash |

---

## UC-002: Act-As Role Lain (Admin)

Admin meniru peran lain untuk keperluan demo cepat tanpa ganti akun.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | admin | Melihat aplikasi persis seperti role lain |

## Preconditions

- Sudah login sebagai `admin`
- Target role bukan `admin` (atau kembali ke asli)

## Postconditions

- `users.act_as` = target role (atau null saat kembali)
- Seluruh hak akses mengikuti peran yang ditiru

## Main Flow

1. **Admin** pilih menu "Act as" → pilih role (mis. `storekeeper`)
2. **System** validasi pemilik role = admin (BR-003)
3. **System** set `act_as = storekeeper`, sesi memakai hak storekeeper
4. **Admin** navigasi fitur → perlakuan seperti storekeeper
5. **Admin** tekan "Kembali ke Admin"
6. **System** set `act_as = null`, hak kembali penuh

## Exception Flows

### EF-1: Non-admin mengirim act_as
- **Trigger:** Request `act_as` dari role lain
- **Step X:** System tolak (403), `act_as` tidak berubah

## Business Rules

- **BR-002:** Hak akses per role
- **BR-003:** Act-as hanya Admin; selama aktif ikuti target role
- **BR-004:** Role hanya 4 nilai

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| users | Update | Kolom act_as |

---

## UC-003: Kelola Data Pengguna

Admin menambah/mengubah user & password untuk keperluan demo.

## Actors

| Actor | Role | Goal |
|-------|------|------|
| Admin | admin | Menyiapkan akun tiap peran |

## Preconditions

- Login sebagai `admin`

## Postconditions

- User baru/terubah tersimpan dengan role valid & password ter-hash

## Main Flow

1. **Admin** buka halaman Pengguna → "Tambah"
2. **Admin** isi nama, email, role, password (≥ 8)
3. **System** validasi unique email + enum role (BR-004) + hash password (BR-005)
4. **System** simpan user
5. **System** tampilkan daftar user ter-update

## Alternative Flows

### AF-1: Email sudah dipakai
- **Trigger:** Email duplikat
- **Step 3a:** System tolak dengan pesan "Email sudah terdaftar"

## Business Rules

- **BR-004, BR-005**

## Data Entities

| Entity | Action | Details |
|--------|--------|---------|
| users | Create/Update | name, email, role, password |

## Notes

- Seed awal menyediakan 1 user per role (lihat BAB-1/SEED-TODO) — halaman ini untuk penambahan sesudahnya.
