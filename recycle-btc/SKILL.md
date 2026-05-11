---
name: recycle-btc
description: BTC→XMR recycle for the eigenwallet ASB maker running in Tyler's homelab k8s cluster. Use when ASB has accumulated BTC and XMR inventory is low (effective max_buy_btc XMR-constrained), to swap BTC back to XMR via a competing maker, capturing ~1–2% spread differential.
allowed-tools: Bash(kubectl *), Bash(curl *), Bash(jq *), Bash(python3 *), Bash(date *), Bash(sleep *), Bash(echo *), Bash(awk *), Bash(grep *), Bash(sed *), Bash(head *), Bash(tail *), Bash(test *), Bash(cat *)
---

# recycle-btc

Take BTC accumulated in the eigenwallet ASB maker and swap it back to XMR via a competing maker, capturing the spread differential. This refills XMR inventory so the maker can keep selling, and earns ~1–2% per cycle net of fees.

The ASB runs in the `eigenwallet` namespace on Tyler's homelab cluster. This skill orchestrates `swap-cli` (the patched taker CLI) against rendezvous-discovered makers.

## When to invoke

- ASB XMR inventory is low — `max_buy_btc` is constrained by XMR balance rather than the configured cap
- ASB BTC balance is well above what's needed to backstop any in-flight maker refunds
- Operator wants to rebalance inventory toward XMR
- After a wave of taker swaps where BTC piled up

## When NOT to invoke

- Pool / cluster health issues — check `kubectl get pods -n eigenwallet` shows everything `Running` and `flux get hr -A` is clean
- Active swap in non-terminal state on the asb side — check `asb-controller get-swaps`
- Mempool fees > ~10 sat/vB — BTC tx fees can eat the recycle margin
- An existing recycle is already running on the `swap-cli-taker` pod (only one can run at a time, the wallet locks force serialization)

## Hard-learned lessons (do not skip these)

1. **Always pass `--monero-node-address http://<monerod-clusterip>:18081`** to `swap buy-xmr` / `swap resume`. The CLI's internal `monero-rpc-pool` connects to random public Monero nodes over Tor and will silently get stuck after finding the maker's XMR lock — the scanner stops advancing and the swap freezes in `xmr lock transaction seen` indefinitely. Using the cluster's local monerod (which is fully synced and reachable in microseconds) avoids this entirely.
2. **The argument needs `http://` scheme.** Bare `<ip>:<port>` produces `Invalid value: relative URL without a base`. Use `http://10.97.93.76:18081` form (the ClusterIP of the `monerod` Service in the `eigenwallet` namespace).
3. **monerod's parser is strict — it wants an IP, not a hostname.** Same was true for the asb's `--proxy` flag. Use the Service ClusterIP, not `monerod` hostname. The ClusterIP is stable for the life of the Service.
4. **Maker selection matters.** Some makers advertise but never lock XMR (ghost makers — observed with EihLFt at v4.5.3 despite having 1.6 BTC inventory and a newer version than us). Bias toward makers that:
   - Have a moderately recent version (≥ v4.1.x; avoid v4.0.x)
   - Have large `max_quantity` (suggests active inventory, not stale advertising)
   - Have a Tor onion address (suggests a real long-running maker, not a quick-spin clearnet experiment)
   - If a maker fails to lock XMR within ~10 min of BTC confirmation, blacklist and pick another counterparty next time
5. **The amnesty path saves you on a no-lock failure.** If the maker never broadcasts XMR lock, the swap auto-cancels after ~3 hours and refunds the BTC minus ~4000 sats in cancel fees (~$3). This is the protocol working as designed; do not panic when it triggers.
6. **The swap-cli pod runs as a long-lived sleep-infinity pod** with the taker wallet mounted from the NFS PV at `/mnt/media/eigenwallet/market-observer/data`. Manifests live ad-hoc (not in the homelab repo) at this point — `kubectl get pod -n eigenwallet swap-cli-taker` to check, recreate from the inline YAML in §"Stand up the taker pod" if missing.

## Procedure

### 1. Sanity checks

```sh
kubectl get pods -n eigenwallet -o wide
kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 bitcoin-balance
kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 monero-balance
kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 get-current-quote
```

Note the current quote price — this is the rate at which we sell XMR. Our recycle needs to buy XMR at a price lower than this to capture spread.

### 2. Scan the market

