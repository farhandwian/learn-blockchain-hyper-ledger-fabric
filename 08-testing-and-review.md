# 08 — Testing & Review: Your Daily Working Checklist

> This is the file you'll use every day. Pin it, print it, or turn it into a PR template.

---

## 8.1 The test pyramid for Fabric

```
                        ▲
                       ╱ ╲       E2E multi-org
                      ╱   ╲      (test-network, 2-3 orgs)
                     ╱─────╲     • slow (minutes)
                    ╱       ╲    • closest to production
                   ╱ INTEGR. ╲   • MANDATORY for: endorsement policy,
                  ╱           ╲    MVCC, private data, upgrades
                 ╱─────────────╲
                ╱               ╲  Integration (1 peer / mock ledger)
               ╱                 ╲ • medium (seconds)
              ╱      UNIT         ╲• the full transaction flow
             ╱                     ╲
            ╱_______________________╲ Unit (mock stub)
                                      • fast (ms)
                                      • business logic, validation,
                                        state machine, authorization

   ⚠️ Common mistake: only writing unit tests.
      MVCC conflicts, endorsement failures, and private-data bugs
      will NEVER show up in a unit test.
```

---

## 8.2 Unit tests — mock stub

Go (`shimtest` / a hand-rolled mock), Node (`sinon-chai` + a fake stub). What matters: the **logic**, not Fabric itself.

What must be tested at this layer:

```
  ✅ Input validation (empty, negative, too long, wrong format)
  ✅ The state machine (every legal transition AND every illegal one)
  ✅ Authorization (every ❌ cell in the authorization matrix)
  ✅ Ownership (not the owner → must fail)
  ✅ Idempotency (create twice → must fail)
  ✅ Schema compatibility (reading old-format JSON)
  ✅ Calculations (split/merge: children's total == parent's total)
```

A good example (Go):

```go
func TestCreateBatch_RejectsNonSupplier(t *testing.T) {
    ctx := newMockCtx()
    ctx.SetMSPID("RetailMSP")          // ← not the supplier

    err := (&SmartContract{}).CreateBatch(ctx, "B1", "Coffee", 100)

    require.Error(t, err)
    require.Contains(t, err.Error(), "only the Supplier")
}

func TestShipBatch_RejectsIllegalTransition(t *testing.T) {
    ctx := newMockCtx()
    ctx.SetMSPID("FarmerMSP")
    ctx.PutState("B1", batchJSON("SOLD", "FarmerMSP"))  // already sold

    err := (&SmartContract{}).ShipBatch(ctx, "B1", "DistMSP")

    require.Error(t, err)   // can't ship something already SOLD
}
```

> 🔑 For every chaincode function, a healthy ratio is: **1 happy path : 3-5 negative tests.**
>
> 🚩 If an AI agent produces tests that are all happy-path, that adds zero confidence. Ask explicitly for negative tests.

---

## 8.3 Integration scenarios you MUST have

This is what separates "tested" from "actually tested." Each of these needs a test on the test-network:

