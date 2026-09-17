# 07a — Chaincode Sketch: Functions Behind the Batch State Machine

> Companion to [07 §7.3](07-supply-chain-case-study.md#73-state-machine--draw-this-first-before-coding). Every arrow in that diagram becomes one function here, written out the way [03 §3.2](03-chaincode.md#32-a-minimal-chaincode-example-go) teaches: validate → check identity → check state precondition → deterministic time → write → emit event.

---

## The state machine, restated as a function list

```
HARVESTED ──ShipBatch──▶ IN_TRANSIT ──ReceiveBatch──▶ RECEIVED ──ProcessBatch──▶ PROCESSED
    ▲                        │                            │                        │
    │                        └──ReceiveBatch(fail)──▶ REJECTED                     │
    │                                                                              │
    └──────────────────────────── ShipBatch (cycle repeats) ◀─────────────────────┘
                                                                                    │
                                                                                    ▼
                                                                                  SOLD  (terminal, function unnamed in §7.3)

RECALLED ◀── reachable from any state above, via RecallBatch
```

| Function | From → To | Actor(s) | Policy (from §7.3 / §7.7) |
|---|---|---|---|
| `CreateBatch` | (none) → `HARVESTED` | Farmer | `OR(FarmerMSP)` |
| `ShipBatch` | `HARVESTED`→`IN_TRANSIT`, `PROCESSED`→`IN_TRANSIT` | sender + recipient | `AND(owner, recipient)` |
| `ReceiveBatch` | `IN_TRANSIT`→`RECEIVED` or `IN_TRANSIT`→`REJECTED` | sender + receiver | `AND(sender, receiver)` |
| `ProcessBatch` | `RECEIVED`→`PROCESSED` | Processor | not in the doc — sketch below defaults to owner-only, see gaps |
| `RecallBatch` | any state → `RECALLED` | Regulator or original owner | `OR(AuditMSP, FarmerMSP)` |
| *(unnamed)* | `PROCESSED`→…→`SOLD` | ? | not specified — the `....` in §7.3's diagram, left as a gap |

---

## Shared types and helpers

The `Batch` struct follows the data model in [§7.4](07-supply-chain-case-study.md#74-data-model--key-design), not the generic example in 03.2 — `qtyKg`, `parentBatchIds`, `originFarmId` are the case study's own field names.

```go
type SmartContract struct {
    contractapi.Contract
}

type Batch struct {
    DocType        string   `json:"docType"` // "batch" — rich queries need this, see 3.8
    ID             string   `json:"id"`
    Product        string   `json:"product"`
    QtyKg          int      `json:"qtyKg"`          // integer, not float — see 3.7
    Status         string   `json:"status"`
    Owner          string   `json:"owner"`          // current holder's MSP ID
    PendingTo      string   `json:"pendingTo,omitempty"` // set by ShipBatch, cleared by ReceiveBatch
    ParentBatchIds []string `json:"parentBatchIds,omitempty"`
    OriginFarmId   string   `json:"originFarmId"`
    UpdatedAt      string   `json:"updatedAt"`
    UpdatedBy      string   `json:"updatedBy"`
}

type BatchInput struct {
    ID    string `json:"id"`
    QtyKg int    `json:"qtyKg"`
}
```

The transition guard from §7.3, unchanged:

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
        if s == to {
            return true
        }
    }
    return false
}
```

`REJECTED` lives under `IN_TRANSIT` in this map, not under `RECEIVED` — there's no separate `RejectBatch` function. A failed quality check is an outcome of `ReceiveBatch` itself, not a new transaction type.

Hand-rolled composite-key indexes (see [3.4](03-chaincode.md#34-key-design--an-architectural-decision-not-a-detail)) need an explicit create **and** delete on every status/owner change, so every function below routes through these two helpers instead of calling `PutState`/`DelState` on the index keys directly:

```go
func putIndex(ctx contractapi.TransactionContextInterface, objType, attr1, attr2 string) error {
    key, err := ctx.GetStub().CreateCompositeKey(objType, []string{attr1, attr2})
    if err != nil {
        return err
    }
    return ctx.GetStub().PutState(key, []byte{0x00})
}

