# 05 — Identity & Access: MSP, ABAC, Endorsement Policy

> Fabric has two **different** authorization questions, and they're often mixed up:
>
> 1. **"Who is allowed to CALL this function?"** → checked inside chaincode (ABAC)
> 2. **"Who must APPROVE this change?"** → endorsement policy
>
> Both must be correct. The first is often forgotten by AI; the second is often filled in carelessly.

---

## 5.1 Where identity comes from

```
   ┌──────────────────┐
   │  Supplier CA     │  the Certificate Authority owned by the Supplier org
   │  (Fabric CA)     │
   └────────┬─────────┘
            │ issues
            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  X.509 CERTIFICATE  (for user "budi")                       │
   │                                                            │
   │    Subject:  CN=budi, OU=client, O=Supplier                │
   │    Issuer:   Supplier CA                                   │
   │    Extension (custom attributes):                          │
   │        role      = "warehouse_manager"                     │
   │        siteCode  = "JKT-01"                                │
   │    Signature: <signed by Supplier CA>                      │
   └────────────────────────────────────────────────────────────┘
            │
            │ stored in
            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  WALLET (in your application's backend)                     │
   │    - certificate + private key                              │
   │    ⚠️ This is a credential. Treat it like a DB password.    │
   └────────────────────────────────────────────────────────────┘
            │
            │ used to sign transaction proposals
            ▼
   ┌────────────────────────────────────────────────────────────┐
   │  The PEER verifies it via the MSP:                          │
   │    "Was this certificate issued by a CA that's valid for    │
   │     which org?"                                             │
   │    → result: MSP ID = "SupplierMSP"                        │
   └────────────────────────────────────────────────────────────┘
```

> 🔑 **MSP (Membership Service Provider)** = the rules that map a certificate → organization + role. What chaincode sees is the **MSP ID** (e.g. `SupplierMSP`), never a person's name.

⚠️ Fabric only verifies **"this identity is valid and belongs to org X."** It knows nothing about your business rules. Every "who is allowed to do what" rule is chaincode's responsibility.

---

## 5.2 A practical identity model for your application

This is a design decision that's often gotten wrong:

```
   OPTION A — one identity per organization (most common)

   ┌─────────────────┐
   │  Supplier's     │  normal user login (JWT/session, in your own DB)
   │  Backend App    │
   │                 │  every transaction to Fabric uses the
   │  [1 Fabric      │  identity "app-supplier"
   │   identity]     │
   └─────────────────┘
   ✅ Simple, easy wallet management
   ⚠️ The ledger only knows "SupplierMSP did this," not who the person was
      → store the userId in the transaction payload if you need a per-user trail


   OPTION B — one identity per user

   ┌─────────────────┐
   │  Backend App    │  every user gets their own certificate
   │                 │  (enrolled with the CA at registration)
   │  [N identities] │
   └─────────────────┘
   ✅ Per-individual audit trail, strong non-repudiation
   ⚠️ Certificate management, revocation (CRL), rotation → real ops overhead
```

For a supply-chain MVP: **Option A + userId in the payload** is usually enough. Move to B only once per-individual auditing is a real requirement.

🚩 Red flag: a Fabric identity shared with the frontend, or a private key that ends up in a repo or a logged environment variable.

---

## 5.3 ABAC — checking identity inside chaincode

This is the thing **most often missing** from AI-generated code. To repeat: Fabric does not enforce your business rules.

```go
// Most common approach: check the organization
mspID, _ := ctx.GetClientIdentity().GetMSPID()
if mspID != "SupplierMSP" {
    return fmt.Errorf("only the Supplier may create a batch")
}

// Check a custom certificate attribute
ok, _ := ctx.GetClientIdentity().AssertAttributeValue("role", "warehouse_manager")
if !ok {
    return fmt.Errorf("requires the warehouse_manager role")
}

// Check ownership — the most important pattern in supply chain
batch := getBatch(ctx, id)
if batch.Owner != mspID {
    return fmt.Errorf("this batch belongs to %s, you are %s", batch.Owner, mspID)
}
```

### Authorization matrix — build this BEFORE coding

This is an artifact **you** should define, not the AI:

```
   ┌──────────────────┬──────────┬─────────────┬──────────┬─────────┐
   │ Function         │ Supplier │ Distributor │ Retailer │ Auditor │
   ├──────────────────┼──────────┼─────────────┼──────────┼─────────┤
   │ CreateBatch      │    ✅    │      ❌     │    ❌    │   ❌    │
   │ ShipBatch        │  ✅ own  │   ✅ own    │    ❌    │   ❌    │
   │ ReceiveBatch     │    ❌    │      ✅     │    ✅    │   ❌    │
   │ RecallBatch      │    ✅    │      ❌     │    ❌    │   ❌    │
   │ ReadBatch        │    ✅    │      ✅     │    ✅    │   ✅    │
   │ ReadPrice (PDC)  │ ✅ if a  │  ✅ if a    │    ❌    │   ❌    │
   │                  │ party    │  party      │          │         │
   └──────────────────┴──────────┴─────────────┴──────────┴─────────┘

   "own" = only for a batch it currently owns
```

🧪 **Required test:** for every ❌ in the table, write a test that attempts it and **confirms it fails**. This is a negative test, and it's almost always forgotten.

---

## 5.4 Endorsement policy — your application's trust model

The question is: *"How many signatures are needed for this change to be valid?"*

