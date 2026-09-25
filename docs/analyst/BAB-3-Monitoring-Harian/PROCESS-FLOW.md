# Process Flows — BAB-3 Monitoring Harian

Proses P2H dan log shift yang menentukan kesiapan alat sebelum operasi.

## PF-001: Alur P2H Harian & Gate Keselamatan

> **Referensi:** detail proses pembuatan WO otomatis (step 5–9) dideskripsikan penuh hanya di **UC-009 (BAB-4)** — flow di bawah hanya ringkasan visual untuk konteks chapter ini. Jangan deskripsikan ulang di tempat lain (anti-duplikasi Check #4).

**Trigger:** Operator membuka shift pagi/malam dan menyiapkan alat.
**Goal:** Alat hanya berjalan bila lulus pemeriksaan harian; temuan kritis langsung jadi Work Order.

**Actors:**
- Operator: mengisi checklist
- Sistem: mengevaluasi & membuat WO bila gagal

### Flow

```
START (alat status = idle)
  |
  v
[1] Operator pilih alat -> form 10 item P2H
  |
  v
[2] Operator isi ok/fail per item -> submit
  |
  +--- semua item OK ---------------------------> [3] hasil = PASS
  |                                                     |
  |                                                     v
  |                                               END (alat siap
  |                                                   "Mulai Operasi"
  |                                                    -> running)
  |
  +--- ada item NON-kritis fail ----------------> [4] hasil = PASS + WARNING
  |                                               (BR-024: coolant, lampu,
  |                                                klakson, APAR, sabuk, spion,
  |                                                tekanan ban)
  |                                                     |
  |                                                     v
  |                                                   END
  |
  +--- ada item SAFETY-CRITICAL fail ------------> [5] BR-023: 1 DB TRANSACTION
        (oli mesin / rem / hidrolik)                   |
                                                       v
                                                [6] checklist hasil = FAIL
                                                       |
                                                       v
                                                [7] equipment = rejected
                                                       |
                                                       v
                                                [8] insert WO AUTO
                                                    trigger_type = p2h_gagal
                                                    status = open, prioritas = high
                                                    assigned_to = null
                                                       |
                                      +----------------+----------------+
                                      |                                 |
                               [9a] sukses                      [9b] GAGAL (DB error)
                                      |                                 |
                                      v                                 v
                               equipment = maintenance           ROLLBACK sampai
                               (BR-038)                         equipment = rejected
                                      |                         (FAIL-SAFE: tidak
                                      v                          pernah kembali idle)
                               END (Foreman notifikasi              |
                               "WO-2026-XXXX dibuat")               v
                                                                END (error ke
                                                                Foreman; retry /
                                                                WO manual)
```

### Step Descriptions

| Step | Actor | Action | System Response |
|------|-------|--------|-----------------|
| 1 | Operator | Pilih alat | Tampilkan 10 item (3 safety-critical ditandai) |
| 2 | Operator | Isi + submit | Validasi BR-022 (1× per hari) |
| 3 | — | — | Simpan pass → alat layak running |
| 4 | — | — | Simpan pass + warning per item fail |
| 5–8 | Sistem | Evaluasi fail kritis | Transaksi: fail → rejected → WO → maintenance |
| 9a | Sistem | Commit | Notifikasi Foreman |
| 9b | Sistem | Error | Rollback ke rejected (stuck, butuh Foreman) |

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | Semua item ok? | Step 3 | Step 4/5 |
| 2 | Ada item kritis fail? | Step 5 | Step 4 |
| 8 | Insert WO sukses? | Step 9a | Step 9b |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 2 | P2H sudah ada hari ini | Tolak + arahkan ke checklist eksisting | Lihat/edit catatan |
| 8 | DB error | Rollback → alat tetap rejected | Foreman retry / buka WO manual |

---

## PF-002: Alur Log Shift & Hour Meter

**Trigger:** Operator mengakhiri/mencatat blok jam operasi.
**Goal:** Jam operasi & hour meter tercatat akurat untuk laporan.

**Actors:**
- Operator: mengisi log
- Sistem: hitung jam & update HM

### Flow

```
START (alat running)
  |
  v
[1] Operator pilih alat + shift (Day/Night) + isi jam & material
  |
  v
[2] System hitung jam_operasi (BR-025)
  |
  +--- jam_operasi > 0 -----> [3] material + tonase wajib konsisten
  |                                 |
  |                                 v
  |                           [4] 1 transaksi: insert log
  |                                + hour_meter += jam_operasi (BR-026)
  |                                 |
  |                                 v
  |                               END
  |
  +--- jam_operasi = 0 ------> [5] alasan_idle WAJIB (BR-027)
                                     |
                          +----------+----------+
                          | isi                 | kosong
                          v                     v
                     insert log            tolak + pesan
                     (HM +0)                  END
```

### Decision Points

| Step | Condition | True Path | False Path |
|------|-----------|-----------|------------|
| 2 | jam_operasi > 0? | Step 3 | Step 5 |
| 5 | alasan_idle terisi? | insert log | tolak |

### Exception Handling

| Step | Exception | System Response | Recovery |
|------|-----------|-----------------|----------|
| 1 | (alat, shift, tanggal) sudah ada log | Tolak duplikat (BR-028) | Edit log eksisting |
| 1 | Jam selesai ≤ jam mulai (sama hari) | Validasi tolak | Perbaiki isian |

## Process Relationship Map

```
PF-001 --fail kritis triggers--> UC-009 (auto-WO, detail di BAB-4)
PF-002 --depends-on--> alat berstatus running (hasil PF-001 pass)
PF-001 --gate--> PF-002 (P2H dulu baru operasi/log)
```