func delIndex(ctx contractapi.TransactionContextInterface, objType, attr1, attr2 string) error {
    key, err := ctx.GetStub().CreateCompositeKey(objType, []string{attr1, attr2})
    if err != nil {
        return err
    }
    return ctx.GetStub().DelState(key)
}
```

---

## `CreateBatch`

```go
func (s *SmartContract) CreateBatch(ctx contractapi.TransactionContextInterface,
    id string, product string, qtyKg int) error {

    // 1) VALIDATE INPUT
    if id == "" || product == "" {
        return fmt.Errorf("id and product are required")
    }
    if qtyKg <= 0 {
        return fmt.Errorf("qtyKg must be > 0, got %d", qtyKg)
    }

    // 2) CHECK CALLER IDENTITY — §7.7: OR(FarmerMSP), a single-org check
    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }
    if mspID != "FarmerMSP" {
        return fmt.Errorf("only FarmerMSP may create a batch, not %s", mspID)
    }

    // 3) CHECK STATE PRECONDITION — must not already exist
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
    now := ts.AsTime().UTC().Format(time.RFC3339)

    batch := Batch{
        DocType: "batch", ID: id, Product: product, QtyKg: qtyKg,
        Status: "HARVESTED", Owner: mspID, OriginFarmId: mspID,
        UpdatedAt: now, UpdatedBy: mspID,
    }
    b, err := json.Marshal(batch)
    if err != nil {
        return err
    }

    // 5) WRITE — main record + owner/status indexes
    if err := ctx.GetStub().PutState(id, b); err != nil {
        return err
    }
    if err := putIndex(ctx, "owner~batch", mspID, id); err != nil {
        return err
    }
    if err := putIndex(ctx, "status~batch", "HARVESTED", id); err != nil {
        return err
    }

    // 6) EMIT EVENT
    return ctx.GetStub().SetEvent("BatchCreated", b)
}
```

---

## `ShipBatch`

Sets a **state-based endorsement policy** on the key (§5.4) so the follow-up `ReceiveBatch` cannot commit without the recipient's peer also endorsing — this is the two-party-confirmation mitigation from §7.2, enforced by Fabric's validation layer, not by chaincode logic.

```go
func (s *SmartContract) ShipBatch(ctx contractapi.TransactionContextInterface,
    id string, toMSP string) error {

    // 1) VALIDATE INPUT
    if id == "" || toMSP == "" {
        return fmt.Errorf("id and toMSP are required")
    }

    // 2) CHECK CALLER IDENTITY — must be the current owner
    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }

    batchJSON, err := ctx.GetStub().GetState(id)
    if err != nil {
        return fmt.Errorf("failed to read state: %v", err)
    }
    if batchJSON == nil {
        return fmt.Errorf("batch %s does not exist", id)
    }
    var batch Batch
    if err := json.Unmarshal(batchJSON, &batch); err != nil {
        return err
    }
    if batch.Owner != mspID {
        return fmt.Errorf("only the current owner (%s) may ship this batch, not %s", batch.Owner, mspID)
    }

    // 3) CHECK STATE PRECONDITION — legal transition; also rejects an
    //    already-IN_TRANSIT batch, since "IN_TRANSIT" has no self-edge in `allowed`
    if !canTransition(batch.Status, "IN_TRANSIT") {
        return fmt.Errorf("cannot ship batch %s from status %s", id, batch.Status)
    }

    // 4) DETERMINISTIC TIME
    ts, err := ctx.GetStub().GetTxTimestamp()
    if err != nil {
        return err
    }

    oldStatus := batch.Status
    batch.Status = "IN_TRANSIT"
    batch.PendingTo = toMSP
    batch.UpdatedAt = ts.AsTime().UTC().Format(time.RFC3339)
    batch.UpdatedBy = mspID

    b, err := json.Marshal(batch)
    if err != nil {
        return err
    }

    // 5) WRITE — record, move the status index, hand the key's endorsement
    //    policy to AND(sender, receiver)
    if err := ctx.GetStub().PutState(id, b); err != nil {
        return err
    }
    if err := delIndex(ctx, "status~batch", oldStatus, id); err != nil {
        return err
    }
    if err := putIndex(ctx, "status~batch", "IN_TRANSIT", id); err != nil {
        return err
    }

    ep, err := statebased.NewStateEP(nil)
    if err != nil {
        return err
    }
    if err := ep.AddOrgs(statebased.RoleTypePeer, mspID, toMSP); err != nil {
        return err
    }
    policy, err := ep.Policy()
    if err != nil {
        return err
    }
    if err := ctx.GetStub().SetStateValidationParameter(id, policy); err != nil {
        return err
    }

    // 6) EMIT EVENT
    return ctx.GetStub().SetEvent("BatchShipped", b)
}
```

---

## `ReceiveBatch`

One function, two outcomes — `qualityPass` decides between the two edges `IN_TRANSIT` has in `allowed`. Ownership only actually moves on a pass; a rejected batch bounces back to the sender rather than transferring.

```go
func (s *SmartContract) ReceiveBatch(ctx contractapi.TransactionContextInterface,
    id string, qualityPass bool) error {

    // 1) VALIDATE INPUT
    if id == "" {
        return fmt.Errorf("id is required")
    }

    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }

    batchJSON, err := ctx.GetStub().GetState(id)
    if err != nil {
        return fmt.Errorf("failed to read state: %v", err)
    }
    if batchJSON == nil {
        return fmt.Errorf("batch %s does not exist", id)
    }
    var batch Batch
    if err := json.Unmarshal(batchJSON, &batch); err != nil {
        return err
    }

    // 2) CHECK CALLER IDENTITY — must be the shipment's declared destination
    //    (§7.3 negative test). The sender's half of the AND(sender, receiver)
    //    endorsement is enforced separately, at the validation layer, using
    //    the policy ShipBatch attached to this key — not checked here.
    if batch.PendingTo != mspID {
        return fmt.Errorf("batch %s was shipped to %s, not %s", id, batch.PendingTo, mspID)
    }

    // 3) CHECK STATE PRECONDITION
    newStatus := "RECEIVED"
    if !qualityPass {
        newStatus = "REJECTED"
    }
    if !canTransition(batch.Status, newStatus) {
        return fmt.Errorf("cannot move batch %s from %s to %s", id, batch.Status, newStatus)
    }

    // 4) DETERMINISTIC TIME
    ts, err := ctx.GetStub().GetTxTimestamp()
    if err != nil {
        return err
    }

    oldStatus, oldOwner := batch.Status, batch.Owner
    batch.Status = newStatus
    batch.PendingTo = ""
    batch.UpdatedAt = ts.AsTime().UTC().Format(time.RFC3339)
    batch.UpdatedBy = mspID
    if qualityPass {
        batch.Owner = mspID // ownership only transfers on a pass
    }

    b, err := json.Marshal(batch)
    if err != nil {
        return err
    }

    // 5) WRITE
    if err := ctx.GetStub().PutState(id, b); err != nil {
        return err
    }
    if err := delIndex(ctx, "status~batch", oldStatus, id); err != nil {
        return err
    }
    if err := putIndex(ctx, "status~batch", newStatus, id); err != nil {
        return err
    }
    if qualityPass {
        if err := delIndex(ctx, "owner~batch", oldOwner, id); err != nil {
            return err
        }
        if err := putIndex(ctx, "owner~batch", mspID, id); err != nil {
            return err
        }
        // policy travels with ownership (§5.4) — now that the receiver holds
        // it alone, drop the two-party requirement until the next ShipBatch
        ep, err := statebased.NewStateEP(nil)
        if err != nil {
            return err
        }
        if err := ep.AddOrgs(statebased.RoleTypePeer, mspID); err != nil {
            return err
        }
        policy, err := ep.Policy()
        if err != nil {
            return err
        }
        if err := ctx.GetStub().SetStateValidationParameter(id, policy); err != nil {
            return err
        }
    }

    // 6) EMIT EVENT
    eventName := "BatchReceived"
    if !qualityPass {
        eventName = "BatchRejected"
    }
    return ctx.GetStub().SetEvent(eventName, b)
}
```

---

## `ProcessBatch`

Handles both merge (many parents → one output, the `BATCH#A,B,C → BATCH#X` example in §7.4) and split (one parent → many outputs) with the same quantity-conservation check from §7.8. Each output is a **new** batch ID that starts life at `PROCESSED` — a design choice this sketch makes since the doc leaves it open (see gaps below).

