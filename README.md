# Belajar Hyperledger Fabric — untuk Reviewer & Validator

Materi ini ditulis untuk **software engineer yang belum pernah menyentuh blockchain**, dan yang tugasnya nanti **bukan menulis chaincode dari nol**, melainkan **mendesain, mengetes, dan memvalidasi** aplikasi supply chain berbasis Hyperledger Fabric.

Karena itu porsinya beda dari tutorial biasa:

| Tutorial biasa | Materi ini |
|---|---|
| Fokus: cara nulis kode | Fokus: cara tahu kode itu **salah** |
| Hafal API | Paham **kenapa** API-nya begitu |
| "Hello world jalan!" | "Kenapa transaksi ini INVALID padahal responsenya sukses?" |

---

## Peta belajar

```
                        MULAI DI SINI
                              |
        +---------------------+---------------------+
        |                                           |
   [ FONDASI ]                                 (skip kalau
        |                                       sudah paham)
        v
  01 Mental Model .............. Fabric itu apa, dan BUKAN apa
        |
        v
  02 Transaction Flow  <<<<  INTI. 80% bug ada di sini
        |
        v
        +---------------+---------------+
        |               |               |
        v               v               v
  03 Chaincode    04 Privasi Data   05 Identitas
   (state,          (channel,        & Akses
    query,           private data,    (MSP, ABAC,
    event)           on/off-chain)    endorsement)
        |               |               |
        +---------------+---------------+
                        |
                        v
              06 Lifecycle & Deploy ...... praktik: test-network
                        |
                        v
              07 Studi Kasus Supply Chain  <<< desain nyata
                        |
                        v
              08 Testing & Review .......  checklist harian Anda
```

---

## Daftar isi

| # | File | Isi | Waktu |
|---|------|-----|-------|
| 01 | [01-mental-model.md](01-mental-model.md) | Fabric itu apa, arsitektur, komponen, kapan TIDAK butuh blockchain | ~45 mnt |
| 02 | [02-transaction-flow.md](02-transaction-flow.md) | **Wajib.** Execute-Order-Validate, read/write set, MVCC, determinisme | ~90 mnt |
| 03 | [03-chaincode.md](03-chaincode.md) | Anatomi chaincode, world state, composite key, query, event | ~60 mnt |
| 04 | [04-privasi-data.md](04-privasi-data.md) | Channel, Private Data Collection, on-chain vs off-chain | ~45 mnt |
| 05 | [05-identitas-akses.md](05-identitas-akses.md) | CA, MSP, wallet, ABAC, endorsement policy | ~60 mnt |
| 06 | [06-lifecycle-deploy.md](06-lifecycle-deploy.md) | Deploy chaincode, upgrade, migrasi skema, baca log | ~60 mnt + praktik |
| 07 | [07-studi-kasus-supply-chain.md](07-studi-kasus-supply-chain.md) | Desain end-to-end aplikasi supply chain | ~60 mnt |
| 08 | [08-testing-dan-review.md](08-testing-dan-review.md) | Cara ngetes, skenario wajib, checklist review, tabel error | ~60 mnt |

Total ~7-8 jam baca + 1 hari praktik.

---

## Cara pakai materi ini

1. **Baca 01 dan 02 dulu, jangan loncat.** File 02 adalah kuncinya. Kalau Anda paham file 02, Anda sudah lebih paham Fabric daripada kebanyakan orang yang cuma ikut tutorial.
2. **File 03-05 boleh dibaca sesuai kebutuhan.**
3. **File 06 dikerjakan sambil buka terminal.** Jangan cuma dibaca.
4. **File 08 dicetak/di-pin.** Itu checklist kerja harian Anda saat me-review kode dari AI agent.

Di setiap file ada bagian **"Red flag saat review"** — itu bagian yang paling relevan buat pekerjaan Anda.

---

## Konvensi penulisan

> 🔑 = konsep kunci, hafalkan
>
> ⚠️ = jebakan umum, sering bikin bug
>
> 🚩 = red flag saat review kode
>
> 🧪 = sesuatu yang harus Anda tes

Versi Fabric yang diacu: **Fabric 2.5 LTS** (lifecycle v2, Gateway API).
