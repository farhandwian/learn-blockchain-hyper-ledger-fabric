# 03 — Chaincode: State, Key, Query, Event

> Target: Anda tidak perlu bisa menulis chaincode dari nol, tapi harus bisa **membaca** chaincode dan menemukan yang salah.

---

## 3.1 Anatomi chaincode

Chaincode itu program biasa. Tidak ada bahasa khusus. Strukturnya:

```
   ┌───────────────────────────────────────────────────────────┐
   │  CHAINCODE (satu package, di-deploy ke channel)           │
   │                                                           │
   │   ┌─────────────────────────────────────────────────┐     │
   │   │  Contract: "BatchContract"                      │     │
   │   │                                                 │     │
   │   │   func CreateBatch(ctx, id, qty)   ──┐          │     │
   │   │   func TransferBatch(ctx, id, to)    │ dipanggil│     │
   │   │   func ReadBatch(ctx, id)            │ dari luar│     │
   │   │   func QueryByOwner(ctx, owner)    ──┘          │     │
   │   └─────────────────────────────────────────────────┘     │
   │                          │                                │
   │                          │ ctx.GetStub()                  │
   │                          ▼                                │
   │   ┌─────────────────────────────────────────────────┐     │
   │   │  STUB — satu-satunya pintu ke ledger            │     │
   │   │   GetState / PutState / DelState                │     │
   │   │   GetStateByRange / GetQueryResult              │     │
   │   │   CreateCompositeKey / SetEvent / GetTxID ...   │     │
   │   └─────────────────────────────────────────────────┘     │
   └───────────────────────────────────────────────────────────┘
```

> 🔑 **Semua yang menyentuh ledger lewat stub.** Kalau ada kode di chaincode yang mengakses sesuatu di luar stub (file, network, jam), itu langsung mencurigakan (lihat 02.6).

---

## 3.2 Contoh chaincode minimal (Go)

Baca ini pelan-pelan. Semua chaincode Fabric bentuknya mirip ini.

```go
type SmartContract struct {
    contractapi.Contract
}

type Batch struct {
    ID        string `json:"id"`
    Product   string `json:"product"`
    Qty       int    `json:"qty"`      // integer, bukan float
    Owner     string `json:"owner"`    // MSP ID pemilik
    Status    string `json:"status"`
    UpdatedAt string `json:"updatedAt"`
}

func (s *SmartContract) CreateBatch(ctx contractapi.TransactionContextInterface,
    id string, product string, qty int) error {

    // 1) VALIDASI INPUT
    if id == "" || product == "" {
        return fmt.Errorf("id dan product wajib diisi")
    }
    if qty <= 0 {
        return fmt.Errorf("qty harus > 0, dapat %d", qty)
    }

    // 2) CEK IDENTITAS PEMANGGIL  ← sering dilupakan AI!
    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }
    if mspID != "SupplierMSP" {
        return fmt.Errorf("hanya Supplier yang boleh membuat batch, bukan %s", mspID)
    }

    // 3) CEK BELUM ADA (idempotency / cegah timpa)
    existing, err := ctx.GetStub().GetState(id)
    if err != nil {
        return fmt.Errorf("gagal baca state: %v", err)
    }
    if existing != nil {
        return fmt.Errorf("batch %s sudah ada", id)
    }

    // 4) WAKTU DETERMINISTIK
    ts, err := ctx.GetStub().GetTxTimestamp()
    if err != nil {
        return err
    }

    batch := Batch{
        ID: id, Product: product, Qty: qty,
        Owner: mspID, Status: "CREATED",
        UpdatedAt: ts.AsTime().UTC().Format(time.RFC3339),
    }

    b, err := json.Marshal(batch)
    if err != nil {
        return err
    }

    // 5) TULIS
    if err := ctx.GetStub().PutState(id, b); err != nil {
        return err
    }

    // 6) EMIT EVENT untuk konsumen off-chain
    return ctx.GetStub().SetEvent("BatchCreated", b)
}
```

### 6 langkah di atas adalah template review Anda

```
   ┌───┬──────────────────────────────┬────────────────────────────────┐
   │ # │ Langkah                      │ Kalau tidak ada?               │
   ├───┼──────────────────────────────┼────────────────────────────────┤
   │ 1 │ Validasi input               │ Data sampah masuk ledger       │
   │   │                              │ PERMANEN. Tidak bisa dihapus.  │
   │ 2 │ Cek identitas pemanggil      │ 🚨 Siapapun bisa apapun        │
   │ 3 │ Cek pre-condition state      │ Overwrite diam-diam, atau      │
   │   │                              │ transisi status ilegal         │
   │ 4 │ Waktu via GetTxTimestamp     │ Non-determinisme               │
   │ 5 │ PutState sekali di akhir     │ (lihat 02.7)                   │
   │ 6 │ SetEvent                     │ Sistem off-chain tidak tahu    │
   │   │                              │ ada perubahan                  │
   └───┴──────────────────────────────┴────────────────────────────────┘
```

