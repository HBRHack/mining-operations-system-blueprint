# SPEC: SIMTAS — Simulasi Sistem Operasional Tambang (Loading–Hauling)

Aplikasi web latihan yang mensimulasikan integrasi 3 modul operasional tambang — monitoring alat (P2H + log shift), inventory spare part, dan maintenance Work Order — yang saling terhubung lewat state machine alat dan transaksi stok atomik. Dibangun sebagai portofolio sekaligus media belajar relasi many-to-many + state machine di Laravel.

> **Dokumen konsolidasi.** Setiap baris di dokumen ini adalah sintesis dari dokumen sumber; **tanpa informasi baru**. Sumber kebenaran tetap: `00-Global/REQUIREMENTS-MATRIX.md` (50 FR), `business-rules.md` (43 BR), `BAB-*/USE-CASE.md` (21 UC), `BAB-*/PROCESS-FLOW.md` (13 PF), `nfr.md` (8 NFR). **Bila ada konflik antara spec.md dan sumber → sumber menang.**

## Document Info

| Field | Value |
|-------|-------|
| **Version** | 1.0 |
| **Last Updated** | 2026-09-25 |
| **Status** | Draft — **SPEC-GATE 8/8 LULUS** (2026-09-25); menunggu approval |
| **Source** | analyst-grill-with-docs interview session (aspek A–G, terkunci 2026-09-25) + koreksi state machine 5-state (CHG-001) |
| **Related ADRs** | Belum ada ADR (belum ada `docs/adr/`); keputusan arsitektur tercatat di `assumptions-constraints.md` (CON-001…007) & `change-management.md` (CHG-001…003) |
| **Sumber sumbernya** | `docs/analyst/question-framework.md`, `00-Global/SRS-MASTER.md` |

---

## SPEC-GATE — Gerbang Validasi Spec

Spec ini WAJIB lolos 8 gate sebelum boleh dipakai sebagai dasar coding/ticket. Tiap gate punya **metode verifikasi yang bisa diulang** (script grep/awk — bukan opini).

| ID | Gate | Kriteria Lolos | Metode Verifikasi | Status |
|----|------|----------------|-------------------|--------|
| **SG-01** | FR coverage | Semua **50 FR** (FR-001…FR-050) muncul di ≥1 User Story | `grep -oE 'FR-[0-9]{3}' spec.md` = 50 unik == baris matrix | ☑ |
| **SG-02** | No orphan reference | Semua ID yang dirujuk spec (FR/UC/BR/PF/NFR/ASM/CON/CHG) **ada definisinya** di dokumen sumber | comm: `referenced − defined` = kosong | ☑ |
| **SG-03** | Count parity | Jumlah ID di spec tidak mengubah total sumber: **50 FR / 43 BR / 21 UC / 13 PF / 8 NFR** | Hitung otomatis vs matrix & business-rules | ☑ |
| **SG-04** | No Mermaid | 0 kemunculan diagram Mermaid (aturan permanen skill) | `grep -ri mermaid spec.md` (kecuali baris gate ini sendiri) = 0 | ☑ |
| **SG-05** | Testability | ≥ **10 Testing Decisions** (target NFR-TEST-001) — masing-masing menunjuk FR/BR | Hitung baris tabel test vs `nfr.md` §NFR-TEST-001 | ☑ |
| **SG-06** | Out-of-scope konsisten | Seksi Out of Scope identik dengan `SRS-MASTER.md §1.2` (explicitly out of scope) — tidak ada yang ditambah/dikurangi | Cross-check manual per baris + grep | ☑ |
| **SG-07** | Bahasa domain | Menggunakan kosakata `00-Global/GLOSSARY.md` (minimal: P2H, Breakdown, Rejected, Ambil Part, Downtime, trigger_type) | `grep` tiap istilah = ada | ☑ |
| **SG-08** | State machine cocok | Diagram state di spec = **5 state** persis seperti CHG-001/PF-003 (`idle, running, rejected, breakdown, maintenance`) — `p2h_ok` BUKAN state | Parse code-block `equipment.status ∈ {...}` = persis 5 nilai; `p2h_ok` 0 kemunculan **di dalam snippet** (kalimat `p2h_ok` = bukan state di luar snippet itu legal) | ☑ |

