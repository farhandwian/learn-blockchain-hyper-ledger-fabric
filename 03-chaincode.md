# 03 — Chaincode: State, Keys, Queries, Events

> Goal: you don't need to be able to write chaincode from scratch, but you must be able to **read** chaincode and spot what's wrong with it.

---

## 3.1 Chaincode anatomy

Chaincode is just an ordinary program. No special-purpose language. Its structure:

```
   ┌───────────────────────────────────────────────────────────┐
   │  CHAINCODE (one package, deployed to a channel)           │
   │                                                           │
   │   ┌─────────────────────────────────────────────────┐     │
   │   │  Contract: "BatchContract"                      │     │
   │   │                                                 │     │
   │   │   func CreateBatch(ctx, id, qty)   ──┐          │     │
   │   │   func TransferBatch(ctx, id, to)    │ called   │     │
   │   │   func ReadBatch(ctx, id)            │ from     │     │
   │   │   func QueryByOwner(ctx, owner)    ──┘ outside  │     │
   │   └─────────────────────────────────────────────────┘     │
   │                          │                                │
   │                          │ ctx.GetStub()                  │
   │                          ▼                                │
   │   ┌─────────────────────────────────────────────────┐     │
   │   │  STUB — the one and only door to the ledger      │     │
   │   │   GetState / PutState / DelState                │     │
   │   │   GetStateByRange / GetQueryResult              │     │
   │   │   CreateCompositeKey / SetEvent / GetTxID ...   │     │
   │   └─────────────────────────────────────────────────┘     │
   └───────────────────────────────────────────────────────────┘
```

> 🔑 **Everything that touches the ledger goes through the stub.** If there's chaincode that reaches outside the stub for anything (a file, the network, the clock), that's immediately suspicious (see 02.6).

---

## 3.2 A minimal chaincode example (Go)

Read this slowly. Nearly all Fabric chaincode has this same shape.

```go
type SmartContract struct {
    contractapi.Contract
}

type Batch struct {
    ID        string `json:"id"`
    Product   string `json:"product"`
    Qty       int    `json:"qty"`      // integer, not float
    Owner     string `json:"owner"`    // owning MSP ID
    Status    string `json:"status"`
    UpdatedAt string `json:"updatedAt"`
}

func (s *SmartContract) CreateBatch(ctx contractapi.TransactionContextInterface,
    id string, product string, qty int) error {

    // 1) VALIDATE INPUT
    if id == "" || product == "" {
        return fmt.Errorf("id and product are required")
    }
    if qty <= 0 {
        return fmt.Errorf("qty must be > 0, got %d", qty)
    }

    // 2) CHECK CALLER IDENTITY  ← the thing AI most often forgets!
    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }
    if mspID != "SupplierMSP" {
        return fmt.Errorf("only the Supplier may create a batch, not %s", mspID)
    }

    // 3) CHECK IT DOESN'T EXIST YET (idempotency / prevent overwrite)
    existing, err := ctx.GetStub().GetState(id)
    if err != nil {
        return fmt.Errorf("failed to read state: %v", err)
    }
    if existing != nil {
        return fmt.Errorf("batch %s already exists", id)
    }

    // 4) DETERMINISTIC TIME
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

    // 5) WRITE
    if err := ctx.GetStub().PutState(id, b); err != nil {
        return err
    }

    // 6) EMIT AN EVENT for off-chain consumers
    return ctx.GetStub().SetEvent("BatchCreated", b)
}
```

### These 6 steps are your review template

```
   ┌───┬──────────────────────────────┬────────────────────────────────┐
   │ # │ Step                         │ What if it's missing?          │
   ├───┼──────────────────────────────┼────────────────────────────────┤
   │ 1 │ Validate input               │ Garbage data enters the ledger │
   │   │                              │ PERMANENTLY. Can't be deleted. │
   │ 2 │ Check caller identity        │ 🚨 Anyone can do anything      │
   │ 3 │ Check the state precondition │ Silent overwrite, or an        │
   │   │                              │ illegal status transition      │
   │ 4 │ Time via GetTxTimestamp      │ Non-determinism                │
   │ 5 │ PutState once, at the end    │ (see 02.7)                     │
   │ 6 │ SetEvent                     │ Off-chain systems never learn  │
   │   │                              │ about the change               │
   └───┴──────────────────────────────┴────────────────────────────────┘
```

