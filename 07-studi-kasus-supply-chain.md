# 07 — Studi Kasus: Aplikasi Supply Chain

> Bagian ini menyatukan semuanya. Kasus: **rantai pasok kopi** dari petani sampai kafe.
>
> Ini adalah bentuk dokumen desain yang seharusnya **Anda** buat sebelum menyuruh AI agent menulis kode.

---

## 7.1 Aktor & organisasi

```
   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
   │  PETANI    │──▶│ PROSESOR   │──▶│DISTRIBUTOR │──▶│   KAFE     │
   │ (FarmerMSP)│   │(ProcMSP)   │   │ (DistMSP)  │   │(RetailMSP) │
   └────────────┘   └────────────┘   └────────────┘   └────────────┘
          │                │                │                │
          └────────────────┴────────┬───────┴────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │  REGULATOR/AUDITOR│  (AuditMSP)
                          │  read-only        │
                          └───────────────────┘
```

---

## 7.2 Yang paling penting: pisahkan alur fisik dan alur data

Ini kesalahan konseptual terbesar dalam proyek supply chain blockchain.

```
   ══════════════ DUNIA FISIK ══════════════
        karung kopi benar-benar berpindah
   Petani ──truk──▶ Prosesor ──truk──▶ Distributor ──▶ Kafe
        │              │                  │             │
        │  ⚠️ Blockchain TIDAK tahu apa-apa tentang ini │
        │              │                  │             │
        ▼              ▼                  ▼             ▼
   ┌─────────────────────────────────────────────────────────┐
   │              TITIK INPUT (oracle)                       │
   │   Manusia scan QR / IoT device / integrasi ERP          │
   │   ⚠️ INI TITIK TERLEMAH DARI SELURUH SISTEM             │
   └─────────────────────────────────────────────────────────┘
        │              │                  │             │
        ▼              ▼                  ▼             ▼
   ══════════════ LEDGER ══════════════
     catatan KLAIM: siapa bilang apa, kapan — tidak bisa diubah
```

> 🔑 **Blockchain tidak membuat data jadi benar. Ia membuat data jadi tidak bisa dibantah.**
>
> Kalau petani input "organik" padahal tidak, ledger akan menyimpan kebohongan itu secara permanen dan sangat rapi.

### Mitigasi oracle problem (harus ada di desain)

| Cara | Contoh |
|---|---|
| Konfirmasi 2 pihak | Penerimaan hanya sah kalau pengirim **dan** penerima sama-sama endorse (`AND`) — kedua pihak harus berbohong bersama |
| Sensor IoT bertanda tangan | Device punya sertifikat sendiri, data suhu/GPS ditandatangani device |
| Verifikasi pihak ketiga | Sertifikat organik dari lembaga sertifikasi, hash-nya di ledger |
| Deteksi anomali off-chain | 500kg keluar dari gudang yang hanya menerima 300kg → alert |
| Konsekuensi ekonomi | Reputasi/penalti on-chain untuk klaim yang terbukti palsu |

🚩 Kalau dokumen desain proyek Anda tidak membahas ini sama sekali, itu red flag di level proyek, bukan level kode.

---

## 7.3 State machine — gambar ini dulu, sebelum coding

```
                    ┌──────────┐
     CreateBatch    │ HARVESTED│  (Petani)
     ───────────▶   └────┬─────┘
                         │ ShipBatch(→Prosesor)
                         │ policy: AND(Farmer, Proc)
                         ▼
                    ┌──────────┐
                    │IN_TRANSIT│
                    └────┬─────┘
                         │ ReceiveBatch
                         │ policy: AND(pengirim, penerima)
                         ▼
                    ┌──────────┐        ┌──────────┐
                    │ RECEIVED │───────▶│ REJECTED │ (mutu tidak lolos)
                    └────┬─────┘        └──────────┘
                         │ ProcessBatch (Prosesor)
                         ▼
                    ┌──────────┐
                    │ PROCESSED│  ← boleh di-split jadi beberapa batch
                    └────┬─────┘
                         │ ShipBatch ... (siklus berulang)
                         ▼
                       ....
                    ┌──────────┐
                    │   SOLD   │  (terminal)
                    └──────────┘

              ┌──────────┐
              │ RECALLED │ ← bisa dari state manapun, hanya oleh
              └──────────┘   Regulator atau pemilik asal
```

