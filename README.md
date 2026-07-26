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
application. The next desktop milestone is a Tauri exact-origin request
injector or native loopback proxy that applies the protected header to
navigation, subresources, and WebSocket upgrades without exposing it to
browser JavaScript.

Start with:

- `ROADMAP.md` for implemented capabilities and the ordered next work;
- `ARCHITECTURE.md` for target boundaries;
- `CONTRACTS.md` for cross-repository interfaces;
- `PLATFORMS.md` for verification rules;
- `DEVELOPMENT_MASTER_PLAN.md` for the historical 0.1 target specification and
  original milestone design.

Implementation belongs in the repository that owns the component. Runtime
archives belong in GitHub Releases, never in Git.
