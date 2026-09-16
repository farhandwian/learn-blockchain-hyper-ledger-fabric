# 01 — Mental Model: Fabric itu Apa (dan Bukan Apa)

> Target: setelah baca ini, Anda bisa menjelaskan Fabric ke bos Anda dalam 2 menit tanpa menyebut kata "crypto".

---

## 1.1 Analogi paling akurat

Lupakan Bitcoin. Bayangkan begini:

```
   Anda punya 4 perusahaan yang harus kerja sama,
   tapi tidak saling percaya 100%.

   CARA LAMA:                          CARA FABRIC:

   Supplier    [DB sendiri]            Supplier    ┐
      | email/API                      Distributor ├─> [ SATU buku besar bersama ]
   Distributor [DB sendiri]            Retailer    │      - tiap baris ditandatangani
      | email/API                      Auditor     ┘      - tidak bisa diubah/dihapus
   Retailer    [DB sendiri]                               - semua punya salinan
      | laporan                                           - aturan ditegakkan otomatis
   Auditor     [minta data ke semua]

   Masalah: data beda-beda,             Masalah hilang: satu versi kebenaran,
   saling tuduh, rekonsiliasi manual    riwayat lengkap, tidak bisa dibantah
```

**Definisi kerja:** Hyperledger Fabric adalah *database terdistribusi yang dimiliki bersama oleh beberapa organisasi, di mana setiap perubahan data harus disetujui (ditandatangani) oleh pihak-pihak yang disepakati, dan seluruh riwayat perubahan tersimpan permanen.*

Itu saja. Tidak ada koin, tidak ada mining, tidak ada spekulasi.

---

## 1.2 Fabric BUKAN Ethereum

Ini penting karena 90% konten blockchain di internet membahas Ethereum, dan intuisinya **salah** kalau dipakai di Fabric.

| | Ethereum / Bitcoin (public) | **Hyperledger Fabric** |
|---|---|---|
| Siapa boleh ikut | Siapa saja, anonim | Hanya org yang diundang, **identitas jelas** |
| Konsensus | Proof of Work / Stake, kompetisi | **Raft** — ordering service, tidak ada kompetisi |
| Biaya transaksi | Gas, bayar pakai token | **Tidak ada gas**, tidak ada token |
| Bahasa smart contract | Solidity (bahasa khusus) | **Go / JavaScript / Java** biasa |
| Data | Publik, semua orang lihat | **Privat**, per-channel, bisa per-org |
| Urutan eksekusi | Order → Execute | **Execute → Order → Validate** ⬅ ini beda besar |
| Throughput | ~15-30 TPS | ~500-2000 TPS (tergantung tuning) |
| Finality | Probabilistik (tunggu N block) | **Deterministik** — commit = final |

> 🔑 Perbedaan yang paling berdampak ke kode Anda adalah baris **Execute → Order → Validate**. Itu isi file 02.

---

## 1.3 Kapan sebenarnya TIDAK butuh blockchain

Ini pertanyaan yang harus Anda bisa jawab, karena sering proyek dipaksakan pakai blockchain padahal Postgres cukup.

```
       Apakah ada BANYAK PIHAK yang menulis data?
                    |
         TIDAK ─────+───── YA
           |                |
           v                v
     Pakai database   Apakah pihak-pihak itu saling percaya penuh?
     biasa. Selesai.        |
                    YA ─────+───── TIDAK
                     |              |
                     v              v
             Pakai DB bersama   Apakah butuh audit trail yang
             + API. Selesai.    tidak bisa dibantah / diubah?
                                       |
                             TIDAK ────+──── YA
                               |             |
                               v             v
                         DB + log biasa   ✅ FABRIC MASUK AKAL
```

Untuk supply chain, jawabannya biasanya **ya di semua cabang** — makanya ini use case klasik Fabric.