🚩 **Step 2 is the one most often missing from AI-generated code.** Fabric only guarantees that *the caller's identity is valid*; it has no idea about your business rule of "only the supplier may create." You have to write that yourself.

---

## 3.3 Stub API you need to know

| Function | Purpose | Note |
|---|---|---|
| `GetState(key)` | read one key | goes into the read-set |
| `PutState(key, val)` | write one key | goes into the write-set |
| `DelState(key)` | remove from world state | history stays on the blockchain |
| `GetStateByRange(start, end)` | fetch a range of keys | ⚠️ phantom read |
| `CreateCompositeKey(prefix, attrs)` | build a structured key | see 3.4 |
| `GetStateByPartialCompositeKey` | query by prefix | ⚠️ phantom read |
| `GetQueryResult(query)` | JSON rich query | **CouchDB only**, ⚠️ phantom read |
| `GetHistoryForKey(key)` | change history of a key | slow, read-only |
| `GetTxTimestamp()` | time (deterministic) | ✅ use this |
| `GetTxID()` | transaction ID (deterministic) | ✅ fine for a unique ID |
| `SetEvent(name, payload)` | emit an event | **1 event per transaction**, last one wins |
| `GetPrivateData(coll, key)` | read private data | see file 04 |
| `InvokeChaincode(name, args, ch)` | call another chaincode | careful, see 3.8 |

⚠️ `SetEvent` may only be called once per transaction. If called twice, the first call silently disappears — no error.

---

## 3.4 Key design — an architectural decision, not a detail

World state is key-value. There are no tables, no JOINs, no automatic indexes. **Key design = query design.**

### Composite keys

```
   CreateCompositeKey("owner~batch", ["DIST1", "BATCH007"])
                          │              │        │
                       objectType     attr 1   attr 2

   Produces (internally):
   \x00owner~batch\x00DIST1\x00BATCH007\x00

   Effect: keys are sorted lexicographically, so they can be scanned by prefix:

   \x00owner~batch\x00DIST1\x00BATCH003\x00   ┐
   \x00owner~batch\x00DIST1\x00BATCH007\x00   ├─ GetStateByPartialCompositeKey
   \x00owner~batch\x00DIST1\x00BATCH019\x00   ┘  ("owner~batch", ["DIST1"])
   \x00owner~batch\x00DIST2\x00BATCH001\x00
```

### A common pattern: hand-rolled indexes

```
   ┌───────────────────────────────────────────────────────────┐
   │  MAIN DATA                                                │
   │    BATCH007  →  {"product":"Coffee","owner":"DIST1", ...} │
   │                                                           │
   │  INDEX (empty value, the KEY is what matters)             │
   │    owner~batch : DIST1 : BATCH007   →  0x00               │
   │    status~batch: SHIPPED: BATCH007  →  0x00               │
   └───────────────────────────────────────────────────────────┘

   ⚠️ This index is NOT automatic. The chaincode must:
      - create the index entry on create
      - DELETE the old index entry & create a new one on update
```

🚩 **Classic red flag:** chaincode changes `owner` from DIST1 to DIST2 but forgets to `DelState` the old index entry. The result: a query for "batches owned by DIST1" still returns a batch that has already moved on. Look for this: for every indexed field, is there a `DelState(oldIndex)` on the update path?

---

## 3.5 LevelDB vs CouchDB

This decision is made when the network is set up and is **hard to change later**.

```
   ┌────────────────────────┬────────────────────────────────────┐
   │  LevelDB (default)     │  CouchDB                           │
   ├────────────────────────┼────────────────────────────────────┤
   │  Pure key-value        │  Document store (JSON)             │
   │  Query: by key only    │  Query: rich queries (Mongo-like)  │
   │         & range        │         WHERE status=X AND owner=Y │
   │  Fast                  │  2-4x slower                        │
   │  Embedded, zero ops    │  A separate container to operate    │
   │  Value can be anything │  Value MUST be valid JSON           │
   └────────────────────────┴────────────────────────────────────┘
```

### ⚠️ If you choose CouchDB: an INDEX is MANDATORY

A rich query with no index does a **full scan**. Fine with 100 records in dev. A timeout with 5 million records in production.

