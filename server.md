## Server configuration
To participate in the WEBCAT integrity verification system, a website MUST advertise a cryptographic enrollment policy via a custom HTTP response header. This policy defines the set of trusted signers, the signature threshold, and the Sigsum-based policy used to validate the transparency logging signed manifest updates.

### 1. `x-webcat` Header

Websites MUST include the `x-webcat` header in all HTTP responses, including:

- `/index.html` or `/` (used for enrollment discovery)
- All subresources (scripts, workers, etc.)

The value of this header MUST be a JSON object with the following structure:

```json
{
  "signers": ["<base64-ed25519-public-key>", "..."],
  "threshold": <integer>,
  "policy": {
    // See "Sigsum Policy Format"
  },
}
```

_TODO_: Clients check this upon first visit, and then cache the policy. Serving it in all HTTP requests adds significant overhead, and is likely avoidable. We should have a speciifc validation path for enrollment, and then possibly attach this metadata to manifest files, which have to be fetched anyway.

#### 1.1 Field Definitions

- `signers`:  
  An array of Ed25519 public keys, base64-encoded. These keys are authorized to sign WebCAT manifest files for the domain.

- `threshold`:  
  An integer ≥ 1 indicating the minimum number of distinct valid signatures required to accept a manifest as valid. The value of `threshold` MUST be less than or equal to the number of entries in `signers`.

- `policy`:  
  A JSON object specifying the transparency log configuration and the associated witness policy required to validate signed checkpoints. This includes the list of witnesses, group definitions, and quorum requirements. See [Sigsum Policy Format](#sigsum-policy-format) for details.

- `max_age`:  
  An integer representing the maximum number of seconds a manifest may remain valid after its signing timestamp. Since different signatures might have different inclusion times, `max_age` is always counted from the oldest one. _TODO: what do we check this against? We could use the infra chain as a distributed timestamping authority, and requiring the inclusion of a timestamped consensus as proof of time. As we'd have a library to verify consensus anyways for the preload list updates, it should be easy to implement._



### 2. Policy Transition Mechanism

To transition from one enrollment policy to another, servers MUST follow a strict protocol to ensure uninterrupted verification across all clients.

#### 2.1 Transition Requirements

When initiating a policy change:

- The new policy MUST be served persistently in the `x-webcat` header.
- The previous policy SHOULD be served in the `x-webcat-prev` header.
- The values of `x-webcat` and `x-webcat-prev` MUST differ.

#### 2.2 Enrollment Observation Period

Once the new policy is published, it enters a transition period during which enrollment systems monitor the header for consistency and stability.

> ⚠️ **During this period, no clients have switched to the new policy yet.**

To prevent enrollment failure:
- The contents of `x-webcat` MUST remain unchanged throughout this period.
- Any modification to `x-webcat` before the transition completes will invalidate the attempt.

_TODO: since anybody can submit a request for enrollment or de-enrollment, so can do the infra chain itself. This allows for periodic list clenaups not to clog browsers, to remove expired or abandone domains. It can also backfire in some ways though. We should probably always send an alert to the whois email when a change is initiated.

#### 2.3 Post-Transition Compatibility

After the transition is accepted:
- Some clients will begin enforcing the new policy.
- Others may still enforce the previous one.
- Eventually, all clients will converge on the new state.

To ensure full compatibility throughout this staggered rollout:
- The server SHOULD continue serving `x-webcat-prev` until all clients are expected to have adopted the new policy.
- Once adoption is widespread, `x-webcat-prev` SHOULD be removed.

#### 2.4 Example

```http
x-webcat: { "signers": [...], "threshold": 2, ... }
x-webcat-prev: { "signers": [...], "threshold": 3, ... }
```

In this example, the server is advertising a new policy requiring 2 signers, while still supporting the older 3-signer policy during the transition window.

### 3. Policy Delegation (Optional)

Websites MAY include an additional header to aid user understanding by referencing another domain’s policy:

```
x-webcat-delegation: <delegated-fqdn>
```

### 4. Sigsum Policy Format

The `policy` object defines how clients verify the authenticity of Sigsum checkpoints used in WEBCAT manifest validation. It specifies:

- the witnesses trusted to co-sign log checkpoints,
- how they are grouped,
- how those groups contribute to validation of individual logs, and
- the overall quorum required for manifest acceptance.

#### 4.1 Structure

A valid `policy` object MUST include the following top-level keys:

- `witnesses`: mapping of witness IDs to Ed25519 public keys
- `groups`: mapping of group names to quorum rules over witnesses or other groups
- `logs`: a list of Sigsum logs
- `quorum`: the final group that determines whether a signed checkpoint set is trusted

```json
{
  "witnesses": {
    "X1": "base64-key-X1",
    "X2": "base64-key-X2",
    "X3": "base64-key-X3",
    "Y1": "base64-key-Y1",
    "Y2": "base64-key-Y2",
    "Y3": "base64-key-Y3",
    "Z1": "base64-key-Z1"
  },
  "groups": {
    "X-witnesses": {
      "2": ["X1", "X2", "X3"]
    },
    "Y-witnesses": {
      "1": ["Y1", "Y2", "Y3"]
    },
    "Z-witnesses": {
      "1": ["Z1"]
    },
    "XY-majority": {
      "2": ["X-witnesses", "Y-witnesses"]
    },
    "Trusted-Bloc": {
      "1": ["XY-majority", "Z-witnesses"]
    }
  },
  "logs": [
    {
      "base_url": "https://log-a.example.org",
      "public_key": "base64-logkey-A",
    },
    {
      "base_url": "https://log-b.example.org",
      "public_key": "base64-logkey-B"
    }
  ],
  "quorum": "Trusted-Bloc"
}
```

_TODO: JSON isn't great for this, given that in the original format was designed to be ordered and parsed line by line, to resolve groups and witnesses dependencies._

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
