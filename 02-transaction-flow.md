# 02 — Transaction Flow: Execute → Order → Validate

> **File terpenting dalam materi ini.** Sekitar 80% bug chaincode berasal dari tidak memahami alur ini.
>
> Target: setelah baca ini Anda bisa menjawab "kenapa transaksi ini gagal padahal kodenya kelihatan benar?"

---

## 2.1 Kenapa alurnya aneh?

Di database biasa:

```
   Client → "UPDATE saldo SET x=100" → DB eksekusi → commit → selesai
```

Di Fabric, masalahnya: **siapa yang boleh mengeksekusi?** Kalau cuma satu server yang eksekusi, org lain harus percaya server itu. Padahal premisnya mereka tidak saling percaya.

Solusi Fabric: **beberapa organisasi mengeksekusi transaksi yang sama secara terpisah, lalu hasilnya dibandingkan.** Kalau semua sepakat, baru di-commit.

```
   Ethereum:   ORDER  →  EXECUTE           (urutkan dulu, baru jalankan)
   Fabric:     EXECUTE → ORDER → VALIDATE  (jalankan dulu di beberapa tempat,
                                            urutkan, lalu cek masih valid?)
```

Trade-off-nya: Fabric jauh lebih cepat & fleksibel, **tapi** muncul kelas bug yang tidak ada di database biasa. Itu yang kita pelajari sekarang.

---

## 2.2 Gambaran besar: 3 fase

```
  ╔═══════════════════════════════════════════════════════════════════════╗
  ║  FASE 1 — EXECUTE (Endorsement)                                       ║
  ║                                                                       ║
  ║   Client kirim proposal ke beberapa peer.                             ║
  ║   Tiap peer menjalankan chaincode dalam mode SIMULASI.                ║
  ║   ⚠️ TIDAK ada yang ditulis ke ledger di fase ini.                    ║
  ║   Hasilnya: READ-SET + WRITE-SET + tanda tangan peer.                 ║
  ╚═══════════════════════════════════════════════════════════════════════╝
                                  │
                                  ▼
  ╔═══════════════════════════════════════════════════════════════════════╗
  ║  FASE 2 — ORDER                                                       ║
  ║                                                                       ║
  ║   Client kirim transaksi (+ semua tanda tangan) ke Orderer.           ║
  ║   Orderer HANYA mengurutkan dan membungkus jadi block.                ║
  ║   Orderer TIDAK melihat isi, TIDAK memvalidasi apapun.                ║
  ╚═══════════════════════════════════════════════════════════════════════╝
                                  │
                                  ▼
  ╔═══════════════════════════════════════════════════════════════════════╗
  ║  FASE 3 — VALIDATE & COMMIT                                           ║
  ║                                                                       ║
  ║   Block dikirim ke SEMUA peer. Tiap peer, untuk tiap transaksi:       ║
  ║     a) Cek tanda tangan cukup? (endorsement policy)                   ║
  ║     b) Cek read-set masih valid? (MVCC check)   ⬅ sumber bug!         ║
  ║   Lolos  → write-set diterapkan ke world state, ditandai VALID        ║
  ║   Gagal  → world state TIDAK berubah, ditandai INVALID                ║
  ║   ⚠️ Dua-duanya tetap masuk ke blockchain.                            ║
  ╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 2.3 Diagram detail (sequence)

```
 Client App        Peer Sup      Peer Dist      Orderer       Semua Peer
     │                │              │             │               │
     │──1. Proposal──▶│              │             │               │
     │──1. Proposal───┼─────────────▶│             │               │
     │                │              │             │               │
     │           [simulasi]     [simulasi]         │               │
     │           chaincode      chaincode          │               │
     │           dijalankan     dijalankan         │               │
     │           terhadap       terhadap           │               │
     │           world state    world state        │               │
     │           saat ini       saat ini           │               │
     │                │              │             │               │
     │◀─2. Response───│              │             │               │
     │   {readSet, writeSet, signature}            │               │
     │◀─2. Response───┼──────────────│             │               │
     │                │              │             │               │
     │ 3. Client BANDINGKAN response:              │               │
     │    readSet & writeSet harus IDENTIK.        │               │
     │    Kalau beda → ⛔ berhenti, jangan kirim.  │               │
     │                                             │               │
     │──────4. Kirim tx + semua signature─────────▶│               │
     │                                             │               │
     │                              [orderer urutkan, bikin block] │
     │                                             │               │
     │                                             │──5. Block────▶│
     │                                             │               │
     │                                    [tiap peer validasi:]    │
     │                                    [ a) endorsement policy ]│
     │                                    [ b) MVCC read-set check]│
     │                                    [ lalu commit / tandai  ]│
     │                                    [ INVALID               ]│
     │                                             │               │
     │◀────────6. Event: tx committed (VALID / INVALID)────────────│
     │                                                             │
     │  ⚠️ Kalau app Anda tidak menunggu event ini,                │
     │     Anda tidak tahu transaksi benar-benar berhasil!         │
