# BRC-Y: Giving an AI Agent Control of a Delivery Vehicle Without Giving It a Wallet

sun-dive (https://github.com/sun-dive)

## Abstract

An agent that spends on Bitcoin is normally given a wallet, which is to say a key — and a key is unbounded
capability: it signs anything, for whoever holds it. Funding it lightly bounds the amount at risk without
bounding the behaviour.

This standard defines a **battery**: a covenant that carries its own fuel, pays each step's fee out of the
value it already holds, and enforces its own successor state. The authority to act correctly sits in the
script rather than in anyone's possession, so an agent can be given **control without custody** — a
credential that advances the covenant and cannot direct value anywhere.

It defines the value rule, the fee ceiling, the authorisation, ownership and control, and the top-up. It does
not define the work.

## Motivation

An autonomous program has to answer a question a signed transaction never does: who pays for the next step,
when nobody is watching? Answering it with a key and a hot wallet makes the agent a custodian, and a
custodian that reads untrusted text for a living. Rotating the key after an incident concedes the point.

`OP_PUSH_TX` removes the need. A covenant that reconstructs the sighash preimage on its own stack constrains
its successor without anyone signing for identity; BRC-226 establishes that technique. What it leaves open is
the fee, because a transaction with no fee does not relay and a keyless covenant has no funding UTXO. This
standard closes it with one move:

> **Let the covenant's own carried value be the fuel, and let the fee come out of it.**

A step is then a complete transaction — one input, one output, no funding, no change — that any stranger can
broadcast. Two properties follow, and they are why this is worth standardising rather than writing once:
the value becomes a **fuel gauge** any observer can read from the UTXO, and the sequence of steps becomes a
**computation trace** that is checked rather than trusted.

### Scope

This standard defines how an on-chain action is paid for and authorised. It does not define what the
covenant computes, what its state means, or how that state is encoded — for the last, see BRC-X.

⚠ It bounds an agent's on-chain actions — writing, paying, proving — **not its thinking**. Overclaim and a
reader will correctly object that the agent is not running on Bitcoin. Stated precisely the claim is
stronger: *the agent thinks wherever it likes; it can only act within the covenant.* A battery does not make
an agent correct, honest or safe to deploy; it removes one capability and leaves every other question
untouched.

**A battery is the first rung of a ladder** and deliberately the most restrictive. Where an agent must
genuinely pay somebody, the same technique extends:

```
1. battery       may only advance its own program, paying its own fee   ← THIS STANDARD
2. payee-bound   may pay only recipients enumerated when it was built
3. rate-bound    caps value per spend or per block, enforced in script
```

Rungs 2 and 3 are out of scope here. A reader who needs them must not reach them by relaxing §4, which is
what makes rung 1 safe.

## Specification

The key words "MUST", "MUST NOT", "SHOULD" and "MAY" are to be interpreted as described in RFC 2119.

### 1. Terminology

- **Fuel** — the satoshi value carried by the covenant's output, and the only thing that pays for its steps.
- **Tick** — one spend that performs one step and re-creates the covenant with the advanced state.
- **Computation trace** — the ordered ticks from genesis to the unspent tip. Each hop is a step whose
  correctness the covenant enforced as a condition of the hop existing, so the trace is a proof to be
  walked rather than a log to be trusted.
- **`V`** — the fuel carried by the output being spent, read by the covenant from the preimage.
- **`MAX_FEE`** — the permanent ceiling on how much fuel one tick may consume.
- **Ticker** — whoever broadcasts a tick. Risks nothing and gains nothing.
- **Sponsor** — whoever adds fuel.
- **Flat** — a covenant whose remaining fuel can no longer pay for a tick.

Two roles are OPTIONAL; a covenant MAY bind either, both or neither to a public key:

- **Driver** — a party whose signature is required to *advance* the covenant. Authority to act, nothing else.
- **Owner** — a party whose signature is honoured for rights over the instance itself, such as retiring it
  and recovering what it holds. Authority that reaches value.

### 2. The tick

```
in0    the covenant           unlocking script = the preimage
out0   the covenant, quined, state advanced, value ≥ V − MAX_FEE
```

`out0` MUST be first. Later outputs MAY be present (§6) and the covenant MUST NOT constrain them beyond
committing to them in the sighash.

⚠ A tick MUST NOT require a funding input. The moment it does, the ticker needs a wallet.

### 3. Authorisation, ownership and control

The covenant MUST authorise the advance by validating the sighash preimage supplied in the unlocking script,
so the successor state is enforced by the script rather than asserted by a signer.

Beyond that, **ownership and control by public key are both permitted**:

| configuration | who may advance | who has rights over value |
|---|---|---|
| ownerless | anybody | nobody — fuel can only become mining fees |
| controlled | one keyed driver | nobody |
| owned | anybody, or one keyed driver | a keyed owner, per the rights the script grants |

Ownerless is the strongest form and the easiest to reason about: irreversibly nobody's, with no upgrade,
pause or recovery. Owned trades some of that for operational reality, which is often correct — a fleet
operator who cannot retire a vehicle owns a liability.

**In every configuration, authority MUST be enumerated in the script:**

1. A driver's authority MUST be limited to advancing the covenant. A driver signature MUST NOT be sufficient
   to direct value to an output of the signer's choosing.
2. Any authority that *can* direct value MUST be gated by a key distinct from the driver's whenever the
   driving credential is held by someone other than the owner.
3. No configuration grants an authority the script does not spell out. There is no implicit administrator.

⚠ **Requirement 2 is the one that fails by accident.** If one key both advances the covenant and authorises
its retirement, issuing that key to an agent issues the retirement power too — and a retirement signed
`SIGHASH_ALL` pays whatever outputs the signer chose. The covenant is sound in that scenario; the key
issuance is the mistake. Bind the roles to different keys before the driving credential leaves your custody.

⚠ **A conforming script contains a signature opcode, and counting them is not a conformance test.** The
preimage check is normally built *from* `OP_CHECKSIG`: the script derives a signature over the pushed
preimage using a constant scalar whose value is published, and `OP_CHECKSIG` recomputes the sighash from the
real transaction, so the spend validates only if the preimage is genuine. State the property as *"no secret
exists"*, never as *"no signature opcodes"* — the second is checkable and false.

⚠ Whatever the genesis does not provide for, nobody can add to *that instance* later. The remedy is to
supersede it — mint a corrected covenant and stop fuelling the old one (§5c) — which costs the original's
identity and leaves it dormant rather than gone. Everything an instance will ever do MUST therefore be
correct when it is broadcast; §10 records a constant that could not be fixed and forced exactly that.

### 4. The value rule

The covenant MUST enforce `out0.value ≥ V − MAX_FEE`. It MUST be a **floor** and MUST NOT be an equality.

This is the most consequential line in the standard. Under a floor, a transaction that *increases* the fuel is
already a valid tick, so a top-up needs no separate code path, no privileged key and no second script —
adding fuel and advancing the state are one atomic operation.

⚠ Under an equality the fuel can only fall. The covenant can never be refuelled by anyone, and it dies
permanently on reaching `MAX_FEE`. This is unamendable and has no symptom until the day it stops.

`MAX_FEE` bounds the worst case; it is not the price. An honest ticker pays only what the network requires.

### 5. The fee ceiling

However `MAX_FEE` is carried, it MUST be derived by **serializing a real worst-case spend and measuring it**,
then applying the relay rate. It MUST NOT be hand-counted, inferred from the locking script alone, or copied
from another covenant. That requirement holds in every configuration, because it is the step that has
actually gone wrong in practice.

```
MAX_FEE ≥ ceil(worstCaseTickBytes × relayRate) × headroom       headroom ~1%
```

⚠ **Set below what the network accepts for the worst case, the covenant is dead on arrival**: no tick can
ever pay enough to relay, and the remaining fuel is stranded. Serializing a real spend is the only permitted method
because it is the only one observed to be right.

**How the ceiling is carried, and how a wrong one is escaped.** Two mechanisms for carrying it, and two
routes out — the second of which requires nothing at all:

| | | what it costs |
|---|---|---|
| **a** | a literal baked into the script | cannot be amended in place; escape via (c) or (d) |
| **b** | a field an owner key may update | amendable in place, at the cost of the ownerless configuration |
| **c** | **supersede: mint a corrected instance, leave the old unfunded** | **nothing, and it needs no key** |
| **d** | retire the old instance by burning it | only tidies the UTXO set; requires an owner path |

**(a)** is the strongest and what the deployed instances use. Nothing can amend it, which is the point: no
key's compromise can change what a tick may cost.

**(b)** is permitted and is an **owner** authority under §3, so it inherits the key-separation rule and MUST
NOT be reachable by a driver credential. ⚠ It is not a theft vector — the covenant still cannot direct value
to a party — but a ticker who also mines captures the fee, so an unnecessarily high ceiling extracts rather
than merely wastes. An implementation offering (b) SHOULD bound the update, by a maximum or a rate limit or
both, and SHOULD state that the ownerless configuration is no longer available to it.

**(c) is the real escape hatch, it is available in every configuration, and it requires no authority
whatsoever.** Mint a corrected instance and simply stop funding the flawed one. It goes flat and stays flat.
Nothing has to be signed, no retirement path has to exist, and none of this depends on the old covenant
cooperating — which matters, because a genuinely ownerless covenant *has* no retirement path to invoke.

⚠ **But abandonment is dormancy, not deletion, and §8 is what makes that true.** A flat covenant resumes on
any top-up from anyone. So an abandoned instance's behaviour remains permanently *available*: a stranger may
refuel a superseded covenant years later and it will run exactly as originally written. You cannot withdraw a
covenant, only decline to feed it. **That is the permanent cost, and it is a reputational one rather than a
financial one** — the flawed thing keeps your name on it and can be woken by anybody.

**(d)** burning is therefore an optimisation, not the mechanism: it reclaims a UTXO and forecloses revival.
It requires an owner path, and adding one for this purpose alone forfeits the ownerless configuration in
exchange for tidiness. ⚠ Where permanence is the point — a monument, a public artefact — omitting a burn is
the correct choice and the stranded satoshi is the price of it.

⇒ So (a) is not "unamendable" in the sense of unrecoverable. It is unamendable **in place**, and the recovery
is to supersede rather than to edit. Implementations SHOULD say which of these they chose, because
a reader cannot tell (a) from (b) without reading the script.

Three sources of size variation, of which **only the first threatens the bound**:

1. **Script variants — dangerous.** Every rule that emits opcodes makes the cheapest spend dearer to mine. A
   variant whose script grew while its ceiling did not is a covenant no node will relay, and it looks healthy
   until the first refusal. The ceiling MUST be re-derived per variant, from a single named measurement
   shared by the builder and by whatever checks the builder.
2. **Extra inputs and outputs — safe.** A sponsored top-up is larger, but the sponsor funds the difference.
   The ceiling bounds what leaves the *covenant*, not what the transaction pays.
3. **Variable-length payload — safe.** Free text in an `OP_RETURN` beside a contribution rides on the
   sponsor's transaction, so arbitrary user input cannot threaten a permanent constant.

⚠ Drift in the safe direction is also a defect: a ceiling a few bytes high overpays every miner forever,
silently. A fee check SHOULD fail on drift in *either* direction.

### 6. Sighash scope and sponsorship

The covenant SHOULD use `SIGHASH_ANYONECANPAY | ALL | FORKID` (`0xc1`). `ANYONECANPAY` excludes other inputs
from the preimage, so a sponsor MAY add a funding input without invalidating the covenant's introspection;
`ALL` still commits every output, so `out0` stays pinned.

A top-up is therefore one transaction that adds fuel, advances the state, and MAY carry trailing outputs —
change, or a record of the contribution.

⚠ Any record carried that way is permanent and unmoderatable. Implementations displaying it SHOULD render it
as inert text and MUST NOT turn it into a live link.

### 7. Bounded steps

Each tick MUST perform a step whose worst case is known when the script is written, because §5 depends on it.
The covenant MUST recompute the successor state from the state in its own `scriptCode` and MUST refuse any
other, so a ticker chooses *whether* to advance it, never *how*.

The computation as a whole need not be bounded. **The loop is not in the script, it is in the trace**: each
iteration is a separate transaction, separately paid for and separately verified. Nothing here gives Script a
backward jump; the trace shows where the loop went.

⚠ An external reference implementation that disagrees with the script by one unit cannot advance the
covenant at all, because the covenant recomputes and refuses. That makes the reference's arithmetic
normative in practice, so it SHOULD be verified against exact integer arithmetic rather than floating point,
and its operation order stated wherever truncation makes it observable.

### 8. Halting and resumption

When `V < MAX_FEE` the covenant is **flat**. A flat covenant MUST NOT be treated as terminated: its state is
intact in its locking script, and a top-up under §4 resumes it at exactly the step it stopped on.
Implementations MUST NOT add an "expired" state, and in the ownerless configuration MUST NOT provide any path
that recovers the fuel to a party.

Remaining steps are `floor(V / MAX_FEE)`, computable by any observer from the UTXO alone.

### 9. State

A covenant SHOULD carry its state as fixed-width fields in its own locking script per BRC-X, under the
three-push header defined there. This standard reserves no record types; the instance in §10 uses `0x07`.

### 10. Deployed evidence

⚠ **No delivery vehicle has been built.** The title names the use case the mechanism is for. What exists on
mainnet is a keyless computation, and a **quarter-mile drag race** as a vehicle proof of concept. Hold the
claim to what those demonstrate.

Every figure below was measured from the chain, not read out of the implementation.

**The battery** performs one Mandelbrot iteration per tick, in Script, in 2³² fixed point.

```
genesis    18e3193687078c40ee9a069a419d00f7b2a9c4374fe66e8d2b8a59d424711edd
           block 962,140 · 2026-08-13 · out0 = 2,100 sat · record 0x07
lock       1,428 B · 9 fields · 39 B of state · scope 0xc1 · MAX_FEE 314
tick 1     289e4a75f05c6c154387a512ae2f0f6b42cc84a4f74e674294f0edb918218c4e
           3,092 B · 1 in / 1 out · unlocking 1,600 B · fee 310 sat = 100.26 sat/KB
```

`MAX_FEE` 314 against that measured 3,092-byte spend puts the ceiling at 101.6 sat/KB, ~1.3% over the floor.

Walking the trace twenty steps reaches `b134dc9f…` in block 962,146, whose locking script is
**byte-identical** to the reference implementation replayed twenty times from genesis. Reading the twentieth
step off the trace and computing it locally are two routes to one answer, and the first costs no arithmetic.

⚠ The two directions are not symmetric. *Verifying* walks **backward** using only the transactions and needs
no index. *Locating the tip* walks **forward**, asking what spent each output — a question a transaction does
not answer. The tip is the step nothing has spent yet.

**§5's cases, on chain.** Thirteen consecutive pure ticks measured 3,092 B each, spread zero, fee 310 sat —
constant because fixed-width state makes the lock, and so the preimage, the same length every time. One
sponsored top-up measured 3,483 B, 2 in / 3 out, **fee 349 sat against a `MAX_FEE` of 314**, violating
nothing: the sponsor paid the difference. A reader who assumes the ceiling bounds the transaction fee will
wrongly conclude that spend should have been refused.

**The vehicle** does have size-varying moves: worst case **3,909 B** by serializing every variant, 3,957 for
a variant carrying one extra rule, with the ceiling re-derived for each. The completed quarter mile is
`ac49ed93…` — 402 m, 46 transactions, 166 KB, a 1,674-byte lock carrying 13 fields in 98 B under BRC-X.

⚠ The vehicle's role separation is **specified but only partly built**. The design assigns an owner signature
to configuring a vehicle and a driver signature to operating it; the deployed proof of concept carries one
key field serving both, which is correct for a driver operating their own vehicle and is not the two-key
arrangement §3 requirement 2 obliges for delegation. Neither that design nor this standard should be read as
describing what is deployed.

**The predecessor is the clearest illustration of §3 and §5.** An earlier battery was minted a day earlier
with an opening iteration cap of 6 — too low, so it drew a blob rather than a Mandelbrot. The cap is a script
constant and no key exists that could amend it, so it could not be patched, paused or recalled. A corrected
covenant had to be minted as a separate genesis; the flawed one still runs, advanceable by anyone,
unstoppable by its author, draining to flat.

★ **That is §5's option (c) in the wild, and it shows precisely what the escape hatch does and does not
cost.** Nothing was lost and nothing was stolen. It was never burnt, either — it is simply not being funded,
which is the whole of the remedy. What it cost was identity: the corrected instance is a different covenant
with a different txid, and references to the old one do not follow.

⚠ And it is dormant rather than gone. Anyone may fuel that first battery tomorrow and it will resume drawing
its blob, correctly, from the exact iteration it stopped on — because §8 guarantees exactly that. A published
covenant cannot be withdrawn; it can only be left alone. Implementations SHOULD expect their first genesis to
be wrong about something, SHOULD treat every baked constant as the whole of the engineering work rather than
as configuration, and SHOULD decide before minting whether they can live with a flawed instance remaining
revivable in public.

Both instances declare `BRC-226` in their genesis `OP_RETURN`. That is a conformance claim, not a number this
proposal requests: a battery *is* a BRC-226 covenant, and what this document adds is the funding model and
the authority rules on top.

The genesis also publishes the field layout, so the artefact is rebuildable from the chain with no reference
to any website:

```
BITCOIN BATTERY v1|cr,ci,zr,zi,i,step,cx,cy,mx|widths 5,5,5,5,2,5,5,5,2|
sign-mag LE|1=2^32|mul first div last|grid 3840x2160|mx0 128 k 128 cap 32767|
maxfee 314|ink esc+1-log2(log abs z) mod 32
```

### 11. Conformance

1. A tick is one input and one output, carries no signature over a secret, and needs no funding input.
   (§2, §3)
2. The value rule is a floor, so a top-up is a valid tick with no separate code path. (§4)
3. `MAX_FEE` is a script literal derived by serializing and measuring a worst-case spend, with headroom. (§5)
4. The covenant recomputes the successor state itself and refuses any other. (§7)
5. A flat covenant resumes at its exact prior step on top-up. (§8)
6. Authority is enumerated in the script, and any authority that can direct value is gated by a key distinct
   from the driver's once the driving credential is delegated. (§3)

⚠ Item 3 is the one whose cost is paid before anyone notices. A `MAX_FEE` below the relay floor yields an
instance that has consumed its funding and can never move again — and where the ceiling is a baked literal
(§5a), the remedy is not an edit but a re-mint, which strands whatever the instance already holds and
abandons its identity. Get it right at genesis or choose §5b knowingly.

## Implementations

| covenant | genesis | lock | state | tick | fee |
|---|---|---|---|---|---|
| Bitcoin Battery | `18e31936…` | 1,428 B | 9 fields / 39 B | 3,092 B | 310 sat |
| Vehicle (proof of concept) | `ac49ed93…` | 1,674 B | 13 fields / 98 B | ≤ 3,909 B | — |

The reference implementation and its test suite are published on chain, so this document cites a payload
rather than a URL:

```
source bundle   078a45523f5ba0597aeffe01296d0543566defc20a56c8b4de109ed783bbab17   138,642 B
```

It was verified by retrieving the archive from the chain and comparing it byte for byte against the local
copy — sixteen files, all CRCs intact.

**Tooling.** §5 asks someone to check a permanent number against a script, and §11 makes that the one thing
that cannot be fixed later. In raw Script it is not a practical check: the ceiling is a two-byte push inside
a thousand opcodes, and the fee it implies depends on a serialization nobody performs by eye. So the honest
failure mode of this standard is a correct rule the deployer could not verify. A decompiler answers it — a
conforming script is read back out of the chain as a short BASIC-dialect program, and that dialect compiles
forward to Script, so the two can be checked against each other. A sibling covenant's shipped script, 1,108
hand-written opcodes, reads back as 253 lines.

- Workbench: https://grafverse.com/basic.html

The reference implementation and its test suite were written by Claude (Opus 5).

## References

- BRC-X — Fixed-Width State in a Covenant's Own Locking Script; the state layout referenced in §9
- BRC-226 — Miner-Enforced Resale-Royalty Covenant Tokens, which establishes the `OP_PUSH_TX`
  self-reconstruction this standard depends on for §3. ★ The contrast is the funding model: BRC-226's spends
  are paid for by the party transacting, and it has a party transacting. This standard addresses the case
  where there is nobody
- BIP 143 — the preimage members `scriptCode`, `hashOutputs` and the spent output's value, read in §2 and §4
- Bitcoin SV opcode semantics for `OP_SPLIT`, `OP_CAT`, `OP_BIN2NUM` and `OP_NUM2BIN`
