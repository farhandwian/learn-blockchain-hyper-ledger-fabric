# 01 — Mental Model

## What it is

A database jointly owned by several companies that don't fully trust each other. Every change must be signed by an agreed set of them, and history can't be edited or deleted.

No coin, no mining, no gas. Forget everything Ethereum-related.

```
   OLD WAY: everyone has their own DB, reconciles by email/report,
            argues when the numbers don't match.

   FABRIC:  one shared ledger. Every entry is signed. Everyone has
            a copy. Rules are enforced automatically.
```

## Fabric vs Ethereum (why your instincts about "blockchain" are wrong here)

| | Ethereum / Bitcoin | **Fabric** |
|---|---|---|
| Who can join | Anyone, anonymous | Only invited orgs, known identity |
| Consensus | Mining / staking | Raft ordering, no competition |
| Cost | Gas, paid in tokens | None |
| Contract language | Solidity | Plain Go / JS / Java |
| Data | Public | Private, per-channel |
| Execution order | Order → Execute | **Execute → Order → Validate** ⬅ see file 02 |

## When you don't actually need this

```
Multiple parties writing data?
  NO  → just use a regular database
  YES → do they fully trust each other?
          YES → shared DB + API is enough
          NO  → do you need an undeniable audit trail?
                  NO  → DB + log
                  YES → Fabric makes sense
```

⚠️ Blockchain guarantees *who claimed what, and that the claim can't be altered afterward*. It does **not** guarantee the claim was true. A supplier can enter a fake temperature reading and the ledger will store that lie permanently and neatly (the "oracle problem" — see file 07).

## Network components

```
                        ┌────────────────┐
                        │ Client App     │
                        │ (your backend) │
                        └────────────────┘
                                 │
        ┬────────────────────────┴───────────────────────┬
        │                                                │
    ORGANIZATION: Supplier            ORGANIZATION: Distributor
┌─────────────────────────────┐    ┌─────────────────────────────┐
│ ┌───────────┐ ┌───────────┐ │    │ ┌───────────┐ ┌───────────┐ │
│ │ Peer 0    │ │ CA        │ │    │ │ CA        │ │ Peer 0    │ │
│ │ Chaincode │ │ (ID card) │ │    │ │ (ID card) │ │ Chaincode │ │
│ │ Ledger    │ └───────────┘ │    │ └───────────┘ │ Ledger    │ │
│ └───────────┘               │    │               └───────────┘ │
└─────────────────────────────┘    └─────────────────────────────┘
        │                                                │
        ┴────────────────────────┬───────────────────────┴
                                 │
                     ┌──────────────────────┐
                     │ ORDERING SERVICE     │
                     │ Job is ONLY to:      │
                     │ order tx into blocks │
                     └──────────────────────┘
```

| Component | Job |
|---|---|
| **Peer** | Stores the ledger, runs chaincode, validates & commits blocks |
| **Orderer** | Only sequences transactions into blocks. Never looks at the content. |
| **CA** | Issues identity certificates. Used once at setup, not on the transaction path. |
| **Channel** | A separate ledger. Orgs outside it have none of its data. |
| **Chaincode** | The smart contract — plain Go/JS code |
| **MSP** | Maps a certificate → organization + role |

> 🔑 The orderer executes nothing. This is the opposite of Ethereum, where miners run the code.

## The ledger = 2 parts

- **Blockchain**: append-only history. Stores VALID *and* INVALID transactions (flagged). Can't be edited.
- **World state**: current key-value snapshot. This is what `GetState()` reads. Can be updated/deleted — but the old value stays in the blockchain forever.

⚠️ `DelState()` only removes it from world state. The value still exists permanently in history. **Never put personal or secret data on the ledger.**

## App flow

```
Frontend → your Backend (holds the Fabric identity/wallet) → Fabric network
```

The frontend never talks to Fabric directly, and never holds a private key. Your backend is still a normal backend: auth, validation, rate limiting are all still your job — Fabric only replaces the shared storage layer.

➡️ **[02 — Transaction Flow](02-transaction-flow.md)** — this is the one that actually matters.
