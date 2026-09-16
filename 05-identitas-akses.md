# 05 — Identitas & Akses: MSP, ABAC, Endorsement Policy

> Di Fabric ada dua pertanyaan otorisasi yang **berbeda**, dan sering dicampur:
>
> 1. **"Siapa yang boleh MEMANGGIL fungsi ini?"** → dicek di dalam chaincode (ABAC)
> 2. **"Siapa yang harus MENYETUJUI perubahan ini?"** → endorsement policy
>
> Keduanya wajib benar. Yang pertama sering dilupakan AI; yang kedua sering diisi asal.

---

## 5.1 Dari mana identitas datang

```
   ┌──────────────────┐
   │  CA Supplier     │  Certificate Authority milik org Supplier
   │  (Fabric CA)     │
   └────────┬─────────┘
            │ menerbitkan
            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  SERTIFIKAT X.509  (untuk user "budi")                     │
   │                                                            │
   │    Subject:  CN=budi, OU=client, O=Supplier                │
   │    Issuer:   CA Supplier                                   │
   │    Extension (atribut custom):                             │
   │        role      = "warehouse_manager"                     │
   │        siteCode  = "JKT-01"                                │
   │    Signature: <ditandatangani CA Supplier>                 │
   └────────────────────────────────────────────────────────────┘
            │
            │ disimpan di
            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  WALLET (di backend aplikasi Anda)                          │
   │    - sertifikat + private key                              │
   │    ⚠️ Ini kredensial. Perlakukan seperti password DB.       │
   └────────────────────────────────────────────────────────────┘
            │
            │ dipakai untuk menandatangani proposal transaksi
            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  PEER memverifikasi lewat MSP:                             │
   │    "Sertifikat ini dikeluarkan CA yang sah untuk org mana?"│
   │    → hasil: MSP ID = "SupplierMSP"                         │
   └────────────────────────────────────────────────────────────┘
```

> 🔑 **MSP (Membership Service Provider)** = aturan yang memetakan sertifikat → organisasi + role. Yang dilihat chaincode adalah **MSP ID** (mis. `SupplierMSP`), bukan nama orang.

⚠️ Fabric hanya memverifikasi **"identitas ini sah dan milik org X"**. Ia tidak tahu apa-apa tentang aturan bisnis Anda. Semua aturan "siapa boleh apa" adalah tanggung jawab chaincode.

---

## 5.2 Model identitas praktis untuk aplikasi Anda

Ini keputusan desain yang sering salah:

```
   PILIHAN A — satu identitas per organisasi (paling umum)

   ┌─────────────────┐
   │  Backend App    │  login user biasa (JWT/session, di DB Anda)
   │  Supplier       │
   │                 │  semua transaksi ke Fabric pakai
   │  [1 identitas   │  identitas "app-supplier"
   │   Fabric]       │
   └─────────────────┘
   ✅ Sederhana, wallet mudah dikelola
   ⚠️ Ledger hanya tahu "SupplierMSP yang melakukan", bukan siapa orangnya
      → simpan userId di payload transaksi kalau butuh jejak per-orang


   PILIHAN B — satu identitas per user

   ┌─────────────────┐
   │  Backend App    │  tiap user punya sertifikat sendiri
   │                 │  (di-enroll ke CA saat registrasi)
   │  [N identitas]  │
   └─────────────────┘
   ✅ Jejak audit per-individu, non-repudiation kuat
   ⚠️ Manajemen sertifikat, revocation (CRL), rotasi → beban ops nyata
```

Untuk MVP supply chain: **Pilihan A + userId di payload** biasanya cukup. Naik ke B kalau audit per-individu benar-benar jadi requirement.

🚩 Red flag: identitas Fabric di-share ke frontend, atau private key masuk ke repo/environment variable yang ter-log.

---

## 5.3 ABAC — cek identitas di dalam chaincode

Ini yang **paling sering hilang** dari kode buatan AI. Ulangi: Fabric tidak menegakkan aturan bisnis Anda.

```go
// Cara paling umum: cek organisasi
mspID, _ := ctx.GetClientIdentity().GetMSPID()
if mspID != "SupplierMSP" {
    return fmt.Errorf("hanya Supplier yang boleh membuat batch")
}

// Cek atribut custom di sertifikat
ok, _ := ctx.GetClientIdentity().AssertAttributeValue("role", "warehouse_manager")
if !ok {
    return fmt.Errorf("butuh role warehouse_manager")
}

// Cek kepemilikan (ownership) — ini pola paling penting di supply chain
batch := getBatch(ctx, id)
if batch.Owner != mspID {
    return fmt.Errorf("batch ini milik %s, Anda %s", batch.Owner, mspID)
}
```