```
  ┌────┬───────────────────────────────────────────────────────────┐
  │ #  │ Scenario                                                  │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 1  │ MVCC CONFLICT                                             │
  │    │ 20 parallel transactions on the same key.                 │
  │    │ Expected: some fail, the client retries, no data is       │
  │    │ lost, no false success shown to the user.                 │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 2  │ ENDORSEMENT POLICY FAILURE                                │
  │    │ Submit a transaction with fewer endorsers than the policy │
  │    │ requires.                                                  │
  │    │ Expected: a clear failure, not a timeout.                 │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 3  │ NON-DETERMINISM                                           │
  │    │ Run with AT LEAST 2 orgs endorsing.                       │
  │    │ Expected: 100 transactions in a row, none fail from a     │
  │    │ mismatch. (a single peer alone won't catch this)          │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 4  │ CROSS-ORG AUTHORIZATION                                   │
  │    │ Org2 tries to modify an asset owned by Org1.               │
  │    │ Expected: fails. For EVERY write function.                │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 5  │ PRIVATE DATA                                               │
  │    │ An org that isn't a collection member tries to read it.    │
  │    │ Expected: fails / empty. AND check the hash is on the     │
  │    │ ledger.                                                    │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 6  │ UPGRADE COMPATIBILITY                                     │
  │    │ Deploy v1 → write data → deploy v2 → READ THE OLD DATA.   │
  │    │ Expected: no crash, sensible default values.               │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 7  │ PEER DOWN                                                 │
  │    │ Kill 1 peer, run a transaction.                            │
  │    │ Expected: matches the endorsement policy (AND → fails,     │
  │    │ OutOf → still works). Confirm the behavior is INTENTIONAL.│
  ├────┼───────────────────────────────────────────────────────────┤
  │ 8  │ EVENT LISTENER RESTART                                    │
  │    │ Kill the listener, send 10 tx, bring it back up.           │
  │    │ Expected: 10 events processed (checkpoint works),          │
  │    │ off-chain DB stays in sync.                                 │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 9  │ QUERY PERFORMANCE                                          │
  │    │ Load 100,000 records, run a rich query.                    │
  │    │ Expected: < 1 second. If not → the index is broken.        │
  ├────┼───────────────────────────────────────────────────────────┤
  │ 10 │ IDEMPOTENCY / DOUBLE SUBMIT                                │
  │    │ Send the same transaction twice (e.g. a user double-clicks)│
  │    │ Expected: the second one fails or has no duplicate effect. │
  └────┴───────────────────────────────────────────────────────────┘
```

🧪 If your team only has time for 3: pick **#1, #3, #4**. Those blow up in production most often.

---

## 8.4 PR review checklist — the full version

Copy this into a PR template.

### A. Determinism (blocker)
```
[ ] No time.Now() / new Date() → use GetTxTimestamp()
[ ] No rand / uuid / Math.random → an ID from the client, or GetTxID()
[ ] No HTTP/fetch/axios/gRPC calls to the outside
[ ] No os.Getenv / process.env / local file reads
[ ] No Go map iteration without sorting keys first
[ ] No goroutines / Promise.all affecting write order
[ ] No floats for money/quantities
```

### B. Authorization & validation (blocker)
```
[ ] EVERY write function checks MSP ID / attributes / ownership
[ ] All input is validated (empty, negative, length, format, enum)
[ ] Status transitions are validated against the state machine
[ ] Existence is checked before create (prevents silent overwrite)
[ ] Errors are returned as errors — no silent `return nil`
```

### C. Concurrency & queries
```
[ ] No hot key (global counter, array index, aggregate record)
[ ] The client has retry + backoff for MVCC_READ_CONFLICT
[ ] Queries (range/rich) are NEVER used to decide a write
[ ] No PutState followed by GetState on the same key
[ ] Composite-key indexes get DelState'd when the indexed field changes
[ ] META-INF/statedb/couchdb/indexes exists if rich queries are used
```

### D. Data & privacy
```
[ ] No PII (name, national ID, address, phone) in world state
[ ] No file/base64/blob in world state — hash + URI only
[ ] Secret data goes through transient + PDC, NEVER a transaction argument
[ ] Endorsing orgs are restricted for any function touching private data
[ ] Value sizes are reasonable (< a few hundred KB)
```

### E. Integration & operations
```
[ ] SetEvent for every meaningful change (max 1 per transaction)
[ ] The client waits for & checks COMMIT status, not just endorsement
[ ] Read functions use evaluateTransaction; write functions use submit
[ ] The new chaincode version can read old-version data (+ tested)
[ ] The sequence number is bumped in the deploy script (not hardcoded)
[ ] The endorsement policy is explicit and matches the agreed matrix
[ ] Identity/wallet never leaves the backend
```

### F. Tests
```
[ ] A negative test exists for every authorization rule
[ ] A negative test exists for every illegal status transition
[ ] An integration test exists with ≥2 orgs endorsing
[ ] A concurrency test exists (MVCC)
[ ] An old-schema compatibility test exists
```

---

## 8.5 Error → cause → action table

Save this. You'll use it often.

