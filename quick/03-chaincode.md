# 03 — Chaincode Reference

Use this when reading chaincode and you don't recognize an API call.

## Shape of a chaincode function

Every write function should have these 6 steps. Missing #2 (auth check) is the most common AI mistake — see [02](02-transaction-flow.md#4-no-authorization-check).

```go
func (s *SmartContract) CreateBatch(ctx contractapi.TransactionContextInterface,
    id string, product string, qty int) error {

    // 1) VALIDATE INPUT
    if id == "" || product == "" {
        return fmt.Errorf("id and product are required")
    }
    if qty <= 0 {
        return fmt.Errorf("qty must be > 0, got %d", qty)
    }

    // 2) CHECK CALLER IDENTITY — the step AI most often skips
    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }
    if mspID != "SupplierMSP" {
        return fmt.Errorf("only the Supplier may create a batch, not %s", mspID)
    }

    // 3) CHECK IT DOESN'T EXIST YET
    existing, err := ctx.GetStub().GetState(id)
    if err != nil {
        return err
    }
    if existing != nil {
        return fmt.Errorf("batch %s already exists", id)
    }

    // 4) DETERMINISTIC TIME — never time.Now()
    ts, err := ctx.GetStub().GetTxTimestamp()
    if err != nil {
        return err
    }

    batch := Batch{ID: id, Product: product, Qty: qty, Owner: mspID,
        Status: "CREATED", UpdatedAt: ts.AsTime().UTC().Format(time.RFC3339)}
    b, _ := json.Marshal(batch)

    // 5) WRITE — once, at the end
    if err := ctx.GetStub().PutState(id, b); err != nil {
        return err
    }

    // 6) EVENT for off-chain consumers
    return ctx.GetStub().SetEvent("BatchCreated", b)
}
```

## Stub API cheat sheet

| Function | Use | Note |
|---|---|---|
| `GetState(key)` | read one key | |
| `PutState(key, val)` | write one key | doesn't update your own subsequent `GetState` in the same tx — use a local variable |
| `DelState(key)` | remove from world state | history stays on-chain forever |
| `GetStateByRange` / `GetQueryResult` | scan / rich query | ⚠️ never use the result to decide a write (phantom reads, see file 02) |
| `CreateCompositeKey(prefix, attrs)` | structured, scannable keys | see below |
| `GetTxTimestamp()` / `GetTxID()` | deterministic time / ID | ✅ always use these, never `time.Now()`/`rand` |
| `SetEvent(name, payload)` | emit an event | max 1 per tx, last call wins |
| `GetPrivateData(coll, key)` | read private data | see [04](04-data-privacy.md) |

## Keys: no tables, no JOIN, no automatic index

World state is pure key-value. **Key design = query design.**

```
CreateCompositeKey("owner~batch", ["DIST1", "BATCH007"])
→ \x00owner~batch\x00DIST1\x00BATCH007\x00   (sorted, prefix-scannable)
```

Common pattern — a hand-rolled index (empty value, only the key matters):
```
owner~batch : DIST1 : BATCH007  →  0x00
```

🚩 **Red flag:** chaincode changes `owner` but doesn't delete the old index entry. Queries for the old owner keep returning batches that already moved. Check: does every indexed field get `DelState`'d on update?

## LevelDB vs CouchDB

- **LevelDB** (default): key/range queries only. Fast, no ops overhead.
- **CouchDB**: rich JSON queries (`WHERE status=X`). 2-4x slower. **Requires an index file** (`META-INF/statedb/couchdb/indexes/*.json`) or it does a full table scan — fine with 100 records, a timeout with 5 million. If you see `GetQueryResult` in chaincode with no index folder, that's a performance time bomb.

## Numbers and money

```go
// ⛔ rounding can differ across platforms
Price float64

// ✅ integer, smallest unit
PriceCents int64
```

## Events → off-chain

Chaincode `SetEvent` → your listener service (with a **checkpoint**, so a restart doesn't lose events) → Postgres/search/notifications. Never build a dashboard that queries the ledger directly — it's slow and bad at aggregation. Events only fire for VALID transactions, so you don't need to filter.

## Other common mistakes

| Anti-pattern | Why it's bad |
|---|---|
| `return nil` on failure | Recorded as VALID even though nothing happened |
| Business logic in the client, chaincode is just CRUD | Client is modifiable — rules binding all parties must live in chaincode |
| Value > a few hundred KB | Store a hash off-chain instead |

## Review checklist

```
[ ] No time.Now / rand / http / getenv / unsorted map iteration
[ ] Every write function checks MSP ID and, where relevant, ownership
[ ] All inputs validated; existence/status checked before write
[ ] No PutState followed by GetState on the same key
[ ] Query results never used to decide a write
[ ] Composite-key indexes deleted on update, not just added on create
[ ] CouchDB index exists if rich queries are used
[ ] Integers for money/quantity, not floats
[ ] SetEvent for every meaningful change
[ ] No personal data or files in state — hash + URI only
```

➡️ **[04 — Data Privacy](04-data-privacy.md)**
