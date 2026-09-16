# 02 — Transaction Flow: Execute → Order → Validate

> **The most important file in this material.** About 80% of chaincode bugs come from not understanding this flow.
>
> Goal: after reading this, you can answer "why did this transaction fail when the code looks correct?"

---

## 2.1 Why is the flow so strange?

In a regular database:

```
   Client → "UPDATE balance SET x=100" → DB executes → commit → done
```

In Fabric, the problem is: **who is allowed to execute it?** If only one server executes it, every other org has to trust that server. But the whole premise is that they don't trust each other.

Fabric's solution: **several organizations execute the same transaction independently, then compare the results.** Only if everyone agrees does it get committed.

```
   Ethereum:   ORDER  →  EXECUTE           (sequence first, then run)
   Fabric:     EXECUTE → ORDER → VALIDATE  (run it in several places first,
                                            sequence it, then check it's still valid)
```

The trade-off: Fabric is much faster and more flexible, **but** it introduces a class of bugs that don't exist in a regular database. That's what we're about to learn.

---

## 2.2 The big picture: 3 phases

```
  ╔═══════════════════════════════════════════════════════════════════════╗
  ║  PHASE 1 — EXECUTE (Endorsement)                                      ║
  ║                                                                       ║
  ║   The client sends a proposal to several peers.                      ║
  ║   Each peer runs the chaincode in SIMULATION mode.                    ║
  ║   ⚠️ NOTHING is written to the ledger during this phase.             ║
  ║   Result: a READ-SET + WRITE-SET + the peer's signature.             ║
  ╚═══════════════════════════════════════════════════════════════════════╝
                                  │
                                  ▼
  ╔═══════════════════════════════════════════════════════════════════════╗
  ║  PHASE 2 — ORDER                                                      ║
  ║                                                                       ║
  ║   The client sends the transaction (+ all signatures) to the Orderer.║
  ║   The Orderer ONLY sequences it and packages it into a block.        ║
  ║   The Orderer does NOT look at the content, does NOT validate.       ║
  ╚═══════════════════════════════════════════════════════════════════════╝
                                  │
                                  ▼
  ╔═══════════════════════════════════════════════════════════════════════╗
  ║  PHASE 3 — VALIDATE & COMMIT                                          ║
  ║                                                                       ║
  ║   The block is sent to ALL peers. For each transaction, every peer:  ║
  ║     a) Checks: enough signatures? (endorsement policy)                ║
  ║     b) Checks: is the read-set still valid? (MVCC check) ⬅ bug source!║
  ║   Pass  → write-set is applied to world state, marked VALID          ║
  ║   Fail  → world state is NOT changed, marked INVALID                 ║
  ║   ⚠️ Both outcomes still get written into the blockchain.            ║
  ╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 2.3 Detailed diagram (sequence)

```
 Client App        Peer Sup      Peer Dist      Orderer        All Peers
     │                │              │             │               │
     │──1. Proposal──▶│              │             │               │
     │──1. Proposal───┼─────────────▶│             │               │
     │                │              │             │               │
     │           [simulate]     [simulate]         │               │
     │           chaincode      chaincode          │               │
     │           runs against   runs against        │               │
     │           current        current             │               │
     │           world state    world state          │               │
     │                │              │             │               │
     │◀─2. Response───│              │             │               │
     │   {readSet, writeSet, signature}            │               │
     │◀─2. Response───┼──────────────│             │               │
     │                │              │             │               │
     │ 3. Client COMPARES the responses:            │               │
     │    readSet & writeSet must be IDENTICAL.     │               │
     │    If they differ → ⛔ stop, don't send.     │               │
     │                                             │               │
     │──────4. Send tx + all signatures───────────▶│               │
     │                                             │               │
     │                              [orderer sequences, builds block]│
     │                                             │               │
     │                                             │──5. Block────▶│
     │                                             │               │
     │                                    [each peer validates:]    │
     │                                    [ a) endorsement policy ]│
     │                                    [ b) MVCC read-set check]│
     │                                    [ then commit / mark    ]│
     │                                    [ INVALID                ]│
     │                                             │               │
     │◀────────6. Event: tx committed (VALID / INVALID)────────────│
     │                                                             │
     │  ⚠️ If your app doesn't wait for this event,                │
     │     you don't actually know if the transaction succeeded!   │
