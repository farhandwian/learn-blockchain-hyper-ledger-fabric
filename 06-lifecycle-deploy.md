# 06 — Lifecycle & Deploy (Hands-On Section)

> Open a terminal. This section is meant to be done, not just read.
>
> Prerequisites: Docker + Docker Compose, Go 1.20+ (optional), Node 18+ (optional), curl, git.

---

## 6.1 Chaincode Lifecycle v2 — why it's this complicated

In Fabric 1.x, a single org could deploy chaincode on its own. That's dangerous: chaincode is a set of rules binding on every party.

In Fabric 2.x, deployment is a **joint approval process**:

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ 1. PACKAGE     — bundle the code into a .tar.gz                 │
   │                  Produces a PACKAGE ID (a hash of the contents)  │
   │                  Done once, then shared                          │
   └──────────────────────────────────────────────────────────────────┘
                                  │
   ┌──────────────────────────────▼───────────────────────────────────┐
   │ 2. INSTALL     — each ORG installs the package on its own peer   │
   │                  Supplier: install ✅                            │
   │                  Distributor: install ✅                         │
   └──────────────────────────────────────────────────────────────────┘
                                  │
   ┌──────────────────────────────▼───────────────────────────────────┐
   │ 3. APPROVE     — each ORG approves the DEFINITION:                │
   │                    { name, version, SEQUENCE, endorsement        │
   │                      policy, collections config, init-required } │
   │                                                                  │
   │                  ⚠️ What gets approved is the DEFINITION, not    │
   │                     the code. Every org must approve the SAME    │
   │                     definition.                                  │
   └──────────────────────────────────────────────────────────────────┘
                                  │
   ┌──────────────────────────────▼───────────────────────────────────┐
   │ 4. COMMIT      — one org commits it to the channel               │
   │                  Only succeeds if enough orgs have approved       │
   │                  (per the channel's LifecycleEndorsement policy) │
   │                  → the chaincode is now ACTIVE                   │
   └──────────────────────────────────────────────────────────────────┘
```

### 🔑 SEQUENCE — the source of 90% of deploy failures

```
   Sequence is the DEFINITION's version number, starting at 1.

   Every time you change ANYTHING about the definition
   (new code, new version, new policy, new collection)
   → sequence MUST go up by 1
   → EVERY org must approve AGAIN with the new sequence
   → then commit again

   ⛔ The most common failure:
      "I changed the code, but the sequence is still 1"
      → nothing happens, the old chaincode keeps running,
        and you're left confused about why your change never showed up.
```

🚩 Red flag in an AI-generated deploy script: the sequence is hardcoded to `1`. It must be read from `peer lifecycle chaincode querycommitted`, then incremented.

---

## 6.2 Hands-on: run the test-network

```bash
# 1. Fetch fabric-samples + binaries + docker images
mkdir -p ~/fabric && cd ~/fabric
curl -sSLO https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/install-fabric.sh
chmod +x install-fabric.sh
./install-fabric.sh --fabric-version 2.5.9 docker samples binary

cd fabric-samples/test-network

# 2. Bring up a 2-org network + channel + CouchDB
./network.sh up createChannel -c mychannel -ca -s couchdb

# 3. See what's running
docker ps --format "table {{.Names}}\t{{.Status}}"
```

You should see:

```
   peer0.org1.example.com     ← Org1's peer
   peer0.org2.example.com     ← Org2's peer
   orderer.example.com        ← the ordering service
   couchdb0 / couchdb1        ← world state
   ca_org1 / ca_org2 / ca_orderer
```

```bash
# 4. Deploy the sample chaincode
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-go -ccl go
```

Read the output slowly — you'll see exactly the 4 lifecycle steps above play out.

---

## 6.3 Hands-on: manual invoke & query

```bash
# Set your environment as Org1 (available in test-network)
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=$PWD/../config/
source ./scripts/envVar.sh
setGlobals 1

# INIT sample data
peer chaincode invoke -o localhost:7050 \
  --ordererTLSHostnameOverride orderer.example.com --tls \
  --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" \
  -C mychannel -n basic \
  --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" \
  --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" \
  -c '{"function":"InitLedger","Args":[]}'

# QUERY (evaluate — never touches the ledger)
peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```

> 🔑 Notice the **two** `--peerAddresses` flags. That's because the test-network's default endorsement policy is `AND(Org1, Org2)`.
>
> 🧪 **Required experiment:** remove one `--peerAddresses` and run it again. You'll see `ENDORSEMENT_POLICY_FAILURE`. Now you know what that error actually looks like.

---

## 6.4 Hands-on: trigger an MVCC conflict yourself

This is the single most valuable experiment in this entire material. Do it.

```bash
# Send 10 transfers on the SAME asset, at the same time
for i in $(seq 1 10); do
  peer chaincode invoke -o localhost:7050 \
    --ordererTLSHostnameOverride orderer.example.com --tls \
    --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" \
    -C mychannel -n basic \
    --peerAddresses localhost:7051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt" \
    --peerAddresses localhost:9051 --tlsRootCertFiles "${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt" \
    -c "{\"function\":\"TransferAsset\",\"Args\":[\"asset1\",\"owner$i\"]}" &
done
wait

# Check the results in the peer log
docker logs peer0.org1.example.com 2>&1 | grep -i "MVCC\|VSCC\|invalid" | tail -20
```

You should see lines like:

```
[gossip.privdata] ... Committed block [12] with 10 transaction(s)
[kvledger] ... Channel [mychannel]: Validation... marked as invalid by state validator.
             Reason code [MVCC_READ_CONFLICT]
```

**Think about this:** `peer chaincode invoke` returned `status:200` for nearly all of them. But only 1 actually got saved. **This is the "false success" trap from file 02.5, in the flesh.**

🧪 Check the final result: `peer chaincode query -C mychannel -n basic -c '{"Args":["ReadAsset","asset1"]}'` — only one owner made it through.

---

## 6.5 Upgrading chaincode & schema migration

```bash
# After changing the code:
./network.sh deployCC -ccn basic -ccp <path> -ccl go -ccv 2.0 -ccs 2
#                                                     ^^^^^^   ^^^^^
#                                                     version  SEQUENCE bumped!
```

### ⚠️ There is no database migration in blockchain

This is a concept you must genuinely understand:

```
   Regular database:            Fabric:
   ─────────────────            ────────────────────────────────────
   ALTER TABLE ...              Doesn't exist.
   UPDATE ... SET ...           Old data STAYS in the old format,
   → every row becomes the new  forever, until a transaction happens
     format immediately         to touch it.
```

```
   World state after upgrading v1 → v2:

   BATCH001  {"id":"...","qty":100}                  ← v1 format (old)
   BATCH002  {"id":"...","qty":50}                   ← v1 format (old)
   BATCH003  {"id":"...","qty":75,"grade":"A"}       ← v2 format (new)

   ⚠️ Chaincode v2 MUST be able to read all three without crashing.
```

**The correct strategy:**

```go
type Batch struct {
    ID    string `json:"id"`
    Qty   int    `json:"qty"`
    Grade string `json:"grade,omitempty"`   // new field: optional!
}

func readBatch(...) (*Batch, error) {
    // ...unmarshal...
    if b.Grade == "" {
        b.Grade = "UNKNOWN"   // default for old data
    }
    return &b, nil
}
```

Ground rules:
- ✅ Adding an optional field — safe
- ✅ Adding a new function — safe
- ⚠️ Changing the meaning of an existing field — dangerous, needs explicit versioning (`"schemaVersion": 2`)
- ⛔ Removing a field / changing its type — will crash when reading old data
- ⛔ Changing the key format — old data becomes unreachable

🚩 **A mandatory question for the AI agent every time a struct changes:** *"How does this new chaincode version read a record written by the old version? Show me the test."*

🧪 **Required upgrade test:** deploy v1 → create data → deploy v2 → **read the old data** → confirm no error and the values make sense.

---

## 6.6 Dev mode: fast iteration (Chaincode as a Service)

A full deploy takes 1-2 minutes. Too slow for development. Fabric 2.x supports **Chaincode as a Service** — chaincode runs as an ordinary process/container that you restart yourself.

```
   NORMAL MODE:                      CaaS MODE (dev):
   ─────────────                     ────────────────
   peer builds & runs the            You run the chaincode yourself
   chaincode container                (go run / npm start / a debugger)
        │                                    │
   change code → package →           change code → restart the process
   install → approve → commit        (2 seconds) ✅
   (1-2 minutes) 😴

                                     ⭐ You can attach a debugger and set breakpoints!
```

An example lives in `fabric-samples/asset-transfer-basic/chaincode-external`. Ask your team whether this is already in use — if not, this is the single biggest productivity improvement you could propose.

---

## 6.7 Reading logs — your main debugging skill

```bash
# Peer log (validation, commits, chaincode errors)
docker logs -f peer0.org1.example.com

# Chaincode container log (your chaincode's println/fmt output)
docker ps --format '{{.Names}}' | grep dev-peer
docker logs -f dev-peer0.org1.example.com-basic_1.0-<hash>

# Orderer log (block issues, Raft)
docker logs -f orderer.example.com

# Raise verbosity (set before bringing the network up)
export FABRIC_LOGGING_SPEC=INFO:kvledger=DEBUG:vscc=DEBUG
```

### Keywords to search for

| Search for this | Meaning |
|---|---|
| `MVCC_READ_CONFLICT` | a concurrency conflict — needs a client retry |
| `ENDORSEMENT_POLICY_FAILURE` | not enough / wrong endorsers |
| `PHANTOM_READ_CONFLICT` | a range query collided with a write |
| `INVALID_OTHER_REASON` | usually a chaincode error |
| `chaincode registered` | chaincode started successfully |
| `Committed block [N]` | a block landed — check how many tx were valid |
| `failed to invoke chaincode name:` | chaincode isn't running / wrong name |
| `proposal response was not successful` | chaincode returned an error during simulation |

🧪 Exercise: deliberately trigger the first three errors and note what the log actually looks like. You'll recognize them instantly once you hit them in production.

---

## 6.8 Cleaning up

```bash
./network.sh down          # stop + remove volumes
docker system prune -a     # if disk is full (careful: removes all images)
```

---

## 6.9 Pre-production deploy checklist

```
  [ ] Sequence is read from querycommitted, not hardcoded
  [ ] Endorsement policy is set explicitly, not left as default
  [ ] Collection config (PDC) got approved by every org too
  [ ] CouchDB indexes exist in META-INF and are verified to be used
  [ ] The new chaincode version is proven to read old-version data (tested)
  [ ] Rollback plan: if v2 breaks, what happens?
      (answer: deploy a v3 that fixes it — state CANNOT be rolled back!)
  [ ] Every org approved before commit is attempted
  [ ] Monitoring in place: certificate expiry, block height, invalid-tx rate
```

> ⚠️ **There is no rollback in blockchain.** If chaincode v2 writes bad data to the ledger, that data is permanent. The fix is a v3 that writes a correction. This is exactly why your role as validator matters so much here — "deploy now, fix later" is far more expensive than in an ordinary application.

➡️ Continue to **[07 — Supply Chain Case Study](07-supply-chain-case-study.md)**.
