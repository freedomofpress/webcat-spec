## Server configuration
To participate in the WEBCAT integrity verification system, a website MUST advertise a cryptographic enrollment policy at a well-known path. This policy defines the trust material and the constraints used to validate signatures, transparency proofs, and optionally provenance information.

### 1. Well-Known Directory

A website publishes its current enrollment information, and keeps its history, under `https://<domain>/.well-known/webcat/`. Monitors and auditors use the history to find anything ever signed for the domain by its hash.

| Path | Content |
|------|---------|
| `enrollment.json` | The current enrollment information, observed by oracles. |
| `bundle.json` | The current enrollment information together with the signed manifest, fetched by the browser extension. |
| `bundle-prev.json` | The previous enrollment information together with a manifest signed under it, served during a policy change. |
| `enrollment.<hash>.json` | Every enrollment information ever promoted to canonical state on the enrollment chain, including the current one. |
| `manifest.sigsum.<checksum>.json` | For Sigsum enrollments: the signed manifest whose Sigsum leaf has this checksum. |
| `manifest.sigstore.<hash>.json` | For Sigstore enrollments: the signed manifest whose Rekor entry has this artifact hash. |

`<hash>` and `<checksum>` are lowercase hex SHA-256. Files under this directory MUST NOT change or be removed while the domain is enrolled. See [auditing.md](auditing.md).

#### 1.0 Canonical JSON

Every hash in this document is over the [OLPC Canonical JSON](https://web.archive.org/web/20251209150702/http://wiki.laptop.org/go/Canonical_JSON) encoding of the object. Reference implementation: `canonicalize.ts` in the WEBCAT extension.

* `enrollment.<hash>.json`: `<hash>` is the SHA-256 of the canonical enrollment JSON, the same value the chain records.
* `manifest.sigsum.<checksum>.json`: `<checksum>` is the leaf checksum: SHA-256 of the Sigsum message, which is the SHA-256 of the canonical `manifest` object. All signers of one manifest share it.
* `manifest.sigstore.<hash>.json`: `<hash>` is the SHA-256 of the canonical `manifest` object, the artifact digest recorded in the Rekor `hashedrekord` entry.

The content is the `manifest` and `signatures` objects of the bundle, as served. It MUST be published no later than the manifest is first served.

#### 1.1 Enrollment discovery

The browser extension learns the canonical enrollment hash of a domain from the preload list (see [enrollment.md](enrollment.md)) and fetches `bundle.json` on the first request to the domain. The enrollment information in the bundle MUST hash to that value. If it does not, the extension fetches `bundle-prev.json` and applies the same check. The bundle is cached for the duration of the browser session.


#### 1.2 Field Definitions (Sigsum)
This endpoint MUST return a JSON object with the following structure:

```json
{
  "type": "sigsum",
  "signers": ["<base64url-ed25519-public-key>", "..."],
  "threshold": <integer>,
  "policy": "<base64url-sigsum-policy>",
  "max_age": <integer>,
  "logs": {
    "<base64url-log-public-key>": "<url>"
  }
}
```

- `type`:
  The enrollment type. For Sigsum enrollments this MUST be set to `"sigsum"`.

- `signers`:
  An array of Ed25519 public keys, base64-encoded. These keys are authorized to sign WEBCAT manifest files for the domain. A signer key MUST sign WEBCAT manifests only: auditors treat every leaf by an enrolled key as a manifest that must be published.

- `threshold`:
  An integer ≥ 1 indicating the minimum number of distinct valid signatures required to accept a manifest as valid. The value of `threshold` MUST be less than or equal to the number of entries in `signers`.

- `policy`:
  A base64-encoded string representing the compiled Sigsum policy. This includes the list of witnesses, group definitions, and quorum requirements. See [Sigsum Policy Format](#sigsum-policy-format) for details.

- `max_age`:
  An integer representing the maximum number of seconds a manifest may remain valid after its signing timestamp. Since different signatures might have different inclusion times, `max_age` is always counted from the oldest one. The timestamp is verified against the CometBFT chain's AppHash as described in the enrollment specification.

- `logs`:
  A mapping of Sigsum log public keys (base64url-encoded) to their corresponding log URLs. The enrollment generator includes this mapping, based on the Sigsum trust policy, to help clients locate logs for the purpose of monitoring and auditing.

#### 1.3 Field Definitions (Sigstore)

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

  The claims SHOULD match an identity that signs WEBCAT manifests only, e.g. by pinning the Build Config URI (`1.3.6.1.4.1.57264.1.18`) to a dedicated workflow. Auditors treat every matching log entry as a manifest that must be published.

* `max_age`
  An integer representing the maximum number of seconds a manifest may remain valid after its certificate issuance timestamp.

### 2. Policy Transition Mechanism

To transition from one enrollment information to another, servers MUST follow a strict protocol to ensure uninterrupted verification across all browser extensions.

#### 2.1 Transition Requirements

When initiating a policy change:

- The new enrollment information MUST be served persistently at `/.well-known/webcat/enrollment.json` and at `/.well-known/webcat/enrollment.<hash>.json`.
- `/.well-known/webcat/bundle.json` MUST carry the new enrollment information together with a manifest signed under it.
- `/.well-known/webcat/bundle-prev.json` SHOULD carry the previous enrollment information together with a manifest signed under it.
- The enrollment information in the two bundles MUST differ.

#### 2.2 Enrollment Observation Period

Once the new enrollment information is published, it enters a transition period during which the enrollment infrastructure monitors the well-known path for consistency and stability.

> ⚠️ **During this period, no browser extension has switched to the new enrollment information yet.** Until the enrollment chain promotes the new hash, `bundle-prev.json` is the one that verifies.

To prevent enrollment failure:
- The contents of `/.well-known/webcat/enrollment.json` MUST remain unchanged throughout this period.
- Any modification of the enrollment information before the transition completes will invalidate the attempt.

_TODO: since anybody can submit a request for enrollment or de-enrollment, so can do the infra chain itself. This allows for periodic list clenaups not to clog browsers, to remove expired or abandone domains. It can also backfire in some ways though. We should probably always send an alert to the whois email when a change is initiated._

#### 2.3 Post-Transition Compatibility

After the transition is accepted:
- Browser extensions with a fresh preload list verify `bundle.json`.
- Browser extensions with an older preload list fall back to `bundle-prev.json`.
- Eventually, all browser extensions converge on the new state.

To ensure full compatibility throughout this staggered rollout:
- The server SHOULD continue serving `/.well-known/webcat/bundle-prev.json` until all preload lists in use are expected to carry the new hash.
- Once adoption is widespread, `/.well-known/webcat/bundle-prev.json` SHOULD be removed.
- `/.well-known/webcat/enrollment.<hash>.json` for the previous enrollment information MUST stay, see [auditing.md](auditing.md).

#### 2.4 Example

```json
// /.well-known/webcat/bundle.json
{ "enrollment": { "type": "sigsum", "signers": [...], "threshold": 2, ... }, "manifest": { ... }, "signatures": { ... } }

// /.well-known/webcat/bundle-prev.json
{ "enrollment": { "type": "sigstore", "trusted_root": { ... }, "claims": { ... }, ... }, "manifest": { ... }, "signatures": [ ... ] }
```

In this example, the server is advertising a new policy requiring 2 Sigsum signers, while still supporting the previous Sigstore policy during the transition window.

To unenroll, the site operator removes `enrollment.json`. Oracles verify that the path returns 404 or 410.

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

