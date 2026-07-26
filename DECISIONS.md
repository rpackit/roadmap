# Architecture decisions

## ADR-001: Tauri desktop shell

Use Tauri rather than Electron for the initial desktop shell. R users should
not need to write Rust; the R package renders maintained templates.

## ADR-002: Target-aware packaging

Desktop, static web, and dynamic server targets have different runtime
constraints and must remain explicit.

## ADR-003: Sidecar artifact checksum

The portable runtime archive has an external metadata sidecar containing its
SHA-256. The embedded `build-manifest.json` records source provenance but not
the enclosing archive checksum, avoiding a circular self-hash.

## ADR-004: Authenticated native loopback reverse proxy

Use an authenticated native loopback reverse proxy for the Tauri-to-Shiny
transport instead of direct WebView request interception. The proxy
authenticates a private per-launch WebView profile separately, injects the
upstream Shiny secret only after request normalization, and covers document,
subresource, fetch, and WebSocket traffic under one fail-closed boundary.
Direct interception does not currently provide a portable documented
WebSocket guarantee, and an unauthenticated proxy would erase the existing
loopback-client boundary. Transport contract version `1`, implementation
constraints, and hard acceptance gates are in `TAURI_SECURE_TRANSPORT.md`.
