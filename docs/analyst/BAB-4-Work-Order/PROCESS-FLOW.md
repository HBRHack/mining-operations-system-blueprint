# Process Flows — BAB-4 Work Order

Jantung integrasi 3 modul: state machine alat 5 status + alur part yang menggerakkan stok inventory.

## PF-003: State Machine Alat End-to-End (5 State)

**Trigger:** P2H, laporan breakdown, pengambilan part, penutupan WO.
**Goal:** Status alat selalu mencerminkan kenyataan lapangan — tanpa dead-end state.

**Actors:**
- Operator: P2H & lapor breakdown
- Foreman: klaim & tutup WO
- Storekeeper: ambil part
- Sistem: memvalidasi & mensinkronkan state

### Flow

```
                         ┌───────────────────────────────────────────┐
                         │            STATE TERSIMPAN (enum)         │
                         │  idle | running | rejected | breakdown    │
                         │             | maintenance                 │
                         └───────────────────────────────────────────┘

   ┌──────┐  P2H pass + Mulai Operasi   ┌─────────┐
   │ idle │ ───────────────────────────►│ running │◄────────────┐
   └──┬───┘                             └──┬──────┘             │
      │ P2H gagal                          │                    │ close WO,
      │ safety-critical                    │ breakdown lapangan │ masih shift
      │ (BR-023)                           │ (>30 mnt, FR-041)  │ (BR-045)
      v                                    v                    │
 ┌──────────┐  auto-WO (BR-036)     ┌───────────┐               │
 │ rejected │ ──── (transisi ─────► │ breakdown │──┐            │
 └──────────┘      singkat)         └───────────┘  │            │
      │                            auto-WO (BR-036)│            │
      │ WO gagal dibuat:                            │            │
      │ STUCK di rejected (fail-safe)               │            │
      │ + error Foreman                             │            │
      └──────────────────────────────┬──────────────┘            │
                                     v                           │
                              ┌──────────────┐   semua part      │
                              │ maintenance  │──taken + close──► ├──┘
                              │ (seluruh     │   (BR-044)        │
                              │  fase WO:    │                   │ close WO,
                              │  open/menung-│                   │ di luar shift
                              │  gua_part/   │ ──────────────────►└──► idle
                              │  in_progress)│                        (BR-045)
                              └──────────────┘

  ATURAN KUNCI:
  • rejected & breakdown = transisi SINGKAT — selama 1 transaksi DB yang sama
  • asal WO TIDAK di status alat → di work_orders.trigger_type (BR-039)
  • maks 1 WO terbuka per alat → maintenance tidak bisa breakdown ke-2 (BR-040)
  • selama maintenance: tombol Mulai Operasi disabled (BR-040 + BR-021)
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Operator | Submit P2H gagal kritis / lapor breakdown | rejected/breakdown → auto-WO → maintenance (1 transaksi) |
| 2 | Sistem | — | `trigger_type` diisi sesuai asal (BR-039) |
| 3 | Foreman | Klaim, rencanakan part | WO open → menunggu_part/in_progress; alat tetap maintenance |
| 4 | Storekeeper | Ambil part | stok berkurang atomik (PF-004) |
| 5 | Foreman | Tutup WO | Semua part taken → closed_at → alat running/idle otomatis (PF-005) |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 1 | WO terbuka sudah ada? | Tolak + notifikasi (BR-040) | Buat WO |
| 1 | Insert WO sukses? | maintenance | Stuck rejected/breakdown (fail-safe) |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 1 | DB error saat auto-WO | Rollback ke rejected/breakdown | Foreman retry / WO manual |
| 4 | Stok kurang | Rollback; WO → menunggu_part | Tunggu stok masuk (PF-004) |
| 5 | Masih ada part planned | Close ditolak + daftar | Ambil part dulu |

---

## PF-004: Alur Ambil Part (Integrasi WO → Inventory)

**Trigger:** Storekeeper menekan "Ambil Part" di detail WO.
**Goal:** Stok berkurang tepat 1×, tidak pernah minus, tidak pernah dobel.

**Actors:**
- Storekeeper: mengeksekusi
- Sistem: validasi atomik

### Flow

```
START (WO punya work_order_part status=planned)
  |
  v
[1] Storekeeper tekan "Ambil Part" (qty dari rencana)
  |
  v
[2] BEGIN DB TRANSACTION
  |
  v
[3] Validasi: stok >= qty ?  (BR-056)
  |
  +--- YA ------------------> [4] insert stock_transaction (out, WO ref)
  |                                   |
  |                                   v
  |                             [5] part.stok -= qty
  |                                   |
  |                                   v
  |                             [6] work_order_part -> taken
  |                                 + stock_transaction_id (BR-042)
  |                                   |
  |                                   v
  |                             [7] Evaluasi status WO (BR-037):
  |                                 semua taken -> in_progress
  |                                 sisa kurang -> menunggu_part
  |                                   |
  |                                   v
  |                             [8] COMMIT
  |                                   |
  |                                   v
  |                                 END (stok berkurang 1x)
  |
  +--- TIDAK (stok < qty) --> [9] ROLLBACK semua
                                    |
                                    v
                              WO -> menunggu_part (BR-043)
                              pesan: "stok tidak cukup"
                                    |
                                    v
                                  END (stok TIDAK berubah)
```

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 3 | stok ≥ qty? | Step 4 (commit) | Step 9 (rollback) |
| 7 | Semua part taken? | in_progress | menunggu_part |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 4–7 | DB error di tengah | Rollback total (NFR-REL-001) | Ulang klik |
| 3 | Stok kurang | Rollback; WO menunggu_part | Stok masuk (BAB-5) |

---

## PF-005: Alur Tutup WO & Pengembalian Alat

**Trigger:** Foreman menutup WO in_progress.
**Goal:** Alat kembali ke jalur operasi tanpa keputusan manual status.

**Actors:**
- Foreman: menutup
- Sistem: validasi & hitung status akhir

### Flow

```
START (WO in_progress)
  |
  v
[1] Foreman tekan "Tutup WO"
  |
  v
[2] Semua work_order_part.status = taken ? (BR-044)
  |
  +--- TIDAK ------------> [3] TOLAK + daftar part belum diambil
  |                               |
  |                               v
  |                             END (arahkan ke PF-004)
  |
  +--- YA ----------------> [4] closed_at = now()
                                  |
                                  v
                            [5] Masih jadwal shift hari ini? (BR-045)
                                  |
                    +-------------+-------------+
                    | YA                      | TIDAK
                    v                         v
              equipment = running        equipment = idle
                    |                         |
                    +-------------+-------------+
                                  v
                            [6] COMMIT (WO closed + alat
                                dalam 1 transaksi)
                                  |
                                  v
                            END (tampilkan downtime
                                 = closed_at - detected_at)
```

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Semua part taken? | Step 4 | Step 3 |
| 5 | `now()` dalam shift Day/Night? | running | idle |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 2 | Ada part planned | Tolak close | Selesaikan PF-004 |
| 5 | — | Selalu ada hasil (running/idle) | — |

## Process Relationship Map

```
PF-003 --pemicu--> PF-004 (WO butuh part)
PF-004 --syarat--> PF-005 (part taken sebelum close)
PF-003 --mencakup--> PF-004 + PF-005 (state machine = gambaran besar)
PF-001 (BAB-3) --triggers--> PF-003 (P2H gagal -> rejected)
```