**Aturan main:** status semua gate harus ☑ sebelum Approval di-sign. Gagal → perbaiki spec (atau spec = konfirmasi bahwa sumber yang perlu diperbaiki — jangan diam-diam override).

---

## Problem Statement

Operasional tambang loading–hauling mengandalkan 3 laporan yang saling terpisah: kondisi alat harian (P2H & jam operasi), kartu stok gudang spare part, dan catatan perbaikan alat. Saat ini ketiganya dicatat terpisah (kertas/Excel), sehingga tiga masalah klasik muncul berulang: **(1)** alat bisa berangkat jalan tanpa P2H yang lolos — item keselamatan gagal tapi tetap beroperasi; **(2)** foreman tidak tahu part tersedia sampai WO gagal dikerjakan, dan stok gudang sering tidak sinkron dengan fisik; **(3)** downtime tidak pernah terukur rapi karena titik mulai (laporan breakdown) dan titik selesai (WO ditutup) tidak pernah dihubungkan. Akibatnya keputusan harian — alat mana yang jalan, part mana yang harus direstock, berapa downtime bulan ini — diambil dari ingatan, bukan data. *(Sumber: `SRS-MASTER.md §1.1`, question-framework Aspek A/B.)*

Solusi yang diminta adalah simulasi web satu-satunya: satu database di mana **P2H yang gagal kritis mengunci alat dan otomatis melahirkan Work Order**, **Work Order yang memakai part mengurangi stok gudang secara atomik**, dan **dashboard menampilkan angka segar dari ketiganya** — sehingga ketiga masalah di atas terjawab oleh sistem yang sama, bukan tiga laporan terpisah. *(Sumber: `SRS-MASTER.md §1.2` in-scope.)*

---

## Solution

Dari sudut pandang pengguna: aplikasi web form-based Bahasa Indonesia dengan 4 peran. **Operator** mengisi P2H sebelum jalan (item safety-critical gagal = alat dikunci `rejected` + WO lahir otomatis), mencatat jam operasi per shift, dan melapor breakdown. **Foreman** memantau board WO, mengklaim & menutup WO — begitu WO ditutup, status alat kembali `running`/`idle` otomatis mengikuti jadwal shift. **Storekeeper** mencatat stok masuk, mengambil part untuk WO (potong stok atomik, tidak boleh minus), dan memantau badge "Stok Menipis". **Admin** mengelola master alat/part/user dan bisa "act as" role lain untuk demo. Semua role melihat dashboard: jumlah alat per status, downtime, top part, KPI safety — real-time, tanpa cache. *(Rujukan: `SRS-MASTER.md §1.2–1.3`, `BAB-4/PROCESS-FLOW.md` PF-003.)*

---

## User Stories

Format: `Sebagai <actor>, saya ingin <fitur>, agar <manfaat>.` — actors dari `raci.md` Stakeholder Directory; istilah dari `GLOSSARY.md`. Tiap story **wajib rujuk FR (wajib), UC, dan BR** — rujukan adalah jembatan ke dokumen sumber, bukan pengganti.

> **Konvensi penomoran:** US-001…US-048 urut per chapter (Global sequential). Gate **SG-01** memverifikasi semua 50 FR tercakup.

### BAB-1 — Auth & Peran

1. **US-001** — Sebagai semua role, saya ingin login dengan email & password, agar data operasional tambang tidak bisa diakses sembarang orang. *(FR-001, FR-005 · UC-001 · BR-001, BR-005)*
2. **US-002** — Sebagai sistem, saya ingin hak akses mengikuti 4 role enum, agar operator tidak bisa membuka halaman kelola master. *(FR-002, FR-004 · UC-001 · BR-002)*
3. **US-003** — Sebagai Admin, saya ingin "act as" role lain, agar bisa demo alur foreman/operator tanpa ganti akun. *(FR-003 · UC-002 · BR-002)*
4. **US-004** — Sebagai Admin, saya ingin menambah/mengubah/nonaktifkan user, agar peran di lapangan tergambar benar di sistem. *(UC-003 · BR-003, BR-004 — tanpa FR khusus, tercakup FR-002)*