```

> 🔑 **Hafalkan langkah 3:** client membandingkan hasil dari semua endorser. Kalau chaincode Anda tidak deterministik, hasilnya beda, dan transaksi mati di sini.

---

## 2.4 Read-set & Write-set — dengan contoh konkret

Ini konsep paling penting. Mari lihat wujud aslinya.

Misal world state saat ini:

```
   KEY        VALUE                          VERSION (block, tx)
   ─────────────────────────────────────────────────────────────
   BATCH001   {"qty": 100, "owner": "SUP"}   (5, 2)
   BATCH002   {"qty":  50, "owner": "SUP"}   (7, 0)
```

Chaincode `TransferBatch("BATCH001", "DIST")` menjalankan:

```go
b := GetState("BATCH001")     // baca
b.owner = "DIST"              // ubah di memori
PutState("BATCH001", b)       // tulis
```

Hasil simulasi yang dikirim peer:

```
   ┌─────────────────────────────────────────────────────────┐
   │ READ-SET   (apa yang dibaca, DAN versinya saat dibaca)  │
   ├─────────────────────────────────────────────────────────┤
   │   BATCH001  →  version (5, 2)                           │
   │                ⚠️ NILAINYA tidak ikut dikirim,          │
   │                   hanya nomor versinya                  │
   └─────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────┐
   │ WRITE-SET  (apa yang akan ditulis)                      │
   ├─────────────────────────────────────────────────────────┤
   │   BATCH001  →  {"qty": 100, "owner": "DIST"}            │
   └─────────────────────────────────────────────────────────┘
```

Saat commit, peer bertanya: **"Apakah BATCH001 masih versi (5,2)?"**

- Ya → terapkan write-set. VALID. ✅
- Tidak (ada yang mengubahnya duluan) → **MVCC_READ_CONFLICT**. INVALID. ❌

> 🔑 Ini adalah **optimistic concurrency control**, persis seperti `UPDATE ... WHERE version = 5` di SQL. Bedanya: di Fabric ini otomatis untuk semua key yang dibaca.

---

## 2.5 Jebakan #1 — MVCC_READ_CONFLICT

Ini error yang paling sering muncul di produksi dan paling sering bikin panik.

```
   Waktu ──────────────────────────────────────────────────────▶

   TX-A:  simulasi ──────────┐
          baca BATCH001 v(5,2)│
                              ├──▶ orderer ──┐
   TX-B:  simulasi ──────────┤              │
          baca BATCH001 v(5,2)│              ├─▶ [ BLOCK 8 ]
                              └──▶ orderer ──┘    berisi TX-A, TX-B
                                                        │
                                                        ▼
                                              Validasi berurutan:
                                              ┌──────────────────────────┐
                                              │ TX-A: BATCH001 masih     │
                                              │       v(5,2)? YA ✅      │
                                              │  → commit, versi jadi    │
                                              │    (8,0)                 │
                                              ├──────────────────────────┤
                                              │ TX-B: BATCH001 masih     │
                                              │       v(5,2)? TIDAK ❌   │
                                              │  → MVCC_READ_CONFLICT    │
                                              │  → INVALID, dibuang      │
                                              └──────────────────────────┘