### Matriks otorisasi — buat ini SEBELUM coding

Ini artefak yang harusnya Anda yang tentukan, bukan AI:

```
   ┌──────────────────┬──────────┬─────────────┬──────────┬─────────┐
   │ Fungsi           │ Supplier │ Distributor │ Retailer │ Auditor │
   ├──────────────────┼──────────┼─────────────┼──────────┼─────────┤
   │ CreateBatch      │    ✅    │      ❌     │    ❌    │   ❌    │
   │ ShipBatch        │  ✅ own  │   ✅ own    │    ❌    │   ❌    │
   │ ReceiveBatch     │    ❌    │      ✅     │    ✅    │   ❌    │
   │ RecallBatch      │    ✅    │      ❌     │    ❌    │   ❌    │
   │ ReadBatch        │    ✅    │      ✅     │    ✅    │   ✅    │
   │ ReadPrice (PDC)  │  ✅ jika │   ✅ jika   │    ❌    │   ❌    │
   │                  │  pihak   │   pihak     │          │         │
   └──────────────────┴──────────┴─────────────┴──────────┴─────────┘

   "own" = hanya untuk batch yang dimilikinya saat ini
```

🧪 **Tes wajib:** untuk setiap ❌ di tabel, tulis satu tes yang mencoba melakukannya dan **memastikan gagal**. Ini disebut negative test dan hampir selalu dilupakan.

---

## 5.4 Endorsement Policy — trust model aplikasi Anda

Pertanyaannya: *"Berapa tanda tangan yang dibutuhkan agar perubahan ini sah?"*

```
   POLICY: AND('SupplierMSP.peer', 'DistributorMSP.peer')

   ┌───────────────────────────────────────────────────────────┐
   │  Transaksi "transfer BATCH007 dari Supplier ke Distributor"│
   │                                                           │
   │  Endorsement dari:                                        │
   │    ✅ Peer Supplier     — "ya, saya setuju melepas"       │
   │    ✅ Peer Distributor  — "ya, saya setuju menerima"      │
   │                                                           │
   │  → VALID. Tidak ada satu pihak pun yang bisa mengklaim    │
   │    transfer secara sepihak.                               │
   └───────────────────────────────────────────────────────────┘

   POLICY: OR('SupplierMSP.peer', 'DistributorMSP.peer')

   ┌───────────────────────────────────────────────────────────┐
   │  ⚠️ Supplier SENDIRIAN bisa mengubah kepemilikan batch    │
   │     menjadi milik Distributor, tanpa persetujuan.         │
   │                                                           │
   │  Kadang ini memang yang diinginkan (mis. supplier         │
   │  mencatat produksinya sendiri). Kadang ini lubang besar.  │
   └───────────────────────────────────────────────────────────┘
```

### Sintaks

| Policy | Arti |
|---|---|
| `OR('A.peer','B.peer')` | cukup satu |
| `AND('A.peer','B.peer')` | dua-duanya wajib |
| `OutOf(2, 'A.peer','B.peer','C.peer')` | minimal 2 dari 3 |
| `AND('A.peer', OR('B.peer','C.peer'))` | bisa bersarang |
| `'A.member'` vs `'A.peer'` vs `'A.admin'` | role dalam org — `peer` paling umum |

### 3 tingkat endorsement policy

```
   ┌─────────────────────────────────────────────────────────────────┐
   │ 1. CHAINCODE-LEVEL  (default, di-set saat commit chaincode)     │
   │    Berlaku untuk SEMUA fungsi dalam chaincode itu.              │
   │    Contoh: "majority of orgs"                                   │
   ├─────────────────────────────────────────────────────────────────┤
   │ 2. STATE-BASED / KEY-LEVEL  (di-set dari dalam chaincode)       │
   │    Berlaku untuk satu KEY tertentu, menimpa policy chaincode.   │
   │    ⭐ Sangat pas untuk supply chain: policy ikut berpindah      │
   │       mengikuti kepemilikan barang.                             │
   ├─────────────────────────────────────────────────────────────────┤
   │ 3. COLLECTION-LEVEL  (di definisi PDC)                          │
   │    Berlaku untuk private data.                                  │
   └─────────────────────────────────────────────────────────────────┘
```

