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

This object describes the application metadata, CSP rules, required file hashes, index/fallback mapping, and Sigsum timestamp.

### Required fields

* `version` (string)
  Application version identifier. When `source` is set, this MUST be the name of a git tag in `source`.

* `default_csp` (string)
  The global Content-Security-Policy to enforce for all paths not covered by `extra_csp`.

* `files` (object)
  Map of relative paths to SHA-256 hashes (base64url).

* `default_index` (string)
  Path within `files` that resolves the root (`/`).

  Example:

  ```json
  "default_index": "/index.html"
  ```

* `default_fallback` (string)
  Path within `files` to serve for non-existent `main_frame` paths. Can be used for SPA catch-alls, rewrites, custom error pages.

  Example:

  ```json
  "default_fallback": "/404.html"
  ```

* `timestamp` (string)
  A Sigsum tree head satisfying the Sigsum policy specified during enrollment, in ASCII text format with escaped newlines (`\\n`).

  Example:

  ```
  "timestamp": "tree_size 12345\\nroot_hash aabbcc...\\ntimestamp 1734567890\\nwitness test1 deadbeef..."
  ```

### Optional fields

* `source` (string)
  URL of the git repository the application is built from, including the protocol (`https://` or `ssh://`). Omitted for closed-source applications. The tag `version` MUST exist in this repository and MUST contain the build recipe described in section 2.

* `artifact` (string)
  HTTP(S) URL template of a release archive for this version. The literal `${version}` is substituted with `version`. Example:

  ```json
  "artifact": "https://github.com/jgraph/drawio/releases/download/${version}/webcat-${version}.tar.gz"
  ```

  See section 3 for the archive format. Manifests sharing a `version` MUST list identical `files`.

* `app` (string, deprecated)
  Replaced by `source`. The browser extension ignores it.

* `wasm` (array of strings)
  List of SHA-256 hashes (base64url) of WebAssembly binaries.

* `extra_csp` (object)
  Map of path prefixes to CSP strings. Longest‐prefix matching applies.
  
  Example:

  ```json
  "extra_csp": {
    "/admin/": "default-src 'self'; ..."
  }
  ```


## 1.2 `signatures` object

Maps Ed25519 public keys to Sigsum proofs. Each key is an Ed25519 public key encoded as base64url.

### Value format

Each value is a Sigsum proof, in ASCII text with escaped newlines.

Example:

```json
"signatures": {
  "f0ab12...": "tree_size 12345\\nroot_hash 9c18...\\nsignature abcd...\\nwitness t1 deadbeef..."
}
```

### Validation rules

* The manifest must contain at least `threshold` valid signature and proofs.
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
