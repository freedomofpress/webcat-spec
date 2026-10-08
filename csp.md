## CSP
This document defines the subset of Content Security Policy (CSP) directives and values that are allowed within manifests, in both `default_csp` and `extra_csp` fields. These restrictions aim to ensure deterministic execution and verifiable integrity of client-side code.

### 1. General constraints

* A policy MUST NOT contain a comma (`,`). Only a single policy per header is accepted; policy lists are rejected.
* Directive names are matched case-insensitively. Keyword sources (`'self'`, `'none'`, ...) are matched case-insensitively.
* If a directive is present, all of its values are validated against the rules below, regardless of `default-src`.
* Hash sources are accepted in the form `'sha256-...'`, `'sha384-...'` or `'sha512-...'`. Nonces (`'nonce-...'`) are never accepted, since their value cannot be covered by a signature.

### 2. CSP Constraints

The following sections define which directives are allowed and which values are acceptable under the WEBCAT policy model.

#### 2.1 `default-src`

Allowed values:

* `'none'`
* `'self'`

`default-src: 'none'` MUST be the only value if used. If `default-src` is not exactly `'none'` (it is `'self'` or absent), then the manifest MUST also explicitly define:

* `script-src`
* `style-src`
* `object-src: 'none'`
* `worker-src`

#### 2.2 `script-src` / `script-src-elem`

Allowed values:

* `'none'`
* `'self'`
* `'wasm-unsafe-eval'`
* `'sha256-...'`, `'sha384-...'`, `'sha512-...'`

Not allowed:

* `'nonce-...'`, `'unsafe-eval'`, `'unsafe-inline'`, `'unsafe-hashes'`, `'strict-dynamic'`
* Any host source, including enrolled origins

Hash sources allow inline scripts whose content is fixed at signing time and thus verifiable. `script-src-elem` follows the same rules; it is required only if `script-src` is absent and `default-src` is not `'none'`.

> **Note**: All scripts must be either served from the enrolled origin (and covered by the manifest) or inlined with a hash. Loading scripts from other origins, even enrolled ones, is not supported.

#### 2.3 `style-src` / `style-src-elem`

Allowed values:

* `'none'`
* `'self'`
* `'sha256-...'`, `'sha384-...'`, `'sha512-...'`
* `'unsafe-inline'`
* `'unsafe-hashes'`

`style-src-elem` follows the same rules; it is required only if `style-src` is absent and `default-src` is not `'none'`.

> **Note**: While `'unsafe-inline'` and `'unsafe-hashes'` are currently permitted due to wide usage, they are discouraged for new applications and may be deprecated in future versions of the specification.

#### 2.4 `object-src`

Allowed value:

* `'none'`

Must be explicitly defined as `'none'` when `default-src` is not `'none'`.

#### 2.5 `frame-src` / `child-src`

Unrestricted. Any value is accepted.

The client delivers every enrolled document with `Origin-Agent-Cluster: ?1`. Frames are isolated from the embedding document by the same-origin policy, also when same-site.

A framed origin that is itself enrolled is verified independently under its own manifest. A framed origin that is not enrolled is not verified at all, and its content is outside WEBCAT's guarantees.

> **Recommendation**: Use iframes to sandbox untrusted or unverifiable content that a verified application still needs: CAPTCHAs, third-party players, payment widgets, rendering of user-supplied HTML, previews, and similar. Always set the `sandbox` attribute on such frames, granting only the flags strictly needed. Never combine `allow-scripts` and `allow-same-origin` on a frame whose content is same-origin with the application, including `srcdoc` frames: such a frame can script the verified document directly. For cross-origin content, `allow-same-origin` only preserves the frame's own origin and is generally required for third-party widgets to function. Data crossing the boundary should go through `postMessage` with an explicit check of `event.origin` and `event.source`.

A server MUST NOT send an `Origin-Agent-Cluster` value other than `?1` for enrolled documents. Such a response is treated as declaring reliance on site-keying and is blocked.

#### 2.6 `worker-src`

Allowed values:

* `'none'`
* `'self'`

Must be defined when `default-src` is not `'none'`.

#### 2.7 Other directives (e.g. `img-src`, `connect-src`, `media-src`, `font-src`)

There are currently no restrictions on the values used for the following directives:

* `img-src`
* `connect-src`
* `media-src`
* `font-src`

Developers are still encouraged to use minimal and self-contained source definitions for transparency and reproducibility.

### 3. Example

```
default-src 'self';
script-src 'self' 'wasm-unsafe-eval' 'sha256-oiHtO61BAW24D+FpLSqz2Jnv6Wv67XDn90HOTlaklfQ=';
style-src 'self' 'unsafe-inline';
object-src 'none';
worker-src 'self';
frame-src https://captcha.example
```