### BAB-2 — Master Data

5. **US-005** — Sebagai Admin, saya ingin CRUD master alat (tipe, model, kapasitas), agar daftar fleet lengkap sebelum musim operasi. *(FR-012 · UC-004 · BR-011…BR-013)*
6. **US-006** — Sebagai Admin, saya ingin unit_code tervalidasi unik & berpola `(EX|DT|DZ|WT)-NN`, agar laporan tidak tercampur antar unit. *(FR-006 · UC-004 · BR-011)*
7. **US-007** — Sebagai sistem, saya ingin alat baru selalu berstatus `idle`, agar tidak ada alat lahir dalam kondisi siap jalan tanpa P2H. *(FR-007 · UC-004 · BR-012)*
8. **US-008** — Sebagai Admin, saya ingin alat ber-riwayat tidak bisa dihapus keras, agar riwayat WO & log shift tidak jadi yatim. *(FR-008 · UC-004 · BR-013)*
9. **US-009** — Sebagai Admin/Storekeeper, saya ingin CRUD master part, agar katalog spare part lengkap. *(FR-049 · UC-005 · BR-014)*
10. **US-010** — Sebagai sistem, saya ingin kode part unik + enum kategori/satuan tervalidasi, agar pencarian & agregasi part konsisten. *(FR-009 · UC-005 · BR-014)*
11. **US-011** — Sebagai sistem, saya ingin part baru lahir dengan stok 0, agar tidak ada stok "langsung penuh" tanpa jejak transaksi. *(FR-010 · UC-005 · BR-015)*
12. **US-012** — Sebagai Storekeeper, saya ingin mengoreksi stok manual (stock opname) sebagai transaksi berketerangan, agar stok fisik & sistem selalu bisa disamakan. *(FR-011 · UC-006 · BR-056, BR-058, BR-061)*

### BAB-3 — Monitoring Harian (P2H & Log Shift)

13. **US-013** — Sebagai Operator, saya ingin P2H hari ini harus `pass` sebelum alat bisa `idle → running`, agar alat tidak jalan tanpa pemeriksaan keselamatan. *(FR-013 · UC-007 · BR-021)*
14. **US-014** — Sebagai sistem, saya ingin P2H dibatasi 1× per alat per hari, agar checklist tidak bisa diulang sampai lolos. *(FR-014 · UC-007 · BR-022)*
15. **US-015** — Sebagai sistem, saya ingin item safety-critical (oli, rem, hidrolik) yang gagal menolak jalan & auto-buat WO, agar kerusakan kritis tertangani sebelum beroperasi. *(FR-015 · UC-007, UC-009 · BR-023, BR-036)*
16. **US-016** — Sebagai Operator, saya ingin kegagalan item non-kritis hanya jadi warning tetap `pass`, agar P2H tidak menghalangi operasi untuk hal remeh. *(FR-016 · UC-008 · BR-024)*
17. **US-017** — Sebagai Operator, saya ingin mengisi log shift dengan jam mulai–selesai (lintas hari dihitung benar), agar jam operasi tercatat akurat. *(FR-017 · UC-010 · BR-025)*
18. **US-018** — Sebagai sistem, saya ingin hour meter bertambah otomatis = jam operasi, agar HM kumulatif tanpa input ganda. *(FR-018 · UC-010 · BR-026)*
19. **US-019** — Sebagai sistem, saya ingin `alasan_idle` wajib saat jam operasi 0, agar alat menganggur punya sebab tercatat. *(FR-019 · UC-010 · BR-027)*
20. **US-020** — Sebagai sistem, saya ingin menolak log shift duplikat per (alat, shift, tanggal), agar rekap jam operasi tidak terhitung dobel. *(FR-020 · UC-010 · BR-028)*
21. **US-021** — Sebagai Admin/Foreman, saya ingin halaman daftar alat + filter status, agar bisa memindai kondisi fleet tanpa membuka dashboard. *(FR-048 · UC-008 · BR-076)*

### BAB-4 — Work Order

