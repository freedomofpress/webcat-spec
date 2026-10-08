# Monitoring and auditing

Monitoring and auditing complement the transparency properties of WEBCAT. Monitoring detects activity: new releases and enrollment changes. Auditing checks each release or enrollment change.

The goal is for the enrollment infrastructure, the transparency logs, and then the enrollment and manifest files to carry the metadata needed to run every verification and reproducibility check programmatically.

The audience is web app developers, site operators, infrastructure operators, researchers, and incident responders.

## 1. Inputs

There are three input sources:

1. The enrollment chain, replayed from genesis on an observer node ([enrollment.md](enrollment.md)). It gives every domain ever enrolled and every enrollment information hash each one had.
2. The transparency logs named in that enrollment information: Sigsum logs from `logs` and the policy, Rekor from Sigstore trusted root. They give every manifest ever signed.
3. The domain's well-known directory ([server.md](server.md)). It gives the content behind every hash found in the first two.

Per manifest, the `artifact` archive and the `source` repository it declares ([manifest.md](manifest.md)) give the files and the code.

## 2. Monitoring

Monitoring detects activity, or searches for it. It runs continuously, or retroactively when requested.

### 2.1 Enrollment history

The history of enrollment information is obtained from the domain itself. For every `(domain, hash)` in the chain history, `/.well-known/webcat/enrollment.<hash>.json` MUST exist. Canonicalized and hashed, it MUST match the hash in the name and in the chain.

### 2.2 Log scan

From the enrollment information obtained in 2.1:
 - For Sigstore, save the pinned claims
 - For Sigsum, save the pinned keys

Then scan the respective logs, also obtained from the enrollment information, for all the leaves. For a Sigsum multisig, match against each key.

* Sigsum: `key_hash` equals the SHA-256 of a key in any historical `signers` list.
* Sigstore: the certificate satisfies the `claims` of any historical enrollment and chains to its `trusted_root`.

The same key or identity may be enrolled on more than one domain. In that case the leaf belongs to all of them. Every new leaf that matches is a release.

### 2.3 Manifest recovery

The manifest behind a leaf is obtained from the domain itself. For every matching leaf, the domain MUST serve:

* Sigsum: `/.well-known/webcat/manifest.sigsum.<checksum>.json`, where `<checksum>` is the leaf checksum.
* Sigstore: `/.well-known/webcat/manifest.sigstore.<hash>.json`, where `<hash>` is the artifact digest of the `hashedrekord` entry.

The canonicalized `manifest` object in the file, hashed, MUST match the value in the log. When the key or identity is enrolled on more than one domain, it is enough that one of them serves the file.

### 2.4 Manifest verification

The file is verified as the browser extension would ([manifest.md](manifest.md)), against every enrollment the domain ever had. Freshness is evaluated at the time of the log entry, not at the time of the check. The manifest is valid if it verifies under at least one enrollment.

## 3. Auditing

Auditing checks each recovered manifest against its source and its artifact. It runs once per manifest and can be repeated at any time.

### 3.1 Availability

If the manifest has `artifact`, the archive for its `version` is downloaded and extracted. Every path in `files` MUST be present and MUST hash to the value in the manifest.

### 3.2 Reproducibility

If the manifest has `source`, the repository is cloned at tag `version` and built with the recipe in [manifest.md](manifest.md) section 2 (TODO). The output is compared as in 3.1.

## 4. Obligations on site operators

Both activities depend on the domain keeping its history available:

* Every `enrollment.<hash>.json`, `manifest.sigsum.<checksum>.json` and `manifest.sigstore.<hash>.json` MUST stay unchanged while the domain is enrolled.
* The manifest file MUST be published no later than the manifest is first served.
* Signer keys and Sigstore identities MUST be used for WEBCAT manifests only.
* One archive per `version` MUST stay available while the domain is enrolled.
* A `version` MUST NOT be reused for a different set of `files`.

## 5. Out of scope

* The leaves of a log that shuts down. Monitors SHOULD keep the leaves they have scanned.
* The enrollment content of a domain that unenrolled and went offline.
* Anything outside `files`.
