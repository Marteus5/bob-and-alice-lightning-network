# Breach Protocol — Tier 4 lab: Bitcoin/Lightning regtest

This spins up one `bitcoind` node in regtest mode plus two LND nodes (Alice and Bob),
all talking to each other over a private Docker network. Nothing here touches real
money — regtest is a throwaway, instantly-mineable chain that exists only inside
these containers.

## 1. Bring it up

```
cd breach-protocol-tier4
docker compose up -d
docker compose logs -f
```

Wait until you see `lnd-alice` and `lnd-bob` log lines mentioning `Waiting for
wallet encryption password` — that means both nodes are up and waiting for their
wallets to be created. Ctrl+C to stop following logs (the containers keep running).

## 2. Create wallets on both LND nodes

Each node needs a wallet created once. This generates a seed phrase — for a
regtest lab you don't need to save it carefully, but the flow is the same as
real Lightning node setup, which is good practice.

```
docker exec -it bp-lnd-alice lncli --network=regtest create
docker exec -it bp-lnd-bob   lncli --network=regtest create
```

Follow the prompts (set any password, skip the seed backup confirmation
questions by answering however prompted — this is throwaway regtest money).

## 3. Get a bitcoind wallet going and mine some blocks

Bitcoind also needs a wallet before it can generate addresses / mine to itself:

```
docker exec -it bp-bitcoind bitcoin-cli -regtest -rpcuser=breachprotocol -rpcpassword=breachprotocol createwallet "miner"

docker exec -it bp-bitcoind bitcoin-cli -regtest -rpcuser=breachprotocol -rpcpassword=breachprotocol -generate 101
```

`-generate 101` mines 101 blocks — you need at least 100 for regtest coinbase
rewards to mature into spendable balance.

## 4. Fund Alice's Lightning wallet

Get a deposit address from Alice's LND node, then send it regtest coins from
bitcoind and mine a confirming block:

```
ALICE_ADDR=$(docker exec bp-lnd-alice lncli --network=regtest newaddress p2wkh | grep -o '"[a-zA-Z0-9]*"' | tr -d '"')

docker exec bp-bitcoind bitcoin-cli -regtest -rpcuser=breachprotocol -rpcpassword=breachprotocol -rpcwallet=miner sendtoaddress $ALICE_ADDR 1.0

docker exec bp-bitcoind bitcoin-cli -regtest -rpcuser=breachprotocol -rpcpassword=breachprotocol -generate 6
```

Check Alice's on-chain balance landed:
```
docker exec bp-lnd-alice lncli --network=regtest walletbalance
```

## 5. Connect Alice and Bob

Get Bob's node URI (pubkey@host:port):
```
docker exec bp-lnd-bob lncli --network=regtest getinfo
```
Copy the `identity_pubkey` value from the output, then connect from Alice
(Bob is reachable at `bp-lnd-bob:9735` on the Docker network):
```
docker exec bp-lnd-alice lncli --network=regtest connect <BOB_PUBKEY>@bp-lnd-bob:9735
```

## 6. Open a channel from Alice to Bob

```
docker exec bp-lnd-alice lncli --network=regtest openchannel --node_key=<BOB_PUBKEY> --local_amt=500000

docker exec bp-bitcoind bitcoin-cli -regtest -rpcuser=breachprotocol -rpcpassword=breachprotocol -generate 6
```

Confirm it's active on both sides:
```
docker exec bp-lnd-alice lncli --network=regtest listchannels
docker exec bp-lnd-bob   lncli --network=regtest listchannels
```

## 7. Send a payment

Create an invoice on Bob's node:
```
docker exec bp-lnd-bob lncli --network=regtest addinvoice --amt=1000
```
Copy the `payment_request` value, then pay it from Alice:
```
docker exec bp-lnd-alice lncli --network=regtest payinvoice <PAYMENT_REQUEST>
```

If that succeeds, you've got a fully working two-node Lightning payment running
entirely on your own machine — this is the "happy path" the game's Tier 4
challenges should build on top of (funding, connecting, channel opening,
routing, and later force-closes / failure diagnostics).

## Cleaning up

```
docker compose down          # stop, keep chain/wallet state
docker compose down -v       # stop and wipe everything — start totally fresh
```

## Next steps for wiring this into the game

- Each of the numbered steps above is a natural "challenge": the grading check
  for step 4, for example, is just polling `lncli walletbalance` until the
  `confirmed_balance` matches what was expected, rather than trusting the
  player's terminal output.
- For a scripted per-run sandbox (matching how Tiers 1–3 already isolate each
  attempt), this whole compose stack can be brought up/torn down per challenge
  attempt via the same Docker-per-run approach already used elsewhere in
  Breach Protocol — `docker compose -p run-<id> up -d` with a unique project
  name keeps each player attempt isolated.
- The `no-macaroons=false` line means LND's authentication is on by default,
  which is realistic — a later "advanced" challenge could involve locating and
  using the macaroon file to authenticate a raw gRPC/REST call instead of
  going through `lncli`, which is a real skill Lightning node operators need.
