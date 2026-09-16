# 06 — Lifecycle & Deploy (Bagian Praktik)

> Buka terminal. Bagian ini dikerjakan, bukan dibaca.
>
> Prasyarat: Docker + Docker Compose, Go 1.20+ (opsional), Node 18+ (opsional), curl, git.

---

## 6.1 Chaincode Lifecycle v2 — kenapa ribet

Di Fabric 1.x, satu org bisa deploy chaincode sendirian. Itu berbahaya: chaincode adalah aturan yang mengikat semua pihak.

Di Fabric 2.x, deploy adalah **proses persetujuan bersama**:

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ 1. PACKAGE     — bungkus kode jadi .tar.gz                       │
   │                  Menghasilkan PACKAGE ID (hash dari isi)         │
   │                  Dilakukan sekali, hasilnya dibagikan            │
   └──────────────────────────────────────────────────────────────────┘
                                  │
   ┌──────────────────────────────▼───────────────────────────────────┐
   │ 2. INSTALL     — tiap ORG install package ke peer-nya            │
   │                  Supplier: install ✅                            │
   │                  Distributor: install ✅                         │
   └──────────────────────────────────────────────────────────────────┘
                                  │
   ┌──────────────────────────────▼───────────────────────────────────┐
   │ 3. APPROVE     — tiap ORG menyetujui DEFINISI:                   │
   │                    { name, version, SEQUENCE, endorsement        │
   │                      policy, collections config, init-required } │
   │                                                                  │
   │                  ⚠️ Yang disetujui adalah DEFINISI, bukan kode.  │
   │                     Semua org harus setuju definisi yang SAMA.   │
   └──────────────────────────────────────────────────────────────────┘
                                  │
   ┌──────────────────────────────▼───────────────────────────────────┐
   │ 4. COMMIT      — satu org meng-commit ke channel                 │
   │                  Berhasil hanya jika cukup org sudah approve     │
   │                  (sesuai LifecycleEndorsement policy channel)    │
   │                  → chaincode AKTIF                               │
   └──────────────────────────────────────────────────────────────────┘
```

### 🔑 SEQUENCE — sumber 90% kegagalan deploy

```
   Sequence adalah nomor urut DEFINISI, dimulai dari 1.

   Setiap kali Anda mengubah APAPUN dari definisi
   (kode baru, versi baru, policy baru, collection baru)
   → sequence WAJIB naik 1
   → SEMUA org harus approve ULANG dengan sequence baru
   → lalu commit ulang

   ⛔ Gagal paling umum:
      "sudah ganti kode, tapi sequence masih 1"
      → tidak terjadi apa-apa, chaincode lama tetap jalan,
        dan Anda bingung kenapa perubahan tidak muncul.
```

🚩 Red flag di script deploy buatan AI: sequence di-hardcode `1`. Harus dibaca dari `peer lifecycle chaincode querycommitted` lalu +1.

---

## 6.2 Praktik: jalankan test-network

```bash
# 1. Ambil fabric-samples + binary + docker image
mkdir -p ~/fabric && cd ~/fabric
curl -sSLO https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/install-fabric.sh
chmod +x install-fabric.sh
./install-fabric.sh --fabric-version 2.5.9 docker samples binary

cd fabric-samples/test-network

# 2. Nyalakan jaringan 2 org + channel + CouchDB
./network.sh up createChannel -c mychannel -ca -s couchdb

# 3. Lihat apa yang jalan
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Yang seharusnya muncul:

```
   peer0.org1.example.com     ← peer Org1
   peer0.org2.example.com     ← peer Org2
   orderer.example.com        ← ordering service
   couchdb0 / couchdb1        ← world state
   ca_org1 / ca_org2 / ca_orderer
```

```bash
# 4. Deploy chaincode contoh
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-go -ccl go
```

Baca output-nya pelan-pelan — Anda akan melihat persis 4 langkah lifecycle di atas.

---

## 6.3 Praktik: invoke & query manual