### Kenapa diagram ini penting untuk Anda

Setiap panah = satu fungsi chaincode + satu aturan otorisasi + satu tes.
Setiap **panah yang TIDAK ada** = satu negative test.

🧪 Contoh negative test yang wajib ada:
- `HARVESTED → SOLD` langsung (lompat state) harus **gagal**
- `SOLD → IN_TRANSIT` (mundur) harus **gagal**
- `RECEIVED` oleh org yang bukan tujuan pengiriman harus **gagal**
- `ShipBatch` untuk batch yang sudah `IN_TRANSIT` harus **gagal**

🚩 Red flag: chaincode yang menulis `batch.Status = newStatus` tanpa memvalidasi bahwa transisi lama→baru itu legal. Ini bug paling umum dan paling merusak di aplikasi supply chain.

Bentuk yang benar:

```go
var allowed = map[string][]string{
    "HARVESTED":  {"IN_TRANSIT", "RECALLED"},
    "IN_TRANSIT": {"RECEIVED", "REJECTED", "RECALLED"},
    "RECEIVED":   {"PROCESSED", "RECALLED"},
    "PROCESSED":  {"IN_TRANSIT", "RECALLED"},
    "SOLD":       {},
}

func canTransition(from, to string) bool {
    for _, s := range allowed[from] {
        if s == to { return true }
    }
    return false
}
```

---

## 7.4 Model data & desain key

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  WORLD STATE (publik untuk semua anggota channel)               │
   ├─────────────────────────────────────────────────────────────────┤
   │  KEY                        VALUE                               │
   │  ─────────────────────────  ──────────────────────────────────  │
   │  BATCH#<uuid>               { docType:"batch", id, product,     │
   │                               qtyKg, status, owner(MSP),        │
   │                               parentBatchIds[], originFarmId,   │
   │                               updatedAt, updatedBy }            │
   │                                                                 │
   │  SHIP#<uuid>                { docType:"shipment", batchIds[],   │
   │                               from, to, status, eta }           │
   │                                                                 │
   │  DOC#<uuid>                 { docType:"document", sha256,       │
   │                               uri, kind:"organic_cert",         │
   │                               issuer, expiresAt }               │
   │                                                                 │
   │  INDEX (composite key, value kosong)                            │
   │  owner~batch : DistMSP : BATCH#abc                              │
   │  status~batch: IN_TRANSIT : BATCH#abc                           │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │  PRIVATE DATA COLLECTION "pricing"  (hanya pihak bertransaksi)  │
   ├─────────────────────────────────────────────────────────────────┤
   │  BATCH#<uuid>               { pricePerKgIdr: 85000,             │
   │                               currency:"IDR", terms:"NET30" }   │
   │                               ↑ integer, bukan float!           │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │  OFF-CHAIN (S3 / MinIO)                                         │
   ├─────────────────────────────────────────────────────────────────┤
   │  Foto batch, sertifikat organik PDF, surat jalan,               │
   │  data sensor mentah, data pribadi petani (nama, NIK, rekening)  │
   │  → di ledger hanya HASH + URI                                   │
   └─────────────────────────────────────────────────────────────────┘
```

### `parentBatchIds` — jantung traceability

Ini yang membuat "tarik mundur asal-usul" bisa dilakukan:

```
        BATCH#A (petani A, 100kg)  ┐
        BATCH#B (petani B,  80kg)  ├──▶ BATCH#X (prosesor, 150kg roasted)
        BATCH#C (petani C,  50kg)  ┘         parentBatchIds: [A, B, C]
                                                     │
                                                     ├──▶ BATCH#X1 (50kg) → Kafe 1
                                                     └──▶ BATCH#X2 (100kg) → Kafe 2
                                                            parentBatchIds: [X]

   Recall: kopi petani B tercemar.
     → telusuri MAJU  dari B: B → X → {X1, X2} → Kafe 1 & Kafe 2
     → tarik hanya 2 kafe itu, bukan seluruh jaringan.

   Konsumen scan QR di Kafe 1:
     → telusuri MUNDUR dari X1: X1 → X → {A, B, C} → 3 petani asal