Indexes are declared as JSON files inside the chaincode package:

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

🚩 **Red flag when reviewing:** `GetQueryResult` shows up in the chaincode but there's no `META-INF/statedb/couchdb/indexes/` folder. That's a performance time bomb.

🧪 **Test:** load 100,000 records and measure rich-query latency. If it's over 1 second, the index is wrong or unused.

---

## 3.6 Events — the bridge to the off-chain world

This is an architecture pattern you will **almost certainly need** in a supply-chain application.

```
   ┌──────────────┐
   │  CHAINCODE   │  SetEvent("BatchShipped", payload)
   └──────┬───────┘
          │ (the event rides along in the block, delivered on a VALID commit)
          ▼
   ┌────────────────────────────────────────────────────────┐
   │  LISTENER (your own Node.js/Go service)                │
   │                                                        │
   │    - subscribes to block/chaincode events               │
   │    - SAVES the last processed block number              │
   │      (a checkpoint) → so it can resume after a restart  │
   └───────────┬────────────────────────────────────────────┘
               │
     ┌─────────┼──────────┬──────────────┐
     ▼         ▼          ▼              ▼
 ┌────────┐ ┌──────┐  ┌────────┐   ┌──────────┐
 │Postgres│ │Elastic│ │ Kafka  │   │Notifications│
 │(reports)│(search)│(integr.)│   │(email/WA) │
 └────────┘ └──────┘  └────────┘   └──────────┘

   All reporting, dashboards, search → come from here.
   DON'T build a dashboard that queries the ledger directly.
```

Why this matters:
- Ledger queries are slow and a poor fit for aggregation.
- An off-chain DB can be indexed, joined, and cached however you like.
- The ledger stays the source of truth; the off-chain DB can be rebuilt from block 0 at any time.

🚩 Red flag: an event listener with no **checkpoint**. If the service restarts, any events that passed by are lost forever and the off-chain data drifts out of sync.

⚠️ Events are only delivered for **VALID** transactions. That's actually a feature — you don't need to filter them yourself.

---

## 3.7 The mistake of storing numbers wrong

```go
// ⛔ float — rounding can differ across platforms/versions
type Item struct { Price float64 }

// ✅ integer, in the smallest unit
type Item struct { PriceCents int64 }   // $150.005 → 15000.5 cents, use an integer-safe unit
```

This is a real problem in supply chain (price, weight, volume). The rule is the same as in financial systems: **never use floats for money or quantities that must be exact.**

---

## 3.8 Other things that often go wrong

| Anti-pattern | Why it's bad |
|---|---|
| `return nil` on a failure condition | The transaction gets recorded as VALID even though it did nothing. Errors must be returned as errors. |
| Storing an entire array in 1 key | Hot key + value size bloat + MVCC conflicts |
| Values > a few hundred KB | Blocks get big, replication slows down. Store a hash, keep the file off-chain. |
| `InvokeChaincode` across channels, then writing | The read-set from the other channel is **not** validated. Only safe for reads. |
| Business logic in the client, chaincode is just CRUD | The client can be modified. Rules that bind all parties MUST live in chaincode. |
| No `docType` in the JSON | Rich queries struggle to distinguish document types |
| Function name doesn't match what the client calls | "function not found" error at runtime |

> 🔑 The **"business logic in the client"** point is the most conceptually important. The test question: *"If one organization modifies their own client application, could they break a business rule?"* If yes, the rule is in the wrong place.

---

## 3.9 Chaincode review checklist

```
  [ ] No time.Now / rand / http / getenv / map iteration
  [ ] Every write function checks the caller's MSP ID or attributes
  [ ] All inputs are validated (empty, negative, format, length)
  [ ] Existence checked before create; status checked before transition
  [ ] Status transitions follow the agreed state machine
  [ ] No PutState followed by GetState on the same key
  [ ] Queries are never used to decide a write
  [ ] Composite-key indexes are updated AND deleted when data changes
  [ ] META-INF/statedb/couchdb/indexes exists if rich queries are used
  [ ] Important numbers use integers, not floats
  [ ] Errors are returned as errors, never a silent return
  [ ] SetEvent for every meaningful change (max 1 per tx)
  [ ] No personal data / large files stored in state
```

➡️ Continue to **[04 — Data Privacy](04-data-privacy.md)**.
