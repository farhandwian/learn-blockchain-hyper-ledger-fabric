# 01 — Mental Model: What Fabric Is (and Isn't)

> Goal: after reading this, you should be able to explain Fabric to your boss in 2 minutes without saying the word "crypto".

---

## 1.1 The most accurate analogy

Forget Bitcoin. Picture this instead:

```
   You have 4 companies that need to work together,
   but don't fully trust each other.

   OLD WAY:                            FABRIC WAY:

   Supplier    [own DB]                Supplier    ┐
      | email/API                      Distributor ├─> [ ONE shared ledger ]
   Distributor [own DB]                Retailer    │      - every entry is signed
      | email/API                      Auditor     ┘      - can't be altered/deleted
   Retailer    [own DB]                                    - everyone has a copy
      | reports                                            - rules enforced automatically
   Auditor     [asks everyone for data]

   Problem: inconsistent data,          Problem gone: one source of truth,
   finger-pointing, manual reconciling  full history, nothing deniable
```

**Working definition:** Hyperledger Fabric is *a distributed database jointly owned by multiple organizations, where every change to the data must be approved (signed) by an agreed-upon set of parties, and the entire history of changes is stored permanently.*

That's it. No coin, no mining, no speculation.

---

## 1.2 Fabric is NOT Ethereum

This matters because 90% of blockchain content online is about Ethereum, and that intuition is **wrong** when applied to Fabric.

| | Ethereum / Bitcoin (public) | **Hyperledger Fabric** |
|---|---|---|
| Who can join | Anyone, anonymous | Only invited orgs, **clear identity** |
| Consensus | Proof of Work / Stake, competitive | **Raft** — ordering service, no competition |
| Transaction cost | Gas, paid in tokens | **No gas**, no tokens |
| Smart contract language | Solidity (special-purpose) | Plain **Go / JavaScript / Java** |
| Data | Public, everyone sees it | **Private**, per-channel, can be per-org |
| Execution order | Order → Execute | **Execute → Order → Validate** ⬅ this is a big difference |
| Throughput | ~15-30 TPS | ~500-2000 TPS (depending on tuning) |
| Finality | Probabilistic (wait N blocks) | **Deterministic** — commit = final |

> 🔑 The difference that impacts your code the most is the **Execute → Order → Validate** row. That's the whole subject of file 02.

---

## 1.3 When you actually DON'T need blockchain

This is a question you should be able to answer, because projects often get forced onto blockchain when Postgres would have been enough.

```
       Do MULTIPLE PARTIES write the data?
                    |
          NO ───────+───── YES
           |                |
           v                v
     Use a regular    Do those parties fully trust each other?
     database. Done.        |
                    YES ────+──── NO
                     |              |
                     v              v
             Use a shared DB   Do you need an audit trail that
             + API. Done.      can't be denied / altered?
                                       |
                              NO ──────+──── YES
                               |             |
                               v             v
                        regular DB + log   ✅ FABRIC MAKES SENSE
```

For supply chain, the answer is usually **yes at every branch** — which is why it's a classic Fabric use case.

⚠️ **But know its limits:** blockchain guarantees *"who claimed what, when, and that it can't be changed afterward."* It does **not** guarantee the claim is true. If a supplier enters "container temperature 4°C" when it was actually 20°C, the blockchain will store that lie neatly and permanently. This is called the **oracle problem**, covered in file 07.

---

## 1.4 Network components

There are 5 things you need to know. Here's the diagram:

```
                        ┌────────────────┐
                        │ Client App     │
                        │ (your backend) │
                        └────────────────┘
                                 │
        ┬────────────────────────┴───────────────────────┬
        │                                                │
    ORGANIZATION: Supplier            ORGANIZATION: Distributor
┌─────────────────────────────┐    ┌─────────────────────────────┐
│ ┌───────────┐ ┌───────────┐ │    │ ┌───────────┐ ┌───────────┐ │
│ │ Peer 0    │ │ CA        │ │    │ │ CA        │ │ Peer 0    │ │
│ │           │ │ (ID card) │ │    │ │ (ID card) │ │           │ │
│ │ Chaincode │ └───────────┘ │    │ └───────────┘ │ Chaincode │ │
│ │ Ledger    │               │    │               │ Ledger    │ │
│ └───────────┘               │    │               └───────────┘ │
└─────────────────────────────┘    └─────────────────────────────┘
        │                                                │
        ┴────────────────────────┬───────────────────────┴
                                 │
                     ┌──────────────────────┐
                     │ ORDERING SERVICE     │
                     │ (Raft, 3-5 nodes)    │
                     │                      │
                     │ Job is ONLY to:      │
                     │ order tx into blocks │
                     └──────────────────────┘
```

The client application (your backend) talks to Peers for endorsement, then to the Orderer to submit the transaction. The CA is only used once, upfront, to issue identities — it doesn't sit on the transaction path.

### Quick reference

