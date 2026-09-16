# 07 — Case Study: A Supply Chain Application

> This section pulls everything together. Case: a **coffee supply chain**, from farmer to café.
>
> This is the shape of design document **you** should produce before ever telling an AI agent to write code.

---

## 7.1 Actors & organizations

```
   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
   │   FARMER   │──▶│ PROCESSOR  │──▶│DISTRIBUTOR │──▶│    CAFÉ    │
   │ (FarmerMSP)│   │(ProcMSP)   │   │ (DistMSP)  │   │(RetailMSP) │
   └────────────┘   └────────────┘   └────────────┘   └────────────┘
          │                │                │                │
          └────────────────┴────────┬───────┴────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │ REGULATOR/AUDITOR │  (AuditMSP)
                          │  read-only        │
                          └───────────────────┘
```

---

## 7.2 The most important thing: separate the physical flow from the data flow

This is the biggest conceptual mistake in blockchain supply-chain projects.

```
   ══════════════ THE PHYSICAL WORLD ══════════════
        sacks of coffee genuinely move around
   Farmer ──truck──▶ Processor ──truck──▶ Distributor ──▶ Café
        │              │                  │             │
        │  ⚠️ Blockchain knows NOTHING about any of this │
        │              │                  │             │
        ▼              ▼                  ▼             ▼
   ┌─────────────────────────────────────────────────────────┐
   │              INPUT POINT (the oracle)                    │
   │   A human scanning a QR code / an IoT device / an ERP    │
   │   integration                                            │
   │   ⚠️ THIS IS THE WEAKEST POINT IN THE WHOLE SYSTEM       │
   └─────────────────────────────────────────────────────────┘
        │              │                  │             │
        ▼              ▼                  ▼             ▼
   ══════════════ THE LEDGER ══════════════
     a record of CLAIMS: who said what, when — and it can't be changed
```

> 🔑 **Blockchain doesn't make data true. It makes data undeniable.**
>
> If a farmer enters "organic" when it isn't, the ledger will store that lie permanently and very neatly.

### Mitigating the oracle problem (must be part of the design)

| Technique | Example |
|---|---|
| Two-party confirmation | Receipt is only valid if the sender **and** the receiver both endorse it (`AND`) — both parties would have to lie together |
| Signed IoT sensors | Devices have their own certificates; temperature/GPS data is signed by the device |
| Third-party verification | An organic certificate from a certifying body, its hash stored on the ledger |
| Off-chain anomaly detection | 500kg leaving a warehouse that only received 300kg → alert |
| Economic consequences | On-chain reputation/penalties for claims later proven false |

🚩 If your project's design document doesn't discuss this at all, that's a red flag at the project level, not the code level.

---

## 7.3 State machine — draw this first, before coding

```
                    ┌──────────┐
     CreateBatch    │ HARVESTED│  (Farmer)
     ───────────▶   └────┬─────┘
                         │ ShipBatch(→Processor)
                         │ policy: AND(Farmer, Proc)
                         ▼
                    ┌──────────┐
                    │IN_TRANSIT│
                    └────┬─────┘
                         │ ReceiveBatch
                         │ policy: AND(sender, receiver)
                         ▼
                    ┌──────────┐        ┌──────────┐
                    │ RECEIVED │───────▶│ REJECTED │ (failed quality check)
                    └────┬─────┘        └──────────┘
                         │ ProcessBatch (Processor)
                         ▼
                    ┌──────────┐
                    │ PROCESSED│  ← may be split into several batches
                    └────┬─────┘
                         │ ShipBatch ... (cycle repeats)
                         ▼
                       ....
                    ┌──────────┐
                    │   SOLD   │  (terminal)
                    └──────────┘

              ┌──────────┐
              │ RECALLED │ ← reachable from any state, only by
              └──────────┘   the Regulator or the original owner
```

### Why this diagram matters to you

Every arrow = one chaincode function + one authorization rule + one test.
Every **missing arrow** = one negative test.

🧪 Negative tests that must exist:
- `HARVESTED → SOLD` directly (skipping states) must **fail**
- `SOLD → IN_TRANSIT` (going backward) must **fail**
- `RECEIVED` by an org that isn't the shipment's destination must **fail**
- `ShipBatch` on a batch that's already `IN_TRANSIT` must **fail**

🚩 Red flag: chaincode that writes `batch.Status = newStatus` without validating that the old→new transition is legal. This is the most common and most damaging bug in supply-chain applications.

The correct shape:

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

