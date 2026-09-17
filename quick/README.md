# Working on a Fabric project through an AI agent

You're right: most Fabric code is stuff an AI writes and fixes fine. This doc is only about the part where that breaks down.

---

## The one thing that makes Fabric different

In a normal app, wrong code **crashes**. You get a stack trace, paste it to the AI, it fixes it. That loop works.

In Fabric, wrong code usually returns **`200 OK` and silently does nothing**.

That's the whole problem. "Analyze the code and fix the bug" requires you to know a bug exists. Fabric's most common failures produce no error, no crash, no failing test — just data that quietly isn't there, noticed three weeks later.

So your job is narrow:

- Know the **5 things that fail silently** (so you know when to point the AI at something).
- Make the **4 decisions the AI can't make** (they're business decisions, not code decisions).
- Everything else — the API, composite keys, CouchDB indexes, SDK wiring — let the AI handle it. You don't need to learn it.

---

## 30 seconds of Fabric

A shared database owned by several companies that don't fully trust each other. Every change is signed by an agreed set of them, and history is append-only.

Three facts, and every silent failure below comes from one of them:

1. **Your code runs on several machines at once, and the results must match byte-for-byte.**
2. **Nothing is saved at the moment your code runs** — it's simulated, then ordered, then validated, then committed. It can be rejected at the end.
3. **Nothing can ever be deleted.**

---

## The 5 silent failures

| # | What | Why AI writes it | How it shows up |
|---|---|---|---|
| 1 | **Non-determinism**<br>`time.Now()`, `rand`, `uuid`, HTTP calls, Go map iteration inside chaincode | It looks like completely normal code | Works perfectly in dev (1 peer). Fails randomly once 2+ orgs endorse. |
| 2 | **Hot key**<br>a global counter, a total, an "index" stored as one array | It's the obvious way to track a total | Works when you click through it by hand. Loses writes under concurrency — one wins, the rest are silently discarded. |
| 3 | **Success ≠ saved**<br>client returns success right after submitting | That's how every other API works | Your API says OK. The data isn't there. |
| 4 | **No authorization check**<br>chaincode doesn't verify who's calling | Fabric *does* verify identity, so it looks handled | Nothing shows up. It works — for everybody, including the wrong people. |
| 5 | **Permanent data**<br>personal data, photos, files written to the ledger | It's just a field on a struct | Nothing, until legal asks you to delete it and you can't. |

**Fabric verifies *who* the caller is. It never enforces *what they're allowed to do*.** That's #4, and it's the one AI misses most often, because the code looks secure.

---

## The 4 decisions AI can't make for you

There's no right answer in the code for these. The AI will pick a default, and the default is usually wrong.

1. **How many organizations must sign each change?** (endorsement policy)
   AI defaults to "any one org." That means one company can unilaterally declare a transfer happened — which deletes the reason you're using blockchain at all. Ask yourself per function: *if this org lied alone, would anyone be harmed?* If yes, require both.

2. **Who's allowed to call each function?**
   Write the grid: functions down the side, organizations across the top, ✅/❌ in each cell. Hand it to the AI. Without it, the AI writes no checks.

3. **What data goes on the shared ledger vs. stays private vs. stays out entirely?**
   Default: IDs, status, quantities, and hashes go on. Prices go in a private collection. Personal data and files stay off entirely — only their hash goes on.

4. **Which status changes are legal?**
   Draw the state machine. `HARVESTED → SHIPPED → RECEIVED → SOLD`. Without it, AI writes `status = newStatus` and your goods can jump from "harvested" straight to "sold," or go backwards.

These four are ~2 hours of your thinking, and they're the actual job.

---

## The lazy version: constrain up front, don't review after

Reviewing AI output for these is work. Preventing them is a copy-paste. Save this as `CLAUDE.md` in the chaincode repo — the agent reads it automatically on every task.