```go
func (s *SmartContract) ProcessBatch(ctx contractapi.TransactionContextInterface,
    parentIds []string, outputs []BatchInput) error {

    // 1) VALIDATE INPUT
    if len(parentIds) == 0 || len(outputs) == 0 {
        return fmt.Errorf("at least one parent and one output batch are required")
    }

    // 2) CHECK CALLER IDENTITY — resolved here as owner-only (see gaps)
    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }

    // 3) CHECK STATE PRECONDITION — every parent must be RECEIVED and
    //    owned by the caller; accumulate qty for the conservation check
    parents := make([]Batch, 0, len(parentIds))
    parentQtyTotal := 0
    for _, pid := range parentIds {
        pJSON, err := ctx.GetStub().GetState(pid)
        if err != nil {
            return fmt.Errorf("failed to read parent %s: %v", pid, err)
        }
        if pJSON == nil {
            return fmt.Errorf("parent batch %s does not exist", pid)
        }
        var parent Batch
        if err := json.Unmarshal(pJSON, &parent); err != nil {
            return err
        }
        if parent.Owner != mspID {
            return fmt.Errorf("parent batch %s is not owned by %s", pid, mspID)
        }
        if !canTransition(parent.Status, "PROCESSED") {
            return fmt.Errorf("parent batch %s cannot move from %s to PROCESSED", pid, parent.Status)
        }
        parents = append(parents, parent)
        parentQtyTotal += parent.QtyKg
    }

    outputQtyTotal := 0
    for _, o := range outputs {
        if o.ID == "" || o.QtyKg <= 0 {
            return fmt.Errorf("invalid output batch %+v", o)
        }
        outputQtyTotal += o.QtyKg
    }
    // quantity conservation — §7.8: children's total must equal parents' total
    if outputQtyTotal != parentQtyTotal {
        return fmt.Errorf("quantity mismatch: parents total %dkg, outputs total %dkg", parentQtyTotal, outputQtyTotal)
    }

    // 4) DETERMINISTIC TIME
    ts, err := ctx.GetStub().GetTxTimestamp()
    if err != nil {
        return err
    }
    now := ts.AsTime().UTC().Format(time.RFC3339)

    // 5) WRITE — close out every parent, then create each output batch
    for _, parent := range parents {
        oldStatus := parent.Status
        parent.Status = "PROCESSED"
        parent.UpdatedAt = now
        parent.UpdatedBy = mspID
        pb, err := json.Marshal(parent)
        if err != nil {
            return err
        }
        if err := ctx.GetStub().PutState(parent.ID, pb); err != nil {
            return err
        }
        if err := delIndex(ctx, "status~batch", oldStatus, parent.ID); err != nil {
            return err
        }
        if err := putIndex(ctx, "status~batch", "PROCESSED", parent.ID); err != nil {
            return err
        }
    }

    for _, o := range outputs {
        existing, err := ctx.GetStub().GetState(o.ID)
        if err != nil {
            return err
        }
        if existing != nil {
            return fmt.Errorf("output batch %s already exists", o.ID)
        }
        child := Batch{
            DocType: "batch", ID: o.ID, QtyKg: o.QtyKg,
            Status: "PROCESSED", Owner: mspID, ParentBatchIds: parentIds,
            UpdatedAt: now, UpdatedBy: mspID,
        }
        cb, err := json.Marshal(child)
        if err != nil {
            return err
        }
        if err := ctx.GetStub().PutState(o.ID, cb); err != nil {
            return err
        }
        if err := putIndex(ctx, "owner~batch", mspID, o.ID); err != nil {
            return err
        }
        if err := putIndex(ctx, "status~batch", "PROCESSED", o.ID); err != nil {
            return err
        }
    }

    // 6) EMIT EVENT — max 1 per transaction (§3.3), so one summary event
    //    rather than one per output
    payload, err := json.Marshal(map[string]interface{}{"parentIds": parentIds, "outputs": outputs})
    if err != nil {
        return err
    }
    return ctx.GetStub().SetEvent("BatchProcessed", payload)
}
```

