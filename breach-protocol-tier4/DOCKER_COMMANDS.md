# Docker commands reference

General Docker commands first, then the ones specific to this regtest lab.

## General

```
docker run hello-world          # sanity check
docker build -t myapp .         # build an image from a Dockerfile
docker run -p 8000:8000 myapp   # run it, mapping a port (host:container)
docker compose up               # spin up multi-container apps (e.g. app + database)
```

Handy day-to-day commands:

```
docker ps                       # running containers (add -a for stopped ones too)
docker images                   # local images
docker logs -f <container>      # follow a container's logs
docker exec -it <container> sh  # open a shell inside a running container
docker stop <container>         # stop a container
docker rm <container>           # remove a stopped container
docker rmi <image>              # remove an image
```

## This lab (`breach-protocol-tier4`)

Run from the `breach-protocol-tier4/` directory.

```
docker compose up -d            # start bitcoind, lnd-alice, lnd-bob in the background
docker compose ps               # check service status
docker compose logs -f          # follow logs from all services
docker compose logs -f lnd-alice  # follow one service
docker compose restart lnd-bob  # restart one service
docker compose down             # stop everything, keep chain/wallet data
docker compose down -v          # stop AND wipe all volumes (totally fresh start)
```

Container names: `bp-bitcoind`, `bp-lnd-alice`, `bp-lnd-bob`.

Talking to the nodes:

```
docker exec -it bp-lnd-alice lncli --network=regtest getinfo
docker exec -it bp-lnd-bob   lncli --network=regtest getinfo
docker exec -it bp-bitcoind  bitcoin-cli -regtest -rpcuser=breachprotocol -rpcpassword=breachprotocol getblockchaininfo
```

Isolated per-attempt copy of the whole stack (unique project name):

```
docker compose -p run-<id> up -d
docker compose -p run-<id> down -v
```

See `README.md` for the full walkthrough (wallets, funding, channels, payments).

## Bob pays Alice back (Lightning)

Prerequisite: the Alice→Bob channel from README steps 5–6 is open. Alice opened it
with 500,000 sat, so at first *all* the liquidity is on Alice's side. Bob can only
send what he has received, and LND makes him keep a channel reserve (~1% of channel
capacity, 5,000 sat here), so first move some balance to Bob.

1. Give Bob enough outbound liquidity (Alice pays Bob 100,000 sat):
   ```
   docker exec bp-lnd-bob lncli --network=regtest addinvoice --amt=100000
   docker exec bp-lnd-alice lncli --network=regtest payinvoice <PAYMENT_REQUEST>
   ```
2. Alice creates an invoice for what Bob will send back (e.g. 20,000 sat):
   ```
   docker exec bp-lnd-alice lncli --network=regtest addinvoice --amt=20000
   ```
3. Bob pays it (copy `payment_request` from step 2):
   ```
   docker exec bp-lnd-bob lncli --network=regtest payinvoice <PAYMENT_REQUEST>
   ```
4. Verify balances on both sides:
   ```
   docker exec bp-lnd-alice lncli --network=regtest channelbalance
   docker exec bp-lnd-bob   lncli --network=regtest channelbalance
   ```

Invoice-free alternative (keysend), from Bob:
```
docker exec bp-lnd-bob lncli --network=regtest sendpayment --dest=<ALICE_PUBKEY> --amt=20000 --keysend
```
(Alice's pubkey: `docker exec bp-lnd-alice lncli --network=regtest getinfo`. If keysend is rejected, add `accept-keysend=true` to `lnd-alice/lnd.conf` and restart her.)

To send **on-chain** coins instead (Bob's wallet must be funded first):
```
docker exec bp-lnd-alice lncli --network=regtest newaddress p2wkh
docker exec bp-lnd-bob   lncli --network=regtest sendcoins --addr=<ALICE_ADDR> --amt=50000
docker exec bp-bitcoind  bitcoin-cli -regtest -rpcuser=breachprotocol -rpcpassword=breachprotocol -generate 1
```
