# Learning Hyperledger Fabric — for Reviewers & Validators

This material is written for a **software engineer who has never touched blockchain before**, whose job will **not be writing chaincode from scratch**, but rather **designing, testing, and validating** a supply-chain application built on Hyperledger Fabric.

> ⚡ **In a hurry, or mostly delegating the coding to an AI agent?** Read **[quick/README.md](quick/README.md)** instead — a ~10 minute condensed version of everything below, plus a ready-to-paste `CLAUDE.md` block. This full version is the deep-dive reference for when you want to actually understand *why*.

That's why this differs from a typical tutorial:

| Typical tutorial | This material |
|---|---|
| Focus: how to write code | Focus: how to tell code is **wrong** |
| Memorize the API | Understand **why** the API works that way |
| "Hello world runs!" | "Why is this transaction INVALID even though the response said success?" |

---

## Learning map

```
                        START HERE
                              |
        +---------------------+---------------------+
        |                                           |
   [ FOUNDATIONS ]                              (skip if
        |                                       you already know)
        v
  01 Mental Model ............... What Fabric is, and is NOT
        |
        v
  02 Transaction Flow  <<<<  CORE. 80% of bugs live here
        |
        v
        +---------------+---------------+
        |               |               |
        v               v               v
  03 Chaincode    04 Data Privacy   05 Identity
   (state,          (channel,        & Access
    query,           private data,    (MSP, ABAC,
    event)           on/off-chain)    endorsement)
        |               |               |
        +---------------+---------------+
                        |
                        v
              06 Lifecycle & Deploy ...... hands-on: test-network
                        |
                        v
              07 Supply Chain Case Study  <<< real-world design
                        |
                        v
              08 Testing & Review .......  your daily checklist
```

---

## Table of contents

| # | File | Contents | Time |
|---|------|-----|-------|
| 01 | [01-mental-model.md](01-mental-model.md) | What Fabric is, its architecture, components, when you DON'T need blockchain | ~45 min |
| 02 | [02-transaction-flow.md](02-transaction-flow.md) | **Mandatory.** Execute-Order-Validate, read/write sets, MVCC, determinism | ~90 min |
| 03 | [03-chaincode.md](03-chaincode.md) | Chaincode anatomy, world state, composite keys, queries, events | ~60 min |
| 04 | [04-data-privacy.md](04-data-privacy.md) | Channels, Private Data Collections, on-chain vs off-chain | ~45 min |
| 05 | [05-identity-access.md](05-identity-access.md) | CA, MSP, wallets, ABAC, endorsement policy | ~60 min |
| 06 | [06-lifecycle-deploy.md](06-lifecycle-deploy.md) | Deploying chaincode, upgrades, schema migration, reading logs | ~60 min + hands-on |
| 07 | [07-supply-chain-case-study.md](07-supply-chain-case-study.md) | End-to-end design of a supply-chain application | ~60 min |
| 08 | [08-testing-and-review.md](08-testing-and-review.md) | How to test, required scenarios, review checklist, error table | ~60 min |

Total: ~7-8 hours of reading + 1 day of hands-on practice.

---

## How to use this material

1. **Read 01 and 02 first, don't skip ahead.** File 02 is the key. If you understand file 02, you already understand Fabric better than most people who just followed a tutorial.
2. **Files 03-05 can be read as needed.**
3. **File 06 is meant to be done, with a terminal open.** Don't just read it.
4. **Print or pin file 08.** It's your daily working checklist when reviewing code from an AI agent.

Every file has a **"Red flag when reviewing"** section — that's the part most directly relevant to your job.

---

## Notation used

> 🔑 = key concept, memorize it
>
> ⚠️ = common trap, frequently causes bugs
>
> 🚩 = red flag when reviewing code
>
> 🧪 = something you must test

Fabric version referenced throughout: **Fabric 2.5 LTS** (lifecycle v2, Gateway API).
