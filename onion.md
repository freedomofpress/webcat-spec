# Onion Service Enrollment

This document specifies the enrollment of Tor v3 onion services into
WEBCAT. The operator signs with a blinded form of the onion's identity key;
the chain verifies an Ed25519 signature and stores a record that only a
party who knows the address can derive or read. Part 1 specifies the proof
and the committed content, Part 2 the record lifecycle, Part 3 admission.
The parts are independent. [Enrollment](enrollment.md) specifies the chain,
canonical state, the snapshot, and the light-client flow. Tor terms are as
in [`rend-spec`](https://spec.torproject.org/rend-spec/). MUST, MUST NOT,
SHOULD, and MAY are as in RFC 2119.

## Part 1. Proof and Content

### 1.1 Privacy

An observer who does not know an address learns nothing about it beyond
the existence of some record. An observer who knows an address can derive
its record, read its committed policy, and observe its history.

### 1.2 Notation

- `H256`, `H512`: SHA-256, SHA-512. `||`: concatenation. `INT_n(x)`:
  `n`-byte big-endian encoding of `x`.
- `B`: the Ed25519 base point. `L`: the order of its prime-order subgroup.
  `p = 2^255 − 19`. Points are encoded per RFC 8032.
- `clamp`: the scalar clamping of RFC 8032 §5.1.5, identical to the X25519
  private-key clamping of RFC 7748.
- `u(Q) = (1 + y) / (1 − y) mod p`: the Montgomery u-coordinate of an
  Edwards point `Q = (x, y)`.

### 1.3 Keys

The operator key is the onion's identity key as stored by Tor in
`hs_ed25519_secret_key`: a 32-byte header followed by

```
s = bytes[0..32]      secret scalar, clamped, little-endian
k = bytes[32..64]     nonce prefix
```

This is the expanded key and MUST NOT be hashed again. Implementations MUST
compute `P = [s]·B` and MUST verify that `P` equals the key encoded in the
address.

```
h   = clamp( H512( "webcat-blind-v1:" || P )[0..32] )
tag = [h] · P                                      signature verification key
key = u(tag)                                       record key
s'  = h · s  mod L                                 blinded signing scalar
k'  = H512( "webcat-blind-nonce-v1:" || k || h )[0..32]
```

The multiplication is over the integer `h` and MUST be computed as
`[8]·[h/8]·P`, exact because `h/8 < L`, so `tag` lies in the prime-order
subgroup for every `P`. Signatures use the RFC 8032 algorithm with
`(s', k')` as the expanded secret key and verify under `tag` with any
Ed25519 verifier. This is Tor's key-blinding scheme
([key blinding](https://spec.torproject.org/rend-spec/keyblinding-scheme.html),
[deriving blinded keys](https://spec.torproject.org/rend-spec/deriving-keys.html))
with a different hash input and no time period.

### 1.4 Submission

The operator publishes the policy at
`http://<address>/.well-known/webcat/enrollment.json` in the format of
[Server](server.md). `policy_hash` is the SHA-256 of the canonicalized
JSON.

```
commitment = H256( "webcat-onion-policy-v1:" || P || policy_hash )     0x00…00 for a tombstone
statement  = "webcat-onion-v1:" || chain_id || tag || commitment || INT_8(height)
sig        = Ed25519.sign( (s', k'), statement )

OnionSubmission {
  tag          32 B
  commitment   32 B
  height        8 B    chain height at which the statement was produced
  sig          64 B
  admission    variable; present when Part 3 requires it
}
```

Only a party who knows `P` can compute or compare a commitment. Such a
party can still confirm policy delegation ([Server](server.md) §3) by
comparing the fetched policy's hash with the delegated clearnet record.

The kind of a submission is determined by state. A `key` is blocked once a
Tombstone has been accepted for it (§2.3).

| State at `key` | `commitment` | Kind |
|---|---|---|
| no record, not blocked | any | Enroll |
| record present | equal to canonical | Renew |
| record present | different, non-zero | Update |
| record present or blocked | zero | Tombstone |
| blocked, no record | non-zero | rejected |

After inclusion the operator MUST derive `key` from the address alone and
verify that the record holds the expected commitment. This is the only
check that detects enrollment at a wrong key.

Validators MUST reject a submission unless all of the following hold:

1. `tag` decodes to a canonically encoded, torsion-free, non-identity
   point.
2. `sig` verifies under `tag` over `statement` with cofactorless
   verification, rejecting non-canonical `S` and small-order `R`.
3. `current_height − onion.freshness_window ≤ height ≤ current_height`.
4. If a record exists at `key`, `height` is greater than its stored
   `height`.
5. If `key` is blocked, the commitment is zero.
6. The rules of Part 2 and Part 3 for the submission's kind hold.

The chain stores the commitment and does not fetch the policy.

### 1.5 Lookup

1. Decode the address per `rend-spec`: base32, 35 bytes, version `0x03`.
   Tor Browser has validated the checksum and rejected torsion-tainted keys.
   The client MUST reject `y ≥ p`, `y = 1`, and `y = p − 1`. Compute `u(P)`.
2. Import `H512("webcat-blind-v1:" || P)[0..32]` as an X25519 private key
   and `u(P)` as an X25519 public key. The derived shared secret is `key`.
3. Locate the leaf at `key` in the onion subtree of the imported snapshot,
   which was verified against `AppHash` at import per
   [Enrollment](enrollment.md).
4. If the commitment is zero, the origin is blocked.
5. Fetch the policy and require
   `H256("webcat-onion-policy-v1:" || P || SHA-256(canonical_json)) == commitment`.

X25519 clamps the private key and multiplies by it as an integer, so step 2
computes `u([h]·P)` exactly. Every step is available in the Web Crypto API.

### 1.6 Storage

Onion records form a canonical subtree under `AppHash`, separate from the
clearnet subtree, with leaves `key → commitment` (64 B). Validators also
keep, per record, the Edwards `tag`, the last accepted `height`, and the
state of Parts 2 and 3, plus the block set of every tombstoned `key`
(32 B each, permanent). This state is not in the snapshot. Submission
history is in the transaction log under `DataHash`.

### 1.7 Security Considerations

- Ownership. A record at `key(P)` requires a signature under `tag(P)`,
  whose secret `h·s` determines `s` because `h` is computable by anyone who
  knows `P` and invertible mod `L`. The property is forgery resistance
  under a public multiplicative related key, the setting of Tor's
  descriptor signing (Hopper, *Proving Security of Tor's Hidden Service
  Identity Blinding Protocol*, 2013). Domain separation keeps the factor
  distinct from Tor's; `k'` keeps nonces independent.
- Derivation. The chain does not verify that `tag` derives from an onion
  key. A record no client derives is inert; Part 3 bounds their number.
- Replay. A signature is valid for one `(chain_id, tag, commitment,
  height)`. Rule 4 rejects older statements while the record exists; after
  a purge rule 3 rejects them, because `freshness_window < record_ttl`.
- Encoding. Rules 1 and 2 remove encoding and signature malleability. A
  torsion-tainted `P` yields a distinct torsion-free `tag`. `tag` and
  `−tag` share a `key` and an owner; two onions share a `key` only if
  `[h(P2)]·P2 = ±[h(P1)]·P1`, a one-bit loss over a hash-collision bound.
- Chain correctness. Clients trust validators for the leaf, as for
  clearnet. The transaction log lets auditors re-verify every record, and
  an operator monitoring its `key` detects a wrong or withheld record.
- Linkability. Commitments differ across records with identical policies.
  The admission path is visible in the submission.

## Part 2. Lifecycle

Durations are chain parameters in blocks. The update cooldown is
`voting_config.delay`, shared with clearnet enrollment.

### 2.1 Expiry and Renewal

An Enroll or Renew sets `expiry = height_accepted + onion.record_ttl`; an
Update sets it at promotion; a Tombstone sets
`expiry = height_accepted + onion.tombstone_ship_ttl`, evaluated against
the parameter's current value at each `EndBlock`. A record past `expiry`
with no pending update is purged at the next `EndBlock`. A submission for a
purged or never-enrolled `key` that is not blocked is an Enroll.

A Renew carries the canonical commitment and refreshes `expiry`. If
`onion.pow_renew_effort` is non-zero, a Renew MUST carry proof of work at
that effort (§3.1).

### 2.2 Update

An Update carries a non-zero commitment different from the canonical one.
It is pending for `voting_config.delay`, during which the canonical
commitment remains in effect; on promotion it replaces the canonical
commitment and sets `expiry`. A later Update with a different commitment
replaces the pending one and restarts the cooldown; one with the same
commitment is a no-op and MUST be rejected at `CheckTx`. A record with a
pending update is not purged.

### 2.3 Tombstone

A Tombstone carries a zero commitment. It immediately replaces the
canonical commitment, ends any pending update, and adds `key` to the block
set permanently: every later non-zero commitment at that `key` MUST be
rejected. An Enroll with a zero commitment blocks a never-enrolled key.

The tombstone leaf is shipped to clients for `onion.tombstone_ship_ttl`
blocks and then purged; the key stays blocked and clients see no record. A
later Tombstone at a blocked key re-creates the leaf. While a leaf is
shipped, a further Tombstone refreshes `expiry` at most once per
`onion.update_window`; any other is a no-op and MUST be rejected at
`CheckTx`. Governance MAY lower `onion.tombstone_ship_ttl` to purge shipped
tombstones; the parameter applies to existing leaves.

After a key compromise every action available to the operator is available
to the attacker. Updates compete and the most recently promoted wins; a
Tombstone overrides any pending update; once tombstoned the operator MUST
migrate to a new address. Operators SHOULD monitor their own `key`.

### 2.4 Update Budget

A record accepts at most `onion.update_budget` renewals and updates per
`onion.update_window` blocks without admission. The window starts at
enrollment and restarts at the first accepted submission after
`onion.update_window` blocks. Beyond the budget a Renew or Update MUST
carry proof of work at the current enrollment effort (§3.1). A Tombstone of
an existing record is not counted and needs no admission.

### 2.5 Parameters

| Parameter | Meaning | Constraint |
|---|---|---|
| `onion.record_ttl` | Record lifetime | > `onion.freshness_window` |
| `onion.freshness_window` | Maximum statement age (§1.4) | |
| `onion.tombstone_ship_ttl` | Blocks a tombstone leaf stays in the snapshot | |
| `onion.update_window`, `onion.update_budget` | Renewals and updates per record per window without admission | |
| `onion.pow_renew_effort` | Proof-of-work effort on every Renew; 0 disables | |

## Part 3. Admission

Admission is required for an Enroll and for renewals and updates beyond the
update budget (§2.4). Admission parameters MUST NOT affect submissions
within the budget or a Tombstone of an existing record. Every admission
proof binds `SHA-256(statement)` and is valid for one submission. Every
path produces the same record. Parameters are set through administrative
voting and take effect at a declared future height.

### 3.1 Proof of Work (REQUIRED)

The puzzle is Equi-X with the effort rule of Tor
[proposal 327](https://spec.torproject.org/hspow-spec/).

```
seed(n)   = block hash at height n · onion.pow_interval
challenge = "webcat-onion-pow-v1" || seed || INT_4(effort) || nonce || SHA-256(statement)
admission = ( seed_height, effort, nonce, solution )
```

`nonce` is 16 bytes and `effort` a 32-bit value, both chosen by the solver.
Validators MUST reject the admission unless `seed_height` is the current or
previous seed boundary, `Equi-X.verify(challenge, solution)` succeeds,
`uint32_be(BLAKE2b-256(challenge || solution)[0..4]) · effort ≤ 2^32 − 1`,
`effort ≥ required(seed_height)`, and fewer than `onion.pow_block_cap`
proof-of-work admissions precede it in the block.

At each seed boundary the chain recomputes, from the admissions of the
interval just closed:

```
required = max( onion.pow_floor,
                clamp( required_prev · admitted / onion.pow_target, required_prev / 4, required_prev · 4 ) )
```

A solution is judged against the requirement at its `seed_height`.
Proposers SHOULD order proof-of-work admissions by declared effort when the
cap binds.

### 3.2 Domain Vouch (OPTIONAL)

A linkable ring signature over the vouch keys of enrolled clearnet domains
proves that one of them vouches, without revealing which. A clearnet policy
MAY carry `onion_vouch_key`, a base64url Ed25519 public key; validators
MUST verify that it is a canonical, torsion-free point when the policy
enters canonical state.

The ring at `ring_height` contains one key per registrable domain: for each
registrable domain (public suffix plus one label, per the Public Suffix
List version `onion.psl_version`) with non-expired enrollments carrying a
vouch key, the key of the enrollment with the fewest labels, then the
lexicographically first FQDN. Members are sorted. `ring_height` MUST be
within `onion.ring_window` of the current height. The signature is CLSAG
(Goodell, Noether, RandomRun, 2019) with a scoped key image:

```
epoch     = current_height / onion.vouch_epoch, or that value minus one
scope     = "webcat-onion-vouch-v1" || INT_8(epoch)
I         = [x] · H_p( scope || X )        signer's key X = [x]·B
message   = chain_id || scope || INT_8(ring_height) || SHA-256(statement)
admission = ( epoch, ring_height, I, signature )
```

`H_p` is RFC 9380 `edwards25519_XMD:SHA-512_ELL2_RO`. Validators MUST
rebuild the ring from state, verify the signature over `message`, reject
the admission if `epoch` is not the current or previous epoch, if `I` is in
the key-image set for `epoch`, or if `onion.vouch_block_cap` vouch
admissions precede it in the block, and record `I` on acceptance. Key-image
sets are kept for the current and previous epoch. Each registrable domain
obtains one admission per epoch, unlinkable across epochs. The security of
the scoped key image is not analyzed in this document.

### 3.3 Invitation (OPTIONAL)

The construction of §3.2 over the Edwards `tag` values of onion records
that are not tombstoned and were enrolled at least `onion.invite_min_age`
blocks before `ring_height`, with signer secret `s'`, period
`onion.invite_period` in place of the epoch, and scope
`"webcat-onion-invite-v1" || INT_8(period)`. Message, checks, and
`onion.invite_block_cap` are as in §3.2. Records can
at most double per `onion.invite_period` after `onion.invite_min_age`,
which SHOULD be at least half of `onion.record_ttl`.

### 3.4 Circuit Breakers

For each path the chain counts admissions over the last
`onion.breaker_window` blocks; above the path's threshold the path is
suspended for `onion.breaker_cooldown` blocks, then re-enabled. Governance
MAY suspend or re-enable a path by parameter change. For the ring paths the
threshold is a fraction of the ring size. Suspension affects only
submissions that require admission.

### 3.5 Parameters

| Parameter | Meaning |
|---|---|
| `onion.pow_enabled`, `onion.vouch_enabled`, `onion.invite_enabled` | Path switches |
| `onion.pow_interval`, `onion.pow_target`, `onion.pow_floor`, `onion.pow_block_cap` | Seed interval, target admissions per interval, minimum effort, per-block cap |
| `onion.vouch_epoch`, `onion.vouch_block_cap`, `onion.ring_window`, `onion.psl_version` | Vouch epoch, per-block cap, maximum ring age, Public Suffix List version |
| `onion.invite_min_age`, `onion.invite_period`, `onion.invite_block_cap` | Inviter minimum age, invitation period, per-block cap |
| `onion.breaker_window`, `onion.breaker_threshold_<path>`, `onion.breaker_cooldown` | Breaker window, thresholds, cooldown |

## Appendix A. Test Vectors

`chain_id` is `webcat-test-1`. `tor_sk` is the payload of
`hs_ed25519_secret_key` after the header, expanded from `seed` as Tor does.

```
seed         = 0101010101010101010101010101010101010101010101010101010101010101
tor_sk       = 58e86efb75fa4e2c410f46e16de9f6acae1a1703528651b69bc176c088bef36e   (s)
               b17a2a2cf3d4a41a8e4e18cc45c8656d558eceddb0adb46bfa088a5f53bed252   (k)
P            = 8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c
address      = rkeohxlubhyzl7ks3mwtzos5olfgocn7dwkbeg7toseadnapn5optfad.onion
u(P)         = 1b1b58dd50ea14b60da17b790cd02754d970c9bab864ebb3c0f3016fe51d3f57
H512(..)[0..32] = e31a2d697481a336b612f25eba39513ac3ba7628a0f96af0c88e70db26c94e7d
h            = e01a2d697481a336b612f25eba39513ac3ba7628a0f96af0c88e70db26c94e7d
s'           = 2e19f8e12f73312c3a23194a14bd71e0aad1323e4b1834c4d89c45180134d40e
k'           = fdce7a16810e4a678039b7890a1670446f41b80a2da8f69bf43ff926b96e53b0
tag          = f73976a9b199453cb26b281a5d37ba2371937e8f217f71d14644f90366776d92
key          = c86bc34a450b892a4ae5c48085043db499210ee7c958fb09403f4453e9e78c67
policy_hash  = 2222222222222222222222222222222222222222222222222222222222222222
commitment   = 9ac1796da0bcdd6ecbba606abaf9bc4f9471cf3ae7f5ff6bf1c598904635e7c9
height       = 110
statement    = 7765626361742d6f6e696f6e2d76313a 7765626361742d746573742d31 <tag> <commitment> 000000000000006e
sig          = e06a7858f5cce82f56edadea3dcc82adbb3b3f0bb0ca31deaaaaba7fd4ee6bc6
               23cc2c4de12f3acecf4a6e1f3480730756e31290943602d453ddfe7b6a09ce08
```
