# 08 — Testing & Review: Checklist Kerja Harian Anda

> Ini file yang akan Anda pakai setiap hari. Pin, cetak, atau jadikan template PR.

---

## 8.1 Piramida tes untuk Fabric

```
                        ▲
                       ╱ ╲       E2E multi-org
                      ╱   ╲      (test-network 2-3 org)
                     ╱─────╲     • lambat (menit)
                    ╱       ╲    • paling mirip produksi
                   ╱ INTEGR. ╲   • WAJIB untuk: endorsement policy,
                  ╱           ╲    MVCC, private data, upgrade
                 ╱─────────────╲
                ╱               ╲  Integration (1 peer / mock ledger)
               ╱                 ╲ • sedang (detik)
              ╱      UNIT         ╲• alur transaksi lengkap
             ╱                     ╲
            ╱_______________________╲ Unit (mock stub)
                                      • cepat (ms)
                                      • logika bisnis, validasi,
                                        state machine, otorisasi

   ⚠️ Kesalahan umum: cuma bikin unit test.
      MVCC conflict, endorsement failure, dan bug private data
      TIDAK AKAN PERNAH muncul di unit test.
```

---

## 8.2 Unit test — mock stub

Go (`shimtest` / mock buatan sendiri), Node (`sinon-chai` + fake stub). Yang penting: **logika**, bukan Fabric-nya.

Yang harus ditest di layer ini:

```
  ✅ Validasi input (kosong, negatif, terlalu panjang, format salah)
  ✅ State machine (setiap transisi legal DAN setiap yang ilegal)
  ✅ Otorisasi (setiap sel ❌ di matriks otorisasi)
  ✅ Kepemilikan (bukan pemilik → harus gagal)
  ✅ Idempotency (create dua kali → gagal)
  ✅ Kompatibilitas skema (baca JSON format lama)
  ✅ Perhitungan (split/merge: total anak == total induk)
```

Contoh yang bagus (Go):

```go
func TestCreateBatch_TolakNonSupplier(t *testing.T) {
    ctx := newMockCtx()
    ctx.SetMSPID("RetailMSP")          // ← bukan supplier

    err := (&SmartContract{}).CreateBatch(ctx, "B1", "Kopi", 100)

    require.Error(t, err)
    require.Contains(t, err.Error(), "hanya Supplier")
}

func TestShipBatch_TolakTransisiIlegal(t *testing.T) {
    ctx := newMockCtx()
    ctx.SetMSPID("FarmerMSP")
    ctx.PutState("B1", batchJSON("SOLD", "FarmerMSP"))  // sudah terjual

    err := (&SmartContract{}).ShipBatch(ctx, "B1", "DistMSP")

    require.Error(t, err)   // tidak boleh mengirim barang yang sudah SOLD
}
```

> 🔑 Untuk tiap fungsi chaincode, rasio sehat: **1 happy path : 3-5 negative test.**
>
> 🚩 Kalau AI agent menghasilkan tes yang isinya cuma happy path, itu tidak menambah keyakinan apa-apa. Minta negative test secara eksplisit.

---

## 8.3 Skenario integrasi yang WAJIB ada

Ini yang membedakan "sudah dites" dengan "benar-benar sudah dites". Masing-masing harus punya tes di test-network:

```
  ┌────┬───────────────────────────────────────────────────────────┐
  │ #  │ Skenario                                                  │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 1  │ MVCC CONFLICT                                             │
  │    │ 20 transaksi paralel ke key yang sama.                    │
  │    │ Harapan: sebagian gagal, client retry, tidak ada data     │
  │    │ hilang, tidak ada sukses palsu ke user.                   │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 2  │ ENDORSEMENT POLICY FAILURE                                │
  │    │ Kirim transaksi dengan endorser kurang dari policy.       │
  │    │ Harapan: gagal dengan error yang jelas, bukan timeout.    │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 3  │ NON-DETERMINISME                                          │
  │    │ Jalankan dengan MINIMAL 2 org endorsing.                  │
  │    │ Harapan: 100 transaksi berturut-turut, tidak ada yang     │
  │    │ gagal karena mismatch. (1 peer saja tidak akan mendeteksi)│
  ├────┼───────────────────────────────────────────────────────────┤
  │ 4  │ OTORISASI LINTAS ORG                                      │
  │    │ Org2 mencoba mengubah aset milik Org1.                    │
  │    │ Harapan: gagal. Untuk SETIAP fungsi write.                │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 5  │ PRIVATE DATA                                              │
  │    │ Org yang bukan anggota collection mencoba baca.           │
  │    │ Harapan: gagal / kosong. DAN cek hash-nya ada di ledger.  │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 6  │ UPGRADE COMPATIBILITY                                     │
  │    │ Deploy v1 → tulis data → deploy v2 → BACA DATA LAMA.      │
  │    │ Harapan: tidak crash, nilai default masuk akal.           │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 7  │ PEER DOWN                                                 │
  │    │ Matikan 1 peer, jalankan transaksi.                       │
  │    │ Harapan: sesuai endorsement policy (AND → gagal,          │
  │    │ OutOf → tetap jalan). Pastikan perilakunya DISENGAJA.     │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 8  │ EVENT LISTENER RESTART                                    │
  │    │ Matikan listener, kirim 10 tx, nyalakan lagi.             │
  │    │ Harapan: 10 event terproses (checkpoint bekerja),         │
  │    │ off-chain DB sinkron.                                     │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 9  │ PERFORMA QUERY                                            │
  │    │ Isi 100.000 record, jalankan rich query.                  │
  │    │ Harapan: < 1 detik. Kalau tidak → index bermasalah.       │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 10 │ IDEMPOTENCY / DOUBLE SUBMIT                               │
  │    │ Kirim transaksi yang sama 2x (mis. user klik 2x).         │
  │    │ Harapan: yang kedua gagal atau tidak berefek ganda.       │
  └────┴───────────────────────────────────────────────────────────┘
```

