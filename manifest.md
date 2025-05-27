## Manifest

This specification defines the manifest format, used to declare the cryptographic integrity and Content Security Policies of a web application.

### 1. Manifest Format
Each enrolled web application must serve a JSON manifest with the following top-level structure:

```json
{
  "manifest": { ... },
  "signatures": { ... }
}
```

#### 1.1 `manifest` object

This object describes the application version, default CSP policy, optional per-path overrides, and expected hashes of relevant resources.

##### Fields:

* `app_url` (string): URL of the upstream project. **Required.**
* `app_version` (string): Version identifier. **Required.**
* `wasm` (array of strings): List of SHA-256 hashes (hex) of WebAssembly binaries. **Optional.**
* `default_csp` (string): The global Content-Security-Policy to enforce. Must be present and must apply to all paths. **Required.**
* `extra_csp` (object): An object mapping URL paths to CSP strings that override the default for specific paths. Keys are paths (e.g., `/viewer/index.html`), and values are CSP policy strings. The strongest matching path prefix determines which CSP is applied — e.g., a policy defined for `/data/` applies to all nested paths like `/data/file.js`, unless a more specific rule (like `/data/manager/index.html`) exists. **Optional.**
* `files` (object): An object mapping relative URL paths to SHA-256 hashes (hex). Keys must include trailing slashes for directories and refer to `index.html` or `index.htm` as applicable. **Required.**

#### 1.2 `signatures` object

This object maps Ed25519 public keys (encoded as multibase, base64url, or raw hex) to their corresponding Sigsum proof objects.

Each value in the object is a Sigsum proof structure containing:

**TODO**

Each manifest must be signed by at least one Ed25519 key.


