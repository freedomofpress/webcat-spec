## Server configuration
To participate in the WEBCAT integrity verification system, a website MUST advertise a cryptographic enrollment policy at a well-known path. This policy defines the trust material and the constraints used to validate signatures, transparency proofs, and optionally provenance information.

### 1. Well-Known Enrollment Path

Websites MUST serve their enrollment policy at:

```
https://<domain>/.well-known/webcat/enrollment.json
```

The enrollment policy is discovered asynchronously by clients during the first HTTP request and cached for the duration of the browser session. This approach eliminates the overhead of including policy data in every HTTP response. However, websites have mechanisms to signal that the information MUST be refreshed.


#### 1.1 Field Definitions (Sigsum)
This endpoint MUST return a JSON object with the following structure:

```json
{
  "type": "sigsum",
  "signers": ["<base64url-ed25519-public-key>", "..."],
  "threshold": <integer>,
  "policy": "<base64url-sigsum-policy>",
  "max_age": <integer>,
  "cas_url": "<url>",
  "logs": {
    "<base64url-log-public-key>": "<url>"
  }
}
```

- `type`:
  The enrollment type. For Sigsum enrollments this MUST be set to `"sigsum"`.

- `signers`:
  An array of Ed25519 public keys, base64-encoded. These keys are authorized to sign WebCAT manifest files for the domain.

- `threshold`:
  An integer ≥ 1 indicating the minimum number of distinct valid signatures required to accept a manifest as valid. The value of `threshold` MUST be less than or equal to the number of entries in `signers`.

- `policy`:
  A base64-encoded string representing the compiled Sigsum policy. This includes the list of witnesses, group definitions, and quorum requirements. See [Sigsum Policy Format](#sigsum-policy-format) for details.

- `max_age`:
  An integer representing the maximum number of seconds a manifest may remain valid after its signing timestamp. Since different signatures might have different inclusion times, `max_age` is always counted from the oldest one. The timestamp is verified against the CometBFT chain's AppHash as described in the enrollment specification.

- `cas_url`:
  The base URL of the Content Addressable Storage (CAS) which will be used to verify artifact availability.
  CAS is used by WEBCAT monitors to retrieve:
    - Sigsum leaves required for manifest verification,
    - signed manifest objects,
    - all immutable resources referenced inside the manifest (e.g., WASM binaries, HTML, JS, CSS, auxiliary files).

- `logs`:
  A mapping of Sigsum log public keys (base64url-encoded) to their corresponding log URLs. The enrollment generator includes this mapping, based on the Sigsum trust policy, to help clients locate logs for the purpose of monitoring and auditing.

#### 1.2 Field Definitions (Sigstore)

Sigstore enrollments use the following structure:

```json
{
  "type": "sigstore",
  "trusted_root": { "...": "..." },
  "claims": {
    "2.5.29.17": "alice@example.com",
    "1.3.6.1.4.1.57264.1.8": "https://token.actions.githubusercontent.com"
  },
  "max_age": <integer>
}
```

* `type`:
  The enrollment type. For Sigstore enrollments this MUST be set to `"sigstore"`.

* `trusted_root`:
  A Sigstore trusted root JSON document.

* `claims`:
  A JSON object mapping certificate extension OIDs to their expected string values.

  Each entry represents a constraint that MUST be satisfied by the signing certificate. All claim conditions are evaluated as a logical **AND**. If any claim fails to match, verification fails. OIDs correspond to X.509 certificate extensions. For example:

  * `"2.5.29.17"` — Subject Alternative Name (SAN).
    This matches any SAN entry (e.g., `rfc822Name`, `URI`, or Fulcio `otherName`) whose value exactly equals the expected string.

  * `"1.3.6.1.4.1.57264.1.8"` — Fulcio OIDC Issuer (V2).

  * `"1.3.6.1.4.1.57264.1.5"` — GitHub Workflow Repository.

  This mechanism allows fine-grained validation of supply chain attributes, regardless of whether they are predefined or custom. In a bring-your-own Sigstore deployment, custom certificate extensions can be specified and validated via this mechanism. The semantic meaning of an OID is determined by the issuing certificate authority referenced in `trusted_root`.

  For a list of Fulcio OIDs used by GitHub Actions, see:
  [https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md)

* `max_age`
  An integer representing the maximum number of seconds a manifest may remain valid after its certificate issuance timestamp.

### 2. Policy Transition Mechanism

To transition from one enrollment policy to another, servers MUST follow a strict protocol to ensure uninterrupted verification across all clients.

#### 2.1 Transition Requirements

When initiating a policy change:

- The new policy MUST be served persistently at `/.well-known/webcat/enrollment.json`.
- The previous policy SHOULD be served at `/.well-known/webcat/enrollment-prev.json`.
- The values of the two files MUST differ.

TODO: describe here or in client validation how to signal a refresh for the client.

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
{ "type": "sigsum", "signers": [...], "threshold": 2, ... }

// /.well-known/webcat/enrollment-prev.json
{ "type": "sigstore", "trusted_root": [...], "issuer": "...", ... }
```

In this example, the server is advertising a new policy requiring 2 signers, while still supporting the older 3-signer policy during the transition window.

### 3. Policy Delegation (Optional)

Websites MAY include an additional header to aid user understanding by referencing another domain’s policy:

```
x-webcat-delegation: <delegated-fqdn>
```

This header is not security critical, and thus its integrity does not need guarantees. This header signals to the browsers that the policy of this websites should match the policy of another domain. A practical example could be: cryptpad.org developes and host a flasgship instance of CryptPad. As the CryptPad developers are expected to define the policy for their product and sign accordingly to it, websites who do not fork and change the would want to enforce the same policy. Thus, the policy of cryptpad.collective.it could be the same of cryptpad.org. the `x-webcat-delegation` header provides this information to the browser, and confirm it by performing a local lookup in the preload list. If the header matches, the browser SHOULD expose this information to the end user in a format TBD based on UX considerations. A basic example could be "Verified by cryptpad.org").

TODO: See https://github.com/freedomofpress/webcat-spec/issues/6

### 4. Sigsum Policy Format

The `policy` object defines how clients verify the authenticity of Sigsum checkpoints used in WEBCAT manifest validation. It specifies:

- the witnesses trusted to co-sign log checkpoints,
- how they are grouped,
- how those groups contribute to validation of individual logs, and
- the overall quorum required for manifest acceptance.

> ⚠️ Experimental

We are currently using the compiled Sigsum policy object as described [here](https://git.glasklar.is/sigsum/project/documentation/-/blob/main/archive/2025-09-12-sketch-compiled-policy.md), implemented originally [here](https://git.glasklar.is/nisse/sigsum-c/-/blob/main/tools/sigsum-compile-policy.c?ref_type=heads) and re-implemented by us in [sigsum-ts](https://github.com/freedomofpress/sigsum-ts).

