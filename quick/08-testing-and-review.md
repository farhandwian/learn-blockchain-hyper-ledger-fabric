# 08 — Testing & Review

Your working checklist. Copy the relevant parts into a PR template.

## The 3 tests that catch almost everything

1. **Run every integration test with 2+ orgs endorsing.** A single peer can't detect non-determinism — there's nothing to compare against. This alone catches most of failure #1 from the README.
2. **Fire 20 parallel transactions at the same record.** Catches hot keys and false-success. Confirm the client retries `MVCC_READ_CONFLICT` and the API never reports success for a write that didn't commit.
3. **For every ❌ in your authorization matrix, a test that attempts it and asserts failure.** AI writes happy-path tests by default — ask for these explicitly. Healthy ratio: 1 happy path : 3-5 negative tests per function.

If you only add three integration tests total: MVCC conflict, cross-org authorization, and an upgrade-compatibility test (deploy v1 → write data → deploy v2 → read the old data without crashing).

## PR review checklist

```
DETERMINISM (blocker)
[ ] No time.Now/rand/HTTP/env reads/unsorted map iteration in chaincode
[ ] No floats for money/quantity

AUTHORIZATION (blocker)
[ ] Every write function checks MSP ID / ownership
[ ] All inputs validated; status transitions checked against the state machine
[ ] Errors returned as errors — no silent `return nil`

CONCURRENCY
[ ] No hot key (global counter, aggregate record)
[ ] Client retries MVCC_READ_CONFLICT with backoff
[ ] Query results never used to decide a write

DATA & PRIVACY
[ ] No PII/files in world state — hash + URI only
[ ] Secret data via transient + PDC, never a plain argument
[ ] Endorsing orgs restricted for private-data functions

INTEGRATION
[ ] Client checks COMMIT status, not just submit response
[ ] Reads use evaluate; writes use submit
[ ] New chaincode version tested against old-version data
[ ] Sequence number not hardcoded

TESTS
[ ] Negative test for every authorization rule and illegal transition
[ ] Integration test with ≥2 orgs endorsing
[ ] Concurrency (MVCC) test
```

## Error → cause → action

| Error | Cause | Action |
|---|---|---|
| `MVCC_READ_CONFLICT` | two tx wrote the same key in the same block | client retry; if frequent → hot key, redesign the key |
| `ENDORSEMENT_POLICY_FAILURE` | too few/wrong endorsers | check `--peerAddresses` / `setEndorsingOrganizations` |
| `ENDORSEMENT_MISMATCH` / "response payload differ" | chaincode isn't deterministic | audit for `time.Now`/`rand`/HTTP/map iteration |
| `chaincode definition not agreed` | orgs approved different definitions | align version/policy/sequence across all orgs |
| `chaincode not found` | not committed, or wrong name/channel | `peer lifecycle chaincode querycommitted` |
| Tx succeeds, data unchanged | called via evaluate, not submit | switch to submitTransaction |
| Slow queries | rich query with no index | add a CouchDB index, verify it's used |
| `identity expired` | certificate expired | re-enroll; add expiry monitoring |
| Private data empty | peer isn't a collection member, or data sent as argument not transient | check collection config |
| Events never arrive | tx was INVALID, or listener has no checkpoint | check tx status; add a checkpoint |

## Questions to ask your AI agent

Point at the failure mode, not the symptom:

- *"Audit this for non-determinism — anything that could differ between two peers."*
- *"Which key here could become a hot key under concurrent load?"*
- *"Who can call this function? Show the check and a test proving others are rejected."*
- *"How does this read data written by the previous chaincode version? Show the test."*
- *"What data here is permanent forever? Should any of it not be?"*
- *"If an org modified their own client app, which business rules could they break?"* — anything they can break is a rule in the wrong place.

⚠️ AI answers confidently whether or not it's correct. Don't accept "it's safe" — ask for a test that fails if the assumption is wrong.

## Definition of done

```
[ ] Determinism + authorization checklist items pass
[ ] 1 happy path + 3+ negative tests per function
[ ] Runs on test-network with ≥2 orgs endorsing
[ ] Endorsement policy explicit, with reasoning documented
[ ] Data placement (on-chain/PDC/off-chain) matches the matrix
[ ] Reading old-version data tested
```

## The one thing to remember

```
Normal app:  deploy → bug → fix → clean up the bad data
Fabric:      deploy → bug → fix → the bad data is on the ledger forever
```

No rollback, no DELETE. That's why this checklist exists, and why it's worth actually using instead of skimming.

⬅️ **[README](README.md)**
