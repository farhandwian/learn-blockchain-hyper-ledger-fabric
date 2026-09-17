# 06 — Lifecycle & Deploy (Hands-On)

Open a terminal for this one. Prereqs: Docker + Compose, curl, git.

## Why deploy is a multi-step approval process

Fabric 1.x let one org deploy chaincode alone — dangerous, since chaincode is a rule binding everyone. Fabric 2.x requires:

```
1. PACKAGE  — bundle code into .tar.gz, get a PACKAGE ID
2. INSTALL  — each org installs it on their own peer
3. APPROVE  — each org approves a DEFINITION (name, version, SEQUENCE,
              endorsement policy...) — approving the definition, not the code
4. COMMIT   — one org commits once enough orgs have approved
```

### 🔑 SEQUENCE — source of most deploy failures

Every time the definition changes (new code, new policy, anything) → sequence must go up by 1 → every org re-approves → commit again. Forget to bump it, and the old chaincode keeps running silently — no error, just confusion about why your change never showed up.

🚩 If an AI-generated deploy script hardcodes sequence `1`, that's wrong — read it from `peer lifecycle chaincode querycommitted` and increment.

## Bring up a test network

```bash
mkdir -p ~/fabric && cd ~/fabric
curl -sSLO https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/install-fabric.sh
chmod +x install-fabric.sh
./install-fabric.sh --fabric-version 2.5.9 docker samples binary
cd fabric-samples/test-network

./network.sh up createChannel -c mychannel -ca -s couchdb
./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-go -ccl go
```

Watch the output — you'll see the 4 lifecycle steps happen for real.

```bash
export PATH=${PWD}/../bin:$PATH FABRIC_CFG_PATH=$PWD/../config/
source ./scripts/envVar.sh && setGlobals 1

peer chaincode invoke -o localhost:7050 --tls \
  --cafile "${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem" \
  -C mychannel -n basic \
  --peerAddresses localhost:7051 --tlsRootCertFiles ".../peer0.org1.example.com/tls/ca.crt" \
  --peerAddresses localhost:9051 --tlsRootCertFiles ".../peer0.org2.example.com/tls/ca.crt" \
  -c '{"function":"InitLedger","Args":[]}'

peer chaincode query -C mychannel -n basic -c '{"Args":["GetAllAssets"]}'
```

> Notice the **two** `--peerAddresses` — the default policy is `AND(Org1, Org2)`. Remove one and rerun: you'll get `ENDORSEMENT_POLICY_FAILURE`. Now you know what that looks like.

## 🧪 Do this: trigger an MVCC conflict yourself

The single most useful 10 minutes in this whole reference — it makes the "success ≠ saved" failure from [02](02-transaction-flow.md) concrete.

```bash
for i in $(seq 1 10); do
  peer chaincode invoke -o localhost:7050 --tls \
    -C mychannel -n basic \
    --peerAddresses localhost:7051 --tlsRootCertFiles ... \
    --peerAddresses localhost:9051 --tlsRootCertFiles ... \
    -c "{\"function\":\"TransferAsset\",\"Args\":[\"asset1\",\"owner$i\"]}" &
done
wait

docker logs peer0.org1.example.com 2>&1 | grep -i "MVCC\|invalid" | tail -20
```

Every invoke returns `status:200`. Only one write actually lands — check with a query. This is exactly why the client must retry MVCC conflicts and never assume 200 means saved.

## Upgrading: there is no schema migration

```bash
./network.sh deployCC -ccn basic -ccp <path> -ccl go -ccv 2.0 -ccs 2
#                                                            ^^ sequence bumped
```

Old data stays in the old format forever, until a transaction touches it. Chaincode v2 must handle both:

```go
type Batch struct {
    Grade string `json:"grade,omitempty"`   // new field — must be optional
}
// on read: if b.Grade == "" { b.Grade = "UNKNOWN" }
```

✅ safe: adding optional fields, new functions. ⛔ dangerous: removing fields, changing types, changing key format — old data becomes unreadable or unreachable.

🚩 Ask the AI, every time a struct changes: *"How does this read data written by the old version? Show the test."*

## Reading logs

```bash
docker logs -f peer0.org1.example.com     # validation, commits, chaincode errors
docker logs -f orderer.example.com        # block/Raft issues
export FABRIC_LOGGING_SPEC=INFO:kvledger=DEBUG:vscc=DEBUG   # more detail
```

| See this | Means |
|---|---|
| `MVCC_READ_CONFLICT` | concurrency conflict — client needs to retry |
| `ENDORSEMENT_POLICY_FAILURE` | wrong/missing endorsers |
| `chaincode not found` | not committed yet, or wrong name/channel |
| `proposal response was not successful` | chaincode returned an error during simulation |

## Pre-production checklist

```
[ ] Sequence read from querycommitted, not hardcoded
[ ] Endorsement policy set explicitly
[ ] New chaincode version proven to read old data (tested)
[ ] Every org approved before commit was attempted
[ ] Monitoring: cert expiry, block height, invalid-tx rate
```

> ⚠️ **There is no rollback.** Bad data written by a buggy v2 stays on the ledger forever — the fix is a v3 that writes a correction. "Ship now, fix later" is far more expensive here than in a normal app.

➡️ **[07 — Supply Chain Case Study](07-supply-chain-case-study.md)**
