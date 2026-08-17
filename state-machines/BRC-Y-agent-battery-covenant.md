# BRC-Y: Giving an AI Agent Control of a Delivery Vehicle Without Giving It a Wallet

sun-dive (https://github.com/sun-dive)

## Abstract

A delivery vehicle has to pay as it goes, and it cannot stop to ask permission. An operator who wants an AI
agent to run one therefore has to answer a question that is currently blocking a great deal of agent work:

> **How do you let software spend money, unsupervised, for weeks, without handing it a wallet?**

Give it a wallet and you have handed over a key, and a key is **unbounded capability**: it will sign
anything, for anyone who comes to hold it. The usual mitigation — fund a small hot wallet and top it up —
bounds the *amount* at risk without bounding the *behaviour*, so what it buys is being robbed in small
amounts, repeatedly. Rotating the key afterwards concedes the same point.

This standard describes the answer already running on mainnet: **the agent is given control, and never
custody.**

A **battery** is a covenant that carries its own fuel. The fee for each step comes out of the value the
covenant already holds, so the agent needs no funding source, and the step itself is authorised by a proof
*about the transaction* rather than by a signature over an identity — so the authority to act correctly is
in the script, not in anybody's possession.

The agent may then be given a **driving credential**: a key the covenant will accept for the single purpose
of advancing it. That key cannot move value anywhere. It cannot pay an address, cannot empty the vehicle,
and cannot be used away from the covenant it names. In the simplest deployments there is no such key at all
and any passer-by may advance the covenant — the fractal in §10 runs that way, and has since August 2026.

What a battery bounds is the **action**, not the amount. The worst a fully compromised agent can do is
spend fuel doing precisely what the covenant was built to do. *"The agent's wallet got drained"* becomes a
category error: there is no wallet, and the failure mode is a **flat battery** rather than a theft — it
stops when the fuel runs out, at the exact step it stopped on, and resumes the moment anyone adds more.

⚠ The operator keeps ownership. This standard permits an **owner** key with rights over the vehicle itself —
retiring it, recovering what it holds — because a fleet operator who cannot retire a vehicle owns a
liability. The whole of the security argument rests on that key being a *different* key from the one the
agent is given, and §3 makes it a requirement rather than a suggestion.

Because each spend is one step whose correctness the covenant enforced before the spend could exist, the
resulting sequence is a **computation trace**: a record that is walked and checked rather than trusted or
replayed. The network is not storing the answer to a computation performed elsewhere — it is performing
the computation, enforcing each step, and leaving the working behind.

The standard defines the value rule, the fee ceiling, the authorisation and the top-up. It does not
define the work.

## Motivation

### A key cannot be made safe by making it small

The stated worry about agents transacting on chain is that the agent's wallet gets drained. The stated
answer is to keep the wallet small and refill it. That answer treats the problem as a quantity when it is a
**capability**: a key authorises any transaction whatsoever, so a compromised agent with a small key is not
a bounded agent, it is an agent that can be robbed in convenient instalments. Rotating the key afterwards
concedes the same point.

The deeper trouble is that the key is a *secret held by the agent*. Agents read untrusted text; that is
most of what they are for. A secret in the possession of something that processes attacker-controlled input
is a secret with a disclosure schedule, and no amount of policy wrapping changes what the key does once
disclosed.

### What a battery changes

`OP_PUSH_TX` removes the key entirely. A covenant that reconstructs the sighash preimage on its own stack
can constrain its own successor without anyone signing anything, because the authorisation is a proof
*about the transaction* rather than a proof of identity. The technique is established; BRC-226 uses it to
enforce a royalty. What this standard does with it is give an agent something to drive that has no
credential attached.

The consequences are worth stating individually, because each answers a question that is otherwise
answered with a policy document:

| the question | with a wallet | with a battery |
|---|---|---|
| what can a compromised agent do? | anything the key allows | advance the program, and nothing else |
| what can be exfiltrated? | the key | there is nothing to exfiltrate |
| what does a prompt injection obtain? | signing capability | no capability; the covenant is the authority |
| what does key rotation cost? | an incident response | there is no key to rotate |
| what is the worst case? | total loss of the balance | the fuel is spent doing the intended work |

⚠ "Fund small and top up" is not wrong — it is simply the wrong *kind* of limit. Under this standard it
becomes a **fuel** limit, chosen for how much work you want done, rather than a **theft** limit chosen for
how much you can afford to lose.

### The battery is the product; the load is a choice

> *A Bitcoin battery can power a Mandelbrot generator, or an AI agent.*

Both are loads on the same covenant, and the difference between them is where the thinking happens. Driving
a fractal, **the chain does the computing** — each spend performs an arithmetic step the script itself
enforces. Driving an agent, **the agent does the thinking and the chain bounds what it may do with the
conclusion**.

★ This is why the fractal is in the document at all. You cannot read your way to believing *"this agent
cannot be robbed"* — the claim is about an absence, and absences are hard to see. But you can watch a
fractal draw itself, on chain, with no key anywhere in existence, advanced by strangers who never needed
permission. The visible load demonstrates the property that the invisible one depends on. Same
covenant, same rules; one of the two happens to be a picture.

The second load is a **vehicle**. A racing covenant carries its own physics as state — position, velocity,
fuel — and each spend advances it one step down a track, consuming fuel to do so. Something outside decides
*how* to drive; the covenant decides what a legal move is, and will not record an illegal one. That is the
agent case with something to look at: the reasoning is external and unconstrained, the action is internal
and bounded.

### Permission to drive is not permission to spend

A delivery vehicle makes the separation concrete, and it is the case this standard exists to serve. An
operator wants an agent to run a van: it must be able to act, repeatedly, for weeks, without supervision —
and it must not be able to do anything else with the money. Those are **two different permissions**, and a
wallet cannot tell them apart, because a key authorises *transactions* rather than *actions*.

A battery separates them, because the two questions are answered in different places:

| capability | where it is decided |
|---|---|
| move value to an arbitrary address | the script — and a conforming one contains **no such path at all** |
| advance the program (drive) | the script — either anybody, or exactly one keyed driver |
| what to do next (route, speed, when) | the agent, unconstrained, off chain |

A driver gate is straightforward: the covenant reads the driver's hash from its own locking script, requires
an offered public key to hash to it, and verifies a signature, so every advance must be authorised by that
one party. The reference implementation does exactly this in its owned configuration, and omits it in its
public one, where any passer-by may advance the vehicle.

★ **So the agent is issued a credential to act, never a credential to spend.** The blast radius of losing it
is that somebody drives your van: bounded, physical, immediately visible, and costing only fuel. The blast
radius of losing a wallet key is the balance. Those are not the same incident, and "the agent needs a key"
silently equates them.

⚠ A driver gate does **not** weaken the battery property, and the two signature checks in such a script must
not be confused. Restricting *who* may advance a covenant is orthogonal to bounding *what* it will accept:
§4 and §5 are untouched by it, and the ladder in Scope widens the second axis, not this one.

⚠ What *does* leak the property is issuing one key for both jobs. A fleet operator legitimately wants the
power to retire a vehicle and recover what it holds — §3 permits exactly that as an **owner** right. But if
the same key also authorises driving, then handing the driving credential to an agent hands over the
recovery power too, and a recovery signed `SIGHASH_ALL` pays whatever address the signer named. The covenant
is not at fault in that scenario and no rule of this standard was broken; the key issuance was the mistake.
**Bind the driver and the owner to different keys before the driving credential leaves your custody.**

⚠ **The deployed instances differ in how strong that claim currently is, and the difference is checkable,
so it is stated rather than glossed.** Measured on the live locking scripts:

| deployed | `OP_CHECKSIG` | of which verify a *secret* key |
|---|---|---|
| battery `18e31936…` | 1 | **0** — the one is `OP_PUSH_TX`'s, against a published constant |
| racing car `e918fa43…` | 2 | **1** — an owner-signed path, retained while the design is proved |
| fuel depot `e889c1f1…` | 2 | **1** |

The battery is in the **ownerless** configuration: its single `OP_CHECKSIG` is `OP_PUSH_TX`'s, so
keylessness there is *structural* — a property of the script, provable by reading it once, and it cannot
rot.

The vehicle and the depot are in the **owned** configuration, which §3 permits, and their second
`OP_CHECKSIG` is the owner's. In the deployed public vehicle that check is reached *only* on the retirement
branch: moving the vehicle requires no signature whatsoever, while retiring it and recovering its value
requires the owner's. The twenty-byte field the depot reads across the boundary is that owner.

★ **The deployed vehicle therefore already separates the two authorities**, in the strongest form available:
driving requires no credential at all, so an agent can be handed the job with nothing to lose and nothing to
steal, while the operator retains a key to the asset itself.

⚠ **The role separation is fully specified but only partly built, and that distinction is stated here rather
than smoothed over.** The design assigns an **owner** signature to configuring a vehicle and a **driver**
signature to launching and operating it — two roles, two keys. The deployed proof of concept carries a single
twenty-byte key field serving both, which is correct for its own case (a driver operating a vehicle they own)
and is not yet the two-key arrangement §3 requirement 2 obliges for delegation. An implementer building the
delegated case MUST bind the roles to distinct keys; the reference implementation is behind its own
specification on this point, and neither the specification nor this standard should be read as describing
what is currently deployed.

What it leaves open is the fee. If no one holds a key, no one holds a funding UTXO either, and a
transaction with no fee does not relay. The step this standard takes is small and it changes the
character of the thing entirely:

> **Let the covenant's own carried value be the fuel, and let the fee come out of it.**

A tick is then a complete, self-contained transaction — one input, one output, no signature, no funding,
no change — that any stranger can broadcast for free. The covenant is not *hosted*, not *operated* and
not *owned*. It simply has a balance, and while the balance lasts it computes.

Two properties fall out of this that are worth naming, because they are the reason to standardise it
rather than to write it once:

- **The value becomes a fuel gauge.** `floor(value / MAX_FEE)` is the number of remaining steps, readable
  by anyone from the UTXO alone. A program you can watch run out of power is legible as a machine in a
  way that an indefinitely-funded service is not.

- **The sequence of spends is a computation trace, and the network is the machine that ran it.** Each
  transaction is one step, and the covenant enforced that step before it could be recorded — so the steps
  are not a log written by a program that was trusted, they are the only steps the rules permitted. To
  verify the computation you do not re-run it. You **walk the trace**, checking at each step that the
  successor is the one the covenant demanded, which is a proof-checking cost rather than an execution
  cost. What is being demonstrated is not that Bitcoin can *store* the result of a computation, but that
  it can *perform and enforce* one, and hand you the working afterwards.

### Scope

This standard defines **how an on-chain action is paid for and authorised**, and nothing else.

It does not define what the covenant computes, what its state fields mean, how many there are, or how
they are encoded — see BRC-X for the last of those. It does not define the geometry, precision or
termination of any particular workload. Those differ for every program worth writing, and a standard that
guessed at them would be wrong for all of them.

⚠ **It bounds an agent's on-chain actions — writing, paying, proving — not its thinking.** Overclaim here and
a reader will correctly object that the agent is not running on Bitcoin. Stated precisely the claim is
stronger:

> *The agent thinks wherever it likes; it can only act within the covenant.*

A battery does not make an agent correct, honest, aligned or safe to deploy. It removes exactly one
capability — the ability to move value anywhere other than forward through its own program — and leaves every
other question about the agent untouched.

**A battery is the first rung of a ladder**, and is deliberately the most restrictive one. Where an agent
must genuinely pay somebody, the same technique extends:

```
1. battery       may only advance its own program, paying its own fee   ← THIS STANDARD
2. payee-bound   may pay only recipients enumerated when it was built
3. rate-bound    caps value per spend or per block, enforced in script
```

Each rung keeps the property that matters — the agent holds no key — while widening what the covenant will
accept. Rungs 2 and 3 are out of scope here and are described in their own proposals; a reader who needs
them should not attempt to reach them by relaxing §4, which is what makes rung 1 safe.

What rung 1 buys is that any covenant written this way can be advanced by any tool, funded by any
stranger, and metered by any observer, with no shared code and no agreement beyond this document.

## Specification

The key words "MUST", "MUST NOT", "SHOULD" and "MAY" are to be interpreted as described in RFC 2119.

### 1. Terminology

- **Pot** — the satoshi value carried by the covenant's output. The fuel.
- **Tick** — one spend of the covenant that performs one step of the computation and re-creates the
  covenant with the advanced state.
- **Computation trace** — the ordered sequence of ticks from the genesis output to the unspent tip. Each
  hop is one step, and the covenant enforced that step's correctness as a condition of the hop existing
  at all. The trace is therefore a *proof* of the computation rather than a log of it, and it is walked —
  step by step — rather than replayed.
- **`V`** — the pot of the output being spent, read by the covenant from the preimage.
- **`MAX_FEE`** — the permanent ceiling on how much value a single tick may remove from the pot.
- **Ticker** — whoever broadcasts a tick. Risks nothing and gains nothing.
- **Load** — what the fuel is spent on. A load may be a computation the script performs itself, or an
  action taken on behalf of an agent that reasons elsewhere.
- **Sponsor** — whoever adds value to the pot.
- **Flat** — the state of a covenant whose pot can no longer pay for a tick.

Two roles are OPTIONAL, and a conforming covenant may bind either, both, or neither to a public key:

- **Driver** — a party whose signature the covenant requires in order to *advance* it. Where no driver is
  bound, any ticker may advance it. A driver's authority is to act, and nothing else.
- **Owner** — a party whose signature the covenant honours for rights over the instance itself, such as
  retiring it and recovering what it holds. An owner's authority extends to value.

⚠ **Driver and owner are different roles and SHOULD be different keys.** They are frequently the same key
in a first implementation, because one party is doing both jobs while a design is being proved — and the
moment a *third party* is handed the driving credential, that convenience becomes a spending credential.
See §3.

### 2. The tick

A conforming tick MUST have this shape:

```
in0    the covenant           unlocking script = the preimage — NO SIGNATURE
out0   the covenant, quined, state advanced, value ≥ V − MAX_FEE
```

`out0` MUST be the first output. Outputs after `out0` MAY be present (see §6); the covenant MUST NOT
constrain them beyond committing to them in the sighash.

⚠ A tick MUST NOT require a funding input. The moment it does, the ticker needs a wallet and a key, and
every property in this document is lost.

### 3. Authorisation, ownership and control

The covenant MUST authorise the *advance itself* by validating the sighash preimage supplied in the
unlocking script — the `OP_PUSH_TX` technique — so that the successor state is enforced by the script rather
than asserted by a signer.

Beyond that, this standard **permits both ownership and control by public key**, and the choice is the
implementer's:

| configuration | who may advance it | who has rights over the value |
|---|---|---|
| **ownerless** | anybody | nobody — the fuel can only become mining fees |
| **controlled** | one keyed driver | nobody |
| **owned** | anybody, or one keyed driver | a keyed owner, per the rights the script grants |

An ownerless, uncontrolled covenant is the strongest form and the one whose properties are easiest to
state: it is irreversibly nobody's, with no administrative path, no upgrade, no pause and no recovery.
A controlled or owned covenant trades some of that for operational reality, which is often the correct
trade — a fleet operator who cannot retire a vehicle owns a liability.

**What MUST hold in every configuration** is that authority is *enumerated in the script*:

1. The driver's authority, where a driver is bound, MUST be limited to advancing the covenant. A driver
   signature MUST NOT be sufficient to direct value to an output of the signer's choosing.
2. Any authority that *can* direct value — a retirement or recovery path — belongs to the owner role and
   MUST be gated by a key distinct from the driver's whenever the driving credential is held by a party
   who is not the owner.
3. No configuration may grant an authority the script does not spell out. There is no implicit
   administrator.

⚠ **Requirement 2 is the one that matters for agents, and it is easy to fail by accident rather than by
design.** If a single key both advances the covenant and authorises its retirement, then issuing that key
to an agent issues the retirement power with it — and a retirement signed `SIGHASH_ALL` commits to whatever
outputs the signer chose, which is to say it can pay the signer. The covenant is still sound; the *key
issuance* is what leaked. Split the roles before handing the driving credential to anything you would not
trust with the balance.

⚠ **Note carefully what that does not say, because the obvious test is the wrong one.** The preimage check
is itself normally implemented *using* `OP_CHECKSIG`: the script derives a signature over the pushed
preimage with a constant scalar whose value is published, pushes the corresponding public point, and lets
`OP_CHECKSIG` recompute the sighash from the actual transaction. The spend validates only if the pushed
preimage is the genuine one. A conforming script therefore **does contain a signature opcode**, and
counting signature opcodes is not a conformance test. The question is only ever whether any key material is
secret. Under this standard none is, and the `OP_CHECKSIG` present is a proof-of-preimage primitive rather
than an authorisation of identity.

⚠ Implementations and their documentation SHOULD state the claim as *"no secret exists"* and MUST NOT state
it as *"the script contains no signature opcodes"*. The second is checkable, is what a reviewer will check,
and is false.

The consequences, in every configuration, are:

1. A ticker cannot profit from ticking and cannot be harmed by it, because it spends none of its own value
   and receives none.
2. The successor state is not a matter of trust. Whoever advances the covenant may choose *whether* to
   advance it, never *how*, because the script recomputes the next state and refuses anything else.
3. The credential an agent is issued — if it is issued one at all — carries the authority to act and no
   authority over value. That is a smaller thing to lose than a key, and losing it has a smaller
   consequence than losing a key.

Additionally, in the ownerless configuration: there is no credential associated with the covenant at all,
so none can be stolen, lost, leaked or subpoenaed, and the set of parties who may advance it is everybody.

⚠ **Whatever the genesis does not provide for, no one can add later.** There is no upgrade path, and in the
ownerless configuration no pause and no recovery either. Everything the covenant will ever do MUST be
correct when the genesis output is broadcast, and implementers SHOULD treat the genesis as a permanent
publication rather than as a deployment. §10 records an instance where a single badly chosen constant could
not be fixed and forced a rebuild.

### 4. The value rule

The covenant MUST enforce:

```
out0.value ≥ V − MAX_FEE
```

It MUST be a **floor**, and MUST NOT be an equality.

This is the single most consequential line in the standard, and the reason is not obvious. Under a floor,
a transaction that *increases* the pot is already a valid tick — the top-up needs no separate code path,
no privileged key and no second script. Adding fuel and advancing the state are the same operation, which
means a contribution and the work it buys are atomic by construction.

⚠ Under an equality, the pot can only ever fall. The covenant cannot be refuelled, by anyone, ever, and
it dies permanently when the pot reaches `MAX_FEE`. This is unamendable, and it has no symptom until the
day the program stops for good.

An honest ticker pays only the fee the network actually requires; `MAX_FEE` bounds the worst case, it is
not the price. A ticker that takes the full ceiling every time is behaving within the rules, and it is
performing the computation while doing so.

### 5. The fee ceiling

`MAX_FEE` MUST be a literal baked into the locking script, and it is therefore permanent.

It MUST be derived by **serializing a real worst-case spend and measuring it**, then applying the network's
relay fee rate. It MUST NOT be hand-counted, estimated from the locking script alone, or copied from
another covenant.

```
MAX_FEE ≥ ceil(worstCaseTickBytes × relayRate) × headroom
```

Implementations SHOULD include modest headroom (on the order of 1%) so that a node counting a byte
differently, or a small future drift in the relay floor, cannot strand the covenant.

⚠ **This bound is unamendable and there is no key that could raise it.** If `MAX_FEE` is set below what
the network will accept for the worst-case tick, the covenant is dead on arrival: no tick can ever pay
enough to relay, no signature exists that could authorise an exception, and the pot is stranded forever.
Deriving it from a serialized spend is the only method this standard permits, because it is the only one
that has ever been observed to be right.

⚠ The worst case is not the average case. The unlocking script — which carries the whole preimage — is
normally the dominant term, and it is easy to omit when reasoning about "the script".

**Which spends vary in size, and which of them threaten the bound.** This is the part that is routinely
got wrong, so it is worth separating into three cases. Only the first is dangerous.

1. **Script variants — dangerous, and the bound MUST be re-derived per variant.** Every rule that emits
   opcodes makes the *cheapest* spend more expensive to mine. A variant whose script grew while its
   ceiling did not is a covenant no node will relay, and it looks exactly like a healthy one until the
   first spend is refused. Implementations MUST derive the ceiling from a single named worst-case
   measurement shared by the builder and by whatever checks the builder — two implementations of "how big
   does this get" is precisely how a bound and the thing it bounds drift apart.

2. **Additional inputs and outputs — not dangerous.** A sponsored top-up is a larger transaction, but the
   sponsor funds the extra bytes. The ceiling bounds what leaves the *pot*, not what the transaction pays,
   so a top-up's fee may legitimately exceed it.

3. **Variable-length payload — not dangerous, for the same reason.** Free-text carried in an `OP_RETURN`
   alongside a contribution rides on the sponsor's transaction. Arbitrary user input therefore cannot
   threaten a permanent constant, which is worth knowing before designing it out.

⚠ **Drift in the safe direction is also a defect.** A ceiling left sitting even a few bytes high causes
every spend to overpay its miner, forever, with no symptom. Implementations SHOULD fail their own fee
check on drift in *either* direction, and treat a bound that can only be revised upward as unfinished.

### 6. Sighash scope and sponsorship

The covenant SHOULD use `SIGHASH_ANYONECANPAY | SIGHASH_ALL | SIGHASH_FORKID` (`0xc1`).

`ANYONECANPAY` excludes the other inputs from the preimage, so a sponsor MAY add a funding input to a
top-up without invalidating the covenant's own introspection. `ALL` still commits every output, so `out0`
remains pinned and the covenant's successor cannot be substituted.

Under this scope, a top-up is one transaction that adds the fuel, advances the state, and MAY carry
trailing outputs — a change output back to the sponsor, or an `OP_RETURN` recording the contribution.
Because the covenant commits to the outputs but constrains only `out0`, those trailing outputs are free
for the application to define.

⚠ Any published record carried this way is permanent and unmoderatable. Implementations displaying such
records to users SHOULD render them as inert text and MUST NOT automatically turn them into live links.

### 7. Bounded computation

Each tick MUST perform a **bounded** step: the work done by one spend MUST have a worst case that is
known when the script is written, because §5 depends on it.

The computation as a whole need not be bounded. The trace may be arbitrarily long, and this is the
mechanism by which a script containing no backward jump performs an unbounded computation: **the loop is
not in the script, it is in the trace.** Each iteration is a separate transaction, separately paid for and
separately verified.

⚠ This is worth stating plainly because it is routinely misread in both directions. Bitcoin Script has no
backward jump, and nothing here gives it one. What the trace shows is *where the loop went*: the iteration
lives in the sequence of enforced steps rather than inside any single script, and the bound on one step is
exactly what makes the unbounded whole payable under §5.

A conforming covenant MUST recompute the successor state itself from the state it reads out of its own
`scriptCode`, and MUST refuse any output whose state is not the one it computed. The ticker therefore has
no discretion: it may choose *whether* to advance the covenant, never *how*.

⚠ An external reference implementation that disagrees with the script by even one unit cannot advance the
covenant at all — the covenant recomputes and refuses. This is a safety property, not a bug, but it makes
the reference implementation's arithmetic normative in practice. Implementations SHOULD verify their
reference against exact integer arithmetic rather than floating point, and SHOULD state the operation
order explicitly where truncation makes it observable.

### 8. Halting and resumption

When `V < MAX_FEE` the covenant is **flat**: no tick can satisfy §4 while paying a relayable fee.

A flat covenant MUST NOT be considered terminated. Its state is intact in its locking script, and a
top-up under §4 resumes it at exactly the step it stopped on. Implementations MUST NOT introduce a
separate "expired" or "closed" state, and MUST NOT provide a path that recovers the pot to any party;
either would reintroduce the privileged actor that §3 removes.

The remaining step count is `floor(V / MAX_FEE)`, computable by any observer from the UTXO alone.

### 9. State

A conforming covenant SHOULD carry its state as fixed-width fields in its own locking script per BRC-X,
under the three-push header defined there, with a record type identifying the field list.

This standard does not reserve or assign record types. The deployed instance in §10 uses `0x07`.

### 10. Worked example — the visible load

⚠ **What is deployed, and what is not.** No delivery vehicle has been built. What exists on mainnet is a
**proof of concept**: a covenant that carries a vehicle's physics as state and runs a **quarter-mile drag
race**, one transaction per move, with every move's legality enforced by the script. The title of this
document names the use case the mechanism is *for*, not an application that has shipped. A reader evaluating
the claim should hold it to what a quarter mile demonstrates — that motion can be bounded, enforced and paid
for by the covenant itself — and no further.

Everything below was measured from the chain rather than read out of the implementation.

The Bitcoin Battery draws a Mandelbrot descent. Each tick performs **one** iteration of `z → z² + c` in
2³² fixed point inside Bitcoin Script, advances the scan when a pixel finishes, and zooms in when a frame
finishes.

The choice of workload is not incidental, though it is not normative either. A fractal was chosen because
it makes the standard's central claim *observable*: the picture advances only when a valid tick exists, so
every pixel drawn is a step that was paid for and enforced, and the whole of it happened with no key in
existence. An agent-driven battery is the same covenant with a different load — an agent that reasons
elsewhere and holds nothing — and it offers a spectator no picture to check. What follows is therefore the
evidence for the mechanism, not a description of the intended application.

```
genesis      18e3193687078c40ee9a069a419d00f7b2a9c4374fe66e8d2b8a59d424711edd
             block 962,140 · 2026-08-13 · out0 = 2,100 sat
locking      1,428 bytes · record 0x07 · 9 fields · 39 bytes of state
grid         3840 × 2160 · fixed point 1.0 = 2³²
scope        0xc1  (ANYONECANPAY | ALL | FORKID)
MAX_FEE      314
```

The first tick, measured:

```
tick         289e4a75f05c6c154387a512ae2f0f6b42cc84a4f74e674294f0edb918218c4e
             1 input · 1 output · no signature
size         3,092 bytes
unlocking    1,600 bytes — the preimage, the successor value, and nothing else
fee          310 sat  =  100.26 sat/KB
```

`MAX_FEE` was fixed at **314** against that measured 3,092-byte spend, which puts the ceiling at
101.6 sat/KB — about 1.3% above the 100 sat/KB relay floor, and the honest fee actually paid was 310.

**A pure tick does not vary in size, and that is a consequence of the state layout rather than luck.**
Thirteen consecutive pure ticks were measured along the trace:

```
13 pure ticks (1-in / 1-out)   3,092 B every one · spread 0 · unlocking 1,600 B · fee 310 sat
one sponsored top-up           3,483 B · 2 in / 3 out · fee 349 sat
```

Because every state field is fixed-width (BRC-X §2), the locking script is the same length at every step,
so the `scriptCode` inside the preimage is too, so the whole transaction is. That is §5 case 1 costing
nothing here — there is only one variant.

⚠ Note the top-up's fee of **349 satoshis against a `MAX_FEE` of 314**, and that nothing is violated:
the sponsor's input paid for the extra bytes, and the covenant's rule bounds what leaves the pot, not what
the transaction pays. This is §5 case 2, on chain. A reader who assumes the ceiling bounds the transaction
fee will conclude, wrongly, that this spend should have been refused.

**The vehicle proof of concept, for contrast, does have size-varying moves.** Its worst case is **3,909
bytes**, measured by serializing every variant, rising to 3,957 for a variant carrying one extra rule
(48 bytes), with the ceiling re-derived for each — §5 case 1, and the reason that case is written as a MUST.
Its `OP_RETURN`-carried contributor marks, up to 60 characters of free text, change no bound at all, being
case 3.

The completed quarter mile is on chain: genesis `ac49ed93…`, 402 metres, **46 transactions** and 166 KB,
finishing in 4.20 s of simulated time. Its locking script is 1,674 bytes and carries 13 state fields in 98
bytes under BRC-X. Two earlier vehicles at 1,636 bytes and a later public one at 1,744 bytes carry the same
field layout unchanged, which is worth noting for its own sake: the script around the state was rewritten
three times in two days without a reader of the state needing to know.

Walking the computation trace twenty steps reaches `b134dc9f…` in block 962,146, still one input and one
output, and its locking script is **byte-identical** to the reference implementation replayed twenty times
from the genesis state.

That identity is what makes the trace worth walking. The reference is not an approximation of the
covenant's behaviour that happens to agree — it agrees because the covenant would have refused anything
else, so reading the twentieth step off the trace and computing the twentieth step locally are two routes
to the same answer, and the first one costs no arithmetic.

⚠ The two directions are not symmetric, and implementers should not discover this the hard way.
*Verifying* a trace walks **backward** from any step to the genesis using only the transactions
themselves, and needs no index. *Locating the tip* walks **forward**, which means asking, at each step,
what spent this output — a question a transaction does not contain the answer to. The tip is simply the
step that nothing has spent yet.

The genesis publishes the field layout in an `OP_RETURN`, so the artefact can be rebuilt from the chain
with no reference to any website:

```
BITCOIN BATTERY v1|cr,ci,zr,zi,i,step,cx,cy,mx|widths 5,5,5,5,2,5,5,5,2|sign-mag LE|
1=2^32|mul first div last|grid 3840x2160|mx0 128 k 128 cap 32767|maxfee 314|
ink esc+1-log2(log abs z) mod 32
```

Everything there is something a reader could not derive from a tip: the field layout needed to parse the
state at all, the grid, the ramp constants governing frames not yet reached, and the recipe that turns an
escape count into a colour. The span is deliberately omitted, being recoverable from the state.

⚠ **The predecessor is the clearest illustration in this document of what §3 and §5 actually cost.** An
earlier instance was minted a day before this one, at a 256×192 grid with an opening iteration cap of 6.
That cap was too low: it drew a blob rather than a Mandelbrot. The cap is a script constant, and by
construction **no key exists that could amend it** — so it could not be patched, deprecated, paused or
recalled. A corrected covenant had to be minted as a separate genesis, and the flawed one is still
running, advanceable by anyone, unstoppable by its own author, and is simply being drained to flat.

This is the price of §3, and it is not hypothetical: *everything the covenant will ever do must be correct
before the genesis is broadcast.* Implementations SHOULD expect their first genesis to be wrong about
something, and SHOULD therefore treat the choice of every baked constant as the whole of the engineering
work rather than as configuration. (For the record, that first instance also labelled itself with a BRC
number that had not been assigned to it, and that label is likewise permanent.)

⚠ **An interoperability note.** Seven of the first twenty ticks were refused by a transaction broadcast
service reporting `461: Non-canonical signature: S value is unnecessarily high`. The rule had been
withdrawn for transactions with a version field greater than 1, and these are version 2 — a miner mined
them unchanged. Implementations SHOULD NOT assume a broadcaster's policy is consensus, and where a
keyless covenant is refused by one endpoint SHOULD try another before concluding the covenant is at
fault.

### 11. Conformance

An implementation conforms if:

1. A tick is one input and one output, carries no signature, and requires no funding input. (§2, §3)
2. The value rule is a floor, `out0.value ≥ V − MAX_FEE`, so that a top-up is a valid tick with no
   separate code path. (§4)
3. `MAX_FEE` is a script literal, derived by serializing and measuring a worst-case spend at the
   network's relay rate, with headroom. (§5)
4. The covenant recomputes the successor state itself and refuses any other. (§7)
5. A flat covenant resumes at its exact prior step on top-up, and no path exists that recovers the pot
   to any party. (§8)
6. No privileged key, owner or administrative action exists at any point in the covenant's life. (§3)

⚠ Item 3 is the one that cannot be fixed later. Every other error in this list produces a covenant that
misbehaves and can be replaced by a better one; a `MAX_FEE` below the relay floor produces a covenant
that has already consumed its funding and can never move again.

## Implementations

Deployed on BSV mainnet:

| covenant | genesis | lock | state | tick | fee |
|---|---|---|---|---|---|
| Bitcoin Battery | `18e31936…` | 1,428 B | 9 fields / 39 B | 3,092 B | 310 sat |

The reference implementation and its test suite are published on chain as an immutable artefact, so that
this document cites a payload rather than a URL:

```
source bundle   078a45523f5ba0597aeffe01296d0543566defc20a56c8b4de109ed783bbab17
```

It was verified by retrieving the archive from the chain and comparing it byte for byte against the
local copy — sixteen files, all CRCs intact.

**Tooling — a permanent constant is only as safe as the script is readable**

Every requirement in §5 asks someone to check a number against a script, and §11 makes that check the one
thing that cannot be fixed later. In raw Script it is not a practical check for most people: the ceiling is
a two-byte push somewhere inside a thousand-odd opcodes, the fee it implies depends on a serialization
nobody can perform by eye, and being wrong is unamendable. The honest failure mode of this standard is
therefore not a bad rule — it is a correct rule that the person deploying the covenant could not verify.

The response was a decompiler. A conforming locking script is read back out of the chain and rendered as a
short BASIC-dialect program, and a program in that dialect compiles forward to Script again, so the two
representations can be checked against each other. A sibling covenant's shipped script — 1,108
hand-written opcodes — reads back end to end as 253 lines.

- Workbench: https://grafverse.com/basic.html

⚠ Implementations SHOULD NOT treat this as ergonomics. A standard that asks for a measured, permanent,
unamendable constant, in a language its users cannot read, has moved the difficulty rather than addressed
it.

The reference implementation and its test suite were written by Claude (Opus 5).

## References

- BRC-X — Fixed-Width State in a Covenant's Own Locking Script, which defines the state layout referenced
  in §9 and used by the deployed instance
- BRC-226 — Miner-Enforced Resale-Royalty Covenant Tokens, which establishes the `OP_PUSH_TX`
  self-reconstruction technique this standard depends on for §3. ★ The contrast is the funding model:
  BRC-226's spends are paid for by the party transacting, and it has a party transacting. This standard
  addresses the case where there is nobody
- BIP 143 — Transaction Signature Verification for Version 0 Witness Program (the preimage members
  `scriptCode`, `hashOutputs` and the spent output's value, all read in §2 and §4)
- Bitcoin SV opcode semantics for `OP_SPLIT`, `OP_CAT`, `OP_BIN2NUM` and `OP_NUM2BIN`