22. **US-022** — Sebagai sistem, saya ingin WO lahir otomatis tanpa approval saat P2H gagal kritis, agar perbaikan langsung punya wadah. *(FR-021 · UC-009 · BR-036)*
23. **US-023** — Sebagai Operator, saya ingin melapor breakdown lewat 1 form sehingga WO terbentuk otomatis, agar laporan lapangan langsung jadi pekerjaan. *(FR-022 · UC-009 · BR-036)*
24. **US-024** — Sebagai sistem, saya ingin status WO hanya mengalir `open → menunggu_part → in_progress → closed`, agar riwayat perbaikan tidak melompat. *(FR-023 · UC-011 · BR-037)*
25. **US-025** — Sebagai sistem, saya ingin alat berstatus `maintenance` seluruh fase WO, agar alat rusak tidak muncul sebagai "siap jalan". *(FR-024 · UC-011 · BR-038)*
26. **US-026** — Sebagai sistem, saya ingin asal WO hanya disimpan di `trigger_type` (`p2h_gagal` / `breakdown_lapangan`), agar 2 KPI safety terbaca dari 1 tabel tanpa duplikasi state. *(FR-025 · UC-009 · BR-039)*
27. **US-027** — Sebagai sistem, saya ingin maksimal 1 WO terbuka per alat, agar dua jalur perbaikan tidak balapan di unit yang sama. *(FR-026 · UC-009 · BR-040)*
28. **US-028** — Sebagai Foreman, saya ingin memisahkan rencana part dari transaksi stok, agar menyusun rencana tidak ikut memotong stok. *(FR-027 · UC-013 · BR-041)*
29. **US-029** — Sebagai Storekeeper, saya ingin "Ambil Part" dieksekusi atomik (validasi → transaksi → kurangi stok → tandai `taken`), agar tidak pernah ada stok terpotong sebagian. *(FR-028 · UC-013 · BR-042, BR-057)*
30. **US-030** — Sebagai sistem, saya ingin menolak pengambilan bila qty > stok dengan rollback total, agar stok gudang tidak pernah minus. *(FR-029 · UC-013 · BR-056)*
31. **US-031** — Sebagai sistem, saya ingin WO berpindah ke `menunggu_part` begitu ada rencana part yang stoknya kurang, agar foreman segera tahu hambatan. *(FR-030 · UC-013 · BR-043)*
32. **US-032** — Sebagai Foreman, saya ingin close WO ditolak selama masih ada part `planned`, agar tidak ada pekerjaan "ditutup" padahal belum lengkap. *(FR-031 · UC-014 · BR-044)*
33. **US-033** — Sebagai sistem, saya ingin status alat setelah close WO ditentukan otomatis dari jam vs jadwal shift (`running` bila shift masih jalan, selain itu `idle`), agar pengembalian alat tidak perlu tebak manual. *(FR-032 · UC-014 · BR-045)*
34. **US-034** — Sebagai Foreman, saya ingin hanya foreman/admin yang boleh klaim, ubah prioritas, dan menutup WO, agar urusan perbaikan tidak diutak-atik role lain. *(FR-033, FR-050 · UC-011 · BR-046)*
35. **US-035** — Sebagai Foreman, saya ingin board daftar WO aktif, agar semua pekerjaan terbuka terpantang dalam satu layar. *(FR-047 · UC-011 · BR-046)*
36. **US-036** — Sebagai Admin, saya ingin downtime WO dihitung `closed_at − detected_at` (berjalan bila belum closed), agar KPI downtime punya rumus tunggal. *(FR-034 · UC-012 · BR-047)*
37. **US-037** — Sebagai sistem, saya ingin breakdown < 30 menit dicatat idle biasa tanpa WO, agar gangguan sepele tidak membanjiri papan perbaikan. *(FR-041 · UC-009 · BR-047)*

### BAB-5 — Inventory