```
   POLICY: AND('SupplierMSP.peer', 'DistributorMSP.peer')

   ┌───────────────────────────────────────────────────────────┐
   │  Transaction "transfer BATCH007 from Supplier to Distributor"│
   │                                                           │
   │  Endorsed by:                                             │
   │    ✅ Peer Supplier     — "yes, I agree to release it"    │
   │    ✅ Peer Distributor  — "yes, I agree to receive it"    │
   │                                                           │
   │  → VALID. Neither party can claim the transfer            │
   │    unilaterally.                                          │
   └───────────────────────────────────────────────────────────┘

   POLICY: OR('SupplierMSP.peer', 'DistributorMSP.peer')

   ┌───────────────────────────────────────────────────────────┐
   │  ⚠️ Supplier ALONE can change a batch's ownership to      │
   │     the Distributor, with no approval needed.             │
   │                                                           │
   │  Sometimes that's actually what you want (e.g. a supplier │
   │  recording its own production). Sometimes it's a huge hole.│
   └───────────────────────────────────────────────────────────┘
```

### Syntax

| Policy | Meaning |
|---|---|
| `OR('A.peer','B.peer')` | either one is enough |
| `AND('A.peer','B.peer')` | both are required |
| `OutOf(2, 'A.peer','B.peer','C.peer')` | at least 2 of 3 |
| `AND('A.peer', OR('B.peer','C.peer'))` | can be nested |
| `'A.member'` vs `'A.peer'` vs `'A.admin'` | roles within an org — `peer` is the most common |

### 3 levels of endorsement policy

```
   ┌─────────────────────────────────────────────────────────────────┐
   │ 1. CHAINCODE-LEVEL  (default, set when the chaincode is committed)│
   │    Applies to EVERY function in that chaincode.                  │
   │    Example: "majority of orgs"                                   │
   ├─────────────────────────────────────────────────────────────────┤
   │ 2. STATE-BASED / KEY-LEVEL  (set from inside chaincode)          │
   │    Applies to one specific KEY, overriding the chaincode policy. │
   │    ⭐ A great fit for supply chain: the policy travels along     │
   │       with the goods' ownership.                                  │
   ├─────────────────────────────────────────────────────────────────┤
   │ 3. COLLECTION-LEVEL  (in the PDC definition)                     │
   │    Applies to private data.                                      │
   └─────────────────────────────────────────────────────────────────┘
```

### State-based endorsement — an elegant pattern for supply chain

```
   BATCH007 is currently held by the Supplier
        │  key-level policy: AND(Supplier, Distributor)
        │  → a transfer needs both parties' approval
        ▼
   [ transfer to Distributor ]
        │  chaincode updates that key's policy to:
        │  AND(Distributor, Retailer)
        ▼
   BATCH007 is now held by the Distributor
        │  → the Supplier can NO LONGER change this batch,
        │    even if the chaincode itself gets bypassed.
```

The code:

```go
ep, _ := statebased.NewStateEP(nil)
ep.AddOrgs(statebased.RoleTypePeer, "DistributorMSP", "RetailerMSP")
policy, _ := ep.Policy()
ctx.GetStub().SetStateValidationParameter(batchID, policy)
```

Why this is powerful: enforcement happens at the **validation layer**, not in chaincode logic. Even if there's a bug in the chaincode, a peer will still reject a transaction lacking the right signatures.

---

## 5.5 Common endorsement-policy mistakes

| Mistake | Consequence |
|---|---|
| `OR(...)` on every function "to keep it simple" | One org can forge a change unilaterally. This defeats the whole reason for using blockchain. |
| `AND(all 6 orgs)` on every function | One peer down = the entire system stops. High latency. |
| Policy doesn't match the endorsers the client actually calls | `ENDORSEMENT_POLICY_FAILURE` — usually shows up as "why does this keep failing?" |
| Forgetting to update the policy when a new org joins | The new org can't transact |
| Chaincode writes private data but the policy doesn't restrict endorsers to collection members | Transaction fails, or data leaks |

### How to choose a policy — a guiding question

For every function, ask:

```
   "If organization X lied about this all by itself,
    would anyone be harmed?"

        YES → needs AND (or OutOf) that includes the harmed party
        NO  → OR / a single org is enough

   Examples:
   - CreateBatch (a farmer recording their own harvest)
     → nobody is harmed  → OR(Farmer) is enough
   - TransferOwnership
     → the recipient could be "shipped to" without knowing → AND(sender, recipient)
   - RecordTemperature from a sensor
     → everyone is harmed if it's faked → AND(owner, auditor/oracle)
```

---

## 5.6 Ops things you need to know

- **Revocation**: a leaked certificate is revoked via the CA's CRL. Peers need their MSP configuration updated. This is a manual process — find out who's responsible for it.
- **Expiry**: certificates have a validity period (often 1 year by default). 🚨 A system that's been running smoothly for a year and then suddenly dies completely is a real, common Fabric incident. Set up certificate-expiry monitoring from day one.
- **TLS**: peer-orderer-client communication uses mTLS with certificates separate from the transaction identity. Don't mix them up.

🧪 Add this to your ops checklist: alert 60 days before any certificate expires.

---

## 5.7 Summary

1. Fabric guarantees *who* someone is; the rules for *what they're allowed to do* are chaincode's job (ABAC).
2. Build an **authorization matrix** before coding, and test every "❌" cell.
3. **The endorsement policy is your trust model** — it's a business decision, not a technical config value. You must decide it, not the AI.
4. **State-based endorsement** lets the policy travel with ownership — the best-fit pattern for supply chain.
5. Monitor certificate expiry from the start.

➡️ Continue to **[06 — Lifecycle & Deploy](06-lifecycle-deploy.md)** (the hands-on section).
