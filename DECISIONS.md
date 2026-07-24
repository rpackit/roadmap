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
