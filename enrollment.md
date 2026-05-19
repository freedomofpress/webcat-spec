# WEBCAT Enrollment Specification

This document explains how domains enroll into WEBCAT and how validators and oracles validate enrollment requests.

The enrollment data are processed and stored in a permissioned CometBFT chain, where the voting network consists of trusted partner orgs (other nonprofits, Tor relay associations, browsers) acting as validators in the CometBFT consensus. Any third party can operate an observer node that allows them to check consistency, behavior, or monitor entries and updates.

TODO: Add description of onion service enrollment, which must be done privately.

# Background

[CometBFT](https://docs.cometbft.com/v1.0/) is a Byzantine Fault Tolerant (BFT) consensus algorithm derived from [Tendermint](https://arxiv.org/abs/1807.04938). It provides a round-based protocol that will reach consensus as long as at least $\gt 2/3$ of the validator voting power is online and honest. The benefit of using CometBFT is that consensus and the networking parts of the chain are handled by an external dependency.

The blockchain state machine runs as an [ABCI](https://docs.cometbft.com/v1.0/) application. The consensus algorithm orders transactions into blocks, and the ABCI application executes them into the application state.

The ABCI interface consists of:
- `CheckTx`: basic validity checks run on transactions prior to them being added to the mempool
- `DeliverTx`: validity checks that run when the transaction is included in a block
- `BeginBlock`/`EndBlock`: hooks to run logic that runs at the beginning and end of a block boundary respectively
- `Commit`: finalizes the application state for that block

After executing all transactions in a block, the Merkleized application state root, the AppHash, is committed into the block header and signed by validators. This will be used by WEBCAT as a verifiable timestamped commitment.

## Timestamp

The CometBFT `AppHash` is a timestamped commitment that serves as a verifiable timestamp to ensure that manifests cannot be signed with future dates.

Each block produced by the enrollment network has a block height (a monatonic counter enforced by consensus), the corresponding `AppHash`, and a consensus timestamp. This data
is included in a verifiable form in the CometBFT [`LightBlock`](https://docs.cometbft.com/v0.34/spec/core/data_structures#lightblock) which contains a list of signatures from the validator set, and a header that contains the `AppHash` and the timestamp. By signing a manifest that includes the `LightBlock`, the manifest signer demonstrates that the manifest could not have existed before that block was finalized. Periodically a CDN will publish the latest `LightBlock`, and clients verify its validity using the signatures provided.

Older `LightBlock` values can be reused in manifests, allowing for backdating. This is acceptable because the security goal is to prevent future-dating, wherein a manifest would be valid longer than it should be.

The chain itself does not enforce expiry; clients check the expiry using the consensus timestamp included in the manifest's `LightBlock`.

### CDN Publishing

The latest [`LightBlock`](https://docs.cometbft.com/v0.38/spec/core/data_structures#lightblock) consensus data is published to a CDN hourly.

The `LightBlock` consists of:

* `Header`: contains the `AppHash`, `ValidatorsHash`, consensus timestamp, block height, and other chain metadata such as the `ChainID`
* `Commit`: contains the `BlockID` (which includes the hash of the header) and a list of `CommitSig`s — each being a validator's signature over a `CanonicalVote` for that block
* `ValidatorSet`: the set of validators and their voting power at that height

See "Light Client Verification" for more detail on the verification performed of the CDN data.

## Enrollment

Chain parameters (configured in the chain's `OracleConfig`):
- `max_enrolled_subdomains`: the maximum number of subdomains per registered domain to rate limit abuse
- `voting_config.quorum`: the number of oracle votes required to reach consensus on an observation
- `voting_config.total`: the total number of authorized oracles
- `voting_config.delay`: the time delay that must elapse before a pending enrollment change is applied to canonical state (this is the "cooldown" period)
- `voting_config.timeout`: the time after which individual votes expire if quorum is not reached
- `observation_timeout`: the maximum age of a blockstamp in an observation transaction

### Enrollment Flow

The enrollment process consists of:

1. Domain owner publishes the domain's enrollment policy (see `server.md`) at:

```
https://<domain>/.well-known/webcat/enrollment.json
```

2. Domain owner (or a frontend acting on their behalf) submits an enrollment request off-chain to the oracle set. This request instructs oracles to observe the domain's enrollment file.

3. Each oracle independently:
   - Fetches the enrollment file from the domain
   - Validates the file and computes its canonical hash
   - Posts an `Observe` transaction to the chain

4. The chain processes observations through a voting mechanism (see below).

To unenroll, domain owners remove the enrollment JSON file, and oracles verify that the path produces a 404 or 410 status code.

### Oracle Observation Process

When an oracle receives an off-chain enrollment request, it:

1. Fetches:

```
https://<domain>/.well-known/webcat/enrollment.json
```

2. Validates:
- The HTTPS certificate is valid (hostname matches, chain trusted, not expired/revoked)
- The response is either:
    - A correctly formatted policy JSON (HTTP 200)
    - A 404 (Not Found) or 410 (Gone) for unenrollment

3. Computes:
- If the file exists: canonical hash of the enrollment JSON (SHA-256 of canonicalized JSON)
- If 404/410: a `NotFound` marker

4. Gets the latest block information from the chain (block height and `app_hash`)

5. Creates and signs an `Observe` transaction containing the observation

6. Broadcasts the signed transaction to the chain

#### Structure

The `Observe` action contains:
- `oracle`: the oracle's identity (ECDSA P-256 public key)
- `observation`:
  - `domain`: the domain being observed (must be a strict subdomain of `zone`)
  - `zone`: the zone (registered domain) under which the domain is being enrolled
  - `hash_observed`: either a SHA-256 hash of the canonicalized enrollment JSON, or `NotFound` for 404/410
  - `blockstamp`:
    - `block_height`: the block height at which the observation was made
    - `app_hash`: the application hash from that block
- `signature`: ECDSA P-256 signature of the transaction

#### `CheckTx`/`DeliverTx`

When processing an `Observe` transaction, the chain validates:

- The oracle is authorized (their public key is in the current `OracleConfig`)
- The blockstamp is not in the future
- The blockstamp is not too old (within `observation_timeout`)
- The blockstamp's `app_hash` matches the recorded `app_hash` for that block height
- The observed domain is a strict subdomain of the specified zone
- If enrolling a new subdomain: the number of subdomains under the registered domain would not exceed `max_enrolled_subdomains`
- If unenrolling: the subdomain exists in canonical state or is pending enrollment

If validation passes, the observation is added to the oracle voting queue as a vote.

#### Voting Mechanism

The chain uses a vote queue to reach consensus on enrollment changes:

1. Each oracle observation is cast as a vote in the voting queue, keyed by subdomain where the value is the observed SHA256 hash.

2. Votes accumulate until a quorum of oracles (configured via chain parameter `voting_config.quorum`) vote for the same hash value for a given subdomain.

3. Once quorum is reached, the winning hash is moved to a pending queue with a timestamp. The pending change waits for a configured cooldown delay period (`voting_config.delay`) before being applied.

4. If a new value for the same subdomain reaches quorum while a pending change is waiting, the new value overwrites the existing pending change and the cooldown delay timer resets. However, if the new value is identical to the existing pending value, the timer is not reset (to prevent DoS attacks that prevent enrollment).

5. Individual votes expire after `voting_config.timeout` if quorum is not reached.

#### `EndBlock`

At the end of each block:

1. We remove votes that have exceeded the timeout period.

2. For any pending enrollment changes where the delay period has elapsed, we promote them to canonical state by updating the canonical hash for that subdomain.

3. Similarly process any pending configuration changes from admin voting.

## Snapshot

The snapshot is:
- a serialization of the canonical enrollment subtree of the application state
- a merkle proof from the root of the canonical enrollment subtree to the AppHash

TODO: Include back of the envelope numbers

The snapshot is deterministic: by using the state at a particular block height, all full nodes will produce identical snapshots.

### Light Client Verification

The snapshot enables users to verify inclusion of a domain using the snapshot, Merkle proof, and latest `LightBlock`. Clients must:
1. Verify the `ValidatorSet` on the `LightBlock` matches the validator set hardcoded in the extension.
2. Verify `Hash(ValidatorSet) == header.ValidatorsHash` to confirm the validator set used for signature verification is the one committed to in the header.
3. Compute `header_hash = MerkleRoot(header)` using  [CometBFT's (Merkle) header hashing scheme](https://github.com/cometbft/cometbft/blob/main/spec/core/data_structures.md#header).
4. Verify `header_hash == commit.BlockID.Hash`, binding the header (and therefore the `AppHash`) to the `BlockID` that validators signed over.
5. For each `CommitSig` with `BlockIDFlag == BLOCK_ID_FLAG_COMMIT`, reconstruct the `CanonicalVote` (using the commit's `Height`, `Round`, `BlockID`, the signature's `Timestamp`, and the `ChainID`) and verify the signature against the corresponding validator's public key.
6. Confirm the signing validators represent >2/3 of total voting power.
7. Extract the `AppHash` from the now authenticated header.
8. Reconstruct the Merklized canonical enrollment subtree from the provided serialized leaves, computing the canonical enrollment root hash.
9. Verify the Merkle proof that demonstrates the canonical enrollment root hash is included in the application state tree, whose root is the `AppHash` from step 7.

Operationally, we scrape the state (`LightBlock`, leaves of canonical state, and merkle proof of inclusion) from a node and push it to a CDN through a serverless cron job.

Optionally, we could compute and publish incremental diffs relative to previous snapshots to reduce bandwidth usage.

## Appendix: ENS Enrollment

WEBCAT supports enrollment of websites accessed through ENS names whose `contenthash` record resolves to immutable content-addressed storage (e.g. IPFS). Only the oracle's resolution path differs from the clearnet enrollment process.

### Enrollment Flow

1. The site author publishes site content to IPFS such that the enrollment policy is reachable at `<CID>/.well-known/webcat/enrollment.json`.
2. The site author sets the ENS `contenthash` record for their name (e.g. `example.eth`) to that CID.
3. The site author submits an enrollment request off-chain to the oracle set, referencing the ENS name.
4. Oracles independently perform the ENS-specific observation process described below.
5. The chain processes observations through the existing voting and cooldown mechanism unchanged.

The value committed to the chain is the canonical hash of `enrollment.json`, not the ENS `contenthash`. Subsequent site content changes do not require re-enrollment. Re-enrollment is only required when the enrollment policy itself changes, such as signer rotation, at which point the new policy must clear the existing cooldown period before clients honor it.

To unenroll, the site author publishes a new CID whose content does not include `.well-known/webcat/enrollment.json` and updates the ENS `contenthash` to point to it. Oracles then submit a `NotFound` observation exactly as for clearnet unenrollment.

### Oracle Observation

For an ENS observation, the oracle:

1. Resolves the ENS `contenthash` record via its own Ethereum RPC provider, yielding a content-addressed identifier (e.g. an IPFS CID).
2. Fetches `<CID>/.well-known/webcat/enrollment.json` via IPFS. The oracle MAY use a local IPFS node or a trusted gateway.
3. Validates and canonicalizes the response exactly as for the clearnet flow.
4. Submits an `Observe` transaction for the ENS name.

### Client Verification

Enrollment can be verified against the committed snapshot only, clients MUST NOT make an additional network round trip during verification.