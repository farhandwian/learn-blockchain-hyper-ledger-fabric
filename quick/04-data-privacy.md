# 04 — Data Privacy

The question for every field: **who's allowed to see this, and is it okay for it to be permanent?**

## Three levels — pick the simplest one that's enough

| Level | What | Cost |
|---|---|---|
| **Channel** | A fully separate ledger. Orgs outside it have zero data, not just "can't view." | High — separate chaincode/config per channel, no easy cross-channel tx. Don't create one per org pair. |
| **Private Data Collection (PDC)** | Same channel, but the actual data only replicates to specific orgs. Only its hash goes on the shared ledger. | Low — slightly more chaincode complexity (transient data) |
| **Off-chain + hash** | Real data lives in S3/DB. Ledger holds only a hash + pointer. | Lowest — mandatory for files/PII |

For most supply-chain apps: **1 channel + a few PDCs.** If an AI agent proposes 6 channels, ask why.

## Private Data Collections (the one you'll use most)

Example: Supplier sells to Distributor at a price Retailer/Auditor shouldn't see.

- **On the shared ledger:** `BATCH007 → {product, qty, owner}` + `hash(privateData) = a3f9c1e8...`
- **Only on Supplier's and Distributor's peers:** `BATCH007 → {price: 10000, terms: "NET30"}`

Retailer/Auditor see the hash — proof the data exists and hasn't been tampered with, without seeing the content. An auditor can request the real data out-of-band and verify it against the on-chain hash.

**Critical: secret data must go through `transient`, never a normal argument** — arguments are permanently recorded on the ledger for everyone.

```javascript
await contract.createTransaction('SellBatch')
  .setTransient({ price_details: Buffer.from(JSON.stringify({price: 10000})) })
  .setEndorsingOrganizations('SupplierMSP', 'DistributorMSP')  // ← mandatory
  .submit('BATCH007', 'DIST1');
```

```go
transient, _ := ctx.GetStub().GetTransient()
ctx.GetStub().PutPrivateData("priceCollection", batchID, transient["price_details"])
```

🚩 Red flags: secret data sent as a plain argument instead of transient (unfixable leak — it's permanent); endorsing orgs not restricted to collection members (a non-member peer can't read the data, so its result will differ → transaction fails).

## On-chain vs off-chain

| ✅ OK on the ledger | ⛔ Never on the ledger |
|---|---|
| IDs, status, timestamps | Photos, PDFs, any file |
| Quantities, product codes | Names, national IDs, phone, email |
| Hashes of documents | Credentials, API keys |
| MSP ID (who did what) | High-frequency raw sensor data |

**Hash + pointer pattern:** upload the file to S3 → compute SHA-256 → submit `{docId, hash, uri}` to the chain. Verify later by re-hashing and comparing. This is also the answer to "what about GDPR/privacy law" — personal data lives off-chain and can be deleted; the on-chain hash becomes meaningless once the original is gone.

## Decision rule

```
Everyone on the channel needs to see it?
  YES → is it a file/PII?  YES → off-chain+hash   NO → world state
  NO  → PDC (almost always the right answer for "some orgs only")
```

➡️ **[05 — Identity & Access](05-identity-access.md)**