```

### Kenapa ini penting buat Anda

- Ini **bukan bug**, ini desain. Optimistic locking memang begitu.
- Tapi ini jadi bencana kalau ada **hot key** — satu key yang di-update banyak transaksi.

### 🚩 Red flag: pola "hot key"

```go
// ⛔ BURUK: counter global
counter := GetState("TOTAL_SHIPMENTS")
counter++
PutState("TOTAL_SHIPMENTS", counter)
// Setiap shipment menyentuh key yang sama.
// 10 shipment bersamaan → 9 gagal.
```

```go
// ✅ BAIK: tidak ada key bersama
PutState("SHIPMENT-"+id, shipment)
// Total dihitung di off-chain DB dari event, bukan di ledger.
```

Pola hot key yang harus Anda cari saat review:
- counter / sequence number global
- "index" yang disimpan sebagai satu key berisi array
- record "summary" atau "aggregate" yang di-update tiap transaksi
- semua transaksi menulis ke record milik satu organisasi yang sama

### Yang harus dilakukan aplikasi

⚠️ **Client wajib punya retry.** Contoh: retry 3x dengan exponential backoff + jitter untuk error `MVCC_READ_CONFLICT`.

🧪 **Skenario tes wajib:** kirim 20 transaksi paralel yang menyentuh key yang sama, pastikan aplikasi tidak kehilangan data dan tidak menampilkan sukses palsu.

---

## 2.6 Jebakan #2 — Chaincode HARUS deterministik

Karena chaincode dijalankan **terpisah di beberapa peer**, hasilnya harus persis sama. Kalau tidak, read-set/write-set beda → client menolak → transaksi tidak pernah terkirim.

```
   Peer Supplier                     Peer Distributor
   ┌─────────────────────┐           ┌─────────────────────┐
   │ writeSet:           │           │ writeSet:           │
   │ {"ts":"10:00:01.123"}│  ≠       │ {"ts":"10:00:01.456"}│
   └─────────────────────┘           └─────────────────────┘
                    │                        │
                    └────────┬───────────────┘
                             ▼
                   Client: "hasilnya beda!"
                   ⛔ Transaksi dibatalkan.
                   (kadang muncul sebagai error yang membingungkan)