🧪 Kalau tim Anda hanya sempat mengerjakan 3: pilih **#1, #3, #4**. Itu yang paling sering meledak di produksi.

---

## 8.4 Checklist review PR — versi lengkap

Salin ini jadi template PR.

### A. Determinisme (blocker)
```
[ ] Tidak ada time.Now() / new Date() → pakai GetTxTimestamp()
[ ] Tidak ada rand / uuid / Math.random → ID dari client atau GetTxID()
[ ] Tidak ada HTTP/fetch/axios/gRPC call ke luar
[ ] Tidak ada os.Getenv / process.env / baca file lokal
[ ] Tidak ada iterasi map Go tanpa sort keys
[ ] Tidak ada goroutine / Promise.all yang mempengaruhi write order
[ ] Tidak ada float untuk uang/kuantitas
```

### B. Otorisasi & validasi (blocker)
```
[ ] SETIAP fungsi write mengecek MSP ID / atribut / kepemilikan
[ ] Semua input divalidasi (kosong, negatif, panjang, format, enum)
[ ] Transisi status divalidasi terhadap state machine
[ ] Cek eksistensi sebelum create (cegah overwrite diam-diam)
[ ] Error dikembalikan sebagai error — tidak ada `return nil` diam-diam
```

### C. Konkurensi & query
```
[ ] Tidak ada hot key (counter global, index array, record agregat)
[ ] Client punya retry + backoff untuk MVCC_READ_CONFLICT
[ ] Query (range/rich) TIDAK dipakai untuk memutuskan write
[ ] Tidak ada PutState lalu GetState pada key yang sama
[ ] Index composite key di-DelState saat field yang diindeks berubah
[ ] Ada META-INF/statedb/couchdb/indexes kalau pakai rich query
```

### D. Data & privasi
```
[ ] Tidak ada PII (nama, NIK, alamat, HP) di world state
[ ] Tidak ada file/base64/blob di world state — hanya hash + URI
[ ] Data rahasia lewat transient + PDC, BUKAN argumen transaksi
[ ] Endorsing org dibatasi untuk fungsi yang menyentuh private data
[ ] Ukuran value wajar (< ratusan KB)
```

### E. Integrasi & operasional
```
[ ] SetEvent untuk setiap perubahan penting (maksimal 1 per transaksi)
[ ] Client menunggu & memeriksa status COMMIT, bukan cuma endorsement
[ ] Fungsi read pakai evaluateTransaction; fungsi write pakai submit
[ ] Chaincode versi baru bisa baca data versi lama (+ ada tesnya)
[ ] Sequence number dinaikkan di script deploy (tidak hardcode)
[ ] Endorsement policy eksplisit dan sesuai matriks yang disepakati
[ ] Identitas/wallet tidak pernah keluar dari backend
```

### F. Tes
```
[ ] Ada negative test untuk setiap aturan otorisasi
[ ] Ada negative test untuk setiap transisi status ilegal
[ ] Ada tes integrasi dengan ≥2 org endorsing
[ ] Ada tes konkurensi (MVCC)
[ ] Ada tes kompatibilitas skema lama
```

---

## 8.5 Tabel error → penyebab → tindakan

Simpan ini. Akan sering dipakai.