🚩 **Langkah 2 adalah yang paling sering hilang dari kode buatan AI.** Fabric hanya memastikan *identitas pemanggil valid*; ia sama sekali tidak tahu aturan bisnis "hanya supplier boleh create". Itu harus Anda tulis.

---

## 3.3 API stub yang perlu Anda kenal

| Fungsi | Gunanya | Catatan |
|---|---|---|
| `GetState(key)` | baca satu key | masuk read-set |
| `PutState(key, val)` | tulis satu key | masuk write-set |
| `DelState(key)` | hapus dari world state | riwayat tetap di blockchain |
| `GetStateByRange(start, end)` | ambil rentang key | ⚠️ phantom read |
| `CreateCompositeKey(prefix, attrs)` | bikin key terstruktur | lihat 3.4 |
| `GetStateByPartialCompositeKey` | query berdasarkan prefix | ⚠️ phantom read |
| `GetQueryResult(query)` | rich query JSON | **CouchDB saja**, ⚠️ phantom read |
| `GetHistoryForKey(key)` | riwayat perubahan sebuah key | lambat, read-only |
| `GetTxTimestamp()` | waktu (deterministik) | ✅ pakai ini |
| `GetTxID()` | ID transaksi (deterministik) | ✅ boleh untuk ID unik |
| `SetEvent(name, payload)` | emit event | **1 event per transaksi**, yang terakhir menang |
| `GetPrivateData(coll, key)` | baca private data | lihat file 04 |
| `InvokeChaincode(name, args, ch)` | panggil chaincode lain | hati-hati, lihat 3.8 |

⚠️ `SetEvent` hanya boleh sekali per transaksi. Kalau dipanggil dua kali, yang pertama hilang tanpa error.

---

## 3.4 Desain key — ini keputusan arsitektur, bukan detail

World state adalah key-value. Tidak ada tabel, tidak ada JOIN, tidak ada index otomatis. **Desain key = desain query Anda.**

### Composite key

```
   CreateCompositeKey("owner~batch", ["DIST1", "BATCH007"])
                          │              │        │
                       objectType     attr 1   attr 2

   Menghasilkan key (secara internal):
   \x00owner~batch\x00DIST1\x00BATCH007\x00

   Efeknya: key ter-URUT secara leksikografis, jadi bisa di-scan per prefix:

   \x00owner~batch\x00DIST1\x00BATCH003\x00   ┐
   \x00owner~batch\x00DIST1\x00BATCH007\x00   ├─ GetStateByPartialCompositeKey
   \x00owner~batch\x00DIST1\x00BATCH019\x00   ┘  ("owner~batch", ["DIST1"])
   \x00owner~batch\x00DIST2\x00BATCH001\x00
```

### Pola umum: index buatan sendiri

```
   ┌───────────────────────────────────────────────────────────┐
   │  DATA UTAMA                                               │
   │    BATCH007  →  {"product":"Kopi","owner":"DIST1", ...}   │
   │                                                           │
   │  INDEX (value kosong, key-nya yang penting)               │
   │    owner~batch : DIST1 : BATCH007   →  0x00               │
   │    status~batch: SHIPPED: BATCH007  →  0x00               │
   └───────────────────────────────────────────────────────────┘

   ⚠️ Index ini TIDAK otomatis. Chaincode harus:
      - membuat entri index saat create
      - MENGHAPUS index lama & membuat yang baru saat update
```

🚩 **Red flag klasik:** chaincode mengubah `owner` dari DIST1 ke DIST2 tapi lupa `DelState` index lama. Hasilnya query "batch milik DIST1" tetap mengembalikan batch yang sudah pindah. Cari pola: setiap field yang diindeks, apakah ada `DelState(indexLama)` di jalur update?

---

## 3.5 LevelDB vs CouchDB

Keputusan ini dibuat saat setup jaringan dan **sulit diubah belakangan**.

```
   ┌────────────────────────┬────────────────────────────────────┐
   │  LevelDB (default)     │  CouchDB                           │
   ├────────────────────────┼────────────────────────────────────┤
   │  Key-value murni       │  Document store (JSON)             │
   │  Query: hanya by key   │  Query: rich query (Mongo-like)    │
   │         & range        │         WHERE status=X AND owner=Y │
   │  Cepat                 │  2-4x lebih lambat                 │
   │  Embedded, no ops      │  Container terpisah, perlu di-ops  │
   │  Value boleh apa saja  │  Value HARUS JSON valid            │
   └────────────────────────┴────────────────────────────────────┘
```

### ⚠️ Kalau pilih CouchDB: INDEX WAJIB

Rich query tanpa index akan melakukan **full scan**. Di dev dengan 100 record, cepat. Di produksi dengan 5 juta record, timeout.

Index dideklarasikan sebagai file JSON di dalam package chaincode:

```
  chaincode/
  └── META-INF/
      └── statedb/
          └── couchdb/
              └── indexes/
                  └── indexOwner.json
```

```json
{
  "index": { "fields": ["docType", "owner"] },
  "ddoc": "indexOwnerDoc",
  "name": "indexOwner",
  "type": "json"
}
```

