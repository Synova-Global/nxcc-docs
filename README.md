# nxcc — NativX Constituent Client

`ghcr.io/synova-global/nxcc` is the agent you run inside your own data center to
launch and operate a **constituent** on the NativX exchange. A constituent is
your own branded capacity token: people can buy it, hold it, and redeem it for
compute that you deliver.

nxcc proves the capacity you actually deliver, so your token stays backed by
real work. You can never issue supply you have not earned, and neither can
anyone else. That guarantee is the reason a buyer is willing to hold your
token, and the whole tool is built to protect it.

*Authorized NativX constituents only. The image is public; authorization comes
from enrollment, not from possession.*

---

## Install and verify

The supported way to run nxcc is the cosign-signed Docker image. There are no
registry credentials — the image is public. Trust comes from the signature:
install [cosign](https://github.com/sigstore/cosign/releases) and verify
**before** you run anything.

```bash
cosign verify \
  --key https://synova.global/.well-known/cosign.pub \
  ghcr.io/synova-global/nxcc:0.8.0

docker pull ghcr.io/synova-global/nxcc:0.8.0
```

- Release tags are bare semver (`:0.8.0` for release `v0.8.0`); `:0.8` and
  `:latest` track the newest release, and every release also publishes an
  immutable `:sha-<commit>` tag for pinning.
- **Current release: `0.8.0`.** It carries the `constituent-agreement/v3`
  onboarding terms and the self-custody onboarding ceremony described below.
  Onboard with this image or newer.
- You need a Linux x86_64 host with Docker (or systemd for the extracted
  binary) and outbound HTTPS to the pinned hosts listed at the bottom. Nothing
  else.

nxcc is locked down by construction: its endpoints are compiled in, it refuses
to start if an endpoint-override environment variable is set, it speaks HTTPS
only against a strict host allow-list, and it holds **no privileged key** — no
mint key, no order entry, no registry write. Its entire write surface is seven
POST endpoints (enroll, onboard, markets, ingest, capacity, reserve, redeem
fulfill). A modified agent is an unauthorized fork and its token is revoked.

## Key custody — two keys, two hosts

Every delivery attestation is **dual-signed**: a *provider* key and an
independent hardware *witness* key. The independence is the point — you cannot
self-attest. Generate the pair on two different hosts:

| Host | Command | Emits |
|------|---------|-------|
| Provider host | `nxcc install --provider <id>` | `<id>.provider.priv/.pub` only |
| Facility/BMS host | `nxcc witness-keygen --provider <id>` | `<id>.witness.priv/.pub` only |

`witness-keygen` refuses to run on a host that already holds a provider key.
(`--dev-colocate-witness` exists for single-host dev/test only.)

Your **settlement wallet's** private key never touches nxcc at all — see the
next section.

---

## Onboarding, start to finish

You onboard your company once, then list one market per class you serve:
`I` (inference), `T` (training), `CO` (connectivity), optionally tagged
`--green`. Each market is a `{provider}-{class}` instrument backing the
matching `COIL-{class}` benchmark.

```bash
# 1. Ask NativX for approval to onboard. This files a request; NativX approves it.
nxcc enroll request --provider <id> --provider-name "<Your Co>" \
              --class I --email ops@yourco.com
nxcc enroll status  --provider <id>   # check back; on approval it stores your token

# 2. Prove the wallet that will receive settlement. NativX does not provide
#    custody: mints and sale proceeds go to a wallet YOU control, and you prove
#    control by signing a one-off challenge with it. Any EIP-191 personal_sign
#    works — MetaMask, a hardware signer, `cast wallet sign`. The signature
#    authorises no transfer and moves no funds.
nxcc enroll wallet-challenge --provider <id>          # prints the text to sign
#    ...sign the printed challenge with your wallet, out of band...
nxcc enroll wallet-attest    --provider <id> --address 0xYourWallet --signature 0x...

# 3. Onboard your company and create your first market. This generates your
#    signing keys, records your acceptance of the constituent agreement (v3),
#    and issues your bearer token. --wallet is a cross-check that the address
#    you proved is the one you meant.
nxcc onboard  --provider <id> --provider-name "<Your Co>" \
              --region <region> --silicon <sku,sku> --accept-terms \
              --wallet 0xYourWallet --class I

# 4. From your wallet, grant the settlement contract a BOUNDED allowance
#    (your token for sells; USDC only if you buy). Your wallet, your
#    signature, your bound — never unlimited. The venue cannot do this for you.

# 5. List more markets whenever you want (no new approval needed).
nxcc markets  --provider <id> --class T
nxcc markets  --provider <id> --class CO --green
```

**Before your first `nxcc run`, confirm you are delivery-ready.** Onboarding
answers `provider_status: active` immediately, but the ingest gateway trusts
your newly registered provider **and witness** keys only after a periodic sync
(~60 s). Poll, then run:

```bash
nxcc status --provider <id> --wait 90   # blocks until delivery_ready
nxcc run    --provider <id>
```

The on-chain registration binds your token to your wallet **permanently**
(write-once). That is a guarantee to you, not a restriction: nobody — including
NativX — can later re-point your mints or your sale proceeds anywhere else.

---

## Attestation — how your delivery is proved

`nxcc run` is a forward-only loop. Every window it measures the capacity you
delivered, signs the measurement with **both** keys, appends it to a local
hash-chained log, and ships it to the index ingest gateway:

- Each window produces a **MeasurementBatch**: provider id, sequence number,
  window bounds, per-workload samples, and a `composite_root` that commits to
  the previous batch — the chain is forward-only and tamper-evident.
- The gateway verifies both signatures and the chain. Verified delivery
  accrues to your **mint ceiling**: the most that may ever be issued against
  what you have delivered. `nxcc mint` shows it (ceiling, not circulating).
- Issuance is server-governed: the on-chain `BocMintAuthority` mints your
  token *only* against BOC-attested verified delivery — **supply ≤ verified
  delivery, always**. nxcc never holds the mint key; nothing you can run
  raises the ceiling except delivering and proving more compute.

Three related attestation surfaces, all optional to start:

- **Declared capacity** (`nxcc capacity declare --units 500 --region us-east`)
  — an advertised availability signal. It never changes what you can issue:
  declare 500, deliver 50, and the market sees an offer of 500 backed by a
  verified 50. The gap is visible, which is the point.
- **Reserved capacity** (`nxcc reserve attest ...`) — attests capacity under
  contract at a committed price for a term; a pricing input for the COIL-T
  index. Price and term are immutable; capacity only revises down; re-audit
  quarterly or the record drops out of the index's inputs.
- **Cost basis** (`nxci-fb-v2` schema, off by default) — an operator-declared
  delivered cost dimension on your samples. Published as a dimension; never
  affects pricing of your delivery or your ceiling.

Run it day to day:

```bash
nxcc run     --provider <id> --all-classes   # one agent feeds every listed market
nxcc status  --provider <id>                 # local agent state
nxcc verify  --provider <id>                 # re-check the local log + BOC pubkey
nxcc mint    --provider <id>                 # verified-delivery ceiling
```

---

## How you get paid

Money reaches you in exactly one way: **buyers pay USDC for your token, and
that USDC lands in your wallet at on-chain settlement.** There is no invoice,
no payout schedule, and no balance held at the venue.

1. **Minting.** The NativX mint authority mints your token to **your**
   settlement wallet, against your verified delivery. Minted tokens sit in
   your wallet — they are yours before any sale.
2. **Sale.** Your token trades on your `{provider}-{class}` market on the
   venue's open order book. You are the natural seller of your own minted
   supply; the bounded allowance you granted at onboarding (step 4) is what
   lets a matched sell actually settle from your wallet.
3. **Settlement.** Every fill settles as **one atomic on-chain swap** on Arc:
   your token moves from your wallet to the buyer, and the buyer's USDC moves
   to your wallet — both legs in a single transaction, or neither. There is no
   settlement risk window and no venue float. A per-fill replay guard means a
   fill can never settle twice. Before any value moves, the settlement
   contract independently re-checks that your circulating supply is fully
   covered by your verified delivery — an unbacked token cannot settle,
   period.
4. **Costs.** Settlement transactions are submitted and paid for by the
   venue's settlement operator — you pay no gas on a sale. Your own
   transactions (granting or topping up the allowance) need a small amount of
   gas in your wallet. Venue trading fees follow the published fee schedule
   (currently 0.02% maker / 0.05% taker) and are reflected at the venue; the
   on-chain swap itself takes no cut.
5. **Watching it.** Every settlement is an on-chain transaction from/to your
   wallet — verifiable on the explorer without trusting anyone. The venue also
   shows your per-wallet trade history, and your **yield accrual**: each daily
   COIL fix values your delivered capacity and accrues per-token yield,
   epoch by epoch. NativX can issue you a private **constituent view** link —
   attested capacity, sales, accrual, open claims — with no venue login
   needed; ask your NativX contact.

**Ops notes that matter:** keep the settlement allowance topped up (sales stop
settling when it runs out — the venue will warn, but the grant is yours to
make); keep a little gas USDC in the wallet; and treat the wallet key with the
care of the revenue it receives. NativX never holds it and can never recover it.

---

## Redemption — what happens when a holder redeems your token

Redemption is how your token keeps its meaning: a holder can always exchange
it for the compute it represents. It is a **delivery event, not a payment
event** — the holder already paid when they bought the token; discharge moves
no money.

The lifecycle, end to end:

1. **A holder opens a claim.** They escrow `amount` of your token into the
   on-chain redemption registry, which records a claim against your provider.
   The tokens leave the holder's wallet but are **not burned yet** — they sit
   in escrow while the claim is open.
2. **You see it.** Open claims against you — holder, amount, time remaining —
   are visible in your constituent view and to the venue. Each claim has a
   TTL fixed per token at registration (at most 365 days).
3. **You deliver the compute.** Two ways:
   - **Manual:** run the workload however you fulfil it, then attest it:
     `nxcc redeem fulfill --provider <id> --claim-id <id> --token <sym> --amount-q8 <n>`.
   - **Automatic (Kubernetes):** run the companion `coil-fulfill-controller`
     in your cluster. It sweeps open claims, provisions a real namespace +
     GPU/CPU Job per claim, measures what was delivered during the run
     window (DCGM for GPU, CPU-core-seconds for CPU), and drives the sealed
     nxcc binary to sign the attestation. The controller never signs; nxcc
     still holds the witness leg.
4. **The attestation is verified twice.** The gateway verifies your
   dual-signed fulfillment (provider + hardware witness). The NativX
   attestation authority then signs the on-chain discharge message binding
   your delivered composite root to that exact claim.
5. **Discharge burns the escrow.** The settlement service submits the
   discharge on-chain; the registry re-verifies the attestation signature and
   **burns** the escrowed tokens. Supply only ever decreases on a *proven*
   delivery — the burn is the on-chain record that the compute was really
   served.
6. **If you don't deliver in time.** After the TTL plus a 1-day grace period,
   the claim can be recycled: the escrow is **refunded to the holder** in
   full. The discharge and recycle windows are disjoint by construction, so a
   valid-but-late delivery inside the grace window can never be front-run by
   a recycle, and a recycle can never strip a claim that is still
   dischargeable. Nobody's value is ever destroyed by a missed claim — but a
   pattern of recycled claims is visible on-chain and to every buyer, and it
   is your token's reputation.

---

## What nxcc cannot do

nxcc is a measurement-and-attestation client, not a control plane. It has no
path to issue, move, or govern supply — by construction, not by policy:

- **No self-mint.** It never holds a mint key. `nxcc mint` only *reads* the
  ceiling.
- **No registry mutation.** It cannot register, re-point, or alter
  instruments, tokens, or key fingerprints.
- **No order entry.** No order book, no trade, no balance to move. It reports
  delivery; it never transacts.
- **No custody.** Your settlement wallet's key never touches it.

## Pinned endpoints

| Purpose | Host |
|---|---|
| Ingest (delivery and redemption) | `https://nxci-ingest.nativx.exchange` |
| Titan reads (staging-compat release) | `https://staging.synova.global` — exactly `/v1/instruments`, `/v1/delivery/inventory`, `/v1/boc/pubkey` |
| Titan reads (production release) | `https://nativx.exchange` — same reads, BOC key at `/v1/titan/boc/pubkey`, separately signed image |
| Onboarding | `https://onboard.synova.global` |
| Auth | bearer token issued by `nxcc onboard`, stored `0600` at `~/.nxcc/keys/<id>.token`; `NXCC_BEARER_TOKEN` overrides |

## House rules

- Do not modify the binary or its pinned endpoints — a changed agent is an
  unauthorized fork and its token gets revoked.
- Your bearer token is per company and short-lived; rotate through onboarding.
- Customer identifiers are SHA-256 blinded before they leave your host. nxcc
  ships opaque hashes, never customer names.

## Support

Enrollment approval, delivery-readiness questions, and constituent-view links
go through NativX constituent onboarding (the contact on your enrollment
approval). Suspected vulnerabilities: **security@nativx.exchange**. Run
`--help` on any nxcc command for the full option set.

---

*This repository holds the public documentation for the `nxcc` container
image. The nxcc source is internal to NativX; possession of the image does not
constitute authorization to operate a constituent — enrollment does.*
