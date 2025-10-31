## Server configuration
To participate in the WEBCAT integrity verification system, a website MUST advertise a cryptographic enrollment policy via a well-known path. This policy defines the set of trusted signers, the signature threshold, and the Sigsum-based policy used to validate the transparency logging signed manifest updates.

### 1. Well-Known Enrollment Path

Websites MUST serve their enrollment policy at:

```
https://<domain>/.well-known/webcat/enrollment.json
```

This endpoint MUST return a JSON object with the following structure:

```json
{
  "signers": ["<base64-ed25519-public-key>", "..."],
  "threshold": <integer>,
  "policy": "<base64-sigsum-policy>",
  "max_age": <integer>
}
```

The enrollment policy is discovered by clients during the enrollment process and cached for the duration of the domain's enrollment. This approach eliminates the overhead of including policy data in every HTTP response.

#### 1.1 Field Definitions

- `signers`:
  An array of Ed25519 public keys, base64-encoded. These keys are authorized to sign WebCAT manifest files for the domain.

- `threshold`:
  An integer ≥ 1 indicating the minimum number of distinct valid signatures required to accept a manifest as valid. The value of `threshold` MUST be less than or equal to the number of entries in `signers`.

- `policy`:
  A base64-encoded string representing the compiled Sigsum policy. This includes the list of witnesses, group definitions, and quorum requirements. See [Sigsum Policy Format](#sigsum-policy-format) for details.

- `max_age`:
  An integer representing the maximum number of seconds a manifest may remain valid after its signing timestamp. Since different signatures might have different inclusion times, `max_age` is always counted from the oldest one. The timestamp is verified against the CometBFT chain's AppHash as described in the enrollment specification.

### 2. Policy Transition Mechanism

To transition from one enrollment policy to another, servers MUST follow a strict protocol to ensure uninterrupted verification across all clients.

#### 2.1 Transition Requirements

When initiating a policy change:

- The new policy MUST be served persistently at `/.well-known/webcat/enrollment.json`.
- The previous policy SHOULD be served at `/.well-known/webcat/enrollment-prev.json`.
- The values of the two files MUST differ.

#### 2.2 Enrollment Observation Period

Once the new policy is published, it enters a transition period during which enrollment systems monitor the well-known path for consistency and stability.

> ⚠️ **During this period, no clients have switched to the new policy yet.**

To prevent enrollment failure:
- The contents of `/.well-known/webcat/enrollment.json` MUST remain unchanged throughout this period.
- Any modification to the enrollment file before the transition completes will invalidate the attempt.

_TODO: since anybody can submit a request for enrollment or de-enrollment, so can do the infra chain itself. This allows for periodic list clenaups not to clog browsers, to remove expired or abandone domains. It can also backfire in some ways though. We should probably always send an alert to the whois email when a change is initiated._

#### 2.3 Post-Transition Compatibility

After the transition is accepted:
- Some clients will begin enforcing the new policy.
- Others may still enforce the previous one.
- Eventually, all clients will converge on the new state.

To ensure full compatibility throughout this staggered rollout:
- The server SHOULD continue serving `/.well-known/webcat/enrollment-prev.json` until all clients are expected to have adopted the new policy.
- Once adoption is widespread, `/.well-known/webcat/enrollment-prev.json` SHOULD be removed.

#### 2.4 Example

```json
// /.well-known/webcat/enrollment.json
{ "signers": [...], "threshold": 2, ... }

// /.well-known/webcat/enrollment-prev.json
{ "signers": [...], "threshold": 3, ... }
```

In this example, the server is advertising a new policy requiring 2 signers, while still supporting the older 3-signer policy during the transition window.

### 3. Policy Delegation (Optional)

Websites MAY include an additional header to aid user understanding by referencing another domain’s policy:

```
x-webcat-delegation: <delegated-fqdn>
```

This header is not security critical, and thus its integrity does not need guarantees. This header signals to the browsers that the policy of this websites should match the policy of another domain. A practical example could be: cryptpad.org developes and host a flasgship instance of CryptPad. As the CryptPad developers are expected to define the policy for their product and sign accordingly to it, websites who do not fork and change the would want to enforce the same policy. Thus, the policy of cryptpad.collective.it could be the same of cryptpad.org. the `w-webcat-delegation` header provides this information to the browser, and confirm it by performing a local lookup in the preload list. If the header matches, the browser SHOULD expose this information to the end user in a format TBD based on UX considerations. A basic example could be "Verified by cryptpad.org").

TODO: See https://github.com/freedomofpress/webcat-spec/issues/6

### 4. Sigsum Policy Format

The `policy` object defines how clients verify the authenticity of Sigsum checkpoints used in WEBCAT manifest validation. It specifies:

- the witnesses trusted to co-sign log checkpoints,
- how they are grouped,
- how those groups contribute to validation of individual logs, and
- the overall quorum required for manifest acceptance.

> ⚠️ Experimental

We are currently using the compiled Sigsum policy object as described [here](https://git.glasklar.is/sigsum/project/documentation/-/blob/main/archive/2025-09-12-sketch-compiled-policy.md), implemented originally [here](https://git.glasklar.is/nisse/sigsum-c/-/blob/main/tools/sigsum-compile-policy.c?ref_type=heads) and re-implemented by us in [sigsum-ts](https://github.com/freedomofpress/sigsum-ts).

#### 4.2 `witnesses`
This field MUST be a dictionary mapping witness identifiers (strings) to base64-encoded Ed25519 public keys.
All witnesses referenced in `groups` MUST be declared in this section.

#### 4.3 `groups`

This field MUST be a dictionary mapping group names (strings) to quorum rules.
Each quorum rule MUST be an object with exactly one key:
  - The key MUST be a stringified integer `n` (e.g., `"1"`, `"2"`, `"3"`)
  - The value MUST be a list of members (strings)

The integer quorum threshold `n` MUST be:
  - Greater than or equal to 1, and
  - Less than or equal to the number of listed members

The group is considered satisfied if at least `n` of the members are valid in the current context.

Each quorum rule MUST list its members, which MAY be:
  - Witness identifiers (from `witnesses`)
  - Group names (allowing for nested logic)

To ensure deterministic validation and prevent circular references, a group definition MAY ONLY reference:
  - Witnesses declared in the `witnesses` section
  - Groups defined on preceding lines in the `groups` object

This rule establishes an evaluation order over the otherwise unordered JSON object. Servers generating the policy MUST order `groups` top-to-bottom such that all dependencies are satisfied as the document is parsed linearly.

#### 4.4 `logs`

This field MUST be a list of objects, each describing a Sigsum log.
Each log object MUST include:
  - `url`: the base URL of the log
  - `key`: the base64-encoded Ed25519 public key of the log

#### 4.5 `quorum`

- This field MUST be a string referencing a group defined in the `groups` section.
- It defines the global witness quorum that must be satisfied for a log checkpoint to be considered valid.

#### 4.6 Validation Semantics

To successfully validate a manifest under this policy:

- At least one log listed in `logs` MUST present a valid checkpoint.
- That checkpoint MUST be co-signed by a set of witnesses satisfying the global `quorum` group.

If no log provides such a checkpoint, the manifest MUST be rejected.
