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
gates. The fixed-runtime, crash-persistence, browser-escape, resource-abuse,
malformed-upstream, and listener-overlap matrix remains open, so the spike is
not a supported app or release-ready transport.

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
