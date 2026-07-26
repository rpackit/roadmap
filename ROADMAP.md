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
- [x] Make dependency plans fail visibly for incomplete lockfiles, incompatible
      locked versions, and `DESCRIPTION Remotes` without exact lockfile
      provenance, without returning possibly credential-bearing remote text
- [x] Detect Shiny layouts, packages, renv, parsed base system calls with
      file-and-line evidence, and static-web risks
- [x] Publish a runnable getting-started article from inspection and dependency
      planning through resource validation and managed cleanup
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
- [x] Reject unsafe dependency plans before runtime copying, verify every
      required DESCRIPTION package constraint after restore or installation,
      record constraint evidence in the bundle manifest, and bind later
      validation to the copied app and installed versions

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
and leakage checks on the current development WebView2 runtime. The listener
gate also proves four exact-loopback traffic paths under three same-port
contenders: IPv4 wildcard, IPv6 v6-only wildcard, and IPv6 dual-stack wildcard.
All 32 probe requests reached the exact proxy and all wildcard accept counts
were zero; bind success is recorded separately from interception. Valid raw
ordinary HTTP and WebSocket baselines also pass, including one safely tunneled
WebSocket frame. Sixteen HTTP and 16 WebSocket malformed or policy-unsafe
response heads all fail with the exact fixed secret-free 502 boundary; all 34
upstream requests carry one valid synthetic credential, all 17 WebSocket
requests have the normalized upgrade shape, and no marker, frame, or
unexpected upgrade reaches downstream. Seven fragmented valid ordinary-HTTP
body/semantics baselines and 23 hostile body/trailer cases also pass. The
baselines cover fixed-length, chunked, close-delimited, bodyless `HEAD` and
`304` responses with nonzero hypothetical lengths, bodyless `204` without
framing, and bodyless `205` with `Content-Length: 0`. The negatives produce 6
exact pre-stream 502 responses,
12 stream fail-closed terminations that either withhold the downstream head or
leave incomplete framing, 1 empty close-delimited limit cutoff, 2 bodyless
malicious-status terminations, and 2 response-splitting attempts exposing only
the first safe response. The split between no-head and incomplete-framing
outcomes is scheduling-dependent. Trailer coverage includes 97 fields against
the configured 96-field maximum. All 30 first upstream requests carry one
valid synthetic credential. Every keep-alive downstream socket receives a
second authenticated request attempt; all 30 physically close before proxy
shutdown, and no second response, attacker marker, or reusable connection
results. Relevant hostile fixtures omit upstream `Connection: close`, proving
proxy-enforced isolation. `204` rejects `Content-Length` or
`Transfer-Encoding`; `205` rejects a nonzero length or any
`Transfer-Encoding` to avoid ambiguous stream/trailer framing; and the
`HEAD`/`204`/`205`/`304` body policy permits no streamed bytes. Authenticated
request uploads now pass a separate real-loopback matrix for total bytes, idle gaps,
minimum sustained rate, total duration, and request trailers: one immediate
bounded upload succeeds; a chunked upload stops before crossing a small test
byte cap; independent idle, below-rate, and over-duration uploads terminate
without a success response; and a parsed chunked trailer becomes a fixed
`502`. The development WebView2 report records all six results, 5/5 bounded
negative terminations, 5/5 valid parsed upstream credentials, and zero
request-probe credential leaks.

Responses now have separate 256 MiB encoded and decoded defaults, a 15-second
non-empty-frame idle deadline, and a 1 KiB/s floor in complete 5-second
windows. At most two ordered `gzip`/`deflate`/`br`/`zstd` layers are decoded
with backpressure; unsafe transforms, malformed encodings, and decoded
overflow fail closed. A real-loopback matrix proves identity and gzip
baselines plus five bounded idle, below-rate, expansion, malformed-gzip, and
unsupported-coding failures. Its expansion fixture maps 67 encoded bytes to
4,128 decoded bytes against a 32-byte cap; all 7 upstream requests are
correctly authenticated with zero credential or attacker-marker leakage. The
recorded WebView2 `150.0.4078.99` run passed the complete matrix and forwarded
zero decoded bytes from the expansion case.

WebSocket tunnels now apply independent raw-byte token buckets in each
direction, defaulting to 8 MiB/s with a one-second burst. A real-loopback
matrix proves a small authenticated baseline plus separate 100-byte upload and
download cases at 100 B/s with a 100 ms burst. Both shaped exchanges complete
boundedly after the required 750 ms minimum; all 3 upstream handshakes are
correctly authenticated and normalized with zero proxy-cookie or
bootstrap-header leakage. Recorded WebView2 `150.0.4078.99` debug and release
runs both measured 997 ms client-to-upstream and 934 ms upstream-to-client.

The active browser-escape matrix also passes on WebView2 `150.0.4078.99`.
It records one external-document navigation callback and one request-layer
network block, one popup denial, one download cancellation with an empty
isolated directory, and one cancelled native-origin external-scheme event. The
scheme is a random per-run volatile current-user URL protocol whose
same-executable handler is self-tested with a scoped canary; cancellation keeps
the marker absent, and the registration is removed and verified absent.
Native readback confirms devtools, browser accelerator keys, and default
context menus disabled; a valid unpacked extension is explicitly rejected as
unsupported; environment and machine/user policy-registry overrides are
absent before and after WebView creation; no `DevToolsActivePort` exists; and
external navigation and popup collectors receive zero requests.

Phase 1 remains unchecked: the reviewed fixed minimum WebView2 runtime and
forced-crash profile persistence are unresolved. The spike is not a generated
application or supported installer.

## Later targets

- [ ] Add shinylive and Docker targets
- [ ] Add macOS portable runtime builders