```

> 🔑 **Memorize step 3:** the client compares the results from all endorsers. If your chaincode isn't deterministic, the results differ, and the transaction dies right here.

---

## 2.4 Read-set & write-set — a concrete example

This is the single most important concept. Let's see what it actually looks like.

Say the current world state is:

```
   KEY        VALUE                          VERSION (block, tx)
   ─────────────────────────────────────────────────────────────
   BATCH001   {"qty": 100, "owner": "SUP"}   (5, 2)
   BATCH002   {"qty":  50, "owner": "SUP"}   (7, 0)
```

Chaincode `TransferBatch("BATCH001", "DIST")` runs:

```go
b := GetState("BATCH001")     // read
b.owner = "DIST"              // change in memory
PutState("BATCH001", b)       // write
```

The simulation result the peer sends back:

```
   ┌─────────────────────────────────────────────────────────┐
   │ READ-SET   (what was read, AND its version at read time)│
   ├─────────────────────────────────────────────────────────┤
   │   BATCH001  →  version (5, 2)                           │
   │                ⚠️ The VALUE itself is NOT included,     │
   │                   only the version number                │
   └─────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────┐
   │ WRITE-SET  (what will be written)                        │
   ├─────────────────────────────────────────────────────────┤
   │   BATCH001  →  {"qty": 100, "owner": "DIST"}             │
   └─────────────────────────────────────────────────────────┘
```

At commit time, the peer asks: **"Is BATCH001 still at version (5,2)?"**

- Yes → apply the write-set. VALID. ✅
- No (someone else already changed it) → **MVCC_READ_CONFLICT**. INVALID. ❌

> 🔑 This is **optimistic concurrency control**, exactly like `UPDATE ... WHERE version = 5` in SQL. The difference: in Fabric this happens automatically for every key you read.

---

## 2.5 Trap #1 — MVCC_READ_CONFLICT

This is the error you'll see most often in production, and the one that causes the most panic.

```
   Time ───────────────────────────────────────────────────────▶

   TX-A:  simulate ──────────┐
          reads BATCH001 v(5,2)
                              ├──▶ orderer ──┐
   TX-B:  simulate ──────────┤              │
          reads BATCH001 v(5,2)              ├─▶ [ BLOCK 8 ]
                              └──▶ orderer ──┘    contains TX-A, TX-B
                                                        │
                                                        ▼
                                              Validated in order:
                                              ┌──────────────────────────┐
                                              │ TX-A: is BATCH001 still  │
                                              │       v(5,2)? YES ✅     │
                                              │  → commit, version      │
                                              │    becomes (8,0)         │
                                              ├──────────────────────────┤
                                              │ TX-B: is BATCH001 still  │
                                              │       v(5,2)? NO ❌      │
                                              │  → MVCC_READ_CONFLICT    │
                                              │  → INVALID, discarded    │
                                              └──────────────────────────┘
```

### Why this matters to you

- This is **not a bug**, it's the design. That's just how optimistic locking works.
- But it becomes a disaster if there's a **hot key** — a single key that many transactions update.

### 🚩 Red flag: the "hot key" pattern

```go
// ⛔ BAD: a global counter
counter := GetState("TOTAL_SHIPMENTS")
counter++
PutState("TOTAL_SHIPMENTS", counter)
// Every shipment touches the same key.
// 10 shipments at once → 9 fail.
```

```go
// ✅ GOOD: no shared key
PutState("SHIPMENT-"+id, shipment)
// The total is computed in an off-chain DB from events, not on the ledger.
```

Hot-key patterns to look for when reviewing:
- a global counter / sequence number
- an "index" stored as a single key holding an array
- a "summary" or "aggregate" record updated on every transaction
- every transaction writing to a record owned by the same single organization

### What the application must do

⚠️ **The client must have retries.** Example: retry 3 times with exponential backoff + jitter for `MVCC_READ_CONFLICT` errors.

🧪 **Required test scenario:** send 20 parallel transactions that touch the same key, and confirm the application doesn't lose data or report a false success.

---

## 2.6 Trap #2 — chaincode MUST be deterministic

Because chaincode runs **independently on multiple peers**, the results must be exactly identical. If not, the read-set/write-set will differ → the client rejects it → the transaction never gets sent.

```
   Peer Supplier                     Peer Distributor
   ┌─────────────────────┐           ┌─────────────────────┐
   │ writeSet:           │           │ writeSet:           │
   │ {"ts":"10:00:01.123"}│  ≠       │ {"ts":"10:00:01.456"}│
   └─────────────────────┘           └─────────────────────┘
                    │                        │
                    └────────┬───────────────┘
                             ▼
                   Client: "the results differ!"
                   ⛔ Transaction aborted.
                   (sometimes shows up as a confusing error)