38. **US-038** — Sebagai sistem, saya ingin setiap transaksi out menjamin stok ≥ 0 (check + rollback), agar saldo gudang tidak pernah negatif. *(FR-035 · UC-015 · BR-056)*
39. **US-039** — Sebagai sistem, saya ingin stok hanya boleh berubah lewat `stock_transaction` append-only, agar setiap perubahan punya jejak pelaku & alasan. *(FR-036 · UC-016 · BR-057, BR-063)*
40. **US-040** — Sebagai Storekeeper, saya ingin form stok masuk (qty > 0 + keterangan), agar penerimaan part dari supplier tercatat. *(FR-038 · UC-015 · BR-059, BR-061)*
41. **US-041** — Sebagai Storekeeper/Foreman, saya ingin kartu stok & riwayat transaksi per part, agar pergerakan tiap part bisa ditelusuri. *(FR-039 · UC-016 · BR-063)*
42. **US-042** — Sebagai Storekeeper, saya ingin badge "Stok Menipis" saat stok ≤ min_stok (di daftar part & kartu stok), agar pengadaan bisa dicegat dini. *(FR-037 · UC-016, UC-020 · BR-060)*
43. **US-043** — Sebagai sistem, saya ingin qty semua transaksi selalu > 0, agar tidak ada transaksi nol/negatif yang merusak aritmetika stok. *(FR-040 · UC-015 · BR-061)*

### BAB-6 — Dashboard & Laporan

44. **US-044** — Sebagai semua role, saya ingin melihat jumlah alat per status dari query live, agar keadaan fleet terbaru tanpa refresh manual. *(FR-042 · UC-017 · BR-076)*
45. **US-045** — Sebagai Foreman/Admin, saya ingin total downtime per rentang tanggal, agar tren ketersediaan alat terukur. *(FR-043 · UC-018 · BR-047)*
46. **US-046** — Sebagai Foreman/Admin, saya ingin 5 part paling sering dipakai, agar perencanaan stok berbasis pemakaian nyata. *(FR-044 · UC-019 · BR-078)*
47. **US-047** — Sebagai Foreman/Admin, saya ingin KPI safety terpisah per `trigger_type` (P2H gagal vs breakdown lapangan), agar dua penyebab gangguan tidak tertumpuk jadi satu angka. *(FR-045 · UC-021 · BR-039)*
48. **US-048** — Sebagai sistem, saya ingin dashboard penuh hanya untuk foreman/admin (operator/storekeeper dapat ringkasan), agar tampilan sesuai tanggung jawab (BR-080). *(FR-046 · UC-017 · BR-080)*

---

## Implementation Decisions

Keputusan implementasi — tanpa path file / snippet kode (mengikuti aturan format; snippet state machine di bawah diizinkan karena meng-encode keputusan yang persis).

### ID-01 — Arsitektur stack & constraint

- **Keputusan:** Laravel 12 + PHP 8.3 + MySQL/MariaDB; Blade + JS minimal (HTMX hanya bila perlu); **tanpa** Livewire/Filament/SPA berat; CRUD manual; UI Bahasa Indonesia, WITA, `d/m/Y`. *(CON-001…CON-006 · `assumptions-constraints.md` · SRS §6)*
- **Mendukung:** US-001…US-048 (seluruhnya — semua story adalah form-based di stack ini).
- **Catatan gate:** ASM-001 (PHP ≥ 8.2 di mesin dev) masih **Open** — wajib diverifikasi `php -v` sebagai langkah pertama eksekusi (lihat `change-management.md`/SRS §7).

### ID-02 — State machine 5 state (keputusan CHG-001 — JANGAN diganggu)

- **Keputusan:** Enum status alat persis 5 nilai; `p2h_ok` **bukan** state (hanya `p2h_checklist.hasil` pass/fail); asal kerusakan disimpan di `work_orders.trigger_type`. Snippet meng-encode keputusan (sumber: PF-003, `change-management.md` CHG-001):

```
equipment.status  ∈ { idle, running, rejected, breakdown, maintenance }

  idle ──(P2H pass, FR-013)──────────────► running
  idle ──(P2H gagal safety-critical,
          FR-015/BR-023)────────────────► rejected ──(auto-WO, FR-021)──┐
  running ─(breakdown >30 mnt, FR-022)───► breakdown ─(auto-WO)──────────┤
                                                                         ▼
                                        maintenance ◄────────────── (WO open)
                                             │
                                             └─(WO closed, FR-032/BR-045)─►
                                                    running (shift masih jalan)
                                                  | idle    (shift habis)

  gagal NON-kritis ──► tetap pass (warning) — FR-016
  breakdown < 30 mnt ──► idle biasa, tanpa WO — FR-041
```