---

## `RecallBatch`

Deliberately skips `canTransition` — §7.3 draws `RECALLED` as reachable from every state, and §7.7 requires `OR` specifically so the recall can't be blocked by the party at fault.

```go
func (s *SmartContract) RecallBatch(ctx contractapi.TransactionContextInterface,
    id string, reason string) error {

    // 1) VALIDATE INPUT
    if id == "" || reason == "" {
        return fmt.Errorf("id and reason are required")
    }

    mspID, err := ctx.GetClientIdentity().GetMSPID()
    if err != nil {
        return err
    }

    batchJSON, err := ctx.GetStub().GetState(id)
    if err != nil {
        return fmt.Errorf("failed to read state: %v", err)
    }
    if batchJSON == nil {
        return fmt.Errorf("batch %s does not exist", id)
    }
    var batch Batch
    if err := json.Unmarshal(batchJSON, &batch); err != nil {
        return err
    }

    // 2) CHECK CALLER IDENTITY — §7.7: OR(AuditMSP, FarmerMSP)
    if mspID != "AuditMSP" && mspID != batch.OriginFarmId {
        return fmt.Errorf("only AuditMSP or the originating farmer may recall batch %s, not %s", id, mspID)
    }

    // 3) CHECK STATE PRECONDITION — no canTransition call on purpose;
    //    only guard against recalling something already recalled
    if batch.Status == "RECALLED" {
        return fmt.Errorf("batch %s is already recalled", id)
    }

    // 4) DETERMINISTIC TIME
    ts, err := ctx.GetStub().GetTxTimestamp()
    if err != nil {
        return err
    }

    oldStatus := batch.Status
    batch.Status = "RECALLED"
    batch.UpdatedAt = ts.AsTime().UTC().Format(time.RFC3339)
    batch.UpdatedBy = mspID

    b, err := json.Marshal(batch)
    if err != nil {
        return err
    }

    // 5) WRITE
    if err := ctx.GetStub().PutState(id, b); err != nil {
        return err
    }
    if err := delIndex(ctx, "status~batch", oldStatus, id); err != nil {
        return err
    }
    if err := putIndex(ctx, "status~batch", "RECALLED", id); err != nil {
        return err
    }

    // 6) EMIT EVENT — the event listener (§3.6) drives the forward-trace-
    //    and-notify workflow from §7.4, not chaincode itself
    return ctx.GetStub().SetEvent("BatchRecalled", b)
}
```

---

## Negative tests these guards must satisfy (from §7.3)

| Test | Exercises |
|---|---|
| `HARVESTED → SOLD` directly must fail | `canTransition` guard (no direct edge exists) |
| `SOLD → IN_TRANSIT` must fail | `ShipBatch`'s `canTransition` call (`allowed["SOLD"]` is empty) |
| `ReceiveBatch` by a non-destination org must fail | `ReceiveBatch`'s `batch.PendingTo != mspID` check |
| `ShipBatch` on a batch already `IN_TRANSIT` must fail | same `canTransition` call — `IN_TRANSIT` has no self-edge |

---

## Open gaps this sketch had to guess at (per §7.9)

1. **`ProcessBatch`'s endorsement policy** — coded above as owner-only (single-org, no counter-signature). Undecided: should the origin farmers also have to endorse a merge that changes their batch's final status?
2. **The `PROCESSED → SOLD` function** — not sketched at all. Name, caller, and whether the buyer (Café) must co-endorse like `ReceiveBatch` does are still undefined.

➡️ Back to **[07 §7.9 — Design exercise](07-supply-chain-case-study.md#79-design-exercise)** to resolve these yourself before handing this to an AI agent.