## 7.4 Data model & key design

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  WORLD STATE (public to every channel member)                   │
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
   │  INDEX (composite key, empty value)                             │
   │  owner~batch : DistMSP : BATCH#abc                              │
   │  status~batch: IN_TRANSIT : BATCH#abc                           │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │  PRIVATE DATA COLLECTION "pricing" (only the transacting parties)│
   ├─────────────────────────────────────────────────────────────────┤
   │  BATCH#<uuid>               { pricePerKgIdr: 85000,             │
   │                               currency:"IDR", terms:"NET30" }   │
   │                               ↑ integer, not float!             │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │  OFF-CHAIN (S3 / MinIO)                                         │
   ├─────────────────────────────────────────────────────────────────┤
   │  Batch photos, organic certificate PDFs, delivery notes,        │
   │  raw sensor data, farmers' personal data (name, national ID,   │
   │  bank account) → the ledger only holds a HASH + URI             │
   └─────────────────────────────────────────────────────────────────┘
```

### `parentBatchIds` — the heart of traceability

This is what makes "trace the origin backward" possible:

```
        BATCH#A (Farmer A, 100kg)  ┐
        BATCH#B (Farmer B,  80kg)  ├──▶ BATCH#X (Processor, 150kg roasted)
        BATCH#C (Farmer C,  50kg)  ┘         parentBatchIds: [A, B, C]
                                                     │
                                                     ├──▶ BATCH#X1 (50kg) → Café 1
                                                     └──▶ BATCH#X2 (100kg) → Café 2
                                                            parentBatchIds: [X]

   Recall: Farmer B's coffee is contaminated.
     → trace FORWARD from B: B → X → {X1, X2} → Café 1 & Café 2
     → recall only those 2 cafés, not the entire network.

   A consumer scans the QR code at Café 1:
     → trace BACKWARD from X1: X1 → X → {A, B, C} → the 3 original farmers
```

🚩 Red flag: a data model with no parent/child relationship. Without it, "traceability" is just an ordinary log with none of the actual value.

⚠️ This traversal can get deep. Don't do unbounded recursion inside chaincode — cap the depth, or better yet: build the graph in an off-chain DB from events, and traverse it there.

---

## 7.5 Data-placement decision matrix

Build this table for **every field** in your system:

| Field | Where | Why |
|---|---|---|
| batchId, status, qtyKg | World state | needs to be seen & verified by everyone |
| owner (MSP ID) | World state | the basis for authorization |
| parentBatchIds | World state | traceability, the core value of the system |
| certificate hash | World state | proof the document hasn't been altered |
| price, terms | **PDC** | a commercial secret between 2 parties |
| farmer name & national ID | **Off-chain** | personal data, must be deletable |
| photos, PDFs | **Off-chain** | size; ledger only holds the hash |
| per-second IoT temperature | **Off-chain** | volume; ledger only holds aggregates/violations |
| real-time GPS | **Off-chain** | volume + driver privacy |
| total stock, reports | **Off-chain (from events)** | aggregation = a hot key if kept on-chain |

🚩 If an AI agent puts `farmerName`, `photoBase64`, or per-second `temperature` into `PutState` — reject it and point to this table.

---

## 7.6 Full system architecture

```
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Farmer Mobile│  │Processor Web │  │Auditor Portal│
  │  (QR scan)   │  │              │  │  (read-only) │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         └─────────────────┼─────────────────┘
                           │ REST / GraphQL + JWT
         ┌─────────────────▼─────────────────────────────────┐
         │  BACKEND API (per organization)                    │
         │                                                   │
         │  • User auth (JWT) — has NOTHING to do with        │
         │    the Fabric identity                             │
         │  • Input validation (first layer)                  │
         │  • MVCC-conflict retry  ⬅ MANDATORY                │
         │  • Fabric identity wallet (secret, on the server)  │
         │  • Upload file → S3, compute hash → submit tx      │
         └───────┬───────────────────────────┬───────────────┘
                 │ Gateway SDK (gRPC/mTLS)   │ read
                 ▼                           ▼
  ┌──────────────────────────────┐  ┌──────────────────────┐
  │      FABRIC NETWORK          │  │  OFF-CHAIN STORE      │
  │  ┌────────────────────────┐  │  │                      │
  │  │ Channel: coffee-chain  │  │  │  Postgres (queries,  │
  │  │  chaincode: coffeecc   │  │  │   reports, traceability│
  │  │  PDC: pricing          │  │  │   graph)             │
  │  └────────────────────────┘  │  │  S3 (docs, photos)   │
  └──────────┬───────────────────┘  │  Redis (cache)       │
             │ chaincode events     └──────────▲───────────┘
             ▼                                 │
  ┌────────────────────────────────────────────┴───────────┐
  │  EVENT LISTENER  (a separate service, with a checkpoint)│
  │   BatchCreated / BatchShipped / BatchReceived / ...     │
  │   → writes to Postgres, sends notifications, triggers ERP│
  └────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────┐
  │  IoT GATEWAY (optional)                                │
  │   temperature/GPS sensors → aggregation + violation    │
  │   detection → only VIOLATIONS go onto the ledger,       │
  │   not every reading                                     │
  └────────────────────────────────────────────────────────┘