- **Mendukung:** US-013, US-015, US-016, US-022, US-023, US-025, US-026, US-033, US-037.
- **Dilarang:** menambah state ke-6 (mis. `p2h_ok`) = pelanggaran CHG-001 → wajib lewat change-management.

### ID-03 — Maks 1 WO terbuka + WO auto tanpa approval

- **Keputusan:** Constraint unique/cek pada WO terbuka per alat (BR-040); WO auto dibuat `status=open`, `prioritas=high`, `assigned_to=null`, approval dilewati (BR-036). *(SRS §7 ASM-003 = Verified)*
- **Mendukung:** US-022, US-023, US-027.

### ID-04 — Integritas stok: satu jalur, atomik, append-only

- **Keputusan:** `part.stok` hanya berubah di dalam transaksi DB yang juga INSERT `stock_transaction` (BR-057); cek `stok − qty ≥ 0` sebelum commit (BR-056); rencana part di tabel `work_order_part` terpisah dari transaksi (BR-041); "Ambil Part" = 1 transaksi DB untuk semua item (BR-042); koreksi stok = transaksi `work_order_id=NULL` berketerangan (BR-058). *(NFR-REL-001 · PF-004 · PF-009 · PF-010 · PF-012)*
- **Mendukung:** US-012, US-028, US-029, US-030, US-038, US-039, US-040, US-043.

### ID-05 — WO status flow & close ceremony

- **Keputusan:** Transisi tertutup `open → menunggu_part → in_progress → closed` (BR-037); close hanya foreman/admin (BR-046); close wajib semua part `taken` (BR-044); hasil close menentukan status alat dari jam vs jadwal shift (BR-045). *(PF-005 · UC-014)*
- **Mendukung:** US-024, US-032, US-033, US-034.

### ID-06 — KPI dari 1 tabel: `trigger_type` sebagai pemisah

- **Keputusan:** Asal WO **hanya** di `work_orders.trigger_type` (`p2h_gagal` | `breakdown_lapangan`, BR-039); KPI safety & filter laporan di-group dari kolom ini — bukan kolom flag terpisah, bukan state tambahan. *(BR-039 · CHG-001 · PF-013)*
- **Mendukung:** US-026, US-047.

### ID-07 — Dashboard live tanpa cache

- **Keputusan:** Semua angka dashboard dihitung dari query agregat per request (BR-076) — tanpa tabel cache; kuncinya eager loading / GROUP BY, bukan cache (NFR-PERF-001 < 2 detik). Filter role per BR-080. *(PF-013)*
- **Mendukung:** US-044, US-045, US-046, US-048.

### ID-08 — Otorisasi: role enum + middleware + act-as

- **Keputusan:** 4 role enum (FR-004), semua route terproteksi middleware (BR-001), act-as = kolom `users.act_as` (bukan sesi ganda), password ≥ 8 hash bcrypt/argon2 (BR-005). *(PF-006)*
- **Mendukung:** US-001, US-002, US-003, US-004.

### ID-09 — Skema data: 10 entitas sumber kebenaran

- **Keputusan:** Skema mengikuti `00-Global/ERD-MASTER.md` (10 entitas) + validasi per `DATA-DICT-MASTER.md` (9 value list). Relasi many-to-many `work_order ↔ part` lewat pivot `work_order_part` (ber-`status planned/taken` — inti latihan). Tidak ada field redundan antar tabel terkait (Validation Gate Check 1 lolos). *(ERD-MASTER · DATA-DICT-MASTER)*
- **Mendukung:** US-005…US-011 (master), US-028…US-031 (pivot), US-039…US-041 (transaksi).

### ID-10 — Data fiktif & seeder realistis

