# 04 — Privasi Data: Channel, Private Data, On/Off-Chain

> Pertanyaan yang harus bisa Anda jawab untuk setiap field data:
> **"Siapa yang boleh melihat ini, dan apakah boleh permanen?"**

---

## 4.1 Masalahnya

Di supply chain, tidak semua orang boleh lihat semua hal:

```
   Supplier menjual ke Distributor seharga  Rp 10.000
   Distributor menjual ke Retailer seharga  Rp 15.000

   ⚠️ Kalau semua data di satu tempat terbuka:
      Retailer tahu margin Distributor 50%  →  Distributor bangkrut
      Supplier tahu markup Distributor      →  minta naik harga

   Tapi Auditor harus bisa membuktikan transaksi itu benar terjadi.
```

Fabric punya 3 tingkat pemisahan. Pilih yang paling sederhana yang cukup.

---

## 4.2 Tiga tingkat privasi

```
  ╔══════════════════════════════════════════════════════════════════════╗
  ║ TINGKAT 1 — CHANNEL                                    (paling kuat) ║
  ║                                                                      ║
  ║   Ledger benar-benar TERPISAH. Org di luar channel tidak punya       ║
  ║   datanya sama sekali — bukan cuma tidak boleh lihat, memang tidak   ║
  ║   ada di disk mereka.                                                ║
  ║                                                                      ║
  ║   Biaya: chaincode & data terpisah, tidak bisa transaksi lintas      ║
  ║          channel dengan mudah, ops jadi ribet.                       ║
  ╚══════════════════════════════════════════════════════════════════════╝

  ╔══════════════════════════════════════════════════════════════════════╗
  ║ TINGKAT 2 — PRIVATE DATA COLLECTION (PDC)              (paling pas)  ║
  ║                                                                      ║
  ║   Satu channel, tapi data rahasia hanya direplikasi ke org tertentu. ║
  ║   Yang masuk ledger bersama hanya HASH-nya.                          ║
  ║                                                                      ║
  ║   Biaya: sedikit lebih rumit di chaincode (transient data).          ║
  ╚══════════════════════════════════════════════════════════════════════╝

  ╔══════════════════════════════════════════════════════════════════════╗
  ║ TINGKAT 3 — OFF-CHAIN + HASH                          (paling murah) ║
  ║                                                                      ║
  ║   Data asli di S3/DB/sistem internal. Di ledger hanya hash + pointer.║
  ║   Wajib untuk file, dokumen, PII.                                    ║
  ╚══════════════════════════════════════════════════════════════════════╝
```

---

## 4.3 Channel — visual

```
   ┌──────────────────────────────────────────────────────────────┐
   │  CHANNEL "supply-utama"                                      │
   │  Anggota: Supplier, Distributor, Retailer, Auditor           │
   │                                                              │
   │    Ledger A:  batch, shipment, status pengiriman             │
   └──────────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────┐
   │  CHANNEL "keuangan-sup-dist"                                 │
   │  Anggota: Supplier, Distributor                              │
   │                                                              │
   │    Ledger B:  invoice, harga, termin pembayaran              │
   │                                                              │
   │    ⛔ Retailer & Auditor TIDAK punya ledger ini sama sekali. │
   └──────────────────────────────────────────────────────────────┘

   Peer Distributor menyimpan DUA ledger.
   Peer Retailer menyimpan SATU ledger.
```

⚠️ **Channel itu mahal secara operasional.** Tiap channel butuh konfigurasi sendiri, chaincode di-deploy sendiri, dan transaksi atomik lintas channel praktis tidak ada. Jangan bikin channel per pasangan org — itu ledakan kombinatorial.

**Kapan pakai channel:** kelompok org yang benar-benar terpisah, bisnisnya beda, jarang berinteraksi. Contoh: satu channel per negara/region.

---

## 4.4 Private Data Collection — ini yang paling sering dipakai

Ini mekanisme yang paling cocok untuk kasus "harga rahasia" di atas.

