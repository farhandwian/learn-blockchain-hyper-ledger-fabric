# 07 — Supply Chain Case Study

A worked example: coffee, farmer → café. Copy this structure for your own design — it's the artifact to hand an AI agent instead of "build me a supply chain chaincode."

## Separate the physical flow from the data flow

The biggest conceptual mistake in these projects:

```
PHYSICAL WORLD:  Farmer → truck → Processor → truck → Distributor → Café
                 (blockchain knows NOTHING about any of this)
                          │
                 INPUT POINT: a human scanning a QR code, an IoT
                 device, an ERP integration
                 ⚠️ THIS IS THE WEAKEST LINK IN THE WHOLE SYSTEM
                          │
LEDGER:          a record of CLAIMS — who said what, undeniable, not
                 necessarily TRUE
```

**Mitigating the oracle problem** (this needs to be in your design doc, not just the code):

| Technique | Example |
|---|---|
| Two-party confirmation | Receipt valid only if sender AND receiver both endorse |
| Signed IoT sensors | Device has its own cert, signs its own readings |
| Third-party certs | Org's organic certificate, hash on-chain |
| Off-chain anomaly detection | 500kg out of a warehouse that received 300kg → alert |

If a project's design doc never mentions this, that's a red flag at the project level.

## State machine — draw this before any code

```
HARVESTED → IN_TRANSIT → RECEIVED → PROCESSED → ... → SOLD (terminal)
                              │
                          REJECTED
   RECALLED ← reachable from any state, only by Regulator/original owner
```

Every arrow = one function + one auth rule + one test. Every **missing** arrow = a required negative test (`HARVESTED → SOLD` directly must fail, `SOLD → IN_TRANSIT` must fail, etc).

🚩 The most common and most damaging bug: chaincode that writes `batch.Status = newStatus` without validating the transition is legal.

```go
var allowed = map[string][]string{
    "HARVESTED": {"IN_TRANSIT", "RECALLED"},
    "IN_TRANSIT": {"RECEIVED", "REJECTED", "RECALLED"},
    "RECEIVED": {"PROCESSED", "RECALLED"},
    "SOLD": {},
}
```

## Data placement matrix

| Field | Where | Why |
|---|---|---|
| batchId, status, qty, owner | World state | needs verification by everyone |
| parentBatchIds | World state | traceability — the actual point of the system |
| price, terms | PDC | commercial secret between 2 parties |
| farmer name, national ID | Off-chain | personal data, must be deletable |
| photos, PDFs | Off-chain | size; ledger holds the hash |
| per-second sensor data | Off-chain | volume; ledger holds only violations |

🚩 If an AI puts `farmerName` or per-second `temperature` into `PutState`, reject it and point to this table.

### `parentBatchIds` — the traceability mechanism

```
BATCH#A(farmer A) ┐
BATCH#B(farmer B) ├→ BATCH#X (processed, parentBatchIds:[A,B,C])
BATCH#C(farmer C) ┘        └→ BATCH#X1 → Café 1

Recall: B is contaminated → trace FORWARD: B → X → X1 → Café 1 only.
Consumer scans QR at Café 1 → trace BACKWARD: X1 → X → {A,B,C}.
```

Without a parent/child relation, "traceability" is just a log with no actual value. Don't do unbounded recursion in chaincode — cap the depth, or build the graph off-chain from events.

## System architecture

```
Mobile/Web clients → Backend API (per org: auth, input validation,
                       MVCC retry, holds the Fabric wallet)
                            │
              ┌─────────────┴──────────────┐
        Fabric Network                Off-chain store
        (channel + PDC)               (Postgres/S3/Redis)
              │
        chaincode events → Event Listener (with checkpoint) → off-chain DB
```

Things to notice: the backend is still a normal backend (Fabric replaces nothing about auth/validation/logging); all heavy reads go through the off-chain DB, never the ledger directly; the wallet never leaves the server; IoT data gets filtered/aggregated before anything touches the ledger.

## Endorsement policy per function

| Function | Policy | Why |
|---|---|---|
| `CreateBatch` | `OR(Farmer)` | self-reported, harms no one |
| `ShipBatch` | `AND(owner, recipient)` | recipient shouldn't be shipped-to blind |
| `ReceiveBatch` | `AND(sender, receiver)` | two-sided confirmation = oracle mitigation |
| `SetPrice` (PDC) | `AND(seller, buyer)` | price is an agreement |
| `RecallBatch` | `OR(Auditor, Farmer)` | must move fast, can't be blocked by the party at fault |

## Easy to forget

- **Correcting mistakes**: no UPDATE/DELETE exists. Need a `CorrectBatch` function that writes a *new* transaction with a reason — old value stays visible in history (a feature, not a bug).
- **Split/merge**: validate quantity conservation (children's total == parent's total).
- **One peer down**: does the system still work? Depends entirely on the endorsement policy.
- **Who actually runs each peer?** If one vendor hosts every org's peer, there's no real reason to use blockchain — Postgres + an audit log would do. Fabric's value only shows up when each org genuinely runs its own node.

➡️ **[08 — Testing & Review](08-testing-and-review.md)**
