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

* `app` (string)
  URL of the upstream project.

* `version` (string)
  Application version identifier.

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
