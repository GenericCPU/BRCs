# BRC-Y: Giving an AI Agent Control of a Delivery Vehicle Without Giving It a Wallet

sun-dive (https://github.com/sun-dive)

## Abstract

An agent that spends on Bitcoin is normally given a wallet, which is to say a key — and a key is unbounded
capability: it signs anything, for whoever holds it. Funding it lightly bounds the amount at risk without
bounding the behaviour.

This standard defines a **battery**: a covenant that carries its own fuel, pays each step's fee from the value
it already holds, and enforces its own successor state. The authority to act correctly sits in the script rather
than in anyone's possession, so an agent can be given **control without custody** — at most a credential that
advances the covenant, which cannot direct value anywhere.

## Motivation

An autonomous program must answer a question a signed transaction never does: who pays for the next step, when
nobody is watching? Answering it with a hot wallet makes the agent a custodian — and a custodian that reads
untrusted text for a living.

`OP_PUSH_TX` removes the key: a covenant reconstructing the sighash preimage on its own stack constrains its
successor without anyone signing for identity, as BRC-226 establishes. What remains open is the fee, since a
transaction with no fee does not relay and a keyless covenant has no funding UTXO. Hence:

> **Let the covenant's own carried value be the fuel, and let the fee come out of it.**

A step is then a complete transaction — one input, one output, no funding, no change — that any stranger can
broadcast. The value becomes a **fuel gauge** readable from the UTXO, and the steps become a **computation
trace**, each hop enforced before it could exist, so it is walked and checked rather than trusted.

### Scope

Defines how an on-chain action is paid for and authorised. Does not define what the covenant computes, or how
its state is encoded — for the latter see BRC-X.

⚠ It bounds an agent's on-chain **actions**, not its reasoning: *the agent thinks wherever it likes; it can only
act within the covenant.* A battery does not make an agent correct, honest or safe to deploy.

This is the most restrictive rung of a ladder; `payee-bound` and `rate-bound` covenants are separate proposals
and MUST NOT be reached by relaxing §4. Capping value *per unit of time* is unavailable — a covenant has no clock
but `nLockTime`, and gating on it restricts honest use identically.

## Specification

The key words "MUST", "MUST NOT", "SHOULD" and "MAY" are as described in RFC 2119.

### 1. Terminology

- **Fuel** — the satoshi value carried by the covenant's output; the only thing that pays for its steps.
- **Tick** — one spend that performs one step and re-creates the covenant with the advanced state.
- **`V`** — the fuel carried by the output being spent, read from the preimage.
- **`MAX_FEE`** — the ceiling on how much fuel one tick may consume.
- **Flat** — a covenant whose remaining fuel can no longer pay for a tick.

Two roles are OPTIONAL; a covenant MAY bind either, both or neither to a public key:

- **Driver** — signature required to *advance* the covenant. Authority to act, nothing else.
- **Owner** — signature honoured for rights over the instance itself, such as retiring it and recovering what it
  holds. Authority that reaches value.

### 2. The tick

```
in0    the covenant           unlocking script = the preimage
out0   the covenant, quined, state advanced, value ≥ V − MAX_FEE
```

`out0` MUST be first; later outputs MAY be present and MUST NOT be constrained beyond being committed to in the
sighash. A tick MUST NOT require a funding input — the moment it does, the ticker needs a wallet.

⚠ Two covenants that each rebuild at output 0 cannot share a transaction. Where they must interact, one MUST
tolerate a fixed output offset and recognise its counterparty by script *shape*, not by a hash of a script at
rest.

### 3. Authorisation, ownership and control

The covenant MUST authorise the advance by validating the preimage supplied in the unlocking script, so the
successor is enforced by the script rather than asserted by a signer. Beyond that, ownership and control by
public key are both permitted:

| configuration | who may advance | rights over value |
|---|---|---|
| ownerless | anybody | nobody — fuel can only become mining fees |
| controlled | one keyed driver | nobody |
| owned | anybody, or one keyed driver | a keyed owner, per the rights the script grants |

**Authority MUST be enumerated in the script:**

1. A driver's authority MUST be limited to advancing the covenant, and MUST NOT suffice to direct value to an
   output of the signer's choosing.
2. Any authority that *can* direct value MUST be gated by a key **distinct from the driver's** whenever the
   driving credential is held by someone other than the owner. One key doing both jobs turns a driving
   credential into a spending credential the moment it is issued.
3. No configuration grants an authority the script does not spell out. There is no implicit administrator.

⚠ A conforming script *contains* a signature opcode — `OP_PUSH_TX` is built from `OP_CHECKSIG` against a constant
whose scalar is published — so counting them is not a conformance test. The property is **"no secret exists"**,
and MUST be tested by parsing script chunks rather than searching hex.

### 4. The value rule and the top-up

The covenant MUST enforce `out0.value ≥ V − MAX_FEE`. It MUST be a **floor**, MUST NOT be an equality.

Under a floor, a transaction that *increases* the fuel is already a valid tick, so a top-up needs no separate
code path, no privileged key and no second script. ⚠ Under an equality the fuel can only fall: the covenant can
never be refuelled by anyone and dies permanently on reaching `MAX_FEE`, with no symptom until it stops.

The covenant SHOULD use `SIGHASH_ANYONECANPAY | ALL | FORKID` (`0xc1`), so a sponsor MAY add a funding input
without invalidating the covenant's introspection while `out0` stays pinned.

⚠ **Anyone can empty it; nobody can take it.** Whoever may advance the covenant may advance it maliciously until
it is flat, but every satoshi that leaves is bounded by this rule and lands where the script says — never at an
address of the attacker's choosing. Mitigation is operational: **fund lightly and top up.** That is a griefing
limit, not a theft limit.

### 5. The fee ceiling

`MAX_FEE` MUST be derived by **serializing a real worst-case spend and measuring it**, then applying the relay
rate with modest headroom. It MUST NOT be hand-counted, inferred from the locking script alone, or copied from
another covenant. It MUST cover the worst spend **any legal variant** can produce, and MUST be measured on the
transaction the covenant **exists for** rather than the simplest one it can make.

```
MAX_FEE ≥ ceil(worstCaseTickBytes × relayRate) × headroom
```

⚠ Set below what the network accepts, no tick can ever pay enough to relay and every satoshi the instance holds
becomes unreachable. `MAX_FEE` MAY be a script literal, which is permanent, or a field an owner key may update,
which forecloses the ownerless configuration; implementations SHOULD state which. A wrong literal is escaped only
by **superseding** the instance — mint a corrected covenant, stop fuelling the old — which needs no key but
forfeits the original's identity and leaves it dormant rather than gone, since §7 lets anyone revive it.

⚠ **This is a custody decision, not configuration.** §4 places no upper bound on what an instance may come to
hold, so the exposure from an error is whatever it accumulates over its life, growing with the covenant's
success. A bug strands fuel as effectively as a bad constant: a state no spend can satisfy need only be
*reachable*, and until reached it is indistinguishable from working code.

⇒ Because a covenant of useful size cannot be read by eye, what gets verified is a mental model of the script,
and one modification moves every offset and depth behind it — so errors surface after minting and funding, when
nothing can be altered. **An implementation MUST be able to read its covenant back, not merely write one.**

### 6. Bounded steps

Each tick MUST perform a step whose worst case is known when the script is written, because §5 depends on it.
The covenant MUST recompute the successor from the state in its own `scriptCode` and MUST refuse any other, so a
ticker chooses *whether* to advance it, never *how*.

The whole computation need not be bounded: **the loop is not in the script, it is in the trace.** Nothing here
gives Script a backward jump.

### 7. Halting, resumption, and destruction

When `V < MAX_FEE` the covenant is **flat**, and a flat covenant MUST NOT be treated as terminated: its state is
intact and a top-up resumes it at exactly the step it stopped on. Remaining steps are `floor(V / MAX_FEE)`.
*Ticks forward, or waits for recharge* is a complete life cycle.

> **A covenant needs a burn path UNLESS permanence is the point of it.**

The test is not "can the money come back" but **bounded and intentional** against **unbounded and incidental**:
one immortal output on purpose is a monument, one per race or per mint is a leak every node carries forever. The
quantity is the number of unspendable outputs, not the value in them.

⚠ **An ownerless instance MUST NOT have a burn path.** A burn needs a signature, a signature needs an owner, and
an owner destroys the property the design exists for — the combination is a covenant free to the first
passer-by. Where a burn is warranted the instance is **owned** under §3 and MUST be built so from genesis. Its
owner key MUST be written once, at genesis, and MUST NOT be writable again; the branch MUST sit below the owner
check and MUST re-verify that the instance is claimed, since an unwritten owner field skips the signature check
rather than failing it.

### 8. Conformance

1. A tick is one input, one output, requires no funding input, and no secret key material exists. (§2, §3)
2. The value rule is a floor, so a top-up is a valid tick with no separate code path. (§4)
3. `MAX_FEE` is derived by serializing and measuring a worst-case spend, covering every legal variant. (§5)
4. The covenant recomputes the successor itself and refuses any other. (§6)
5. A flat covenant resumes at its exact prior step on top-up. (§7)
6. Authority is enumerated in the script, and any authority that can direct value is gated by a key distinct
   from the driver's once the driving credential is delegated. (§3)

⚠ Item 3 cannot be fixed later and MUST be satisfied **before the genesis is broadcast**. ★ Test the refusals
harder than the acceptances, with a control that withholds the flag under test: **a rule no test has provoked is
a rule no test has examined.**

## Implementations

Deployed on BSV mainnet; every figure measured from the chain.

| | genesis | lock | state | tick | fee |
|---|---|---|---|---|---|
| Bitcoin Battery — one Mandelbrot iteration per tick | `18e31936…` | 1,428 B | 9 fields / 39 B | 3,092 B | 310 sat |
| Vehicle proof of concept — a quarter-mile drag race | `ac49ed93…` | 1,674 B | 13 fields / 98 B | ≤ 3,909 B | — |

The battery's `MAX_FEE` is 314 against that measured 3,092-byte spend: 101.6 sat/KB, ~1.3% over the floor.
Walking its trace twenty steps reaches `b134dc9f…`, whose locking script is byte-identical to the reference
implementation replayed twenty times from genesis.

⚠ **No delivery vehicle has been built.** The title names the use case the mechanism is for.

The reference implementation and its test suite are published on chain, so this document cites a payload rather
than a URL — verified by retrieving the archive and comparing it byte for byte:

```
source bundle   078a45523f5ba0597aeffe01296d0543566defc20a56c8b4de109ed783bbab17   138,642 B
```

§5's reader requirement is met by a decompiler rendering a deployed locking script as a short BASIC-dialect
program which compiles forward again, so the two can be checked against each other: 1,108 opcodes read back as
253 lines. Workbench: https://grafverse.com/basic.html — a separate proposal is forthcoming.

The reference implementation and its test suite were written by Claude (Opus 5).

## References

- BRC-X — Fixed-Width State in a Covenant's Own Locking Script; the state layout used by both instances above
- BRC-226 — Miner-Enforced Resale-Royalty Covenant Tokens, establishing the `OP_PUSH_TX` self-reconstruction this
  depends on. The contrast is the funding model: BRC-226's spends are paid for by the party transacting
- BIP 143 — the preimage members `scriptCode`, `hashOutputs` and the spent output's value, read in §2 and §4
- Bitcoin SV opcode semantics for `OP_SPLIT`, `OP_CAT`, `OP_BIN2NUM` and `OP_NUM2BIN`