Spin up an ephemeral `swap-cli list-sellers` pod (the scan takes ~4 min total — 3 min Tor bootstrap + 60s scan window):

```sh
kubectl run -n eigenwallet swap-cli-scan --restart=Never \
  --image=ghcr.io/tylerjw/eigenwallet-swap-cli:4.5.3 \
  -- -d /tmp/swap-cli list-sellers --electrum-rpc tcp://electrs:50001 --enable-tor --wait-seconds 60

# Wait ~4 min then read logs
sleep 270
kubectl logs -n eigenwallet swap-cli-scan \
  | awk '/^\[/{found=1} found{print} /^\]/{print; exit}' > /tmp/sellers.json
kubectl delete pod -n eigenwallet swap-cli-scan --grace-period=1
```

### 3. Analyze + pick counterparty

Fetch CEX reference + rank against our quote:

```sh
KRAKEN_ASK=$(curl -s "https://api.kraken.com/0/public/Ticker?pair=XMRXBT" | python3 -c "import sys,json; print(list(json.load(sys.stdin)['result'].values())[0]['a'][0])")
KUCOIN_ASK=$(curl -s "https://api.kucoin.com/api/v1/market/orderbook/level1?symbol=XMR-BTC" | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['bestAsk'])")
OUR_QUOTE=$(kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 get-current-quote | grep -oE '0\.[0-9]+' | head -1)

python3 <<EOF
import json
mid = ($KRAKEN_ASK + $KUCOIN_ASK) / 2
our = $OUR_QUOTE
sellers = json.load(open('/tmp/sellers.json'))
print(f'CEX mid: {mid:.8f} BTC/XMR   Our quote: {our:.8f} ({(our/mid-1)*100:+.2f}% vs mid)')
print()
print(f'{"rk":<3} {"peer":<22} {"price":<14} {"vs CEX":<8} {"vs us":<8} {"max BTC":<10} {"ver":<6} {"tor"}')
rows = []
for s in sellers:
    p = s['quote']['price'] / 1e8
    mx = s['quote']['max_quantity'] / 1e8
    if mx == 0 or p == 0: continue
    rows.append((p, s['peer_id'], mx, s.get('version', '?'), '/onion3/' in s['multiaddr']))
rows.sort()
for i, (p, peer, mx, ver, tor) in enumerate(rows[:10], 1):
    print(f'{i:<3} {peer[:20]:<22} {p:<14.8f} {(p/mid-1)*100:+.2f}%   {(p/our-1)*100:+.2f}%   {mx:<10.4f} {ver:<6} {"yes" if tor else "no"}')
EOF
```

**Counterparty pick heuristic:** prefer rank 1 IF it has v4.1.1+ and tor-reachable AND max ≥ our intended recycle amount × 2. Otherwise skip to the next-cheapest matching these criteria.

### 4. Decide recycle amount

- **Min:** at least 0.005 BTC (typical maker `min_quantity`)
- **Max:** what asb can spare while keeping a buffer for refunds/fees (rule of thumb: leave 0.02 BTC + 0.001 per active swap in asb)
- **Sweet spot:** 0.05 BTC (~10 XMR back, ~$30–50 spread captured net of ~$5 fees)

### 5. Ensure taker pod exists

```sh
kubectl get pod -n eigenwallet swap-cli-taker 2>/dev/null | grep -q Running || \
  kubectl apply -f - <<'YAML'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: eigenwallet-taker-data
spec:
  capacity: { storage: 1Gi }
  accessModes: [ReadWriteMany]
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.1.195
    path: /mnt/media/eigenwallet/market-observer/data
  mountOptions: [nfsvers=4]
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: taker-data
  namespace: eigenwallet
spec:
  accessModes: [ReadWriteMany]
  storageClassName: ""
  volumeName: eigenwallet-taker-data
  resources: { requests: { storage: 1Gi } }
---
apiVersion: v1
kind: Pod
metadata:
  name: swap-cli-taker
  namespace: eigenwallet
spec:
  restartPolicy: Never
  containers:
    - name: swap-cli
      image: ghcr.io/tylerjw/eigenwallet-swap-cli:4.5.3
      command: ["sleep", "infinity"]
      stdin: true
      tty: true
      volumeMounts:
        - { name: data, mountPath: /data }
  volumes:
    - name: data
      persistentVolumeClaim: { claimName: taker-data }
YAML

kubectl wait --for=condition=Ready pod/swap-cli-taker -n eigenwallet --timeout=60s
```

