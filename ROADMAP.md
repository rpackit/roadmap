# Roadmap

This is the current-state source of truth for rpackit. Checked items are
implemented and verified at the scope stated here. Toolchain readiness means
the required external tools are present; it does not mean a corresponding
native build API or artifact already exists.

**Last reviewed:** 2026-07-27

## Target maturity

| Target | Inspection or planning | Implemented output |
|---|---|---|
| Portable desktop resources | Implemented | Versioned resource bundle with portable R and authenticated managed Shiny lifecycle |
| Native Tauri desktop app | Phases 1-3 complete; native packaging is next | Generated application-specific source compiles and passes the real-R/WebView lifecycle; no installer |
| Static web | Compatibility assessment only | Builder not yet implemented |
| Dynamic server | Compatibility assessment only | Builder not yet implemented |

The currently runnable R-package sequence extends from
`prepare_desktop()` and `validate_desktop_bundle()` through
`generate_tauri_app()` and `validate_tauri_project()`. The generated project
stamps the maintained native shell with application resources, identity,
version, optional icon, and authenticated launch-contract metadata. The source
compiles and passes the real proxy/R/WebView lifecycle gate, but it is not yet
a native installer.

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

1. [x] Prove the authenticated native loopback reverse proxy in a Windows Tauri
       transport spike, including one-time `B` bootstrap and host-only
       HttpOnly `P` cookie authentication, HTTP/subresource/fetch/WebSocket
       coverage, redirect isolation, and every hard gate in
       `TAURI_SECURE_TRANSPORT.md`.
2. [x] Integrate the protocol-2 launcher, deterministic cleanup, and maintained
       native Tauri WebView shell around the validated resource contract.
   - [x] Add strict non-executing schema-1/protocol-2 resource validation.
   - [x] Add bounded protocol-2 event decoding and terminal sequence tracking.
   - [x] Add suspended process creation, explicit inherited-handle allowlisting,
         unnamed kill-on-close Job ownership, no-breakaway policy, and
         fail-before-execution assignment.
   - [x] Add validated explicit Unicode child-environment construction with
         case-insensitive replacement/removal, Windows ordering, exact
         double-NUL serialization, Debug redaction, and zeroization so the
         real-R owner can strip ambient R/rpackit configuration without
         mutating the parent process.
   - [x] Compose the native layers against an executable synthetic runtime:
         token consumption, bounded pipes, exact runtime/listener ownership,
         authenticated readiness, health polling, graceful/forced shutdown,
         zero-active-Job accounting, owner-drop cleanup, negative gates, and
         retryable preservation of unexpected audit entries.
   - [x] Capture reported runtime identity by PID plus creation time and verify
         its exact IPv4-loopback listener through Windows owner-PID tables.
   - [x] Atomically create protected current-account-plus-SYSTEM session,
         token, and control objects with exact DACL readback and non-recursive
         cleanup.
   - [x] Drive those layers through a real bundled R launch, authenticated
         readiness, graceful/forced shutdown, and repeated `hello-shiny`
         lifecycle/crash matrices.
   - [x] Compose the authenticated proxy and R lifecycle under one native
         owner: bind/classify the random browser origin before R, share one
         `S`/`P`/`B` set, expose only native `P`/`B` launch handles after
         readiness, stop proxy traffic before forced cleanup, and pass both
         synthetic and released-R/`hello-shiny` composition gates.
   - [x] Add pre-R WebView2 identity/runtime/policy preflight, one hidden
         hardened WebView with an exact per-launch profile, native `B`
         bootstrap, exact `P` cookie verification, authenticated document
         readiness, close interception, and bounded
         cookie/browsing-data/window/profile cleanup.
   - [x] Pass the
         [reviewed full-owner gate](https://github.com/rpackit/rpackit-tauri/actions/runs/30237185375)
         against published portable R 4.6.1 and pinned `hello-shiny`, including
         graceful R/proxy/Job/session cleanup and exact WebView profile removal.
3. [x] Generate the maintained Tauri project from one validated resource
       bundle, including application-specific metadata, assets, and launch
       configuration.
   - [x] Pass the
         [generated-project gate](https://github.com/rpackit/rpackit/actions/runs/30239483026):
         compile the generated `hello-shiny` source, run it through the native
         owner, verify secret-free evidence, and remove runner-scoped runtime
         and build storage.
4. [ ] Package hello-shiny as a native Windows desktop executable and verify it
       on a clean machine without system R.
5. [ ] Generate a release workflow that publishes the verified native artifact,
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

The cross-process forced-crash profile matrix also passes on WebView2
`150.0.4078.99`: the child verifies `P` in a populated private profile, is
forcibly terminated without reaching its graceful-cleanup sentinel, and the
parent reopens the same profile and old hostname with no reusable cookie,
destroys the WebView, removes the directory, and records no secret shape.

Phase 1 is now complete. The full Debug and Release matrices also pass on the
exact reviewed x64 Fixed Version Runtime `149.0.4022.98`. Native startup and
the runner verify the pinned Microsoft archive, expanded 259-file tree,
executable identity and signer, trusted runtime-folder selection, and exact
loaded version. Both fixed reports pass every development gate, contain no
secret shape or unproven gate, and set `phase1_release_ready` to true.
WebSocket shaping measured 1,007/934 ms in Debug and 1,006/921 ms in Release,
with 3/3 valid normalized handshakes and no credential leakage. The spike is
still not a generated application or supported installer; real launcher
lifecycle is Phase 2.

Phase 2 now has independently tested native startup and application-owner
foundations in `rpackit-tauri`: strict schema-1 resource loading, protocol-2
decoding, suspended Job assignment, explicit sanitized-environment
construction, exact process/listener identity, protected token/control files,
an authenticated proxy/runtime owner, and a maintained Tauri
WebView/window/profile owner. The
[reviewed full-owner run](https://github.com/rpackit/rpackit-tauri/actions/runs/30237185375)
passed with the SHA-256-pinned portable R 4.6.1 Release and pinned
`hello-shiny`: native interpreter/package loading, one-time bootstrap,
authenticated direct, proxied, and real WebView content, credential denial,
graceful and forced close, owner drop, runtime crash, timeout/takeover,
proxy/descendant Job cleanup, hostile-profile isolation, private-session
cleanup, cookie deletion, window destruction, and exact profile removal.
Phase 2 is complete. Phase 3 now also generates and validates
application-specific source from the versioned
`windows-template-v1.0.0` template. The generated `hello-shiny` project
compiles and passes the real released-R/WebView owner gate. Native executable
packaging and clean-machine verification remain the next milestone.

## Later targets

- [ ] Add shinylive and Docker targets
- [ ] Add macOS portable runtime builders