```bash
# Set environment sebagai Org1 (ada di test-network)
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
source ./scripts/envVar.sh
setGlobals 1

# INIT data contoh
peer chaincode invoke -o localhost:7050 \
  --ordererTLSHostnameOverride orderer.example.com --tls \
  --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" \
  -C mychannel -n basic \
  --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" \
  --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" \
  -c '{"function":"InitLedger","Args":[]}'

# QUERY (evaluate — tidak masuk ledger)
peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```

> 🔑 Perhatikan **dua** `--peerAddresses`. Itu karena endorsement policy default test-network adalah `AND(Org1, Org2)`.
>
> 🧪 **Eksperimen wajib:** hapus satu `--peerAddresses` dan jalankan lagi. Anda akan melihat `ENDORSEMENT_POLICY_FAILURE`. Sekarang Anda tahu wujud error itu.

---

## 6.4 Praktik: bikin MVCC conflict sendiri

Ini eksperimen paling berharga di seluruh materi. Lakukan.

```bash
# Kirim 10 transfer ke aset yang SAMA, bersamaan
for i in $(seq 1 10); do
  peer chaincode invoke -o localhost:7050 \
    --ordererTLSHostnameOverride orderer.example.com --tls \
    --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" \
    -C mychannel -n basic \
    --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" \
    --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" \
    -c "{\"function\":\"TransferAsset\",\"Args\":[\"asset1\",\"owner$i\"]}" &
done
wait

# Lihat hasilnya di log peer
docker logs peer0.org1.example.com 2>&1 | grep -i "MVCC\|VSCC\|invalid" | tail -20
```

Anda akan melihat baris seperti:

```
[gossip.privdata] ... Committed block [12] with 10 transaction(s)
[kvledger] ... Channel [mychannel]: Validation... marked as invalid by state validator.
             Reason code [MVCC_READ_CONFLICT]
```

**Renungkan:** `peer chaincode invoke` tadi mengembalikan `status:200` untuk hampir semuanya. Tapi hanya 1 yang benar-benar tersimpan. **Inilah jebakan "sukses palsu" dari file 02.5, dalam wujud nyata.**

🧪 Cek hasil akhirnya: `peer chaincode query -C mychannel -n basic -c '{"Args":["ReadAsset","asset1"]}'` — ownernya cuma satu.

---

## 6.5 Upgrade chaincode & migrasi skema

```bash
# Setelah mengubah kode:
./network.sh deployCC -ccn basic -ccp <path> -ccl go -ccv 2.0 -ccs 2
#                                                     ^^^^^^   ^^^^^
#                                                     versi    SEQUENCE naik!
```

### ⚠️ Tidak ada migrasi database di blockchain

Ini konsep yang harus benar-benar Anda pahami:

```
   Database biasa:              Fabric:
   ─────────────────            ────────────────────────────────────
   ALTER TABLE ...              Tidak ada.
   UPDATE ... SET ...           Data lama TETAP dalam format lama,
   → semua row jadi format baru selamanya, sampai ada transaksi
                                yang menyentuhnya.
```

```
   World state setelah upgrade v1 → v2:

   BATCH001  {"id":"...","qty":100}                  ← format v1 (lama)
   BATCH002  {"id":"...","qty":50}                   ← format v1 (lama)
   BATCH003  {"id":"...","qty":75,"grade":"A"}       ← format v2 (baru)

   ⚠️ Chaincode v2 HARUS bisa membaca ketiganya tanpa crash.
```

**Strategi yang benar:**

```go
type Batch struct {
    ID    string `json:"id"`
    Qty   int    `json:"qty"`
    Grade string `json:"grade,omitempty"`   // field baru: opsional!
}

func readBatch(...) (*Batch, error) {
    // ...unmarshal...
    if b.Grade == "" {
        b.Grade = "UNKNOWN"   // default untuk data lama
    }
    return &b, nil
}
```

Aturan main:
- ✅ Menambah field opsional — aman
- ✅ Menambah fungsi baru — aman
- ⚠️ Mengubah arti field yang ada — berbahaya, butuh versioning eksplisit (`"schemaVersion": 2`)
- ⛔ Menghapus field / mengubah tipe — akan crash saat baca data lama
- ⛔ Mengubah format key — data lama jadi tidak terjangkau

