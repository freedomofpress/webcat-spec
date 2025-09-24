# WEBCAT Enrollment Specification

This document explains how domains enroll into WEBCAT and how validators and oracles validate enrollment requests.

The enrollment data are processed and stored in a permissioned CometBFT chain, where the voting network consists of trusted partner orgs (other nonprofits, Tor relay associations, browsers) acting as validators in the CometBFT consensus. Any third party can operate an observer node that allows them to check consistency, behavior, or monitor entries and updates.

# Background

[CometBFT](https://docs.cometbft.com/v1.0/) is a Byzantine Fault Tolerant (BFT) consensus algorithm derived from [Tendermint](https://arxiv.org/abs/1807.04938). It provides a round-based protocol that will reach consensus as long as at least $\gt 2/3$ of the validator voting power is online and honest. The benefit of using CometBFT is that consensus and the networking parts of the chain are handled by an external dependency.

The blockchain state machine runs as an ABCI application. The consensus algorithm orders transactions into blocks, and the ABCI application executes them into the application state.

The ABCI interface consists of:
- `CheckTx`: basic validity checks run on transactions prior to them being added to the mempool
- `DeliverTx`: validity checks that run when the transaction is included in a block
- `BeginBlock`/`EndBlock`: hooks to run logic that runs at the beginning and end of a block boundary respectively
- `Commit`: finalizes the application state for that block

After executing all transactions in a block, the Merkleized application state root, the AppHash, is committed into the block header and signed by validators. This will be used by WEBCAT as a verifiable timestamped commitment.

## Timestamp

The CometBFT AppHash is a timestamped commitment, so it can serve as the timestamp to ensure that manifests cannot be signed with future dates.

Each block produced by the enrollment network has a block height (a monatonic counter enforced by consensus), the corresponding AppHash, and a consensus timestamp. By signing a manifest that includes `(AppHash, height, timestamp)`, the manifest signer demonstrates that the manifest could not have existed before that block was finalized.

Previous proofs (older AppHash values) can be reused in manifests, allowing for backdating. This is acceptable because the security goal is to prevent future-dating, wherein a manifest would be valid longer than it should be.

The chain itself does not enforce expiry, clients check the expiry using the consensus timestamp.

The consensus data will be published to a CDN with the following structure:

```json
{
  "height": 12345,
  "app_hash": "abc123def456...",
  "timestamp": 1640995200,
  "block_time": "2025-05-01T00:00:00Z"
}
```

This allows clients to verify the `AppHash` against the chain and check manifest expiry using the consensus timestamp.

## Enrollment

Chain parameters:
- `MAX_NUMBER_SUBDOMAINS`: the maximum number of subdomains per domain to rate limit abuse.
- `NUM_REQUIRED_ORACLES`: the number of oracle observations that must post a response to the chain
- `COOLDOWN_BLOCKS`: the number of blocks that must elapse before the domain is allowed to move from the pending set to the active set.
- `TIMEOUT_BLOCKS`: if insufficient oracle observations arrive within this number of blocks, the enrollment request expires.

### `EnrollmentRequest`

Invariants:
- Non-concurrent: There cannot be multiple enrollment requests for a domain.

The enrollment process involves:
1. User places `enrollment.json` at:

```
https://<domain>/.well-known/webcat/enrollment.json
```

2. User submits an `EnrollmentRequest` to the chain.
3. Transaction validation (`CheckTx`/`DeliverTx`)

To unenroll, clients must remove the enrollment JSON and oracles verify that path produces a 404 or 410.

#### Structure

An `EnrollmentRequest` contains (TODO):
- domain:
- subdomains:
- `signers`: a list of Ed25519 public keys

#### `CheckTx`/`DeliverTx`

We validate:
- The maximum number of subdomains per domain has not been reached.

The pending set tracks domains awaiting oracle observations. Each entry contains:

- `domain`: the domain name
- `request_height`: the block height when the request was submitted
- `enrollment_data`: the signing data from the enrollment request
- `oracle_observations`: oracle responses received so far (TODO do we need this?)

Upon a new validated enrollment request:
- If no pending request exists for the domain, we add it to the pending set.
- If a pending request already exists for that domain, we remove the existing pending entry, add a new entry with the current block height as `request_height` with `oracle_observations` set to empty.

Note that a domain can be in the active set and there can be an update in the pending set.

#### `EndBlock`

We iterate through all new oracle observations in the block and:
- Add the observation to the appropriate entry in the pending set.

We iterate through all domains in the pending set and verify:
- At least `NUM_REQUIRED_ORACLES` matching observations have been submitted, i.e. all responses agree on the `response_hash` provided in the oracle observation
- `COOLDOWN_BLOCKS` have elapsed since the request height
- `TIMEOUT_BLOCKS` has not yet elapsed

If both checks pass, we promote the domain from the pending to active set, i.e. removing from the pending set, and adding to the active set.

### `OracleObservation`

For each new domain in the pending set, oracles:

1. Fetch:

```
https://<domain>/.well-known/webcat/enrollment.json
```

2. Validate:
- The HTTPS certificate is valid (hostname matches, chain trusted, not expired/revoked).
- The response is either:
    - A correctly formatted policy JSON
    - A 404 or 410 (Gone) for unenrollment

3. Returns:
- Hash of the file (or a marker for 404/410)
- Success/Failure

4. Post observation to the chain.

#### Structure

The `OracleObservation` should contain:
- `domain`: the domain being observed
- `request_height`: block height of the original enrollment request
- `observation_height`: block height when this observation was made
- `http_status_code`: code returned on fetch, e.g. 200, 404, 410
- `response_hash`: SHA256 hash of the enrollment.json file, or Null if 404/410 or error occurs
- `oracle_id`: public key of the oracle making this observation
- `signature`: Ed25519 signature of the observation data

## Snapshot

The snapshot is:
- a serialization of the enrollment subtree of the application state
- a merkle proof to the AppHash

TODO: Include back of the envelope numbers

The snapshot is deterministic: by using the state at a particular block height, all full nodes will produce identical snapshots.

The snapshot enables users to verify inclusion of a domain using only the snapshot and Merkle proof.

Operationally, we'll scrape the state from a node and push it to a CDN. This can be done through a serverless cron job.

Optionally, we could compute and publish incremental diffs relative to previous snapshots to reduce bandwidth usage.