⚠️ **Tapi sadari batasnya:** blockchain menjamin *"siapa yang mengklaim apa, kapan, dan tidak bisa diubah belakangan"*. Ia **tidak** menjamin klaimnya benar. Kalau supplier input "suhu kontainer 4°C" padahal aslinya 20°C, blockchain akan menyimpan kebohongan itu dengan sangat rapi dan permanen. Ini disebut **oracle problem**, dibahas di file 07.

---

## 1.4 Komponen jaringan

Ada 5 hal yang perlu Anda kenal. Ini diagramnya:

```
```
      ORGANISASI: Supplier                    ORGANISASI: Distributor
 ┌───────────────────────────────┐      ┌───────────────────────────────┐
 │                               │      │                               │
 │   ┌──────────┐   ┌─────────┐  │      │  ┌─────────┐   ┌──────────┐   │
 │   │  Peer 0  │   │   CA    │  │      │  │   CA    │   │  Peer 0  │   │
 │   │          │   │ kartu ID│  │      │  │ kartu ID│   │          │   │
 │   │ ┌──────┐ │   └─────────┘  │      │  └─────────┘   │ ┌──────┐ │   │
 │   │ │Chain-│ │                │      │                │ │Chain-│ │   │
 │   │ │code  │ │                │      │                │ │code  │ │   │
 │   │ └──────┘ │                │      │                │ └──────┘ │   │
 │   │ ┌──────┐ │                │      │                │ ┌──────┐ │   │
 │   │ │Ledger│ │                │      │                │ │Ledger│ │   │
 │   │ └──────┘ │                │      │                │ └──────┘ │   │
 │   └────┬─────┘                │      │                └─────┬────┘   │
 └────────┼──────────────────────┘      └──────────────────────┼────────┘
          │                                                    │
          └──────────────────────┬─────────────────────────────┘
                                 │
                       ┌─────────▼──────────┐
                       │  ORDERING SERVICE  │
                       │  (Raft, 3-5 node)  │
                       │                    │
                       │  Tugasnya HANYA:   │
                       │  mengurutkan tx    │
                       │  jadi block        │
                       └────────────────────┘

   Aplikasi client (backend Anda) bicara ke Peer untuk endorsement,
   lalu ke Orderer untuk mengirim transaksi.
```

### Penjelasan singkat

| Komponen | Analogi | Tugasnya |
|---|---|---|
| **Peer** | Server database milik satu org | Simpan ledger, jalankan chaincode, validasi & commit block |
| **Orderer** | Notaris / antrean | **Hanya** menentukan urutan transaksi dan membungkusnya jadi block. Tidak tahu isi transaksi. |
| **CA** (Certificate Authority) | Bagian HRD yang bikin kartu pegawai | Menerbitkan sertifikat X.509 untuk user & peer |
| **Channel** | Grup WhatsApp | Ledger terpisah. Org yang tidak di channel benar-benar tidak punya datanya. |
| **Chaincode** | Stored procedure / smart contract | Kode Go/JS yang boleh mengubah state |
| **MSP** | Aturan "kartu pegawai mana yang sah" | Memetakan sertifikat → organisasi & role |

> 🔑 **Orderer tidak mengeksekusi apapun.** Ia buta terhadap isi transaksi. Ini beda total dengan Ethereum di mana miner mengeksekusi kode.

---

## 1.5 Ledger = 2 bagian (sering disalahpahami)

Ini konsep yang wajib benar sejak awal.

```
                         LEDGER (di setiap peer)
   ┌────────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │  A) BLOCKCHAIN — file append-only, riwayat lengkap                 │
   │                                                                    │
   │   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐               │
   │   │Block 0 │──▶│Block 1 │──▶│Block 2 │──▶│Block 3 │──▶ ...        │
   │   │genesis │   │ tx,tx  │   │ tx,tx  │   │ tx,tx  │               │
   │   └────────┘   └────────┘   └────────┘   └────────┘               │
   │                                                                    │
   │   • Tidak bisa diubah, tidak bisa dihapus                          │
   │   • Tiap block punya hash block sebelumnya                         │
   │   • Menyimpan tx VALID **dan** tx INVALID (ditandai)  ⬅ penting!  │
   │                                                                    │
   ├────────────────────────────────────────────────────────────────────┤
   │                                                                    │
   │  B) WORLD STATE — database key-value, kondisi TERKINI              │
   │                                                                    │
   │   ┌──────────────┬──────────────────────────────────────────────┐  │
   │   │ KEY          │ VALUE (JSON)                                 │  │
   │   ├──────────────┼──────────────────────────────────────────────┤  │
   │   │ BATCH001     │ {"status":"SHIPPED","owner":"DIST1", ...}     │  │
   │   │ BATCH002     │ {"status":"CREATED","owner":"SUP1",  ...}     │  │
   │   │ SHIPMENT-77  │ {"batches":["BATCH001"], "eta":"..."}        │  │
   │   └──────────────┴──────────────────────────────────────────────┘  │
   │                                                                    │
   │   • Bisa di-update & dihapus (tapi riwayatnya tetap di blockchain) │
   │   • Ini yang dibaca chaincode saat GetState()                      │
   │   • Implementasi: LevelDB (default) atau CouchDB                   │
   │                                                                    │
   └────────────────────────────────────────────────────────────────────┘

   Hubungan keduanya:
   World State = hasil "memutar ulang" semua transaksi valid di blockchain.
   Kalau world state rusak, peer bisa membangunnya ulang dari blockchain.