```

🚩 Red flag: model data tanpa relasi parent/child. Tanpa ini, "traceability" cuma jadi log biasa dan tidak ada nilainya.

⚠️ Penelusuran ini bisa dalam. Jangan lakukan rekursi tanpa batas di chaincode — batasi kedalaman, atau lebih baik: bangun graph-nya di off-chain DB dari event, dan telusuri di sana.

---

## 7.5 Matriks keputusan data

Buat tabel ini untuk **setiap field** di sistem Anda:

| Field | Di mana | Kenapa |
|---|---|---|
| batchId, status, qtyKg | World state | perlu dilihat & diverifikasi semua pihak |
| owner (MSP ID) | World state | dasar otorisasi |
| parentBatchIds | World state | traceability, inti nilai sistem |
| hash sertifikat | World state | bukti dokumen tidak diubah |
| harga, termin | **PDC** | rahasia komersial antara 2 pihak |
| nama & NIK petani | **Off-chain** | data pribadi, harus bisa dihapus |
| foto, PDF | **Off-chain** | ukuran; ledger hanya hash |
| suhu per detik dari IoT | **Off-chain** | volume; ledger hanya agregat/pelanggaran |
| GPS real-time | **Off-chain** | volume + privasi driver |
| total stok, laporan | **Off-chain (dari event)** | agregasi = hot key kalau on-chain |

🚩 Kalau AI agent menaruh `farmerName`, `photoBase64`, atau `temperature` per-detik ke `PutState` — tolak dan tunjukkan tabel ini.

---

## 7.6 Arsitektur sistem lengkap

```
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Mobile Petani│  │ Web Prosesor │  │Portal Auditor│
  │  (scan QR)   │  │              │  │  (read-only) │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         └─────────────────┼─────────────────┘
                           │ REST / GraphQL + JWT
         ┌─────────────────▼─────────────────────────────────┐
         │  BACKEND API (per organisasi)                     │
         │                                                   │
         │  • Auth user (JWT) — TIDAK ada hubungannya        │
         │    dengan identitas Fabric                        │
         │  • Validasi input (lapis pertama)                 │
         │  • Retry MVCC conflict  ⬅ WAJIB                   │
         │  • Wallet identitas Fabric (rahasia, di server)   │
         │  • Upload file → S3, hitung hash → submit tx      │
         └───────┬───────────────────────────┬───────────────┘
                 │ Gateway SDK (gRPC/mTLS)   │ read
                 ▼                           ▼
  ┌──────────────────────────────┐  ┌──────────────────────┐
  │      FABRIC NETWORK          │  │  OFF-CHAIN STORE     │
  │  ┌────────────────────────┐  │  │                      │
  │  │ Channel: coffee-chain  │  │  │  Postgres (query,    │
  │  │  chaincode: coffeecc   │  │  │   laporan, graph      │
  │  │  PDC: pricing          │  │  │   traceability)      │
  │  └────────────────────────┘  │  │  S3 (dokumen, foto)  │
  └──────────┬───────────────────┘  │  Redis (cache)       │
             │ chaincode events     └──────────▲───────────┘
             ▼                                 │
  ┌────────────────────────────────────────────┴───────────┐
  │  EVENT LISTENER  (service terpisah, punya checkpoint)  │
  │   BatchCreated / BatchShipped / BatchReceived / ...    │
  │   → tulis ke Postgres, kirim notifikasi, trigger ERP   │
  └────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────┐
  │  IoT GATEWAY (opsional)                                │
  │   sensor suhu/GPS → agregasi + deteksi pelanggaran     │
  │   → hanya PELANGGARAN yang masuk ledger, bukan semua   │
  └────────────────────────────────────────────────────────┘