```

### What to notice in this diagram

1. **The backend is still an ordinary backend.** Auth, rate limiting, validation, logging — all still your job. Fabric replaces none of it.
2. **All heavy reads go through the off-chain DB**, never a direct ledger query.
3. **MVCC retry lives in the backend**, and it must be tested.
4. **The wallet lives on the server**, never on the client.
5. **IoT filters first** — don't write 86,400 data points per day per container onto the ledger.

---

## 7.7 Endorsement policy per function

This is a table you must decide, not the AI:

| Function | Policy | Reason |
|---|---|---|
| `CreateBatch` | `OR(FarmerMSP)` | a farmer recording their own harvest; harms no one else |
| `ShipBatch` | `AND(owner, recipient)` | the recipient shouldn't be "shipped to" without knowing |
| `ReceiveBatch` | `AND(sender, receiver)` | two-sided confirmation = oracle-problem mitigation |
| `SetPrice` (PDC) | `AND(seller, buyer)` | price is an agreement |
| `RecallBatch` | `OR(AuditMSP, FarmerMSP)` | a recall must move fast, can't be blocked by the party at fault |
| `RecordViolation` (IoT) | `AND(owner, AuditMSP)` | an owner shouldn't be able to cover up their own violation |
| All reads | — | evaluate, no policy needed |

Plus **state-based endorsement**: every time ownership changes hands, that batch's key-level policy moves with it (see 05.4). Effect: a party that has already released the goods can no longer change that data.

---

## 7.8 Things that usually get forgotten in supply-chain projects

```
  [ ] Correcting input mistakes — humans WILL enter things wrong.
      There's no UPDATE/DELETE in blockchain.
      → you need a CorrectBatch function that records the correction
        as a NEW transaction, with a reason + who made it.
        The old value stays visible in history. That's a feature,
        not a bug.

  [ ] Splitting & merging batches — 100kg becomes 3 sacks; 3 farmers'
      batches become 1 lot. Quantity conservation must be validated
      (children's total == parent's total).

  [ ] Onboarding a new org — a new distributor joins 6 months from now.
      Needs a channel config update + endorsement policy change +
      re-approval. Who has the authority? How long does it take?

  [ ] One peer down — does the system still work?
      Depends on the endorsement policy. AND(everyone) = no.

  [ ] Realistic volume — how many transactions per day?
      1,000 batches/day x 5 transitions = 5,000 tx/day. Very light.
      100,000 QR scans/day? That's a read, never touches the ledger. Fine.
      Per-second sensor data? ⛔ Must be filtered first.

  [ ] Backup & disaster recovery — a peer can rebuild world state from
      the blockchain, but PRIVATE DATA is not on the blockchain.
      If every peer holding a collection is lost, that data is GONE.
      → private data needs its own separate backup.

  [ ] Cost & ops — who actually runs each org's peer?
      If one vendor hosts every peer, is there still a reason to use
      blockchain at all?  ⬅ an honest question worth asking
```

That last point is serious: many "blockchain projects" end up as a single company running every node. In that case, Postgres with an audit log would technically be enough. Fabric's value only shows up when **each org genuinely runs its own peer**.

---

## 7.9 Design exercise

Before moving to file 08, try doing this yourself (30 minutes):

1. Draw the state machine for the product you'll actually be working on.
2. Build the authorization matrix (functions × organizations).
3. Build the data-placement matrix (fields × on-chain/PDC/off-chain).
4. Decide the endorsement policy per function + the reasoning.
5. List 10 negative tests for transitions that **must not** happen.

These five artifacts are **input for your AI agent**. With them, the quality of the generated code will be far better than the prompt "build me a supply-chain chaincode."

➡️ Continue to **[08 — Testing & Review](08-testing-and-review.md)**.