🚩 **Red flag saat review:** ada `GetQueryResult` di chaincode tapi tidak ada folder `META-INF/statedb/couchdb/indexes/`. Ini bom waktu performa.

🧪 **Tes:** isi 100.000 record, ukur latency rich query. Kalau > 1 detik, index-nya salah atau tidak terpakai.

---

## 3.6 Event — jembatan ke dunia off-chain

Ini pola arsitektur yang **hampir pasti Anda butuhkan** di aplikasi supply chain.

```
   ┌──────────────┐
   │  CHAINCODE   │  SetEvent("BatchShipped", payload)
   └──────┬───────┘
          │ (event ikut masuk block, terkirim saat commit VALID)
          ▼
   ┌────────────────────────────────────────────────────────┐
   │  LISTENER (service Node.js/Go milik Anda)              │
   │                                                        │
   │    - subscribe block/chaincode event                   │
   │    - SIMPAN nomor block terakhir yang diproses         │
   │      (checkpoint) → agar bisa resume setelah restart   │
   └───────────┬────────────────────────────────────────────┘
               │
     ┌─────────┼──────────┬──────────────┐
     ▼         ▼          ▼              ▼
 ┌────────┐ ┌──────┐  ┌────────┐   ┌──────────┐
 │Postgres│ │Elastic│ │ Kafka  │   │ Notifikasi│
 │(report)│ │(cari) │ │(integr)│   │ (email/WA)│
 └────────┘ └──────┘  └────────┘   └──────────┘

   Semua reporting, dashboard, pencarian → dari sini.
   JANGAN bikin dashboard yang query langsung ke ledger.
```

Kenapa penting:
- Query ledger lambat dan tidak cocok untuk agregasi.
- Off-chain DB bisa di-index, di-join, di-cache sesuka hati.
- Ledger tetap jadi sumber kebenaran; off-chain DB bisa dibangun ulang dari block 0 kapan saja.

🚩 Red flag: listener event tanpa **checkpoint**. Kalau service restart, event yang lewat hilang selamanya dan data off-chain jadi tidak sinkron.

⚠️ Event hanya terkirim untuk transaksi **VALID**. Ini justru bagus — Anda tidak perlu memfilter sendiri.

---

## 3.7 Kesalahan menyimpan angka

```go
// ⛔ float — pembulatan bisa beda antar platform/versi
type Item struct { Price float64 }

// ✅ integer dalam satuan terkecil
type Item struct { PriceCents int64 }   // Rp 15.000,50 → 1500050
```

Ini masalah nyata di supply chain (harga, berat, volume). Aturannya sama seperti sistem keuangan: **jangan pernah float untuk uang atau kuantitas yang harus persis.**

---

## 3.8 Hal lain yang sering salah

| Anti-pattern | Kenapa buruk |
|---|---|
| `return nil` saat kondisi gagal | Transaksi tercatat VALID padahal tidak melakukan apa-apa. Error harus dikembalikan sebagai error. |
| Menyimpan seluruh array dalam 1 key | Hot key + ukuran value membengkak + MVCC conflict |
| Value > beberapa ratus KB | Block jadi besar, replikasi lambat. Simpan hash, file di off-chain. |
| `InvokeChaincode` lintas channel lalu menulis | Read-set dari channel lain **tidak** divalidasi. Hanya aman untuk read. |
| Logika bisnis di client, chaincode cuma CRUD | Client bisa dimodifikasi. Aturan yang mengikat semua pihak HARUS di chaincode. |
| Tidak ada `docType` di JSON | Rich query jadi susah membedakan jenis dokumen |
| Nama fungsi tidak konsisten dengan yang dipanggil client | Error "function not found" saat runtime |

> 🔑 Poin **"logika bisnis di client"** paling penting secara konseptual. Pertanyaan uji: *"Kalau salah satu organisasi memodifikasi aplikasi client mereka, apakah mereka bisa melanggar aturan bisnis?"* Kalau ya, aturannya salah tempat.

---

## 3.9 Checklist review chaincode

```
  [ ] Tidak ada time.Now / rand / http / getenv / iterasi map
  [ ] Setiap fungsi write mengecek MSP ID atau atribut pemanggil
  [ ] Validasi semua input (kosong, negatif, format, panjang)
  [ ] Cek eksistensi sebelum create; cek status sebelum transisi
  [ ] Transisi status mengikuti state machine yang disepakati
  [ ] Tidak ada PutState lalu GetState pada key yang sama
  [ ] Query tidak dipakai untuk memutuskan write
  [ ] Index composite key di-update DAN dihapus saat data berubah
  [ ] Ada META-INF/statedb/couchdb/indexes kalau pakai rich query
  [ ] Angka penting pakai integer, bukan float
  [ ] Error dikembalikan sebagai error, bukan return diam-diam
  [ ] Ada SetEvent untuk setiap perubahan penting (maks 1 per tx)
  [ ] Tidak ada data pribadi / file besar disimpan di state
```

➡️ Lanjut ke **[04 — Privasi Data](04-privasi-data.md)**.
