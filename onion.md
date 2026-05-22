# Onion Service Enrollment

This document specifies how Tor v3 onion services enroll into WEBCAT. The
goals differ from clearnet enrollment in two ways: the chain MUST NOT learn
the `.onion` identity, and submission MUST come with a proof of operator
control over the service.

The construction reuses Ed25519 blinded keys (as already used by Tor for
hidden-service descriptors) together with a zero-knowledge proof binding the
WEBCAT enrollment to the operator's master onion key.

This document does not redefine Tor's blinded-key derivation,
encoding conventions, or HSDir lookup. We adopt the notation and
constructions from the Tor specification by reference:

- [`rend-spec`](https://spec.torproject.org/rend-spec/) — Tor v3
  hidden-service rendezvous, descriptors, and HSDir protocol.
- [`rend-spec/keyblinding-scheme`](https://spec.torproject.org/rend-spec/keyblinding-scheme.html)
  — Ed25519 key blinding, in particular the definitions of
  `BLIND_STRING`, `N`, the base-point `B`, and the period parameters
  used in §2 and §4 below.
- [`rend-spec/deriving-keys`](https://spec.torproject.org/rend-spec/deriving-keys.html)
  — derivation of `KP_hs_blind` and `KS_hs_blind` from the master key.

Where this spec uses a Tor symbol (`BLIND_STRING`, `N`, `INT_n(x)`,
`period_num`, `period_length`), it has the meaning given in the
references above. The only new symbol we introduce is the WEBCAT
blinding factor `h_wc` and the keys it derives.

See [Enrollment](enrollment.md) for the surrounding chain machinery; this
document specifies only the onion-specific differences.

## 1. Scope and Threat Model

### 1.1 Goals

- Onion privacy. The chain (validators, oracles, observers, browsers
  downloading the snapshot) MUST NOT learn the `.onion` address or its master
  public key `KP_hs_id` of any enrolled service. Only the operator and clients
  who already know the `.onion` learn the binding.
- List correctness. Only the operator of a given `.onion` can produce an
  enrollment for it. The chain MUST be able to reject submissions whose
  WEBCAT key claim is not bound to a currently descriptor-published onion
  service key.
- Descriptor presence. Enrollment requires that a v3 descriptor for
  the same `.onion` is currently published on the appropriate HSDirs
  for the current Tor period. This does not prove the service
  answers connections; it only proves that someone holding
  `KS_hs_blind` (and therefore `KS_hs_id`) has recently uploaded a
  descriptor.
- Spam mitigation. Submitters MUST hold `KS_hs_id`; the cost of
  generating the zero-knowledge proof and maintaining a published descriptor
  acts as the rate limit. Generating a proof already takes ~8 minutes
  of CPU time and several GB of RAM (Section 4.2), which is a
  meaningful rate limit on its own. If empirically this turns out to
  be insufficient, an explicit proof-of-work or stake-based
  mechanism can be layered on top of the submission without changing
  the rest of the protocol.
- Efficient lookup. Browsers that already know a `.onion` can find the
  matching enrollment in `O(1)` from the address alone, without scanning the
  list.
- Monitorability and auditability. Anyone who knows a `.onion` can
  monitor its enrollment by deriving `KP_wc_blind` locally and watching
  the chain entry at that key (changes, blocks, expiries). Anyone —
  regardless of which `.onion`s they know — can audit the chain
  globally by re-verifying every stored proof and signature against
  the audit log (Section 6.2).

### 1.2 Non-goals

- Recovering from a compromised master onion key. As with vanilla Tor v3,
  once `KS_hs_id` leaks there is no key rotation — the only available action
  is to block (Section 5.3).
- Defending against an actively malicious Tor network. We rely on Tor's
  HSDir consensus only to attest that the descriptor was uploaded.
- Proving the onion service is reachable. A descriptor on the HSDirs
  is necessary for users to *find* the service but does not prove the
  service itself answers connections.

### 1.3 Trust split

The construction layers two cryptographic claims:

| Claim | Mechanism | Failure mode |
|---|---|---|
| Operator owns `KP_hs_id` | Ed25519 signatures with both `KS_wc_blind` and `KS_hs_blind` over the submission (Section 3.2) | Classical-crypto break (Ed25519, SHA3). |
| WEBCAT blinded key is bound to the same `KP_hs_id` as the Tor blinded key | Zero-knowledge proof (Section 4) | ZK-proof soundness break would allow an attacker holding *some* Ed25519 keys to enroll those as disjoint WEBCAT entries the chain accepts as if they belonged to a single `.onion`. Because the double signature still pins each accepted bundle to keys the submitter actually owns, the damage is bounded to list integrity (and spam mitigation, as one could reuse the same descriptor once per day). |

*Note: the second claim is allowed to fail more gracefully than the first. A ZK soundness break does not let the attacker forge `.onion` takeovers, because the per-submission double signature still binds the bundle to keys the attacker holds.*

## 2. Cryptographic Primitives

We use SHA3-256 (denoted `H`) and Ed25519 / Curve25519 as in Tor v3.
`L` is the order of Curve25519's prime-order subgroup, `||` denotes
byte concatenation, and `INT_n(x)` is the `n`-byte big-endian encoding
of `x`.

For an Ed25519 master keypair `(KS_hs_id, KP_hs_id)`, two blinding
factors are used.

Tor's blinding factor `h_tor` (reproduced for readability; the
authoritative definition is
[`rend-spec/keyblinding-scheme`](https://spec.torproject.org/rend-spec/keyblinding-scheme.html)):

```
h_tor = clamp(H(BLIND_STRING || KP_hs_id || s || B || N))

BLIND_STRING = "Derive temporary signing key" || INT_1(0)
N            = "key-blind" || INT_8(period_num) || INT_8(period_length)
B            = the Ed25519 base point as the textual ASCII decimal
               tuple "(x_decimal, y_decimal)" — 158 bytes including
               the parentheses, comma, and space. NOT the canonical
               32-byte compressed encoding. This matches Tor's
               reference implementation (`little-t-tor`).
s            = optional secret; empty for plain v3 onion services
clamp        = standard Ed25519 scalar clamping (bit 0 cleared,
               bit 254 set, bit 255 cleared)
```

`h_tor` rotates with the Tor period and matches what Tor itself uses
to derive `KP_hs_blind`.

WEBCAT's blinding factor `h_wc`:

```
h_wc = clamp(H("webcat-blind-v1:" || KP_hs_id))
```

`h_wc` is independent of the Tor period. The same Ed25519 scalar
clamping applied to `h_tor` (RFC 8032 §5.1.5: bits 0, 1, 2 cleared;
bit 254 set; bit 255 cleared) is also applied here. Clamping is not
required by any external protocol — Tor does not specify `h_wc` — but
it gives the WEBCAT side the same cofactor-safety guarantee that
Tor's blinded keys enjoy, so that any plaintext computation of
`KP_wc_blind = [h_wc] · KP_hs_id` (notably the browser-side lookup
derivation in §3.4) is safe against torsion-tainted inputs without
having to perform its own subgroup check.

From the two blinding factors:

```
KP_wc_blind  = [h_wc]  · KP_hs_id     (WEBCAT lookup key, static per .onion)
KS_wc_blind  = [h_wc]  · KS_hs_id     (held only by operator; signs the policy)
KP_hs_blind  = [h_tor] · KP_hs_id     (Tor blinded key for the current period;
                                       already used by Tor for descriptors)
KS_hs_blind  = [h_tor] · KS_hs_id     (held only by operator; already used by
                                       Tor to sign descriptors)
```

Both signing keys are listed here so the WEBCAT / Tor sides are
symmetric — each blinded keypair `(KP_*_blind, KS_*_blind)` has a
public half (visible to the chain and to anyone who knows the
`.onion`, respectively) and a secret half held only by the operator.
The Tor blinded keypair is the same one Tor already maintains; the
WEBCAT blinded keypair is new and is what the operator uses to sign
the enrollment policy. The precise Ed25519 derivation of the secret
halves follows [`rend-spec/keyblinding-scheme`](https://spec.torproject.org/rend-spec/keyblinding-scheme.html);
the `[h] · K` notation here is shorthand for the relation that holds
between the public keys.

Scalars are reduced mod `L` implicitly when multiplying a prime-order
point.

## 3. Enrollment Submission

### 3.1 Operator-side flow

1. Operator publishes the policy at `https://<.onion>/.well-known/webcat/enrollment.json`
   following the clearnet [Server](server.md) format.
2. Operator derives `KP_wc_blind` and `KP_hs_blind` for the current Tor
   period.
3. Operator generates a zero-knowledge proof (Section 4) over the
   statement that both blinded keys derive from the same master key.
4. Operator constructs and signs a `statement` (Section 3.2) and submits it
   off-chain to the oracle set. The oracle set does not fetch the `.onion`
   policy; it validates the cryptographic bundle and descriptor publication
   as described in Section 3.3.

### 3.2 Statement format

```
statement = "webcat-onion-statement-v1:"
          || KP_wc_blind          // 32 bytes, Ed25519 canonical
          || KP_hs_blind          // 32 bytes, Ed25519 canonical
          || INT_8(period_num)
          || INT_8(period_length)
          || policy_hash          // SHA-256 of canonicalized enrollment.json

sig_wc  = Ed25519.sign(KS_wc_blind, statement)
sig_tor = Ed25519.sign(KS_hs_blind, statement)
```

For normal enrollments, renewals, and changes, `policy_hash` is the non-zero
SHA-256 hash of the canonicalized enrollment policy. A block submission uses
the same statement format with `policy_hash = 0x00…00` (32 zero bytes).
The lifecycle kind (`Enroll`, `Renew`, `Change`, or `Block`) is not signed as
a separate field; it is derived deterministically from `policy_hash` and the
current chain state under the rules in Section 5.

Under a sound ZK proof, `sig_wc` alone (combined with the proof)
already establishes that the submitter holds `KS_hs_id`. We require
`sig_tor` as well, however, to preserve a usable security property
even under a ZK soundness break: it forces the submitter to hold the
secret for whichever `KP_hs_blind` they put in the bundle, so a
forger cannot abuse an unrelated victim's descriptor into looking
like a "vouch" for their submission.

### 3.3 Chain-side validation (`CheckTx` / `DeliverTx`)

The chain accepts the submission iff:

- `sig_wc` verifies under `KP_wc_blind` over `statement`.
- `sig_tor` verifies under `KP_hs_blind` over `statement`.
- `period_num` is the current Tor time period (within a small tolerance
  for boundary submissions — see TODO below).
- The zero-knowledge proof verifies under the WEBCAT-onion verifying
  key against the public-input vector flattened from `KP_wc_blind`,
  `KP_hs_blind`, `period_num`, and `period_length`.
- An oracle (analogous to the clearnet oracle set) successfully
  retrieves a v3 descriptor for `KP_hs_blind` from the appropriate
  HSDirs for the current period. The oracle does not decrypt the
  descriptor and does not attempt to connect to the service.

The chain records `policy_hash` but does not fetch the `.onion` policy or
verify that the hash matches the well-known resource. Clients that already
know the `.onion` perform that check when they fetch the policy and manifest.

### 3.4 Lookup

A browser that knows a `.onion` reconstructs the lookup key locally:

```
KP_hs_id    = base32_decode(domain).first(32)
h_wc        = H("webcat-blind-v1:" || KP_hs_id)
KP_wc_blind = [h_wc] · KP_hs_id
```

It then looks `KP_wc_blind` up in the canonical-state subtree of the
snapshot (Section 6.1), checks the Merkle proof against `AppHash` per
the [Enrollment](enrollment.md) §Light Client Verification flow, and
reads the `policy_hash`. That's enough to fetch and verify the
manifest. The chain never sees the `.onion`.

The full submission bundle (ZK proof, signatures, `KP_hs_blind`) is
not needed for this lookup; it lives in a separate audit-log
subtree (Section 6.2) that browsers do not download.

## 4. Zero-Knowledge Proof

The proof asserts knowledge of `KP_hs_id` such that, given the public
inputs `(KP_wc_blind, KP_hs_blind, period_num, period_length)`:

1. There exists an Ed25519 point `pk_point` whose canonical 32-byte
   encoding is `pk_bytes`,
2. `pk_point` lies on the Ed25519 curve and in its prime-order subgroup,
3. `KP_wc_blind = [h_wc(pk_bytes)] · pk_point`,
4. `KP_hs_blind = [h_tor(pk_bytes, period_num, period_length)] · pk_point`,

where `h_wc` and `h_tor` are the blinding-factor derivations from §2.

### 4.1 Inputs

- Public inputs: `KP_wc_blind`, `KP_hs_blind`, `period_num`,
  `period_length`. Validators, oracles, third-party
  re-verifiers can check the proof from these values alone.
- Private witness: `KP_hs_id`.

This document specifies only the relation above and its public/private
input split. The chain commits to a concrete proof system, parameters,
and verifying key as part of its consensus-relevant trust material so
that every validator and every external auditor reaches the same
accept/reject decision for a given submission. Choice of proof system
is out of scope for this spec; see §4.2 for what has been measured to
be feasible.

### 4.2 Prototype benchmarks

A reference prototype implements this relation with an in-circuit
SHA3-256 and Ed25519 gadget. Measurements on entry level Apple Silicon
hardware are reproduced here only as evidence that the relation is
practical to prove and cheap to verify.

| | value |
|---|---|
| Trusted setup (one-time) | ~8 min, ~7 GB peak RAM |
| Proving key | ~3.0 GB |
| Verifying key | ~3.7 KB |
| Prover time | ~7–8 min, ~6 GB peak RAM |
| Verifier time | ~80 ms |
| Proof size (compressed) | 192 B |
| Public input vector | 70 field scalars |

Production deployment selects, fixes, and audits a concrete proof
system separately; the figures above should be expected to shift with
that choice. Choice of proof system is part of that selection, see §7.

### 4.3 Failure consequences

If the proof system is sound, the chain's invariant *"one chain entry =
one master onion key"* holds.

If the proof is broken (a soundness break in the proof gadget),
an attacker who controls *some* set of Ed25519 keypairs can enroll
those as ghost entries: a random `KP_wc_blind` paired with a
`KP_hs_blind` from an `.onion` the attacker actually controls. They
cannot impersonate any *specific* operator's service (the attacker
cannot sign `sig_wc` under a victim's `KP_wc_blind`), and they cannot
abuse a third party's descriptor as a vouch (the attacker cannot
sign `sig_tor` under a victim's `KP_hs_blind`).

Because the chain stores the full cryptographic submission artifacts
(Section 6), an operator who already knows their own `.onion` can
locally compute `KP_wc_blind` and inspect the canonical state at that
key. Any third party can also re-verify every proof and signature
against the chain's stored data. This makes validator misbehavior
publicly detectable, even if it cannot detect a soundness break in the
proof gadget itself.

The HSDir-descriptor observation is not part of the canonical record
and cannot be replayed later (descriptors rotate with the Tor period
and HSDirs do not retain old ones); it is checked once at submission
time by the oracle set.

## 5. Lifecycle Policies

### 5.1 Expiry, renewal, and re-enrollment

Every accepted submission sets the record's `expiry` to 6 months
from the submission time (in the case of a pending change, from its
promotion time — see §5.2). A record whose `expiry` has passed and
which is not protected by either a pending change (§5.2) or an active
block (§5.3) is purged at the next `EndBlock`.

- Renewal. A submission whose `policy_hash` equals the current
  canonical `policy_hash` is accepted immediately and only
  refreshes `expiry`. There is no observation period: the operator
  can renew at any time before expiry.
- Re-enrollment. A submission for a `KP_wc_blind` not currently
  present in canonical state, because it was never enrolled, was
  purged after expiry, or was purged after a 6-month block (§5.3) —
  is treated as a first enrollment. Accepted immediately;
  `expiry := submission_time + 6 months`.

Because `KP_wc_blind` is stable per `.onion`, the chain detects
renewals (same lookup key, new submission) without learning the
underlying `.onion`. Renewals and re-enrollments each cost a fresh
zero-knowledge proof and signed statement.

### 5.2 Policy change

- A change submission is identified by a non-zero `policy_hash` that
  differs from the current canonical `policy_hash` for an existing
  `KP_wc_blind`.
- The change enters a pending state and remains pending for a
  1-week observation period before promotion to canonical state.
  During the wait, the previous canonical policy remains in effect.
- On promotion, the canonical `policy_hash` is replaced and the
  record's `expiry` is set to `promotion_time + 6 months`.
- The pending entry is publicly observable, so monitors of the chain
  can detect a change attempt against a domain they care about
  (§5.5).
- If a second change submission arrives while a pending change is
  waiting, it overwrites the pending change and resets the 1-week
  timer. The only way to interrupt this loop is the escape hatch
  (§5.3).
- Pending preserves the record. While a pending change exists,
  the canonical record's nominal `expiry` is *ignored* for the
  purpose of purging: the record is held alive until the pending
  change either promotes (which sets a fresh `expiry`) or is replaced
  by a block (§5.3).

### 5.3 Compromise block

- The operator can submit a block by signing the normal statement format
  with `policy_hash = 0x00…00` (32 zero bytes). A block immediately sets the
  canonical record's policy to that reserved sentinel value and locks it for
  6 months.
- While blocked, any submission for that `KP_wc_blind` with a
  non-zero `policy_hash` is rejected. There is no path out of a
  block other than waiting for its 6-month expiry. A fresh block
  submission (`policy_hash = 0`) is treated as a renewal under §5.1
  and refreshes the 6-month expiry, useful when an operator wants to
  keep a `KP_wc_blind` indefinitely blocked after a confirmed key
  compromise.
- After the block's `expiry` passes, the record is purged on the
  same schedule as any other expired record (§5.1). The `KP_wc_blind`
  can then be enrolled again (presumably by a new keypair, since a
  key compromise was the trigger). Keeping the purge schedule uniform
  prevents the canonical list from growing unboundedly.
- A block submission ends any pending change (§5.2) immediately and
  replaces canonical state with the block. This is the only mechanism
  that ends the pending-change cycle.

### 5.4 Race semantics

The lifecycle reflects the fact that once an Ed25519 master key is
compromised, there is no recovery, as every action a legitimate operator
can take, the attacker can also take. We therefore design the lifecycle
as a one-way race in which the legitimate operator's only winning move
is the escape hatch:

- The legitimate operator and an attacker holding the same key can both
  submit changes; the change with the most recent successful 1-week
  observation wins.
- The block escape hatch takes precedence over any pending change.
- Once a key is blocked, the legitimate operator MUST migrate to a new
  `.onion` (and therefore a new `KP_hs_id`).

### 5.5 Detection surface

Because `KP_wc_blind` is stable per `.onion`, the chain provides the
following observable events to anyone watching:

- A new enrollment at a `KP_wc_blind` not previously seen (first
  enrollment, or re-enrollment after a 6-month block expired).
- A renewal at an existing `KP_wc_blind` (same key, refreshed expiry).
- A pending change at an existing `KP_wc_blind` (new `policy_hash`,
  1-week timer running).
- A block at an existing `KP_wc_blind`.

The chain never reveals the `.onion`; clients who already know the
`.onion` map it to its `KP_wc_blind` locally and check the chain's
state for that lookup key.

## 6. Chain Storage

State is split into two Merkle subtrees, both committed under `AppHash`,
serving two different audiences:

| | canonical state (§6.1) | audit log (§6.2) |
|---|---|---|
| Audience | Browsers, lookup clients | Monitors, third-party auditors |
| Key | `KP_wc_blind` (32 B) | `(KP_wc_blind, block_height)` |
| Value | `policy_hash` (32 B) | Full submission bundle |
| Mutability | Overwritten on change; purged at expiry | Append-only |
| Per leaf | 64 B | ~400 B |

The split lets browsers download a small canonical snapshot for normal
lookups, while preserving every cryptographic artifact ever accepted
for independent re-verification.

### 6.1 Canonical-state subtree

Keyed by `KP_wc_blind`. Holds exactly one value:

```
KP_wc_blind  →  policy_hash    (32 B)
```

That is the entire browser-visible record.

### 6.2 Audit-log subtree

Append-only. Keyed by `(KP_wc_blind, block_height)` so every accepted
submission for every onion has a unique entry. Holds the full
cryptographic bundle:

```
KP_hs_blind        → 32 B   (period-specific Tor blinded key)
period_num         → 8 B
sig_wc             → 64 B   (Ed25519 under KS_wc_blind,  over statement)
sig_tor            → 64 B   (Ed25519 under KS_hs_blind,  over statement)
zk_proof           → ~192 B (compressed; concrete size depends on §4.2)
policy_hash        → 32 B   (the policy this submission attested)
```

`period_length` is the fixed Tor period length and is not stored per
submission.

The audit log is not purged when the corresponding canonical
record is purged. The canonical record may disappear at expiry, but
its history of accepted submissions remains in the audit log for as
long as the chain retains it (Section 6.4).