| Component | Analogy | Job |
|---|---|---|
| **Peer** | A database server owned by one org | Stores the ledger, runs chaincode, validates & commits blocks |
| **Orderer** | Notary / queue | **Only** decides the order of transactions and packages them into blocks. Doesn't know what's inside them. |
| **CA** (Certificate Authority) | The HR department that issues employee badges | Issues X.509 certificates for users & peers |
| **Channel** | A group chat | A separate ledger. Orgs not in the channel truly have none of its data. |
| **Chaincode** | Stored procedure / smart contract | Go/JS code allowed to change state |
| **MSP** | The rulebook for "which badge is valid" | Maps a certificate → organization & role |

> 🔑 **The orderer executes nothing.** It's blind to transaction content. This is completely different from Ethereum, where miners execute the code.

---

## 1.5 The ledger = 2 parts (commonly misunderstood)

This concept has to be correct from day one.

```
                         LEDGER (on every peer)
   ┌────────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │  A) BLOCKCHAIN — an append-only file, the full history             │
   │                                                                    │
   │   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐               │
   │   │Block 0 │──▶│Block 1 │──▶│Block 2 │──▶│Block 3 │──▶ ...        │
   │   │genesis │   │ tx,tx  │   │ tx,tx  │   │ tx,tx  │               │
   │   └────────┘   └────────┘   └────────┘   └────────┘               │
   │                                                                    │
   │   • Cannot be altered, cannot be deleted                           │
   │   • Every block holds the hash of the previous block               │
   │   • Stores VALID **and** INVALID tx (flagged)  ⬅ important!       │
   │                                                                    │
   ├────────────────────────────────────────────────────────────────────┤
   │                                                                    │
   │  B) WORLD STATE — a key-value database, the CURRENT condition      │
   │                                                                    │
   │   ┌──────────────┬──────────────────────────────────────────────┐  │
   │   │ KEY          │ VALUE (JSON)                                 │  │
   │   ├──────────────┼──────────────────────────────────────────────┤  │
   │   │ BATCH001     │ {"status":"SHIPPED","owner":"DIST1", ...}     │  │
   │   │ BATCH002     │ {"status":"CREATED","owner":"SUP1",  ...}     │  │
   │   │ SHIPMENT-77  │ {"batches":["BATCH001"], "eta":"..."}        │  │
   │   └──────────────┴──────────────────────────────────────────────┘  │
   │                                                                    │
   │   • Can be updated & deleted (but its history stays on the chain) │
   │   • This is what chaincode reads via GetState()                    │
   │   • Backing store: LevelDB (default) or CouchDB                    │
   │                                                                    │
   └────────────────────────────────────────────────────────────────────┘

   The relationship between them:
   World State = the result of "replaying" every valid transaction on the blockchain.
   If world state gets corrupted, a peer can rebuild it from the blockchain.
```

### Practical consequences

- ⚠️ **"Delete" doesn't erase history.** `DelState()` only removes the entry from world state. The old value stays on the blockchain forever. → Never store personal/secret data on the ledger.
- 🔑 **Normal queries read world state**, not the blockchain. Fast.
- 🔑 **`GetHistoryForKey()`** reads the blockchain to see a key's history. Slow — don't use it on a hot path.
- ⚠️ **INVALID transactions still get written into a block.** So "transaction landed in a block" ≠ "transaction succeeded." This is trap #1, covered in full in file 02.

---

## 1.6 The big picture: how a Fabric application flows

```
   ┌──────────────┐
   │  Frontend    │  React / mobile
   └──────┬───────┘
          │ ordinary REST / GraphQL
   ┌──────▼───────────────────────────────────────────┐
   │  Backend App (Node.js / Go / Java)               │
   │                                                  │
   │   ┌──────────────────────────────────────┐       │
   │   │  Fabric Gateway SDK                  │       │
   │   │  - holds the identity (wallet)       │       │
   │   │  - submit / evaluate transactions    │       │
   │   └───────────────┬──────────────────────┘       │
   └───────────────────┼──────────────────────────────┘
                       │ gRPC (mTLS)
              ┌────────▼─────────┐
              │   FABRIC NETWORK │
              └──────────────────┘

   Important notes:
   • The frontend NEVER talks to Fabric directly.
   • Your backend is still a normal backend: auth, validation,
     rate limiting, logging — all still your job.
   • Fabric only replaces the "jointly trusted storage layer".
```

> 🚩 **Red flag when reviewing:** if an AI agent generates code that puts a user's private key in the frontend, or has the frontend connect directly to a peer — reject it. The Fabric identity belongs to the backend.

---

## 1.7 Summary worth pinning up

1. Fabric = a multi-organization shared database with signatures and a permanent history.
2. No tokens, no gas, no mining.
3. The ledger has 2 parts: the **blockchain** (history, permanent) + **world state** (current condition, key-value).
4. The orderer only sequences transactions, it doesn't execute them.
5. Channel = a separate ledger = the strictest privacy boundary.
6. Blockchain guarantees a *claim can't be altered*, not that the *claim is true*.

➡️ Continue to **[02 — Transaction Flow](02-transaction-flow.md)**. This is the most important file; set aside 90 minutes.