```

### Tabel: dilarang vs boleh

| ⛔ DILARANG di chaincode | Kenapa | ✅ Gantinya |
|---|---|---|
| `time.Now()`, `new Date()` | tiap peer beda waktu | `stub.GetTxTimestamp()` — sama di semua peer |
| `rand`, `uuid.New()` | jelas beda | ID dibuat client, atau `stub.GetTxID()` |
| HTTP call ke API luar | respons bisa beda / down | data dikirim sebagai parameter transaksi |
| Iterasi `map` di Go | urutan map di Go **acak by design** | ambil keys → `sort.Strings()` → iterasi |
| `os.Getenv()`, baca file lokal | tiap peer beda environment | parameter transaksi atau state |
| Baca DB eksternal | idem | idem |
| Goroutine / concurrency | urutan tidak deterministik | jangan |
| Float arithmetic sensitif | pembulatan bisa beda antar platform | integer (simpan dalam satuan terkecil) |
| `GetHistoryForKey` untuk memutuskan write | riwayat tidak masuk read-set | hindari |

> 🚩 **Ini item #1 di checklist review Anda.** Grep kode chaincode untuk: `time.Now`, `Date(`, `rand`, `http`, `fetch`, `axios`, `os.Getenv`, `Math.random`. Kalau ada di dalam fungsi chaincode → tolak.

⚠️ Jahatnya: bug non-determinisme sering **intermiten**. Di dev dengan 1 peer, semuanya jalan mulus. Baru meledak di staging dengan 3 org.

---

## 2.7 Jebakan #3 — `GetState` tidak melihat `PutState` sendiri

```go
PutState("BATCH001", []byte(`{"qty":50}`))

v, _ := GetState("BATCH001")
// ⚠️ v masih berisi NILAI LAMA dari world state!
// Bukan {"qty":50}.
```

```
   Dalam satu transaksi:

   ┌──────────────────────────────────────────────────┐
   │  WORLD STATE (read-only selama simulasi)         │
   │     BATCH001 = {"qty":100}                       │
   └──────────────────────────────────────────────────┘
         ▲ GetState baca dari sini          │
         │                                  │
         │                                  ▼
   ┌─────┴────────────────────────────────────────────┐
   │  WRITE-SET (buffer, belum ke mana-mana)          │
   │     BATCH001 = {"qty":50}                        │
   └──────────────────────────────────────────────────┘
              PutState nulis ke sini
```

**Aturan:** simpan objek di variabel lokal, kerjakan semua perubahan di sana, `PutState` sekali di akhir.

🚩 Red flag: pola `PutState(...)` lalu `GetState(...)` pada key yang sama dalam satu fungsi.

---

## 2.8 Jebakan #4 — Phantom read (range & rich query)

Read-set berisi **key spesifik + versinya**. Tapi hasil dari *query* (range query / CouchDB rich query) **tidak** dicatat sebagai kondisi yang divalidasi.

```
   TX-A (simulasi jam 10:00):
     "Ambil semua BATCH dengan status=AVAILABLE"  → dapat [B1, B2]
     "Jumlahnya 2, boleh kirim"                   → PutState(SHIPMENT)

   TX-B (jam 10:00:00.5, block yang sama):
     Membuat BATCH B3 dengan status=AVAILABLE

   Saat commit TX-A:
     MVCC check hanya melihat B1 dan B2 — masih versi yang sama? YA.
     ✅ TX-A dinyatakan VALID.

   ⚠️ Padahal asumsi "hanya ada 2 batch" sudah salah.
      Ini disebut PHANTOM READ. Fabric TIDAK melindungi dari ini.
```

**Aturan praktis:**

| Kegunaan query | Aman? |
|---|---|
| Menampilkan daftar ke user (read-only) | ✅ Aman |
| Laporan / dashboard | ✅ Aman (lebih baik dari off-chain DB) |
| **Menentukan apakah sebuah write boleh dilakukan** | ❌ **TIDAK AMAN** |

🚩 Red flag: `GetStateByRange` / `GetQueryResult` diikuti `PutState` dalam fungsi yang sama, di mana keputusan menulis bergantung pada hasil query.

✅ Cara benar: desain key sehingga keputusan bisa diambil dari `GetState` pada key spesifik. Contoh: alih-alih "cari batch yang available", client yang menentukan batch mana, chaincode cukup cek `GetState(batchID).status == AVAILABLE`.

---

## 2.9 Jebakan #5 — "Sukses" palsu

Ini menyebabkan bug data yang sangat sulit dilacak.

```
   ┌────────────────────────────────────────────────────────────┐
   │  contract.submitTransaction("Transfer", "B1", "DIST")      │
   │       return 200 OK  ✅                                    │
   └────────────────────────────────────────────────────────────┘
                              │
        Apa artinya ini? ──────┼──── Artinya HANYA:
                              │      "endorsement berhasil dan
                              │       tx sudah diterima orderer"
                              │
                              ▼
             BUKAN berarti sudah masuk world state!
             Transaksi masih bisa jadi INVALID saat validasi.
```

**Yang benar:**
- Gunakan **Fabric Gateway API** (v2.4+) dan tunggu status commit, bukan cuma endorsement.
- Di Node.js Gateway: `submitTransaction()` sudah menunggu commit, tapi cek status-nya; `submitAsync()` memberi Anda kontrol untuk menunggu sendiri.
- Untuk baca data: gunakan `evaluateTransaction()` (query, tidak masuk ledger), bukan `submitTransaction()`.

🚩 Red flag: kode yang langsung `return { success: true }` setelah submit tanpa memeriksa status commit.

🧪 Tes: matikan salah satu peer endorser, atau paksa MVCC conflict, lalu pastikan API Anda mengembalikan error — bukan sukses.

---

## 2.10 Submit vs Evaluate

Perbedaan yang wajib benar:

```
   ┌──────────────────────────────┬──────────────────────────────┐
   │  EVALUATE (query)            │  SUBMIT (invoke)             │
   ├──────────────────────────────┼──────────────────────────────┤
   │  Jalan di 1 peer saja        │  Jalan di beberapa peer      │
   │  Tidak ke orderer            │  Ke orderer, masuk block     │
   │  Tidak mengubah state        │  Mengubah state              │
   │  Cepat (ms)                  │  Lambat (~1-2 detik)         │
   │  Untuk: GET / laporan        │  Untuk: CREATE/UPDATE/DELETE │
   └──────────────────────────────┴──────────────────────────────┘

   ⚠️ Kalau chaincode punya PutState tapi dipanggil via evaluate,
      write-set-nya dibuang diam-diam. Data tidak tersimpan,
      tapi tidak ada error. Bug hantu.
```

🚩 Red flag: fungsi yang menulis state dipanggil dengan `evaluateTransaction`.

---

## 2.11 Latihan mandiri

Jawab tanpa melihat ke atas. Kalau bisa semua, Anda siap lanjut.

1. Apa isi read-set, dan kenapa nilai datanya tidak ikut disimpan di sana?
2. Kenapa `time.Now()` dilarang, tapi `stub.GetTxTimestamp()` boleh?
3. TX-A dan TX-B sama-sama membaca dan menulis `BATCH001`, dikirim bersamaan. Apa yang terjadi? Siapa yang harus menanganinya?
4. Chaincode melakukan rich query "cari semua batch kosong", lalu menulis. Kenapa ini tidak aman?
5. API Anda mengembalikan 200. Apakah data pasti sudah masuk ledger? Kenapa?
6. Peer A dan peer B menghasilkan write-set berbeda untuk transaksi yang sama. Di titik mana transaksi ini berhenti, dan apa yang biasanya jadi penyebabnya?

<details>
<summary>Jawaban singkat</summary>

1. Key + nomor versi. Yang divalidasi hanya "apakah versinya masih sama", jadi nilai tidak perlu — lebih hemat dan cukup untuk deteksi konflik.
2. `time.Now()` beda di tiap peer → write-set beda. `GetTxTimestamp()` diambil dari header transaksi yang dibuat client, jadi identik di semua peer.
3. Salah satu gagal dengan `MVCC_READ_CONFLICT`. Aplikasi client yang harus retry.
4. Phantom read — hasil query tidak masuk read-set, jadi perubahan yang membuat asumsi query salah tidak terdeteksi saat validasi.
5. Tidak. 200 hanya berarti endorsement sukses & diterima orderer. Masih bisa INVALID saat validasi.
6. Berhenti di client (langkah 3, sebelum ke orderer). Penyebab: chaincode tidak deterministik.
</details>

---

## 2.12 Ringkasan — tempel di monitor

```
  1. Chaincode dijalankan di BANYAK peer  → harus DETERMINISTIK
  2. Validasi pakai versi read-set        → siapkan RETRY untuk MVCC conflict
  3. Query tidak masuk read-set           → jangan pakai query untuk keputusan write
  4. GetState tidak lihat PutState sendiri → pakai variabel lokal
  5. Response sukses ≠ ter-commit         → selalu cek status commit
  6. Submit ≠ Evaluate                    → tulis pakai submit, baca pakai evaluate
```

➡️ Lanjut ke **[03 — Chaincode](03-chaincode.md)**.