🚩 **Pertanyaan wajib ke AI agent setiap kali struct berubah:** *"Bagaimana chaincode versi baru ini membaca record yang ditulis versi lama? Tunjukkan tesnya."*

🧪 **Tes upgrade wajib:** deploy v1 → buat data → deploy v2 → **baca data lama** → pastikan tidak error dan nilainya masuk akal.

---

## 6.6 Dev mode: iterasi cepat (Chaincode as a Service)

Deploy penuh butuh 1-2 menit. Untuk development, terlalu lambat. Fabric 2.x mendukung **Chaincode as a Service** — chaincode jalan sebagai proses/container biasa yang Anda restart sendiri.

```
   MODE NORMAL:                      MODE CaaS (dev):
   ─────────────                     ────────────────
   peer build & jalankan             Anda jalankan chaincode sendiri
   container chaincode               (go run / npm start / debugger)
        │                                    │
   ubah kode → package →             ubah kode → restart proses
   install → approve → commit        (2 detik) ✅
   (1-2 menit) 😴

                                     ⭐ Bisa pasang breakpoint!
```

Contohnya ada di `fabric-samples/asset-transfer-basic/chaincode-external`. Tanyakan ke tim apakah ini dipakai — kalau belum, ini peningkatan produktivitas terbesar yang bisa Anda usulkan.

---

## 6.7 Membaca log — kemampuan debugging utama Anda

```bash
# Log peer (validasi, commit, error chaincode)
docker logs -f peer0.org1.example.com

# Log chaincode container (println/fmt dari chaincode Anda)
docker ps --format '{{.Names}}' | grep dev-peer
docker logs -f dev-peer0.org1.example.com-basic_1.0-<hash>

# Log orderer (masalah block, Raft)
docker logs -f orderer.example.com

# Naikkan verbosity (set sebelum network up)
export FABRIC_LOGGING_SPEC=INFO:kvledger=DEBUG:vscc=DEBUG
```

### Kata kunci yang dicari

| Cari ini | Artinya |
|---|---|
| `MVCC_READ_CONFLICT` | konflik concurrency — perlu retry di client |
| `ENDORSEMENT_POLICY_FAILURE` | tanda tangan kurang / endorser salah |
| `PHANTOM_READ_CONFLICT` | range query bertabrakan dengan write |
| `INVALID_OTHER_REASON` | biasanya error chaincode |
| `chaincode registered` | chaincode berhasil start |
| `Committed block [N]` | block masuk — cek berapa tx valid |
| `failed to invoke chaincode name:` | chaincode tidak jalan / nama salah |
| `proposal response was not successful` | chaincode return error saat simulasi |

🧪 Latihan: sengaja bikin ketiga error pertama, catat wujud lognya. Nanti saat produksi Anda langsung kenal.

---

## 6.8 Bersih-bersih

```bash
./network.sh down          # matikan + hapus volume
docker system prune -a     # kalau disk penuh (hati-hati: hapus semua image)
```

---

## 6.9 Checklist deploy sebelum production

```
  [ ] Sequence dibaca dari querycommitted, bukan di-hardcode
  [ ] Endorsement policy di-set eksplisit, bukan pakai default
  [ ] Collection config (PDC) ikut di-approve semua org
  [ ] Index CouchDB ada di META-INF dan terverifikasi terpakai
  [ ] Chaincode v-baru terbukti bisa baca data v-lama (ada tesnya)
  [ ] Rollback plan: kalau v2 rusak, apa yang dilakukan?
      (jawaban: deploy v3 yang memperbaiki — TIDAK BISA rollback state!)
  [ ] Semua org sudah approve sebelum commit dicoba
  [ ] Monitoring: expiry sertifikat, tinggi block, rasio tx invalid
```

> ⚠️ **Tidak ada rollback di blockchain.** Kalau chaincode v2 menulis data rusak ke ledger, data itu permanen. Perbaikannya adalah v3 yang menulis koreksi. Ini alasan kenapa peran Anda sebagai validator itu penting — di sini "deploy dulu, perbaiki nanti" jauh lebih mahal daripada di aplikasi biasa.

➡️ Lanjut ke **[07 — Studi Kasus Supply Chain](07-studi-kasus-supply-chain.md)**.