```markdown
# Hyperledger Fabric chaincode — hard rules

## Never (these fail silently in production, not in dev)
- No `time.Now()`, `new Date()` — use `stub.GetTxTimestamp()`
- No `rand`, `uuid`, `Math.random` — use `stub.GetTxID()` or an ID from the client
- No HTTP/fetch/axios calls, no reading env vars or files
- No iterating a Go map without sorting the keys first
- No floats for money or quantity — use integers in the smallest unit
- No global counters, totals, or array-valued "index" keys (causes lost writes
  under concurrency) — aggregate off-chain from events instead
- No `GetState` on a key after `PutState` on it in the same function
- Never use a range/rich query result to decide whether a write is allowed
- No personal data, photos, or files in state — store a SHA-256 hash + URI
- Never `return nil` on a failure path — return a real error

## Always
- Every write function checks the caller: `ctx.GetClientIdentity().GetMSPID()`,
  and checks ownership where relevant
- Validate every input (empty, negative, format, allowed enum values)
- Validate the status transition against the state machine before writing
- Check the record doesn't already exist before creating it
- Emit one `SetEvent` per transaction for meaningful changes
- Client code must wait for and check COMMIT status, not just endorsement
- Reads use `evaluateTransaction`, writes use `submitTransaction`
- Client retries `MVCC_READ_CONFLICT` with backoff

## Tests
For every function: 1 happy path + at least 3 negative tests.
Negative tests must cover: wrong org calling it, illegal status transition,
and calling it on a record the caller doesn't own.
```

Then give it the four decisions from above as the task context, not just "build a supply chain chaincode."

---

## The 3 tests that catch almost everything

Insist on these. They're what turns silent failures into loud ones.

1. **Run every test with 2+ organizations endorsing.** A single peer can never detect non-determinism — the results have nothing to be compared against. This one setting catches failure #1 entirely.
2. **Fire 20 parallel transactions at the same record.** Catches #2 and #3. Most should fail with `MVCC_READ_CONFLICT`; confirm the client retries and that your API never reports success for one that didn't commit.
3. **For every ❌ in your authorization grid, a test that attempts it and asserts failure.** Catches #4. AI writes happy-path tests by default — ask for these explicitly.

If you only do one: **#1.** It's a config change, not a test.

---

## Prompts that work better than "fix the bug"

Point at the failure mode, not the symptom:

- *"Audit this chaincode for non-determinism. Anything that could produce different results on two different peers."*
- *"Which keys here could become hot keys under concurrent load? What happens with 100 simultaneous transactions?"*
- *"For each function, who can call it? Show me the check and the test proving other orgs are rejected."*
- *"What data here is permanent on the ledger forever? Should any of it not be?"*
- *"If one org modified their own client app, which business rules could they break?"* — anything they can break is a rule living in the wrong place.

⚠️ AI answers these confidently whether or not it's right. Don't accept "it's safe" — ask for a test that fails if the assumption is wrong.

---

## The one thing to internalize

```
Normal app:  deploy → bug → fix → clean up the bad data
Fabric:      deploy → bug → fix → the bad data is on the ledger forever
```

There's no rollback and no `DELETE`. "Ship it and fix it later" costs much more here than you're used to. That's the entire reason your review step exists, and why the four decisions are worth doing slowly and up front.

---

## Reference (only when you hit something)

Don't read these front to back. Look something up when you need it.

| File | Open it when |
|---|---|
| [01-mental-model.md](01-mental-model.md) | You need to explain Fabric to someone, or decide if it's even the right tool |
| [02-transaction-flow.md](02-transaction-flow.md) | You want to actually understand *why* the 5 failures happen |
| [03-chaincode.md](03-chaincode.md) | You're reading chaincode and don't recognize an API call |
| [04-data-privacy.md](04-data-privacy.md) | Deciding where a specific field should live |
| [05-identity-access.md](05-identity-access.md) | Writing the authorization grid or picking endorsement policies |
| [06-lifecycle-deploy.md](06-lifecycle-deploy.md) | Deploying, upgrading, or a deploy failed. Also "trigger an MVCC conflict yourself" — 10 minutes that makes failure #3 real |
| [07-supply-chain-case-study.md](07-supply-chain-case-study.md) | Starting your own design — copy the structure |
| [08-testing-and-review.md](08-testing-and-review.md) | You got an error and want the cause (the error → cause → action table) |