```
   Transaksi: Supplier jual BATCH007 ke Distributor, harga Rp 10.000

   ┌────────────────────────────────────────────────────────────────────┐
   │  YANG MASUK LEDGER BERSAMA (semua org di channel lihat ini)        │
   │                                                                    │
   │    BATCH007  →  {"product":"Kopi","qty":100,"owner":"DIST1"}       │
   │                                                                    │
   │    hash(privateData BATCH007) = a3f9c1e8...                        │
   │    ▲                                                               │
   │    └── Retailer & Auditor lihat HASH ini. Mereka tahu ADA data     │
   │        rahasia dan tahu data itu tidak diubah — tapi tidak tahu    │
   │        isinya.                                                     │
   └────────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────┐   ┌─────────────────────────────┐
   │ Peer SUPPLIER               │   │ Peer DISTRIBUTOR            │
   │ ┌─────────────────────────┐ │   │ ┌─────────────────────────┐ │
   │ │ private DB (SideDB)     │ │   │ │ private DB (SideDB)     │ │
   │ │  BATCH007 →             │ │   │ │  BATCH007 →             │ │
   │ │  {"price":10000,        │ │   │ │  {"price":10000,        │ │
   │ │   "terms":"NET30"}      │ │   │ │   "terms":"NET30"}      │ │
   │ └─────────────────────────┘ │   │ └─────────────────────────┘ │
   └─────────────────────────────┘   └─────────────────────────────┘

   ┌─────────────────────────────┐   ┌─────────────────────────────┐
   │ Peer RETAILER               │   │ Peer AUDITOR                │
   │ ┌─────────────────────────┐ │   │ ┌─────────────────────────┐ │
   │ │ private DB              │ │   │ │ private DB              │ │
   │ │  (kosong untuk          │ │   │ │  (kosong)               │ │
   │ │   collection ini)       │ │   │ │                         │ │
   │ └─────────────────────────┘ │   │ └─────────────────────────┘ │
   └─────────────────────────────┘   └─────────────────────────────┘
```

### Kenapa hash-nya berguna

Auditor bisa minta data asli dari Supplier via jalur lain (email, API), lalu **menghitung hash-nya sendiri** dan membandingkan dengan hash di ledger. Kalau cocok → data itu asli dan tidak diubah sejak transaksi terjadi. Ini "bukti tanpa membocorkan".

### Cara datanya masuk (penting!)

Data rahasia **tidak** dikirim sebagai argumen transaksi biasa — argumen tercatat di ledger untuk semua orang. Dikirim lewat **transient field**:

```javascript
// Client
await contract.createTransaction('SellBatch')
  .setTransient({
     price_details: Buffer.from(JSON.stringify({ price: 10000, terms: 'NET30' }))
  })
  .setEndorsingOrganizations('SupplierMSP', 'DistributorMSP')  // ← wajib!
  .submit('BATCH007', 'DIST1');
```

```go
// Chaincode
transient, _ := ctx.GetStub().GetTransient()
raw := transient["price_details"]
// ... validasi ...
ctx.GetStub().PutPrivateData("priceCollection", batchID, raw)
```

### 🚩 Red flag PDC saat review

```
  [ ] Data rahasia dikirim sebagai ARGUMEN biasa, bukan transient
      → argumen tersimpan permanen di blockchain untuk SEMUA org.
        Ini kebocoran yang tidak bisa diperbaiki.

  [ ] Endorsing organizations tidak dibatasi
      → peer yang tidak punya collection ikut meng-endorse,
        transaksi gagal atau bocor.

  [ ] Tidak ada validasi isi transient di chaincode
      → sampah masuk private DB.

  [ ] blockToLive tidak dipertimbangkan
      → PDC bisa diatur auto-purge setelah N block (berguna untuk GDPR).
        Kalau butuh permanen, set 0 secara sadar.

  [ ] Chaincode membandingkan private data di logika yang dijalankan
      peer yang tidak punya data itu → hasil beda antar peer →
      endorsement mismatch.
```

⚠️ **Jebakan halus:** peer yang tidak jadi anggota collection tidak bisa membaca data itu. Kalau chaincode Anda melakukan `GetPrivateData` di fungsi yang di-endorse oleh peer non-anggota, hasilnya beda → transaksi gagal. Selalu batasi endorser untuk fungsi yang menyentuh private data.

---

## 4.5 On-chain vs Off-chain