| Error / symptom | Most likely cause | Action |
|---|---|---|
| `MVCC_READ_CONFLICT` | two tx wrote the same key in the same block | retry on the client; if frequent → a hot key, redesign the key |
| `PHANTOM_READ_CONFLICT` | a range query collided with a write | avoid queries on the write path |
| `ENDORSEMENT_POLICY_FAILURE` | too few/wrong endorsers, or a policy mismatch | check `--peerAddresses` / `setEndorsingOrganizations` |
| `ENDORSEMENT_MISMATCH` / "response payload differ" | **the chaincode isn't deterministic** | audit for determinism (checklist A) |
| `chaincode definition not agreed` | orgs approved different definitions (version/policy/sequence) | align the approve parameters across every org |
| `chaincode not found` / `not defined` | not yet committed, or wrong name/channel | `peer lifecycle chaincode querycommitted` |
| `INVALID_OTHER_REASON` | chaincode returned an error during validation | check the chaincode container log |
| Transaction succeeds but data doesn't change | it was called via **evaluate**, not submit | switch to submitTransaction |
| Slow queries (seconds) | a rich query with no index | add an index in META-INF, verify it's used |
| `identity expired` / TLS handshake fails | **an expired certificate** | re-enroll; set up expiry monitoring |
| Private data is empty despite being written | the peer isn't a collection member, or the data went through an argument instead of transient | check the collection config & transient usage |
| Events never arrive | the tx was INVALID, or the listener has no checkpoint | check the tx status; add a checkpoint |
| Every transaction times out | the orderer is down / lost Raft quorum | check the orderer log |

---

## 8.6 Questions to ask your AI agent

This is your "weapon" as a reviewer. Ask these every time you receive code:

```
  1. "Show me that this function is deterministic. What happens if
      it runs on 3 peers at once?"

  2. "Which key here could become a hot key? What happens if 100
      transactions arrive at the same time?"

  3. "Who exactly can call this function? Show me the check, and show
      me a test that proves everyone else is rejected."

  4. "How does this chaincode version read data written by the
      previous version? Show me the test."

  5. "If organization X modified their own client application, which
      business rules could they break?"

  6. "What data here will be permanent on the ledger forever? Should
      any of it not be permanent?"

  7. "What's the endorsement policy for this function, and why was it
      chosen? Who gets hurt if one party lies?"

  8. "What happens if this transaction gets sent twice?"
```

⚠️ Pay attention to the answers. AI tends to be very confident. **Never accept an answer without a runnable test.** If the AI says "it's already safe," ask for a test that fails if that assumption is wrong.

---

## 8.7 Definition of Done

A new feature can be considered done when:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  [ ] Checklist 8.4 sections A & B fully pass (blockers)       │
  │  [ ] Unit tests: 1 happy path + at least 3 negative tests     │
  │  [ ] Runs on the test-network with ≥2 orgs endorsing          │
  │  [ ] The relevant scenarios from 8.3 have been run            │
  │  [ ] The endorsement policy is written explicitly + its       │
  │       reasoning is documented                                  │
  │  [ ] The data-placement matrix (on-chain / PDC / off-chain)   │
  │       is updated                                               │
  │  [ ] The state machine diagram is updated if a new status     │
  │       was added                                                 │
  │  [ ] Reading data from the previous chaincode version was     │
  │       tested                                                    │
  └──────────────────────────────────────────────────────────────┘
```

---

## 8.8 Closing principle

```
   In a normal application:  deploy → bug found → fix → data gets cleaned up
   In Fabric:                deploy → bug found → fix → BAD DATA STAYS
                                                          ON THE LEDGER FOREVER
```

That's exactly why your role — reviewing and validating, not typing the code — matters **more** in a blockchain project than in an ordinary one.

You don't need to write chaincode faster than the AI. You need to be able to answer the question the AI can't: **"is this actually correct for our business, and what happens if it's wrong?"**

---

## Further reading

| Source | What it's for |
|---|---|
| https://hyperledger-fabric.readthedocs.io | official docs; read the *Key Concepts* section |
| `fabric-samples/test-network` | your practice network |
| `fabric-samples/asset-transfer-basic` | the basic CRUD pattern |
| `fabric-samples/asset-transfer-private-data` | the PDC + transient pattern |
| `fabric-samples/asset-transfer-abac` | the attribute-based authorization pattern |
| `fabric-samples/asset-transfer-sbe` | state-based endorsement (policy travels with ownership) |
| `fabric-samples/off_chain_data` | the event listener → off-chain DB pattern |

⬅️ Back to **[README](README.md)**