### 6. Get the taker's BTC deposit address

```sh
TAKER_ADDR=$(kubectl exec -n eigenwallet swap-cli-taker -- swap -d /data deposit-address --electrum-rpc tcp://electrs:50001 2>&1 | grep -oE 'bc1[a-z0-9]+' | head -1)
echo "Taker BTC: $TAKER_ADDR"
```

This is deterministic from `seed.pem`, so it'll be the same every recycle (`bc1qxprq6nf8kk00lhtqcn7kc7km2ylhtzsk38wyz3` as of 2026-05-10).

### 7. Withdraw BTC from ASB to taker

```sh
AMOUNT="0.05 BTC"   # adjust based on §4
kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 withdraw-btc $TAKER_ADDR "$AMOUNT"
# Capture the TXID from the output, then track confirmation:
TXID=...
curl -s "https://mempool.space/api/tx/$TXID/status"
```

The asb's BTC wallet picks low fees (~1.2 sat/vB) — fine in calm mempool, may be slow during congestion. Wait for at least 1 confirmation before proceeding.

### 8. Launch buy-xmr against the chosen counterparty

```sh
MONEROD_IP=$(kubectl get svc -n eigenwallet monerod -o jsonpath='{.spec.clusterIP}')
ASB_XMR_ADDR=$(kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 monero-address | grep -oE '4[0-9A-Za-z]{94,}' | head -1)
SELLER_PEER=...   # peer_id from §3 ranking

kubectl exec -n eigenwallet swap-cli-taker -- sh -c "
nohup swap -d /data buy-xmr \
  --seller $SELLER_PEER \
  --receive-address $ASB_XMR_ADDR \
  --monero-node-address http://$MONEROD_IP:18081 \
  --electrum-rpc tcp://electrs:50001 \
  --enable-tor \
  > /data/recycle-\$(date +%Y%m%d-%H%M%S).log 2>&1 &
echo \"PID \$!\"
"
```

Detached via `nohup` + `&` so it survives kubectl-exec disconnects. The swap will take ~30-60 min in the happy path.

### 9. Monitor

```sh
# unique states reached so far (good progress sign)
kubectl exec -n eigenwallet swap-cli-taker -- sh -c "grep -aE 'Advancing state' /data/recycle-*.log | sed 's/\x1b\[[0-9;]*m//g' | grep -oE 'state=[a-z _]+' | sort -u"

# any BTC txs broadcast (lock, cancel, redeem)
kubectl exec -n eigenwallet swap-cli-taker -- sh -c "grep -aE 'Published Bitcoin transaction' /data/recycle-*.log | sed 's/\x1b\[[0-9;]*m//g' | sed -E 's/.*txid=([a-f0-9]+).*kind=([a-z_]+).*/[\\\\2] \\\\1/' | sort -u"

# final state
kubectl exec -n eigenwallet swap-cli-taker -- sh -c "grep -aE 'Swap completed' /data/recycle-*.log | sed 's/\x1b\[[0-9;]*m//g' | sed -E 's/.*state=([a-z_ ]+).*/state=\\\\1/' | sort -u"
```

**Expected state progression for happy path:**

