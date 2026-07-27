# rpackit roadmap

This repository owns the architecture and cross-repository contracts for the
rpackit multi-repository project. [`ROADMAP.md`](ROADMAP.md) is the
current-state source of truth: a checked item means the behavior or artifact
exists and has the stated verification evidence.

The current desktop resource contract uses launcher protocol 2: a fresh
default 256-bit CSPRNG-generated credential is handed off through an
account-private one-time file with verified Windows or POSIX permissions,
Shiny enforces it for dynamic HTTP, static resources, and WebSocket traffic,
and readiness requires a post-bind event plus an authenticated probe. New
manifests declare this authentication and set
`network_token_enforced` to `true`. Legacy protocol-1 bundles remain
inspectable but cannot be launched.

Dependency preparation is fail-closed as well: required lockfile records and
DESCRIPTION package constraints are checked before runtime copying, installed
versions are checked again before atomic publication, and `Remotes` requires
reviewed lockfile provenance instead of silently falling back to a repository
package.

This is the authenticated backend contract, not yet a complete native desktop
application. The pre-release
[`rpackit-tauri`](https://github.com/rpackit/rpackit-tauri) Phase 1 spike
implements the selected authenticated native loopback reverse proxy. Native
code uses a third, one-time secret for the exact bootstrap request; the HTTP
response creates a host-only HttpOnly proxy-session cookie, and the proxy
injects the separate Shiny secret only after authenticating later document,
subresource, fetch, and WebSocket traffic. Direct request interception and an
unauthenticated proxy are excluded from the accepted design. The generated
executable will own protocol-2 launch state directly rather than serializing an
R-returned secret. Transport contract version `2` defines the hard acceptance
gates. Listener-overlap plus the strict malformed response-head and streamed
response-body/trailer matrices now pass on the development runtime. The body
matrix includes correct bodyless `HEAD`/`204`/`205`/`304` semantics, excess
trailer fields, close-delimited overflow, and keep-alive reuse attempts after
every raw case. Authenticated request uploads now also have tested byte, idle,
minimum-rate, total-time, and trailer limits. Responses now have separate
encoded and decoded byte caps, idle and sustained-rate gates, and bounded
`gzip`/`deflate`/`br`/`zstd` decoding with a real-loopback expansion matrix.
WebSocket tunnels now have independent 8 MiB/s directional token buckets with
one-second bursts and a real-loopback baseline plus separate upload/download
rate evidence. The active browser-escape matrix now passes on WebView2
`150.0.4078.99`: external document access is blocked before network, popup and
download creation are denied, external-scheme events are cancelled, native
settings and extension rejection are verified, override sources are absent,
and external collectors receive no escape requests. The forced-crash matrix
and the complete Debug/Release matrix on reviewed Fixed Version Runtime
`149.0.4022.98` also pass. Both fixed reports verify the Microsoft
package/tree and exact loaded version, contain no unproven gate, and record
`phase1_release_ready: true`. Phase 1 is complete, but the spike is not a
generated or supported app; real launcher lifecycle begins in Phase 2.

The current implementation has since completed the native lifecycle and source
generation milestones. `rpackit` now generates and validates
application-specific Tauri source from the versioned
`windows-template-v1.0.0` template, and the
[generated-project gate](https://github.com/rpackit/rpackit/actions/runs/30239483026)
compiles and runs generated `hello-shiny` through the released portable-R and
native WebView owner lifecycle. Native installer packaging and clean-machine
verification remain next.

Start with:

- `ROADMAP.md` for implemented capabilities and the ordered next work;
- `ARCHITECTURE.md` for target boundaries;
- `CONTRACTS.md` for cross-repository interfaces;
- `TAURI_SECURE_TRANSPORT.md` for the accepted native transport, threat model,
  forwarding invariants, lifecycle, and release gates;
- `DECISIONS.md` for recorded architecture choices and their rationale;
- `PLATFORMS.md` for verification rules;
- `DEVELOPMENT_MASTER_PLAN.md` for the historical 0.1 target specification and
  original milestone design.

Implementation belongs in the repository that owns the component. Runtime
archives belong in GitHub Releases, never in Git.
