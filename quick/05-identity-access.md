# 05 — Identity & Access

Two **different** questions, often conflated:

1. **"Who can CALL this function?"** → checked inside chaincode (ABAC) — see [03](03-chaincode.md)
2. **"Who must APPROVE this change?"** → endorsement policy — this file

## Where identity comes from

A CA issues each user/app an X.509 certificate (org, role, custom attributes) → held in a **wallet in your backend** (treat it like a DB password, never expose to the frontend) → used to sign proposals → the peer maps the cert to an **MSP ID** via the org's MSP config.

> 🔑 Chaincode only ever sees the MSP ID (e.g. `SupplierMSP`), never a person's name. Fabric verifies the identity is valid — it enforces nothing about what that identity is allowed to do.

**Practical model:** one Fabric identity per organization (not per user) is usually enough for an MVP — put the actual userId in the transaction payload if you need a per-person audit trail. Per-user identities add real certificate-management overhead; only do it if that's a real requirement.

## Decision 1: the authorization matrix

Build this **before** any code gets written. This is your job, not the AI's:

```
                  Supplier  Distributor  Retailer  Auditor
CreateBatch          ✅         ❌          ❌        ❌
ShipBatch          ✅ own     ✅ own        ❌        ❌
ReceiveBatch          ❌         ✅          ✅        ❌
ReadBatch             ✅         ✅          ✅        ✅
```

For every ❌, write a test that attempts it and asserts failure. In chaincode:

```go
mspID, _ := ctx.GetClientIdentity().GetMSPID()
if mspID != "SupplierMSP" { return fmt.Errorf("only Supplier may create a batch") }

// ownership check
if batch.Owner != mspID { return fmt.Errorf("batch belongs to %s", batch.Owner) }

// custom attribute check
ok, _ := ctx.GetClientIdentity().AssertAttributeValue("role", "warehouse_manager")
```

## Decision 2: endorsement policy (the trust model)

The question: **how many signatures make a change valid?**

```
AND(Supplier, Distributor)  → both must sign a transfer. Neither can
                               claim it happened unilaterally.
OR(Supplier, Distributor)   → Supplier ALONE can move ownership to
                               Distributor. Sometimes intended (e.g.
                               self-reported production), often a hole.
```

| Syntax | Meaning |
|---|---|
| `OR('A.peer','B.peer')` | either is enough |
| `AND('A.peer','B.peer')` | both required |
| `OutOf(2, 'A','B','C')` | at least 2 of 3 |

**How to pick it, per function:** *if org X lied about this alone, would anyone be harmed?* Yes → `AND` including the harmed party. No → `OR` is fine.

```
CreateBatch (farmer records own harvest)  → nobody harmed        → OR(Farmer)
TransferOwnership                          → recipient could be   → AND(sender, receiver)
                                              forced to "receive"
RecordTemperature from sensor              → both sides harmed    → AND(owner, auditor)
                                              if faked
```

**Common mistakes:** `OR` everywhere "to keep it simple" (defeats the purpose of using blockchain at all); `AND(all orgs)` everywhere (one peer down halts everything).

**State-based endorsement** (advanced, worth knowing exists): a chaincode can set a *per-key* policy that overrides the chaincode default — e.g. after a transfer, only the new owner + next recipient can sign further changes to that specific batch, even the original owner is locked out. Useful for supply chain since the policy travels with ownership. See `ctx.GetStub().SetStateValidationParameter(key, policy)`.

## Ops facts worth knowing

- Certificates **expire** (often 1 year default). A system that's been silent and fine for a year suddenly dying completely is a real, common Fabric incident — alert 60 days before expiry.
- Revocation happens via the CA's CRL; peers need their MSP config updated. Find out who owns that process.

## Summary

```
1. Fabric verifies WHO. Chaincode must enforce WHAT THEY CAN DO.
2. Build the authorization matrix before coding; test every ❌.
3. Endorsement policy = your trust model. A business decision — yours, not the AI's.
```

➡️ **[06 — Lifecycle & Deploy](06-lifecycle-deploy.md)**