```

### Table: forbidden vs allowed

| ⛔ FORBIDDEN in chaincode | Why | ✅ Use instead |
|---|---|---|
| `time.Now()`, `new Date()` | each peer's clock differs | `stub.GetTxTimestamp()` — identical on every peer |
| `rand`, `uuid.New()` | obviously different | an ID from the client, or `stub.GetTxID()` |
| HTTP calls to an external API | response can differ / be down | send the data as a transaction parameter instead |
| Iterating a `map` in Go | Go map order is **randomized by design** | grab the keys → `sort.Strings()` → iterate |
| `os.Getenv()`, reading local files | environment differs per peer | a transaction parameter or state |
| Reading an external DB | same issue | same issue |
| Goroutines / concurrency | non-deterministic ordering | don't |
| Sensitive float arithmetic | rounding can differ across platforms | use integers (store in the smallest unit) |
| `GetHistoryForKey` to decide a write | history isn't part of the read-set | avoid |

> 🚩 **This is item #1 on your review checklist.** Grep the chaincode for: `time.Now`, `Date(`, `rand`, `http`, `fetch`, `axios`, `os.Getenv`, `Math.random`. If any show up inside a chaincode function → reject it.

⚠️ The nasty part: non-determinism bugs are often **intermittent**. In dev with 1 peer, everything runs smoothly. It only blows up in staging with 3 orgs.

---

## 2.7 Trap #3 — `GetState` doesn't see your own `PutState`

```go
PutState("BATCH001", []byte(`{"qty":50}`))

v, _ := GetState("BATCH001")
// ⚠️ v still contains the OLD value from world state!
// Not {"qty":50}.
```

```
   Within a single transaction:

   ┌──────────────────────────────────────────────────┐
   │  WORLD STATE (read-only during simulation)       │
   │     BATCH001 = {"qty":100}                       │
   └──────────────────────────────────────────────────┘
         ▲ GetState reads from here            │
         │                                     │
         │                                     ▼
   ┌─────┴────────────────────────────────────────────┐
   │  WRITE-SET (a buffer, not committed anywhere yet)│
   │     BATCH001 = {"qty":50}                        │
   └──────────────────────────────────────────────────┘
              PutState writes here
```

**Rule:** keep the object in a local variable, make all your changes there, and call `PutState` exactly once at the end.

🚩 Red flag: the pattern `PutState(...)` followed by `GetState(...)` on the same key within one function.

---

## 2.8 Trap #4 — phantom reads (range & rich queries)

The read-set contains **specific keys + their versions**. But the results of a *query* (range query / CouchDB rich query) are **not** recorded as a condition that gets validated.

```
   TX-A (simulated at 10:00):
     "Get all BATCHes with status=AVAILABLE"    → gets [B1, B2]
     "Count is 2, OK to ship"                   → PutState(SHIPMENT)

   TX-B (at 10:00:00.5, same block):
     Creates BATCH B3 with status=AVAILABLE

   At commit time for TX-A:
     the MVCC check only looks at B1 and B2 — same version? YES.
     ✅ TX-A is marked VALID.

   ⚠️ Except the assumption "there are only 2 batches" is now wrong.
      This is called a PHANTOM READ. Fabric does NOT protect against it.
```

**Practical rule:**

| Query use case | Safe? |
|---|---|
| Displaying a list to the user (read-only) | ✅ Safe |
| Reports / dashboards | ✅ Safe (better done from an off-chain DB) |
| **Deciding whether a write is allowed** | ❌ **NOT SAFE** |

🚩 Red flag: `GetStateByRange` / `GetQueryResult` followed by `PutState` in the same function, where the write decision depends on the query result.

✅ Correct approach: design your keys so the decision can be made from `GetState` on a specific key. Example: instead of "find an available batch," have the client decide which batch, and let chaincode simply check `GetState(batchID).status == AVAILABLE`.

---

## 2.9 Trap #5 — false "success"

This causes data bugs that are extremely hard to track down.

```
   ┌────────────────────────────────────────────────────────────┐
   │  contract.submitTransaction("Transfer", "B1", "DIST")      │
   │       returns 200 OK  ✅                                   │
   └────────────────────────────────────────────────────────────┘
                              │
        What does this mean? ─┼──── It ONLY means:
                              │      "endorsement succeeded and
                              │       the tx was accepted by the orderer"
                              │
                              ▼
             It does NOT mean it's in world state yet!
             The transaction can still turn out INVALID during validation.
```

**The correct approach:**
- Use the **Fabric Gateway API** (v2.4+) and wait for commit status, not just endorsement.
- In the Node.js Gateway: `submitTransaction()` already waits for commit, but check its status; `submitAsync()` gives you manual control over waiting.
- To read data: use `evaluateTransaction()` (a query, doesn't touch the ledger), not `submitTransaction()`.

🚩 Red flag: code that immediately does `return { success: true }` after submitting, without checking commit status.

🧪 Test: kill one of the endorsing peers, or force an MVCC conflict, and confirm your API returns an error — not success.

---

## 2.10 Submit vs Evaluate

A distinction you must get right:

```
   ┌──────────────────────────────┬──────────────────────────────┐
   │  EVALUATE (query)            │  SUBMIT (invoke)             │
   ├──────────────────────────────┼──────────────────────────────┤
   │  Runs on 1 peer only          │  Runs on multiple peers      │
   │  Never reaches the orderer   │  Goes to the orderer, in a block│
   │  Doesn't change state        │  Changes state                │
   │  Fast (ms)                   │  Slow (~1-2 seconds)          │
   │  For: GET / reports           │  For: CREATE/UPDATE/DELETE   │
   └──────────────────────────────┴──────────────────────────────┘

   ⚠️ If a chaincode function has PutState but gets called via evaluate,
      its write-set is silently discarded. No data gets saved,
      but there's no error either. A ghost bug.
```

🚩 Red flag: a function that writes state being called with `evaluateTransaction`.

---

## 2.11 Self-check exercise

Answer without scrolling up. If you can answer all of these, you're ready to move on.

1. What's in a read-set, and why isn't the actual data value stored there too?
2. Why is `time.Now()` forbidden, but `stub.GetTxTimestamp()` allowed?
3. TX-A and TX-B both read and write `BATCH001`, sent at the same time. What happens? Whose job is it to handle that?
4. Chaincode runs a rich query "find all empty batches," then writes. Why is this unsafe?
5. Your API returns 200. Is the data guaranteed to be in the ledger? Why or why not?
6. Peer A and Peer B produce different write-sets for the same transaction. At what point does this transaction stop, and what's the usual cause?

<details>
<summary>Short answers</summary>

1. The key + a version number. Only "is the version still the same" gets validated, so the value itself isn't needed — it's cheaper and sufficient for conflict detection.
2. `time.Now()` differs on every peer → the write-sets differ. `GetTxTimestamp()` comes from the transaction header the client created, so it's identical on every peer.
3. One of them fails with `MVCC_READ_CONFLICT`. The client application must retry.
4. A phantom read — the query result isn't part of the read-set, so a change that invalidates the query's assumption goes undetected during validation.
5. No. 200 only means endorsement succeeded and it was accepted by the orderer. It can still be marked INVALID during validation.
6. It stops at the client (step 3, before it reaches the orderer). Cause: the chaincode isn't deterministic.
</details>

---

## 2.12 Summary — pin this to your monitor

```
  1. Chaincode runs on MULTIPLE peers      → it MUST be DETERMINISTIC
  2. Validation uses read-set versions     → build in RETRY for MVCC conflicts
  3. Queries aren't part of the read-set   → never use a query to decide a write
  4. GetState doesn't see its own PutState → use a local variable
  5. A success response ≠ committed        → always check commit status
  6. Submit ≠ Evaluate                     → write via submit, read via evaluate
```

➡️ Continue to **[03 — Chaincode](03-chaincode.md)**.
