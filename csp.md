## CSP
This document defines the subset of Content Security Policy (CSP) directives and values that are allowed within manifests, in both `default_csp` and `extra_csp` fields. These restrictions aim to ensure deterministic execution and verifiable integrity of client-side code.

### 1. CSP Constraints

The following sections define which directives are allowed and which values are acceptable under the WebCAT policy model.

A single CSP string MUST NOT contain a comma: a comma encodes multiple policies in one header, and the client rejects any `default_csp`/`extra_csp` value containing one.

Where a directive supports both a plain form (e.g. `script-src`, `style-src`) and its `-elem` variant (`script-src-elem`, `style-src-elem`), the same allowed/disallowed values apply to both. The plain directive, when defined, governs its `-elem` counterpart by inheritance; an explicitly defined `-elem` directive MUST independently satisfy the same constraints.

#### 1.1 `default-src`

Allowed values:

* `'none'`
* `'self'`

When `default-src 'none'` is used to satisfy the "must be none" condition below, `'none'` MUST be its only value.

If `default-src` is not `'none'`, then the manifest MUST also define:

* `script-src`
* `style-src`
* `object-src: 'none'`
* `worker-src`
* and either `child-src` or `frame-src`

#### 1.2 `script-src`

Allowed values:

* `'none'`
* `'self'`
* `'wasm-unsafe-eval'`
* `'sha256-...'` (and `'sha384-...'` / `'sha512-...'`) — the hash of an allowed inline script

Not allowed:

* `nonce-...`, `'unsafe-eval'`, `'unsafe-inline'`

> *Note*: Nonce-based sources and `'unsafe-eval'`/`'unsafe-inline'`/`'strict-dynamic'` remain disallowed: nonces are per-response and cannot be verified statically, and the others defeat deterministic script analysis.

The same allowed and disallowed values apply to `script-src-elem`. A defined `script-src` governs `script-src-elem` by inheritance; if a separate `script-src-elem` is present, it MUST satisfy the same constraints.

#### 1.3 `style-src`

Allowed values:

* `'none'`
* `'self'`
* `sha256-...` (and `sha384-...` / `sha512-...`)
* `'unsafe-inline'`
* `'unsafe-hashes'`
* External URLs only if they point to domains that are also enrolled in WEBCAT

The same values apply to `style-src-elem`.

> **Note**: While `'unsafe-inline'` and `'unsafe-hashes'` are currently permitted due to wide usage, they are discouraged for new applications and may be deprecated in future versions of the specification.

#### 1.4 `object-src`

Allowed value:

* `'none'`

Must be explicitly defined as `'none'` when `default-src` is not `'none'`.

> **Note**: Requires more input to understand whether there are casesd where objects are stricly needed.

#### 1.5 `frame-src` / `child-src`

Allowed values:

* `'none'`
* `'self'`
* `blob:`
* `data:`
* External URLs **only if** they point to domains that are also enrolled in WebCAT

At least one of these two must be defined if `default-src` is not `'none'`.

> **Note**: If any external origin is listed, its enrollment is verified at runtime by the client.

#### 1.6 `worker-src`

Allowed values:

* `'none'`
* `'self'`

Must be defined when `default-src` is not `'none'`.

#### 1.7 Other directives (e.g. `img-src`, `connect-src`, `media-src`, `font-src`)

There are currently no restrictions on the values used for the following directives:

* `img-src`
* `connect-src`
* `media-src`
* `font-src`

Developers are still encouraged to use minimal and self-contained source definitions for transparency and reproducibility.
