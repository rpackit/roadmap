# Roadmap

This is the current-state source of truth for rpackit. Checked items are
implemented and verified at the scope stated here. Toolchain readiness means
the required external tools are present; it does not mean a corresponding
native build API or artifact already exists.

**Last reviewed:** 2026-07-26

## Target maturity

| Target | Inspection or planning | Implemented output |
|---|---|---|
| Portable desktop resources | Implemented | Versioned resource bundle with portable R and authenticated managed Shiny lifecycle |
| Native Tauri desktop app | Phase 1 transport spike in progress | No generated app or installer |
| Static web | Compatibility assessment only | Builder not yet implemented |
| Dynamic server | Compatibility assessment only | Builder not yet implemented |

The currently runnable desktop sequence is
`prepare_desktop()` → `validate_desktop_bundle()` →
`start_desktop_app()` → `desktop_app_launch_config()` →
`desktop_app_status()` → `stop_desktop_app()`. It produces executable
resources and an authenticated native-shell handoff contract, not a native
installer or Tauri application.

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
- [x] Implement launcher protocol 2 with a fresh default 256-bit
      CSPRNG-generated session credential, delivered through a
      current-account-private one-time `--token-file` that is consumed before
      app or port validation
- [x] Restrict and verify Windows DACLs for the current account plus SYSTEM,
      and verify POSIX directory mode 0700 and credential mode 0600
- [x] Keep rpackit from placing the credential itself in child arguments,
      environment variables, URLs, manifests, generated lifecycle-event
      fields, returned status, or redaction-safe print output; redact launcher
      errors and public log/event views
- [x] Enforce the `Shiny-Shared-Secret` request header for dynamic HTTP, static
      HTTP, and WebSocket session acceptance, closing unauthenticated upgraded
      sockets before an app session starts
- [x] Gate readiness on a post-bind `listening` event and a successful
      authenticated HTTP probe
- [x] Record the authentication descriptor and
      `network_token_enforced = true` in new manifests; continue validating
      matching legacy protocol-1 bundles while refusing to launch them
- [x] Preserve Shiny directory-app `DESCRIPTION` semantics and retain a
      retryable managed handle whenever private-file cleanup is incomplete
- [x] Publish a reproducible hello-shiny Windows quickstart with runtime
      SHA-256 verification
- [x] Resolve verified portable R entries from the runtime registry, verify
      SHA-256 before extraction, and reuse atomic cache entries offline
- [x] Reject incompatible `renv.lock` or DESCRIPTION R requirements before
      copying a runtime or installing application packages

The implemented protocol is a secure backend launch contract. A stock browser
cannot attach the protected header to top-level navigation and WebSocket
upgrades. `desktop_app_launch_config()` therefore supplies a secret-bearing
development/third-party handoff rather than a browser-facing credential API.
A generated Tauri application does not serialize that R-returned secret; as
runtime owner it implements launcher protocol 2 directly and retains its
launch state in native memory. Its transport is an authenticated loopback
reverse proxy: it keeps the Shiny secret out of the WebView and separately
authenticates the WebView before forwarding HTTP or WebSocket traffic. A bare
loopback proxy and a direct interceptor are not accepted substitutes. See
transport contract version `2` in `TAURI_SECURE_TRANSPORT.md`.

## Ordered next work

1. [ ] Prove the authenticated native loopback reverse proxy in a Windows Tauri
       transport spike, including one-time `B` bootstrap and host-only
       HttpOnly `P` cookie authentication, HTTP/subresource/fetch/WebSocket
       coverage, redirect isolation, and every hard gate in
       `TAURI_SECURE_TRANSPORT.md`.
2. [ ] Integrate the protocol-2 launcher, Windows Job Object ownership, and
       deterministic cleanup, then generate the maintained Tauri project around
       the validated resource contract.
3. [ ] Package hello-shiny as a native Windows desktop executable and verify it
       on a clean machine without system R.
4. [ ] Generate a release workflow that publishes the verified native artifact,
       checksum, signing status, and build provenance.

Phase 1 progress is tracked in
[`rpackit-tauri`](https://github.com/rpackit/rpackit-tauri). Its pre-release
Windows harness now implements the three-secret `S`/`P`/`B` transport and has
exercised authenticated bootstrap, host-only cookie delivery, HTTP assets and
fetches, streaming, redirects, WebSocket traffic, cross-instance isolation,
and leakage checks on the current development WebView2 runtime. Phase 1
remains unchecked: the reviewed fixed minimum WebView2 runtime,
forced-crash profile persistence, browser escape-path attempts,
HTTP/WebSocket resource-abuse and malformed-upstream cases, and Windows
wildcard-listener overlap are not all resolved. The spike is not a generated
application or supported installer.

## Later targets

- [ ] Add shinylive and Docker targets
- [ ] Add macOS portable runtime builders