- **Keputusan:** Semua data 100% fiktif (CON-005); seeder ≥ 5 alat, ≥ 10 part (target realistis 15), 2 shift × 30 hari log, 10–15 WO historis, 10 item P2H (3 safety-critical) — diminta user sebagai tahap 2. *(SRS §6 CON-005 · question-framework Aspek F)*
- **Mendukung:** seluruh UAT demo (US-001…US-048) membutuhkan data awal.

---

## Testing Decisions

**Definisi test yang baik** (NFR-TEST-001): menguji **perilaku eksternal** — input → hasil teramati (halaman/DB state), bukan detail implementasi internal. Semua test menunjuk FR/BR eksplisit agar traceability dua arah terjaga (format: FR yang diuji).

**Modul yang diuji:** logika bisnis inti (state machine, transaksi stok atomik, otorisasi, aturan close) — bukan tampilan Blade. **Prior art:** test khas Laravel Feature Test (HTTP + database transaction assertion, `RefreshDatabase`).

**Target:** ≥ 10 test (NFR-TEST-001) — tabel di bawah berisi **14** kandidat, diprioritaskan berdasarkan (business value × technical risk) ala utility tree.

| # | Test (skenario → ekspektasi) | Rujukan | Kelas |
|---|------------------------------|---------|-------|
| T-01 | P2H gagal item safety-critical → alat `rejected` + WO baru terbentuk `open/high/null-assignee` | FR-015, FR-021, FR-025 · BR-023, BR-036 | Feature |
| T-02 | P2H gagal item **non-kritis** → alat TETAP bisa `running` (hanya warning) | FR-016 · BR-024 | Feature |
| T-03 | `idle → running` tanpa P2H hari ini → **ditolak** | FR-013 · BR-021 | Feature |
| T-04 | WO kedua dibuat saat masih ada WO terbuka → **ditolak** (maks 1 WO) | FR-026 · BR-040 | Feature |
| T-05 | Ambil Part qty > stok → seluruh item rollback, stok tak berubah, tak ada baris transaksi yatim | FR-029, FR-035 · BR-056, BR-042 | Feature (NFR-REL-001) |
| T-06 | Ambil Part 2 item sukses → 2 baris `stock_transaction` out + stok berkurang + part `taken` dalam 1 commit | FR-028 · BR-042, BR-057 | Feature (NFR-REL-001) |
| T-07 | Close WO dengan 1 part masih `planned` → **ditolak** | FR-031 · BR-044 | Feature |
| T-08 | Close WO sukses saat shift berjalan → alat `running`; saat shift habis → alat `idle` | FR-032 · BR-045 | Feature |
| T-09 | Operator coba tutup WO → **403** (hanya foreman/admin) | FR-033, FR-002 · BR-046, BR-002 | Feature (NFR-SEC-001) |
| T-10 | Koreksi stok out melebihi stok → ditolak, tidak ada baris transaksi, stok tidak berubah | FR-011, FR-035 · BR-056, BR-058 | Feature |
| T-11 | Log shift duplikat (alat, shift, tanggal) → ditolak | FR-020 · BR-028 | Feature |
| T-12 | Breakdown < 30 menit → TIDAK ada WO, alat kembali/log idle | FR-041 · BR-047 | Feature |
| T-13 | KPI safety: seed WO `p2h_gagal` + `breakdown_lapangan` → dashboard menampilkan 2 angka terpisah sesuai `trigger_type` | FR-045 · BR-039 | Feature |
| T-14 | Akses halaman proteksi tanpa sesi → redirect `/login?redirect=…` | FR-001 · BR-001 | Feature (NFR-SEC-001) |

**Alokasi NFR ke test:** NFR-REL-001 → T-05, T-06 · NFR-SEC-001 → T-09, T-14 · NFR-PERF-001 → micro-benchmark dashboard manual (bukan unit test) · NFR-USAB-001 → review label UI manual · NFR-TEST-001 → test suite ini sendiri (≥ 10) · NFR-PORT-001 → jalankan suite di mesin fresh · NFR-MAINT-001 → cek komentar modul state machine & stok · NFR-SEC-002 → data fiktif (konfirmasi seeder, CON-005).

---

## Out of Scope

