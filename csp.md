## CSP
This document defines the subset of Content Security Policy (CSP) directives and values that are allowed within manifests, in both `default_csp` and `extra_csp` fields. These restrictions aim to ensure deterministic execution and verifiable integrity of client-side code.

### 1. CSP Constraints

The following sections define which directives are allowed and which values are acceptable under the WebCAT policy model.

#### 1.1 `default-src`

Allowed values:

* `'none'`
* `'self'`

If `default-src` is not `'none'`, then the manifest MUST also define:

* `object-src: 'none'`
* `worker-src`
* and either `child-src` or `frame-src`

#### 1.2 `script-src`

Allowed values:

* `'none'`
* `'self'`
* `'wasm-unsafe-eval'`

Not allowed:

* `sha256-...`, `nonce-...`, or `unsafe-eval`

> **Note**: The use of hash-based or nonce-based script sources is disallowed due to interference with WebAssembly runtime instrumentation and deterministic script analysis.

#### 1.3 `style-src`

Allowed values:

* `'none'`
* `'self'`
* `sha256-...`
* `'unsafe-inline'`
* `'unsafe-hashes'`

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
