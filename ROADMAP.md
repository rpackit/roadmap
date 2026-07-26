# Roadmap

This is the current-state source of truth for rpackit. Checked items are
implemented and verified at the scope stated here. Toolchain readiness means
the required external tools are present; it does not mean a corresponding
native build API or artifact already exists.

**Last reviewed:** 2026-07-25

## Target maturity

| Target | Inspection or planning | Implemented output |
|---|---|---|
| Portable desktop resources | Implemented | Versioned resource bundle with portable R and managed Shiny lifecycle |
| Native Tauri desktop app | Toolchain readiness only | Not yet implemented |
| Static web | Compatibility assessment only | Builder not yet implemented |
| Dynamic server | Compatibility assessment only | Builder not yet implemented |

The currently runnable desktop sequence is
`prepare_desktop()` → `validate_desktop_bundle()` →
`start_desktop_app()` → `desktop_app_status()` → `stop_desktop_app()`. It
produces executable resources, not a native installer or Tauri application.

## Delivered foundation

- [x] Create organization profile
- [x] Establish roadmap and contracts
- [x] Create R package, runtime index, Windows builder, and examples repositories

## Delivered app inspection

- [x] Implement `doctor()` to report platform and external-tool readiness
- [x] Implement `check_app()`
- [x] Implement non-executing dependency plans from R syntax, DESCRIPTION, and
      renv.lock
- [x] Detect Shiny layouts, packages, renv, system calls, and static-web risks
- [x] Require the actual Tauri CLI and native prerequisites before `doctor()`
      reports the Tauri toolchain as ready

`doctor()` does not claim that Tauri project generation or native packaging is
implemented; those remain separate roadmap items below.

## Delivered Windows runtime

- [x] Build from the official CRAN installer
- [x] Configure a runtime-local package library
- [x] Verify a path containing spaces and relocation
- [x] Generate ZIP, SHA-256, and metadata sidecar
- [x] Publish the first explicitly unsigned GitHub prerelease

## Delivered desktop resource lifecycle

- [x] Prepare atomic, versioned desktop resource bundles with portable R
- [x] Restore hello-shiny dependencies and smoke-test its bundled launcher over
      HTTP without using system R
- [x] Manage the bundled Shiny process with versioned startup events, HTTP
      readiness, graceful control-file shutdown, and verified wrapper/runtime
      cleanup
- [x] Publish a reproducible hello-shiny Windows quickstart with runtime
      SHA-256 verification
- [x] Resolve verified portable R entries from the runtime registry, verify
      SHA-256 before extraction, and reuse atomic cache entries offline
- [x] Reject incompatible `renv.lock` or DESCRIPTION R requirements before
      copying a runtime or installing application packages

The current token is a correlation/bootstrap value, not network
authentication. Status and manifests explicitly report
`network_token_enforced = false`.

## Ordered next work

1. [ ] Enforce the per-launch session token for HTTP and WebSocket traffic
       without exposing it in status, events, logs, or manifests.
2. [ ] Generate a Tauri project around the validated resource contract.
3. [ ] Package hello-shiny as a native Windows desktop executable and verify it
       on a clean machine without system R.
4. [ ] Generate a release workflow that publishes the verified native artifact,
       checksum, signing status, and build provenance.

## Later targets

- [ ] Add shinylive and Docker targets
- [ ] Add macOS portable runtime builders