| Error / gejala | Penyebab paling mungkin | Tindakan |
|---|---|---|
| `MVCC_READ_CONFLICT` | dua tx menulis key sama di block yang sama | retry di client; kalau sering → hot key, desain ulang key |
| `PHANTOM_READ_CONFLICT` | range query bertabrakan dengan write | hindari query di jalur write |
| `ENDORSEMENT_POLICY_FAILURE` | endorser kurang/salah, atau policy tidak sesuai | cek `--peerAddresses` / `setEndorsingOrganizations` |
| `ENDORSEMENT_MISMATCH` / "response payload differ" | **chaincode tidak deterministik** | audit determinisme (checklist A) |
| `chaincode definition not agreed` | definisi antar org beda (versi/policy/sequence) | samakan parameter approve di semua org |
| `chaincode not found` / `not defined` | belum commit, atau nama/channel salah | `peer lifecycle chaincode querycommitted` |
| `INVALID_OTHER_REASON` | chaincode return error saat validasi | lihat log chaincode container |
| Transaksi sukses tapi data tidak berubah | dipanggil via **evaluate**, bukan submit | ganti ke submitTransaction |
| Query lambat (detik) | rich query tanpa index | tambah index di META-INF, verifikasi terpakai |
| `identity expired` / TLS handshake gagal | **sertifikat kedaluwarsa** | enroll ulang; pasang monitoring expiry |
| Private data kosong padahal sudah ditulis | peer bukan anggota collection, atau data lewat argumen bukan transient | cek collection config & transient |
| Event tidak sampai | tx-nya INVALID, atau listener tidak punya checkpoint | cek status tx; tambah checkpoint |
| Semua transaksi timeout | orderer down / kehilangan quorum Raft | cek log orderer |

---

## 8.6 Pertanyaan yang harus Anda ajukan ke AI agent

Ini "senjata" Anda sebagai reviewer. Tanyakan setiap kali menerima kode:

```
  1. "Tunjukkan bahwa fungsi ini deterministik. Apa yang terjadi kalau
      dijalankan di 3 peer sekaligus?"

  2. "Key mana yang bisa jadi hot key di sini? Apa yang terjadi kalau
      100 transaksi datang bersamaan?"

  3. "Siapa saja yang bisa memanggil fungsi ini? Tunjukkan pengecekannya
      dan tunjukkan tes yang membuktikan pihak lain ditolak."

  4. "Bagaimana chaincode versi ini membaca data yang ditulis versi
      sebelumnya? Tunjukkan tesnya."

  5. "Kalau organisasi X memodifikasi aplikasi client mereka, aturan
      bisnis mana yang bisa mereka langgar?"

  6. "Data apa saja di sini yang akan permanen selamanya di ledger?
      Apakah ada yang seharusnya tidak permanen?"

  7. "Endorsement policy untuk fungsi ini apa, dan kenapa itu yang
      dipilih? Siapa yang dirugikan kalau satu pihak berbohong?"

  8. "Apa yang terjadi kalau transaksi ini dikirim dua kali?"
```

⚠️ Perhatikan jawabannya. AI cenderung sangat percaya diri. **Jangan terima jawaban tanpa tes yang bisa dijalankan.** Kalau AI bilang "sudah aman", minta tes yang gagal kalau asumsinya salah.

---

## 8.7 Definition of Done

Sebuah fitur baru boleh dianggap selesai kalau:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  [ ] Checklist 8.4 bagian A & B lolos semua (blocker)         │
  │  [ ] Unit test: 1 happy path + minimal 3 negative test        │
  │  [ ] Berjalan di test-network dengan ≥2 org endorsing         │
  │  [ ] Skenario 8.3 yang relevan sudah dijalankan               │
  │  [ ] Endorsement policy ditulis eksplisit + alasannya         │
  │       terdokumentasi                                          │
  │  [ ] Tabel keputusan data (on-chain / PDC / off-chain)        │
  │       ter-update                                              │
  │  [ ] State machine diagram ter-update kalau ada status baru   │
  │  [ ] Sudah dites baca data dari versi chaincode sebelumnya    │
  └──────────────────────────────────────────────────────────────┘
```

---

## 8.8 Prinsip penutup

```
   Di aplikasi biasa:   deploy → ada bug → perbaiki → data dibersihkan
   Di Fabric:           deploy → ada bug → perbaiki → DATA RUSAK TETAP
                                                       DI LEDGER SELAMANYA
```

Itulah kenapa peran Anda — yang mereview dan memvalidasi, bukan yang mengetik kode — justru **lebih penting** di proyek blockchain daripada proyek biasa.

Anda tidak perlu bisa menulis chaincode lebih cepat dari AI. Anda perlu bisa menjawab pertanyaan yang tidak bisa dijawab AI: **"apakah ini benar untuk bisnis kita, dan apa yang terjadi kalau salah?"**

---

## Bacaan lanjutan

| Sumber | Untuk apa |
|---|---|
| https://hyperledger-fabric.readthedocs.io | dokumentasi resmi; baca bagian *Key Concepts* |
| `fabric-samples/test-network` | jaringan latihan |
| `fabric-samples/asset-transfer-basic` | pola CRUD dasar |
| `fabric-samples/asset-transfer-private-data` | pola PDC + transient |
| `fabric-samples/asset-transfer-abac` | pola otorisasi berbasis atribut |
| `fabric-samples/asset-transfer-sbe` | state-based endorsement (policy ikut kepemilikan) |
| `fabric-samples/off_chain_data` | pola event listener → off-chain DB |

⬅️ Kembali ke **[README](README.md)**
