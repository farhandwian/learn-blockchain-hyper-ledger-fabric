# 02 — Transaction Flow: Execute → Order → Validate

This explains *why* the 5 silent failures in the README happen. Read this once, properly.

## Why the flow is weird

A normal DB: `Client → execute → commit → done`.

Fabric's problem: who's trusted to execute it? Solution: **several orgs execute the same transaction independently, then compare results.** Only if they match does it get committed.

```
  PHASE 1 — EXECUTE   Client sends a proposal to several peers.
                       Each peer SIMULATES the chaincode — nothing is
                       written yet. Result: a read-set + write-set + signature.
  PHASE 2 — ORDER      Client sends the signed tx to the Orderer, which
                       only sequences it into a block. Doesn't look at content.
  PHASE 3 — VALIDATE   Every peer checks: (a) enough signatures? (b) is the
                       read-set still valid? → commit as VALID, or reject
                       as INVALID. ⚠️ Both outcomes go into the block.
```

The client **compares the results from all peers before sending anything**. If your chaincode isn't deterministic, the results differ and the transaction dies right there — often with a confusing error, not a clean one.

## Read-set / write-set, concretely

```go
b := GetState("BATCH001")     // read
b.owner = "DIST"
PutState("BATCH001", b)       // write
```

The peer's simulation returns:
- **read-set**: `BATCH001 → version (5,2)` — just the version, not the value
- **write-set**: `BATCH001 → {"owner":"DIST"}`

At commit, the peer asks: *"Is BATCH001 still at version (5,2)?"* Yes → apply, VALID. No → `MVCC_READ_CONFLICT`, INVALID. This is optimistic locking, like `UPDATE ... WHERE version = 5`.

---

## The 5 failures, in detail

### 1. Non-determinism

Chaincode runs independently on multiple peers — results must match exactly.

| ⛔ Forbidden | Why | ✅ Instead |
|---|---|---|
| `time.Now()` | differs per peer | `stub.GetTxTimestamp()` |
| `rand`, `uuid.New()` | differs | `stub.GetTxID()` or client-supplied ID |
| HTTP calls | can differ / be down | pass data as a tx parameter |
| Iterating a Go `map` | order is randomized | sort keys first |
| `os.Getenv()`, local files | differs per peer | tx parameter or state |
| Float arithmetic | rounding can differ | integers, smallest unit |

Grep chaincode for: `time.Now`, `Date(`, `rand`, `http`, `fetch`, `os.Getenv`, `Math.random`. Works fine in dev (1 peer), fails intermittently once 2+ orgs endorse — because a single peer has nothing to compare against.

### 2. Hot key (MVCC conflicts)

```go
// ⛔ every shipment touches the same key → 10 at once, 9 fail
counter := GetState("TOTAL_SHIPMENTS")
counter++
PutState("TOTAL_SHIPMENTS", counter)

// ✅ no shared key; total computed off-chain from events
PutState("SHIPMENT-"+id, shipment)
```

Look for: global counters, an "index" stored as one array, a summary/aggregate record updated on every transaction. The client must retry `MVCC_READ_CONFLICT` (backoff + jitter) — this isn't a bug to fix, it's expected behavior to handle.

### 3. Success ≠ saved

```
contract.submitTransaction(...) → returns 200 OK
```

This ONLY means endorsement succeeded and the orderer accepted it. It can still turn out INVALID during validation (e.g. an MVCC conflict). Use the Fabric Gateway API and check **commit status**, not just the submit response.

Also: `evaluateTransaction` (query) runs on 1 peer, never reaches the orderer, and any `PutState` inside it is silently discarded — no error, no data. Reads use evaluate, writes use submit. Never the reverse.

### 4. No authorization check

Fabric verifies *who* the caller is (their MSP ID). It enforces nothing about *what they're allowed to do* — that's 100% chaincode's job, and AI-written chaincode frequently just... doesn't check.

```go
mspID, _ := ctx.GetClientIdentity().GetMSPID()
if mspID != "SupplierMSP" {
    return fmt.Errorf("only the Supplier may create a batch")
}
```

Also check **ownership**, not just org membership: `if batch.Owner != mspID { return error }`.

### 5. Phantom reads (bonus trap, same family)

A range/rich query's *result* isn't part of the read-set — only specific keys are. So a chaincode function that queries "find all X" and then writes based on the count is unsafe: another transaction can invalidate that assumption without triggering a conflict. Never use a query result to decide whether a write is allowed — only `GetState` on a specific key is safe for that.

---

## Self-check

1. Why is `time.Now()` forbidden but `GetTxTimestamp()` fine? *(one is per-peer, the other comes from the tx header — identical everywhere)*
2. Your API returns 200. Is the data in the ledger? *(no — only means endorsement + orderer accepted it)*
3. Two transactions write the same key at the same second. What happens? *(one gets MVCC_READ_CONFLICT; the client must retry)*

## Summary

```
  1. Runs on multiple peers        → must be DETERMINISTIC
  2. Validated by version          → client must RETRY on MVCC conflict
  3. Queries aren't in the read-set → never use one to decide a write
  4. Success response ≠ committed  → always check commit status
  5. Fabric checks identity only   → chaincode must check authorization
```

➡️ **[03 — Chaincode](03-chaincode.md)**