*(Identik dengan `SRS-MASTER.md §1.2 "Explicitly out of scope"* — **SG-06**, tidak boleh ditambah/dikurangi di spec ini.)*

- Sambungan eksternal: GPS alat berat, fuel system, HRD/absensi, notifikasi WA/SMS/email, API pihak ketiga.
- Ekspor laporan ke Excel/PDF (v2).
- Procurement/pembelian part, faktur, accounting.
- Multi-cabang/multi-tambang, multi-tenant.
- Fitur keamanan lanjutan: 2FA, email verification, audit log formal.
- Pelaporan breakdown manual via WhatsApp/kertas (disederhanakan jadi 1 form digital).

---

## Further Notes

- **Asumsi terbuka (wajib dicek saat mulai coding):** ASM-001 (PHP ≥ 8.2 — gate pertama eksekusi), ASM-002 (P2H 1×/hari). Sisanya Verified. *(Sumber: `assumptions-constraints.md` §Risk Register.)*
- **Constraint keras:** CON-001…CON-006 Hard; CON-007 (deploy) Soft. *(Sumber: `assumptions-constraints.md`.)*
- **Riwayat perubahan requirement:** CHG-001 (state machine 5-state, Major), CHG-002 (renumber FR-049/050, Minor), CHG-003 (UC-020 + rujukan, Moderate). Spec ini mencerminkan **v1.1** requirement. *(Sumber: `change-management.md`.)*
- **Tidak ada open question** — aspek A–G terkunci 2026-09-25 (SRS §8).
- **Dependency:** seeder realistis (tahap 2) dibutuhkan sebelum UAT bermakna; ≥10 test (tahap 4) sebelum dashboard (tahap 5).
- **Risk items:** lihat Risk Register di `assumptions-constraints.md` (4 risiko, mayoritas Low/High-impact).

---

## Traceability Matrix

> FR lengkap per story ada di kolom rujukan masing-masing US di atas (diparse otomatis oleh SG-01). Tabel ini memetakan **US → keputusan implementasi → test** per blok (mengelompokkan story sechapter agar terbaca; per-FR detail = `REQUIREMENTS-MATRIX.md`).

| User Story (blok) | FR | Implementation Decision | Testing Decision |
|--------------------|----|-------------------------|------------------|
| US-001…US-004 (Auth) | FR-001…FR-005 | ID-08 | T-09, T-14 |
| US-005…US-012 (Master) | FR-006…FR-012, FR-049 | ID-09, ID-10 | T-10 |
| US-013…US-016 (P2H gate) | FR-013…FR-016 | ID-02 | T-01, T-02, T-03 |
| US-017…US-021 (Log shift) | FR-017…FR-020, FR-048 | ID-09 | T-11 |
| US-022…US-027 (WO auto & state) | FR-021…FR-026 | ID-02, ID-03, ID-06 | T-01, T-04 |
| US-028…US-031 (Ambil part) | FR-027…FR-030 | ID-04 | T-05, T-06 |
| US-032…US-035 (Close & board) | FR-031…FR-033, FR-047, FR-050 | ID-05 | T-07, T-08, T-09 |
| US-036…US-037 (Downtime & <30m) | FR-034, FR-041 | ID-06 | T-12 |
| US-038…US-043 (Inventory) | FR-035…FR-040 | ID-04 | T-05, T-10 |
| US-044…US-048 (Dashboard) | FR-042…FR-046 | ID-06, ID-07 | T-13 |

**Cakupan dua arah (wajib):** 50 FR → tercakup US mana (SG-01) · 48 US → tanpa FR yatim (SG-02) · 10 keputusan implementasi → minimal 1 US & ≥1 test mengacu.

---

## Approval

| Role | Name | Date | Status |
|------|------|------|--------|
| Product Owner (= Pemilik proyek, SH-001) | — | — | Pending — menunggu SG-01…SG-08 ☑ |
| Tech Lead (= Dev, SH-001) | — | — | Pending |
| QA Lead (= Tester, SH-001) | — | — | Pending |

> **Prasyarat sign-off:** tabel SPEC-GATE di atas semua ☑ + `change-management.md` v1.1 direview user. Kegagalan gate = spec belum boleh dipakai dasar ticket/coding.