```

### Konsekuensi praktis

- ⚠️ **"Delete" tidak menghapus riwayat.** `DelState()` hanya menghapus dari world state. Nilai lamanya tetap ada di blockchain selamanya. → Jangan pernah menyimpan data pribadi/rahasia di ledger.
- 🔑 **Query normal membaca world state**, bukan blockchain. Cepat.
- 🔑 **`GetHistoryForKey()`** membaca blockchain untuk melihat riwayat sebuah key. Lambat, jangan dipakai di jalur panas.
- ⚠️ **Transaksi INVALID tetap masuk block.** Jadi "transaksi masuk block" ≠ "transaksi berhasil". Ini jebakan #1, dibahas tuntas di file 02.

---

## 1.6 Alur besar sebuah aplikasi Fabric

```
   ┌──────────────┐
   │  Frontend    │  React / mobile
   └──────┬───────┘
          │ REST / GraphQL biasa
   ┌──────▼───────────────────────────────────────────┐
   │  Backend App (Node.js / Go / Java)               │
   │                                                  │
   │   ┌──────────────────────────────────────┐       │
   │   │  Fabric Gateway SDK                  │       │
   │   │  - pegang identitas (wallet)         │       │
   │   │  - submit / evaluate transaction     │       │
   │   └───────────────┬──────────────────────┘       │
   └───────────────────┼──────────────────────────────┘
                       │ gRPC (mTLS)
              ┌────────▼─────────┐
              │   FABRIC NETWORK │
              └──────────────────┘

   Catatan penting:
   • Frontend TIDAK pernah bicara langsung ke Fabric.
   • Backend Anda tetap backend biasa: auth, validasi, rate limit,
     logging, semuanya masih tugas Anda.
   • Fabric hanya menggantikan "layer penyimpanan yang dipercaya bersama".
```

> 🚩 **Red flag saat review:** kalau AI agent menghasilkan kode yang menaruh private key user di frontend, atau membuat frontend connect langsung ke peer — tolak. Identitas Fabric dipegang backend.

---

## 1.7 Ringkasan yang harus nempel

1. Fabric = database bersama multi-organisasi dengan tanda tangan dan riwayat permanen.
2. Tidak ada token, tidak ada gas, tidak ada mining.
3. Ledger punya 2 bagian: **blockchain** (riwayat, permanen) + **world state** (kondisi terkini, key-value).
4. Orderer hanya mengurutkan, tidak mengeksekusi.
5. Channel = ledger terpisah = batas privasi paling tegas.
6. Blockchain menjamin *klaim tidak bisa diubah*, bukan *klaim itu benar*.

➡️ Lanjut ke **[02 — Transaction Flow](02-transaction-flow.md)**. Ini file terpenting; siapkan 90 menit.