1. `quote has been requested`
2. `execution setup done`
3. `btc lock ready to publish`
4. `btc is locked` (BTC lock tx broadcast)
5. `xmr lock transaction candidate found` (we received Alice's transfer proof)
6. `xmr lock transaction seen` (we observed her lock tx on-chain)
7. `encrypted signature is sent` (we authorized Alice's BTC redeem)
8. `btc is redeemed` (Alice took the BTC)
9. `xmr redeem tx is constructed` + `xmr redeem tx is published`
10. **`xmr is redeemed`** ← terminal success

### 10. Verify completion

```sh
kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 monero-balance
kubectl exec -n eigenwallet swap-cli-taker -- swap -d /data balance --electrum-rpc tcp://electrs:50001 | grep -i balance
```

XMR balance should have increased by ~the maker's quoted XMR amount. Taker BTC balance should be near zero.

## Failure modes and recovery

### Stuck at `xmr lock transaction seen` (Monero scanner not advancing)

Symptom: state machine sits in `xmr lock transaction seen` for >10 min after the BTC tx confirms. Monero scanner's last `Fetched blocks` is the block where Alice's lock was found, no further activity.

Cause: swap-cli's internal `monero-rpc-pool` failed to find a working public Monero node and the scanner can't fetch new blocks.

Fix: kill + resume with explicit `--monero-node-address`.

```sh
kubectl exec -n eigenwallet swap-cli-taker -- sh -c 'pkill -9 swap; sleep 2'

MONEROD_IP=$(kubectl get svc -n eigenwallet monerod -o jsonpath='{.spec.clusterIP}')
SWAP_ID=...   # from the existing recycle log: grep swap_id

kubectl exec -n eigenwallet swap-cli-taker -- sh -c "
nohup swap -d /data resume \
  --swap-id $SWAP_ID \
  --monero-node-address http://$MONEROD_IP:18081 \
  --electrum-rpc tcp://electrs:50001 \
  --enable-tor \
  > /data/recycle-resume-\$(date +%Y%m%d-%H%M%S).log 2>&1 &
echo \"resume PID \$!\"
"
```

If you launched the original buy-xmr with the `--monero-node-address` flag (per §8), this stuck state should not occur.

### Alice never locks XMR (maker ghosts us)

Symptom: state stays at `btc is locked` for hours. No `xmr lock transaction candidate found` ever appears.

The swap-cli will eventually auto-trigger the "amnesty" cancel path after ~3 hours:
- A `kind=cancel` BTC tx gets broadcast
- The cancel tx reclaims our BTC minus ~4000 sat in cancel fees
- Swap terminates with state `btc amnesty is confirmed`

No action needed — just wait it out. After completion, the taker wallet will hold the refunded BTC (less fees). Use `swap withdraw-btc` to return it to the asb, or leave it staged for the next recycle.

**Add the ghost peer to your mental blacklist for next time.**

### BTC withdraw tx stuck unconfirmed for hours

The asb sets very low BTC fees (~1.2 sat/vB). In congested mempools this can mean hours of waiting. Options:
- Wait it out (mempool eventually clears)
- RBF replace the tx with a higher fee (requires the asb wallet to have signaled RBF, which I haven't verified)
- Cancel and retry when mempool is calmer

### swap-cli pod won't start / crashes

Check the taker NFS path is reachable: `kubectl exec -n eigenwallet deploy/electrs -- ls /electrs-data/` should work (sanity), and the taker PV/PVC should be Bound. If the NFS mount fails, the pod will be stuck in `ContainerCreating`.

## Cleanup (optional)

Once the recycle completes successfully, the taker pod can stay (it's small) or be deleted:

```sh
# Verify drained, then optionally delete pod (PVC + PV stay; data survives on NFS)
kubectl exec -n eigenwallet swap-cli-taker -- swap -d /data balance --electrum-rpc tcp://electrs:50001 | grep -i balance
kubectl delete pod -n eigenwallet swap-cli-taker
```

The pod is cheap to recreate from the YAML in §5, so deleting it is fine for tidiness. Don't delete the PVC or PV — those preserve the taker wallet identity (seed.pem) across pod recreations.

## Quick command reference

| Operation | Command |
|---|---|
| Current ASB BTC | `kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 bitcoin-balance` |
| Current ASB XMR | `kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 monero-balance` |
| ASB Monero addr (for receive) | `kubectl exec -n eigenwallet deploy/asb-controller -- asb-controller --url=http://asb:9944 monero-address` |
| monerod ClusterIP | `kubectl get svc -n eigenwallet monerod -o jsonpath='{.spec.clusterIP}'` |
| Taker BTC balance | `kubectl exec -n eigenwallet swap-cli-taker -- swap -d /data balance --electrum-rpc tcp://electrs:50001` |
| Taker BTC deposit addr | `kubectl exec -n eigenwallet swap-cli-taker -- swap -d /data deposit-address --electrum-rpc tcp://electrs:50001` |
| Withdraw from taker → ASB | `kubectl exec -n eigenwallet swap-cli-taker -- swap -d /data withdraw-btc --electrum-rpc tcp://electrs:50001 <asb-btc-deposit-addr>` |
| Monitor swap log states | `kubectl exec -n eigenwallet swap-cli-taker -- sh -c "grep 'Advancing state' /data/recycle-*.log \| sed 's/\\x1b\\[[0-9;]*m//g' \| grep -oE 'state=[a-z _]+' \| sort -u"` |
| Kill in-flight swap | `kubectl exec -n eigenwallet swap-cli-taker -- pkill -9 swap` |