### State-based endorsement — pola yang elegan untuk supply chain

```
   BATCH007 sedang dipegang Supplier
        │  key-level policy: AND(Supplier, Distributor)
        │  → transfer butuh persetujuan kedua pihak
        ▼
   [ transfer ke Distributor ]
        │  chaincode meng-update policy key itu menjadi:
        │  AND(Distributor, Retailer)
        ▼
   BATCH007 sekarang dipegang Distributor
        │  → Supplier TIDAK BISA LAGI mengubah batch ini,
        │    bahkan kalau chaincode-nya di-bypass.
```

Kodenya:

```go
ep, _ := statebased.NewStateEP(nil)
ep.AddOrgs(statebased.RoleTypePeer, "DistributorMSP", "RetailerMSP")
policy, _ := ep.Policy()
ctx.GetStub().SetStateValidationParameter(batchID, policy)
```

Kenapa ini kuat: penegakannya di **layer validasi**, bukan di logika chaincode. Bahkan kalau ada bug di chaincode, peer akan menolak transaksi yang tidak punya tanda tangan yang benar.

---

## 5.5 Kesalahan umum endorsement policy

| Kesalahan | Akibat |
|---|---|
| `OR(...)` untuk semua fungsi karena "biar gampang" | Satu org bisa memalsukan perubahan sepihak. Ini menghapus alasan pakai blockchain. |
| `AND(semua 6 org)` untuk semua fungsi | Satu peer down = seluruh sistem berhenti. Latency tinggi. |
| Policy tidak match dengan endorser yang dipanggil client | `ENDORSEMENT_POLICY_FAILURE` — sering muncul sebagai "kok gagal terus" |
| Lupa update policy saat menambah org baru | Org baru tidak bisa bertransaksi |
| Chaincode menulis private data tapi policy tidak membatasi endorser ke anggota collection | Transaksi gagal atau data bocor |

### Cara memilih policy — pertanyaan pemandu

Untuk setiap fungsi, tanyakan:

```
   "Kalau organisasi X berbohong sendirian tentang ini,
    apakah ada yang dirugikan?"

        YA  → butuh AND (atau OutOf) yang melibatkan pihak yang dirugikan
        TIDAK → OR / single org cukup

   Contoh:
   - CreateBatch (supplier mencatat produksinya sendiri)
     → tidak ada yang dirugikan  → OR(Supplier) cukup
   - TransferOwnership
     → penerima bisa "dipaksa" menerima barang  → AND(pengirim, penerima)
   - RecordTemperature dari sensor
     → semua pihak dirugikan kalau palsu  → AND(pemilik, auditor/oracle)
```

---

## 5.6 Hal ops yang perlu Anda tahu

- **Revocation**: sertifikat yang bocor dicabut lewat CRL di CA. Peer perlu update konfigurasi MSP-nya. Ini proses manual — tanyakan siapa yang bertanggung jawab.
- **Expiry**: sertifikat punya masa berlaku (default sering 1 tahun). 🚨 Sistem yang berjalan tenang selama setahun lalu tiba-tiba mati total adalah kejadian nyata dan umum di Fabric. Pastikan ada monitoring expiry sertifikat sejak hari pertama.
- **TLS**: komunikasi peer-orderer-client pakai mTLS dengan sertifikat terpisah dari identitas transaksi. Jangan tertukar.

🧪 Tambahkan ke checklist ops: alert 60 hari sebelum sertifikat manapun expired.

---

## 5.7 Ringkasan

1. Fabric menjamin *identitas siapa*; aturan *boleh apa* adalah tugas chaincode (ABAC).
2. Buat **matriks otorisasi** sebelum coding, dan tes setiap sel yang "❌".
3. **Endorsement policy adalah trust model** — ini keputusan bisnis, bukan konfigurasi teknis. Anda yang harus menentukannya, bukan AI.
4. **State-based endorsement** membuat policy berpindah mengikuti kepemilikan — pola paling cocok untuk supply chain.
5. Monitor masa berlaku sertifikat sejak awal.

➡️ Lanjut ke **[06 — Lifecycle & Deploy](06-lifecycle-deploy.md)** (bagian praktik).
