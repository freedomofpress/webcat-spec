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
  Map of absolute paths to SHA-256 hashes (base64url). Every key begins with `/` (e.g. `"/index.html"`, `"/js/app.js"`).

* `default_index` (string)
  Relative filename appended to a directory request path to resolve it. A request for `/` resolves to `files["/" + default_index]`, and a request for `/dir/` resolves to `files["/dir/" + default_index]`.

  Example:

  ```json
  "default_index": "index.html"
  ```

* `default_fallback` (string)
  Absolute path (WITH leading slash) within `files`, served for non-existent `main_frame` paths. Can be used for SPA catch-alls, rewrites, custom error pages. Note the asymmetry with `default_index`: `default_fallback` is a full `files` key, while `default_index` is a bare filename.

  Example:

  ```json
  "default_fallback": "/404.html"
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

* For Sigsum, the manifest must contain at least `threshold` valid signatures and proofs; extra or invalid entries are ignored as long as the threshold of valid ones is met.
* For Sigstore, the manifest verifies if at least one bundle satisfies the enrollment `claims` and certificate freshness.
* Signatures are verified over the canonicalized `manifest` object.
* Each proof must satisfy the enrollment policy.
