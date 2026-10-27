# Manifest

This specification defines the manifest format used to declare the cryptographic integrity and Content Security Policies of a web application.

## 1. Manifest Format

Each enrolled web application must serve a JSON manifest with the following top-level structure:

```json
{
  "manifest": { ... },
  "signatures": { ... }
}
```

## 1.1 `manifest` object

This object describes the application metadata, CSP rules, required file hashes, fallbacks, and Sigsum timestamp.

### Required fields

* `manifest_schema` (integer)
  The version of this manifest format. The current version is `1`: directory entries in `files`, `fallback_hashes`, `source` and `artifact`. Manifests without the field predate versioning: they carry `default_index`, `default_fallback` and `app`, and MAY also carry the version 1 fields next to them. The field is part of the signed content.

* `version` (string)
  Application version identifier, a [semantic version](https://semver.org/spec/v2.0.0.html) such as `1.4.0`. When `source` is set, this MUST be the name of a git tag in `source`. A leading `v`, as in existing git tags, is accepted but advised against.

* `default_csp` (string)
  The global Content-Security-Policy to enforce for all paths not covered by `extra_csp`.

* `files` (object)
  Map of absolute paths to SHA-256 hashes (base64url). A directory path, ending in `/`, is listed like any other path, with the hash of the document the server serves for it; `/` itself is such a path.

  Example:

  ```json
  "files": { "/": "<hash of index.html>", "/index.html": "<hash of index.html>", "/docs/": "<hash of docs/index.html>", "/docs/index.html": "<hash of docs/index.html>" }
  ```

* `wasm` (array of strings)
  List of SHA-256 hashes (base64url) of WebAssembly binaries. The field MUST be present but MAY be an empty array (`[]`) when the application ships no WebAssembly.

### Conditionally required fields

* `timestamp` (string) — REQUIRED for Sigsum manifests, omitted for Sigstore manifests.
  A Sigsum cosigned tree head satisfying the Sigsum policy specified during enrollment, in ASCII text format with escaped newlines (`\\n`). For Sigstore manifests, freshness is derived from the signing certificate's issuance time instead (see [Server](server.md) `max_age`).

  Example:

  ```
  "timestamp": "tree_size 12345\\nroot_hash aabbcc...\\ntimestamp 1734567890\\nwitness test1 deadbeef..."
  ```

### Optional fields

* `fallback_hashes` (array of strings)
  SHA-256 hashes (base64url) of documents accepted at any path on a `main_frame` navigation, checked before the path's own entry: the catch-all page of a single-page application, a custom error page. Subresources never fall back. Each fallback MUST be served at `/.well-known/webcat/hashes/<hash>` ([server.md](server.md)), so that what the manifest accepts can be found by its hash.

  Example:

  ```json
  "fallback_hashes": ["<hash of 404.html>"]
  ```

* `default_index` (string, deprecated) and `default_fallback` (string, deprecated)
  Replaced by directory entries in `files` and by `fallback_hashes`. `default_index` is a bare filename appended to a directory path, `default_fallback` an absolute `files` key served for `main_frame` paths not in `files`. Only manifests without `manifest_schema` carry them; browser extensions from 3.2.0 derive the replacements from them when `fallback_hashes` is absent, and earlier ones require them. A manifest with `manifest_schema` MUST NOT include them.

* `source` (string)
  URL of the git repository the application is built from, including the protocol (`https://` or `ssh://`). Omitted for closed-source applications. The tag `version` MUST exist in this repository and MUST contain the build recipe described in section 2.

* `artifact` (string)
  HTTP(S) URL template of a release archive for this version. The literal `${version}` is substituted with `version`. Example:

  ```json
  "artifact": "https://github.com/jgraph/drawio/releases/download/${version}/webcat-${version}.tar.gz"
  ```

  See section 3 for the archive format. Manifests sharing a `version` MUST list identical `files`.

* `app` (string, deprecated)
  Replaced by `source`. Only manifests without `manifest_schema` carry it; it MUST be ignored. A manifest with `manifest_schema` MUST NOT include it.

* `extra_csp` (object)
  Map of path prefixes to CSP strings. Longest‐prefix matching applies.
  
  Example:

  ```json
  "extra_csp": {
    "/admin/": "default-src 'self'; ..."
  }
  ```


## 1.2 `signatures` object

The shape of `signatures` depends on the enrollment type.

### Sigsum: object of proofs

For Sigsum enrollments, `signatures` is an object mapping Ed25519 public keys (base64url) to Sigsum proofs. Each value is a Sigsum proof, in ASCII text with escaped newlines.

Example:

```json
"signatures": {
  "f0ab12...": "tree_size 12345\\nroot_hash 9c18...\\nsignature abcd...\\nwitness t1 deadbeef..."
}
```

### Sigstore: array of bundles

For Sigstore enrollments, `signatures` is an array of Sigstore bundles (the serialized `@sigstore/bundle` form, either a message-signature or DSSE bundle):

```json
"signatures": [
  { "mediaType": "application/vnd.dev.sigstore.bundle...", "verificationMaterial": { ... }, "messageSignature": { ... } }
]
```

A manifest carries one form or the other, never both, matching its enrollment type.

### Validation rules

* For Sigsum, the manifest must contain at least `threshold` valid signatures and proofs. Entries for keys outside `signers` are ignored; an invalid proof from an enrolled key fails verification.
* For Sigstore, the manifest verifies if at least one bundle satisfies the enrollment `claims` and certificate freshness.
* Signatures are verified over the canonicalized `manifest` object.
* Each proof must satisfy the enrollment policy.

## 2. Reproducible builds

A manifest that sets `source` claims that `files` can be rebuilt from that repository at tag `version`. Anyone can then confirm that the signed code is the open source one. Auditors check it ([auditing.md](auditing.md)).

### 2.1 Build recipe

TODO. The repository at the tag must contain a recipe that produces the web root in an output directory with a single command.

### 2.2 Verification

For every path in `files`, the SHA-256 of `out/<path>` MUST equal the manifest hash. Extra files in `out` are ignored.

## 3. Release archive

The archive keeps every released version available after the website has moved on. Any manifest ever signed can then be checked against the files it describes.

The `artifact` URL MUST serve a gzip-compressed tar archive whose root is the web root: every path in `files` at the same relative path. Extra files are ignored.

Archives MUST remain available for every version ever served.

A closed-source application sets `artifact` without `source`.
