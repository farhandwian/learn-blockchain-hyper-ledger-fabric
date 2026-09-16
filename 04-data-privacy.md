# 04 — Data Privacy: Channels, Private Data, On-Chain vs Off-Chain

> The question you must be able to answer for every data field:
> **"Who is allowed to see this, and is it okay for it to be permanent?"**

---

## 4.1 The problem

In a supply chain, not everyone is meant to see everything:

```
   Supplier sells to Distributor at   $10,000
   Distributor sells to Retailer at   $15,000

   ⚠️ If all this data sits in one open place:
      Retailer learns Distributor's 50% margin  →  Distributor gets undercut
      Supplier learns Distributor's markup      →  demands a higher price

   But the Auditor still needs to be able to prove the transaction really happened.
```

Fabric has 3 levels of separation. Pick the simplest one that's sufficient.

---

## 4.2 Three levels of privacy

```
  ╔══════════════════════════════════════════════════════════════════════╗
  ║ LEVEL 1 — CHANNEL                                       (strongest) ║
  ║                                                                      ║
  ║   The ledger is genuinely SEPARATE. An org outside the channel has  ║
  ║   none of its data at all — not just "not allowed to see it," it   ║
  ║   simply isn't on their disk.                                       ║
  ║                                                                      ║
  ║   Cost: separate chaincode & data, cross-channel transactions are   ║
  ║         practically nonexistent, ops overhead goes up.              ║
  ╚══════════════════════════════════════════════════════════════════════╝

  ╔══════════════════════════════════════════════════════════════════════╗
  ║ LEVEL 2 — PRIVATE DATA COLLECTION (PDC)                (best fit)   ║
  ║                                                                      ║
  ║   One channel, but secret data is only replicated to specific orgs. ║
  ║   Only its HASH goes into the shared ledger.                        ║
  ║                                                                      ║
  ║   Cost: slightly more chaincode complexity (transient data).        ║
  ╚══════════════════════════════════════════════════════════════════════╝

  ╔══════════════════════════════════════════════════════════════════════╗
  ║ LEVEL 3 — OFF-CHAIN + HASH                            (cheapest)    ║
  ║                                                                      ║
  ║   Real data lives in S3/DB/an internal system. Only a hash + pointer║
  ║   goes on the ledger. Mandatory for files, documents, PII.          ║
  ╚══════════════════════════════════════════════════════════════════════╝
```

---

## 4.3 Channels — visualized

```
   ┌──────────────────────────────────────────────────────────────┐
   │  CHANNEL "main-supply"                                       │
   │  Members: Supplier, Distributor, Retailer, Auditor           │
   │                                                              │
   │    Ledger A:  batches, shipments, delivery status            │
   └──────────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────┐
   │  CHANNEL "finance-sup-dist"                                  │
   │  Members: Supplier, Distributor                              │
   │                                                              │
   │    Ledger B:  invoices, prices, payment terms                │
   │                                                              │
   │    ⛔ Retailer & Auditor have NONE of this ledger at all.   │
   └──────────────────────────────────────────────────────────────┘

   The Distributor's peer stores TWO ledgers.
   The Retailer's peer stores ONE ledger.
```

⚠️ **Channels are expensive operationally.** Each channel needs its own configuration, its own chaincode deployment, and atomic cross-channel transactions barely exist. Don't create a channel per pair of orgs — that's a combinatorial explosion.

**When to use a channel:** for a group of orgs that are genuinely separate, run a different business, and rarely interact. Example: one channel per country/region.

---

## 4.4 Private Data Collections — the one you'll use most

This is the mechanism best suited to the "secret price" case above.

```
   Transaction: Supplier sells BATCH007 to Distributor, price $10,000

   ┌────────────────────────────────────────────────────────────────────┐
   │  WHAT GOES INTO THE SHARED LEDGER (every org on the channel sees)  │
   │                                                                    │
   │    BATCH007  →  {"product":"Coffee","qty":100,"owner":"DIST1"}    │
   │                                                                    │
   │    hash(privateData BATCH007) = a3f9c1e8...                        │
   │    ▲                                                               │
   │    └── Retailer & Auditor see this HASH. They know secret data    │
   │        EXISTS and hasn't been tampered with — but not what's      │
   │        inside it.                                                  │
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
   │ Peer RETAILER                │   │ Peer AUDITOR                │
   │ ┌─────────────────────────┐ │   │ ┌─────────────────────────┐ │
   │ │ private DB              │ │   │ │ private DB              │ │
   │ │  (empty for              │ │   │ │  (empty)                │ │
   │ │   this collection)       │ │   │ │                         │ │
   │ └─────────────────────────┘ │   │ └─────────────────────────┘ │
   └─────────────────────────────┘   └─────────────────────────────┘
```

### Why the hash is useful

The auditor can request the real data from the supplier through some other channel (email, an API), **compute the hash themselves**, and compare it to the hash on the ledger. If it matches → the data is genuine and hasn't changed since the transaction. This is "proof without disclosure."

### How the data actually gets in (important!)

Secret data is **never** sent as a plain transaction argument — arguments are recorded on the ledger for everyone to see. It's sent through a **transient field** instead:

```javascript
// Client
await contract.createTransaction('SellBatch')
  .setTransient({
     price_details: Buffer.from(JSON.stringify({ price: 10000, terms: 'NET30' }))
  })
  .setEndorsingOrganizations('SupplierMSP', 'DistributorMSP')  // ← mandatory!
  .submit('BATCH007', 'DIST1');
```

```go
// Chaincode
transient, _ := ctx.GetStub().GetTransient()
raw := transient["price_details"]
// ... validate ...
ctx.GetStub().PutPrivateData("priceCollection", batchID, raw)
```

### 🚩 PDC red flags when reviewing

```
  [ ] Secret data sent as a regular ARGUMENT instead of transient
      → the argument is permanently stored on the blockchain for
        EVERY org. This leak can never be fixed.

  [ ] Endorsing organizations not restricted
      → a peer that doesn't hold the collection ends up endorsing,
        causing failures or leaks.

  [ ] No validation of the transient payload in chaincode
      → garbage ends up in the private DB.

  [ ] blockToLive not considered
      → a PDC can be configured to auto-purge after N blocks (useful
        for GDPR). If it needs to be permanent, set it to 0 deliberately.

  [ ] Chaincode compares private data in logic run by a peer that
      doesn't have that data → results differ between peers →
      endorsement mismatch.
```

⚠️ **A subtle trap:** a peer that isn't a member of the collection cannot read that data. If your chaincode calls `GetPrivateData` inside a function endorsed by a non-member peer, results will differ → the transaction fails. Always restrict the endorsers for any function that touches private data.

---

## 4.5 On-chain vs off-chain

The most practical rule, and the one most often violated by AI agents.

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  OK on the ledger                                                │
   ├──────────────────────────────────────────────────────────────────┤
   │  ✅ IDs, references, status, timestamps                          │
   │  ✅ Quantities, product codes, location (city/warehouse)         │
   │  ✅ HASHes of documents/files                                    │
   │  ✅ Pointers to off-chain storage (URL, object key)               │
   │  ✅ Who did what (MSP ID, not a person's name)                   │
   └──────────────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────────┐
   │  ⛔ NEVER on the ledger                                          │
   ├──────────────────────────────────────────────────────────────────┤
   │  ❌ Photos, PDFs, certificates, invoices (any file)              │
   │  ❌ Personal data: names, national IDs, addresses, phone, email  │
   │  ❌ Credentials, API keys, any kind of secret                    │
   │  ❌ Data that might need to be deleted (GDPR / privacy law)      │
   │  ❌ High-frequency raw sensor data (per-second readings)         │
   │     → aggregate only, or store a hash of the batch                │
   └──────────────────────────────────────────────────────────────────┘
```

### The hash + pointer pattern

```
   ┌──────────────┐        1. upload the file
   │  Client App  │────────────────────────────┐
   └──────┬───────┘                            ▼
          │                          ┌────────────────────┐
          │  2. compute SHA-256      │  S3 / MinIO / IPFS │
          │                          │                    │
          │                          │  cert-2024-001.pdf │
          │  3. submit tx:           └────────────────────┘
          │     { docId:  "cert-2024-001",
          │       hash:   "9f2a3c...",
          │       uri:    "s3://certs/cert-2024-001.pdf",
          │       issuer: "SupplierMSP" }
          ▼
   ┌────────────────────────────────────────────┐
   │  LEDGER — small, permanent, auditable      │
   └────────────────────────────────────────────┘

   Verifying later: download the file → re-hash it → compare to the ledger.
   Match     → the file is genuine, unaltered.
   Mismatch  → the file was tampered with.
   Missing   → the file is gone, but the proof it once existed stays recorded.
```

> 🔑 This answers the question "what about privacy laws like GDPR?" — personal data lives off-chain (and can be deleted); the ledger only holds a hash, which becomes meaningless once the original data is gone.

---

## 4.6 Choosing: a decision tree

```
             Should EVERY channel member be able to see this data?
                              │
              YES ────────────┼───────────── NO
              │                              │
              ▼                              ▼
      Is it a file / PII?         How many orgs may see it?
              │                              │
      NO ─────┼─── YES           ┌───────────┼──────────────┐
      │       │                  │           │              │
      ▼       ▼               A small      A small      Nearly all
   World   Off-chain          number,      number,      orgs, only
   state   + hash             OFTEN        RARELY       1-2 excluded
                              transacting  transacting        │
                              together     together           ▼
                               │            │              PDC too
                               ▼            ▼
                            separate       PDC
                            CHANNEL     (usually
                                        the right
                                        choice)
```

For most supply-chain applications: **1 channel + several PDCs** is the right answer. If your AI agent proposes 6 channels, ask why.

---

## 4.7 Summary

1. Channel = total separation, expensive operationally. Use it for business groups that are genuinely separate.
2. PDC = secret data lives only on specific peers, its hash sits in the shared ledger. This is your main tool for prices and commercial terms.
3. Secret data goes in via **transient**, never as an argument.
4. Files and PII **never** go on the ledger — store a hash + pointer.
5. The ledger is permanent: any decision to put data there can't be undone.

➡️ Continue to **[05 — Identity & Access](05-identity-access.md)**.