Aturan paling praktis, dan yang paling sering dilanggar AI agent.

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  BOLEH di ledger                                                 │
   ├──────────────────────────────────────────────────────────────────┤
   │  ✅ ID, referensi, status, timestamp                             │
   │  ✅ Kuantitas, kode produk, lokasi (kota/gudang)                 │
   │  ✅ HASH dari dokumen/file                                       │
   │  ✅ Pointer ke penyimpanan off-chain (URL, object key)           │
   │  ✅ Siapa melakukan apa (MSP ID, bukan nama orang)               │
   └──────────────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────────┐
   │  ⛔ JANGAN di ledger                                             │
   ├──────────────────────────────────────────────────────────────────┤
   │  ❌ Foto, PDF, sertifikat, invoice (file apapun)                 │
   │  ❌ Data pribadi: nama, NIK, alamat, no. HP, email               │
   │  ❌ Kredensial, API key, secret apapun                           │
   │  ❌ Data yang mungkin harus dihapus (GDPR / UU PDP)              │
   │  ❌ Data sensor mentah frekuensi tinggi (per detik)              │
   │     → agregat saja, atau simpan hash batch-nya                   │
   └──────────────────────────────────────────────────────────────────┘
```

### Pola hash + pointer

```
   ┌──────────────┐        1. upload file
   │  Client App  │────────────────────────────┐
   └──────┬───────┘                            ▼
          │                          ┌────────────────────┐
          │  2. hitung SHA-256       │  S3 / MinIO / IPFS │
          │                          │                    │
          │                          │  cert-2024-001.pdf │
          │  3. submit tx:           └────────────────────┘
          │     { docId:  "cert-2024-001",
          │       hash:   "9f2a3c...",
          │       uri:    "s3://certs/cert-2024-001.pdf",
          │       issuer: "SupplierMSP" }
          ▼
   ┌────────────────────────────────────────────┐
   │  LEDGER — kecil, permanen, bisa diaudit    │
   └────────────────────────────────────────────┘

   Verifikasi nanti:  download file → hash ulang → bandingkan dengan ledger.
   Cocok  → file asli, tidak diubah.
   Beda   → file sudah dimanipulasi.
   Hilang → file hilang, tapi buktinya pernah ada tetap tercatat.
```

> 🔑 Ini menjawab pertanyaan "bagaimana blockchain kalau ada UU Perlindungan Data Pribadi?" — data pribadi disimpan off-chain (bisa dihapus), yang di ledger hanya hash yang jadi tidak bermakna setelah data aslinya hilang.

---

## 4.6 Memilih: pohon keputusan

```
             Data ini boleh dilihat SEMUA anggota channel?
                              │
              YA ─────────────┼───────────── TIDAK
              │                              │
              ▼                              ▼
      Ini file / PII?           Berapa org yang boleh lihat?
              │                              │
     TIDAK ───┼─── YA          ┌─────────────┼──────────────┐
      │       │                │             │              │
      ▼       ▼             Sebagian     Sebagian      Hampir semua
   World   Off-chain        kecil,       kecil,        org, cuma
   state   + hash           SERING       JARANG        1-2 dikecualikan
                            transaksi    transaksi           │
                            bareng       bareng              ▼
                               │            │           PDC juga
                               ▼            ▼
                            CHANNEL       PDC
                            terpisah    (paling
                                         sering
                                         dipilih)
```

Untuk kebanyakan aplikasi supply chain: **1 channel + beberapa PDC** adalah jawaban yang benar. Kalau AI agent Anda mengusulkan 6 channel, tanya kenapa.

---

## 4.7 Ringkasan

1. Channel = pemisahan total, mahal secara ops. Pakai untuk kelompok bisnis yang benar-benar terpisah.
2. PDC = data rahasia hanya di peer tertentu, hash-nya di ledger bersama. Ini alat utama untuk harga & syarat komersial.
3. Data rahasia masuk lewat **transient**, bukan argumen.
4. File dan PII **tidak pernah** masuk ledger — simpan hash + pointer.
5. Ledger permanen: setiap keputusan menaruh data di sana tidak bisa dibatalkan.

➡️ Lanjut ke **[05 — Identitas & Akses](05-identitas-akses.md)**.