```

### Yang harus Anda perhatikan dari diagram ini

1. **Backend tetap backend biasa.** Auth, rate limit, validasi, logging — semua masih tugas Anda. Fabric tidak menggantikan apapun dari itu.
2. **Semua pembacaan berat lewat off-chain DB**, bukan query ledger.
3. **Retry MVCC ada di backend**, dan harus diuji.
4. **Wallet di server**, tidak pernah di client.
5. **IoT memfilter dulu** — jangan tulis 86.400 titik data per hari per kontainer ke ledger.

---

## 7.7 Endorsement policy per fungsi

Ini tabel yang harus Anda putuskan, bukan AI:

| Fungsi | Policy | Alasan |
|---|---|---|
| `CreateBatch` | `OR(FarmerMSP)` | petani mencatat panennya sendiri; tidak merugikan pihak lain |
| `ShipBatch` | `AND(pemilik, penerima)` | penerima tidak boleh "dikirimi" tanpa tahu |
| `ReceiveBatch` | `AND(pengirim, penerima)` | konfirmasi dua sisi = mitigasi oracle problem |
| `SetPrice` (PDC) | `AND(penjual, pembeli)` | harga adalah kesepakatan |
| `RecallBatch` | `OR(AuditMSP, FarmerMSP)` | penarikan harus bisa cepat, tidak boleh diblokir pihak yang bersalah |
| `RecordViolation` (IoT) | `AND(pemilik, AuditMSP)` | pemilik tidak boleh menutupi pelanggaran sendiri |
| Semua read | — | evaluate, tidak butuh policy |

Plus **state-based endorsement**: setiap kali kepemilikan berpindah, key-level policy batch itu ikut berpindah (lihat 05.4). Efeknya: pihak yang sudah melepas barang tidak bisa lagi mengubah datanya.

---

## 7.8 Hal yang biasanya terlupa di proyek supply chain

```
  [ ] Koreksi kesalahan input — manusia PASTI salah input.
      Tidak ada UPDATE/DELETE di blockchain.
      → butuh fungsi CorrectBatch yang mencatat koreksi sebagai
        transaksi BARU, dengan alasan + siapa yang mengoreksi.
        Nilai lama tetap terlihat di riwayat. Ini fitur, bukan bug.

  [ ] Split & merge batch — 100kg jadi 3 karung; 3 petani jadi 1 lot.
      Konservasi kuantitas harus divalidasi (total anak == total induk).

  [ ] Onboarding org baru — distributor baru gabung 6 bulan lagi.
      Perlu update channel config + endorsement policy + approve ulang.
      Siapa yang punya wewenang? Berapa lama prosesnya?

  [ ] Satu peer down — apakah sistem masih jalan?
      Bergantung endorsement policy. AND(semua) = tidak.

  [ ] Volume realistis — berapa transaksi/hari?
      1000 batch/hari x 5 transisi = 5000 tx/hari. Sangat ringan.
      100.000 scan QR/hari? Itu read, tidak masuk ledger. Aman.
      Sensor per detik? ⛔ Harus difilter.

  [ ] Backup & disaster recovery — peer bisa rebuild world state dari
      blockchain, tapi PRIVATE DATA tidak ada di blockchain.
      Kalau semua peer anggota collection hilang, data itu HILANG.
      → private data wajib punya backup terpisah.

  [ ] Biaya & ops — siapa yang menjalankan peer tiap org?
      Kalau semua peer di-hosting satu vendor, apakah masih ada
      alasan pakai blockchain?  ⬅ pertanyaan jujur yang harus ditanya
```

Poin terakhir itu serius: banyak "proyek blockchain" berakhir sebagai satu perusahaan yang menjalankan semua node. Kalau begitu, secara teknis Postgres dengan audit log sudah cukup. Nilai Fabric muncul justru ketika **tiap org benar-benar menjalankan peer-nya sendiri**.

---

## 7.9 Latihan desain

Sebelum lanjut ke file 08, coba kerjakan sendiri (30 menit):

1. Gambar state machine untuk produk yang akan Anda kerjakan.
2. Buat matriks otorisasi (fungsi × organisasi).
3. Buat tabel keputusan data (field × on-chain/PDC/off-chain).
4. Tentukan endorsement policy per fungsi + alasannya.
5. Daftar 10 negative test dari transisi yang **tidak boleh** terjadi.

Kelima artefak ini adalah **input untuk AI agent Anda**. Dengan ini, kualitas kode yang dihasilkan akan jauh lebih baik daripada prompt "buatkan chaincode supply chain".

➡️ Lanjut ke **[08 — Testing & Review](08-testing-dan-review.md)**.
