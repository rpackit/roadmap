# Tauri Secure Transport

**Status:** accepted architecture with a pre-release Windows transport spike

**Transport contract version:** `2`

**Last reviewed:** 2026-07-26

This document specifies how a generated Tauri application implements the
authenticated Shiny launcher protocol. It is a security contract and a release
gate. The pre-release Windows Phase 1 reference spike lives in
[`rpackit-tauri`](https://github.com/rpackit/rpackit-tauri); it is an
acceptance harness, not a generated application, supported installer, or
release-ready transport. The existing R-level
`desktop_app_launch_config()` is a development and third-party handoff; it is
not a secret-transport mechanism for the generated application.

## Decision

The generated desktop application will use an **authenticated native loopback
reverse proxy**. Direct WebView request-header injection is excluded from the
baseline architecture.

The browser-facing origin and the protected Shiny origin remain different. A
third credential authenticates only the first native bootstrap navigation:

```text
native shell
  one exact bootstrap request with X-Rpackit-Bootstrap: B
             |
             v
native loopback reverse proxy
  fixed bootstrap response + host-only Set-Cookie for P
             |
             v
Tauri WebView
  http://rpackit-<launch-nonce>.localhost:<proxy-port>
             |
             | HttpOnly proxy-session cookie P
             v
native loopback reverse proxy
  fixed upstream + injected Shiny-Shared-Secret S
             |
             v
protected Shiny server
  http://127.0.0.1:<upstream-port>
```

This is not a bare convenience proxy. A proxy that accepts unauthenticated
loopback requests would undo the protection already provided by the
`Shiny-Shared-Secret` boundary.

## Why direct header injection is excluded

The native shell must cover the initial document, every HTTP subresource or
fetch, and every WebSocket upgrade. The currently documented APIs do not
provide that complete, durable contract:

- Tauri's `WebviewBuilder::on_web_resource_request` can change responses for
  the `tauri` custom URI protocol; it is not a general external-HTTP request
  interceptor. `on_navigation` permits or rejects navigation but does not add
  request headers.
- wry's URL/header methods document headers for a supplied navigation. They do
  not promise that the headers are inherited by all subresources and
  WebSocket upgrades.
- WebView2 exposes `WebResourceRequested` for network interception, but
  WebSocket request interception remains the subject of an official tracked
  issue. An enum value or a platform-specific raw COM callback is not a
  portable security guarantee.
- A Tauri global forward-proxy setting is not this design. The shell must
  navigate to a dedicated reverse-proxy origin whose only possible upstream is
  the one Shiny process for the current launch.

A direct injector may be reconsidered only when a stable official API promises
document, subresource, fetch, and WebSocket coverage on every supported
platform; the generated application pins compatible minimum WebView/runtime
versions; and the same end-to-end negative and leakage tests below pass. Until
then, generator or platform code must not silently substitute an interceptor
for the authenticated reverse proxy.

Contract version 2 does use WebView2's native
`NavigateWithWebResourceRequest` operation for the one fixed bootstrap
navigation. That narrowly scoped operation carries `B` once; it is not a
general request interceptor, does not carry `S`, and is never relied on for
application documents, subresources, fetches, or WebSocket upgrades.

## Security boundary and secrets

Each launch creates three independent 256-bit random values and an independent
nonce of at least 128 random bits with the operating-system cryptographic
random-number generator:

- `S`, the **upstream Shiny secret**, is written through the existing
  current-account-private, one-time token-file contract. The launcher and the
  native proxy hold it. The proxy sends it upstream as exactly one
  `Shiny-Shared-Secret` header. It never enters the browser.
- `P`, the **proxy-session secret**, is held by the native process and the
  WebView cookie store. The browser sends it only as an HttpOnly session
  cookie to the exact proxy host. It is never sent upstream.
- `B`, the **bootstrap secret**, is held by the native process and the proxy.
  Native WebView2 code sends it in exactly one bootstrap request header. The
  proxy compares and consumes it atomically; it never enters JavaScript, a
  URL, a cookie, application content, or the upstream request.
- `N`, the **launch nonce**, creates the random browser-facing hostname. It is
  never derived from `S`, `P`, or `B` and is not an authentication credential.

None of `S`, `P`, or `B` may appear in JavaScript, URLs, command-line arguments,
environment variables, manifests, lifecycle events, resource files, log
messages, errors, rpackit-generated crash annotations or output, or
redaction-safe diagnostics. The native process should minimize copies, compare
credentials without data-dependent early exit, and zeroize its in-memory
copies on shutdown on a best-effort basis. Release builds must disable
automatic collection or upload of memory-containing native or WebView crash
dumps unless a tested scrubber can exclude all three secrets.

The preferred browser-facing host is a fresh
`rpackit-<N>.localhost` name for every launch. Cookies are not
port-scoped, so a stable `127.0.0.1` or `localhost` cookie namespace would
allow stale or concurrent desktop instances to share ambient credentials.
The actual WebView2 resolver must return only IPv4 or IPv6 loopback addresses,
and the proxy must bind compatible loopback listeners before navigation.
Resolution of the generated `.localhost` name, cookie scope and persistence,
and WebSocket cookie behavior are hard end-to-end gates. There is no silent
fallback to a stable loopback host if one of those gates fails.

## Authenticated bootstrap and HttpOnly cookie flow

The proxy exposes exactly one route that is exempt from `P` authentication:
`GET` or `HEAD /__rpackit_bootstrap` with an empty query. It is not
unauthenticated. It requires exactly one `X-Rpackit-Bootstrap: B` header from
the native request. Encoded, duplicated, dot-segment, slash-variant, or
query-bearing forms are rejected rather than canonicalized.

After common admission, that route:

- compares `B` without data-dependent early exit and atomically consumes it on
  the one successful bootstrap request;
- returns 401 for missing, wrong, malformed, duplicated, or replayed `B`,
  without setting `P`, dialing upstream, or echoing any credential;
- returns only a fixed, non-navigating loading document owned by rpackit;
- returns no body for `HEAD`;
- contains no application content, dynamic state, secret, or upstream data;
- never dials the upstream server;
- returns
  `Set-Cookie: rpackit_proxy_v1=P; Path=/; HttpOnly; SameSite=Strict` with no
  `Domain`, `Expires`, or `Max-Age`, so the WebView network stack creates a
  host-only session cookie;
- remains subject to the same exact-host, request-target, framing, header-size,
  connection, timeout, and secret-free-error rules as authenticated routes;
- uses a restrictive Content Security Policy, `Referrer-Policy: no-referrer`,
  `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, and no external
  resources.

The startup sequence is:

1. Create the WebView hidden with a private, per-launch profile that is never
   reused. Incognito/private mode is preferred when the supported WebView API
   proves it behaves correctly.
2. Native Rust code constructs one exact `GET` request for the proxy-origin
   bootstrap URL with WebView2 `CreateWebResourceRequest`, adds only
   `X-Rpackit-Bootstrap: B`, and invokes
   `NavigateWithWebResourceRequest`. JavaScript and a general request
   interceptor are not involved.
3. The proxy validates and atomically consumes `B`, then returns the fixed
   loading document and host-only `Set-Cookie` response above. `B` is never
   forwarded or retained for another request.
4. After the document loads, native code reads the cookie store and requires
   exactly one matching `P` with host-only scope, path `/`, HttpOnly,
   `SameSite=Strict`, and session lifetime.
5. Native code navigates the same WebView normally to `/`.
6. Show the window only after the authenticated application root is ready.

The HTTP response is the cookie-creation boundary. The Windows spike found
that the high-level Tauri cookie setter did not establish a verifiable
host-only cookie, consistent with WebView2's native cookie-manager operation
requiring a specified domain when adding a cookie directly. A normal
`Set-Cookie` response without `Domain` lets the WebView network stack create
the required host-only cookie without exposing `P` to JavaScript. The proxy
session cookie name is reserved to rpackit. The cookie is explicitly removed
and the per-launch profile is cleared during normal shutdown; the random host
prevents an old cookie from authenticating a later launch even after a crash.

Host-only and in-memory session behavior are requirements, not assumptions
about the current Tauri/wry/WebView2 cookie wrappers. The Windows transport
spike must prove that the cookie is not sent to a child hostname, has the
required flags, is absent after WebView/profile recreation, and cannot be
reused from disk after a crash. Host-only scope, HttpOnly, path `/`, and
`SameSite=Strict` must always pass; profile isolation cannot compensate for an
incorrect cookie scope or flag. If direct session-cookie behavior cannot prove
absence after recreation or crash, the private/incognito per-launch profile
becomes the authoritative persistence boundary and must pass those two tests.
Any failed scope/flag gate, or failure of both persistence mechanisms, rejects
this bootstrap design.

## WebView escape controls

The in-window navigation allowlist contains only the exact proxy origin.
Popup and new-WebView creation are denied by default independently of ordinary
navigation callbacks. Non-HTTP schemes are denied unless an explicit native
handler owns them. External HTTP or HTTPS links may open only in the operating
system browser after an explicit native decision and never receive proxy
cookies or headers. Downloads are denied or mediated by a native allowlist and
destination policy. Release builds disable devtools, context-menu escape
features, extensions, and remote debugging wherever the pinned runtime exposes
those controls.

## Threat model

### In scope

The design must resist:

- arbitrary unauthenticated loopback clients, including processes running as
  other local accounts;
- requests induced by external or cross-site pages;
- DNS, `Host`, absolute-URI, redirect, cookie, and request-header confusion;
- use of the reverse proxy as an open or general forward proxy;
- duplicate or smuggled authentication headers and cookies;
- leakage of `S`, `P`, or `B` through browser-visible state or generated
  diagnostics;
- startup races, graceful-close failures, crashes, child-process changes, and
  orphaned R descendants;
- resource exhaustion that would turn malformed requests into unbounded
  buffering or task creation.

### Explicitly out of scope

The boundary does not isolate the application from a malicious process running
as the same user, an administrator, a debugger, privileged loopback packet
capture, raw process-memory or operating-system crash dumps available to those
actors, or untrusted R/package code already executing in the
credential-bearing process. Availability-only denial of service is also out of
scope. Loopback HTTP is deliberately not treated as TLS.

These exclusions do not relax the no-leakage rules for normal product paths.

## HTTP forwarding invariants

Every request, including the bootstrap request, must first pass common
admission:

1. Accept only HTTP/1.1 on a compatible loopback listener, the exact generated
   `Host` plus proxy port, and an origin-form request target. Reject malformed
   or duplicate `Host`, `CONNECT`, `TRACE`, absolute-form and authority-form
   targets, non-WebSocket upgrades, and `h2c`.
2. Validate the raw request line, header syntax, duplicates, and framing before
   interpreting credentials or forwarding anything. Reject obsolete folding,
   conflicting or duplicate `Content-Length`, any
   `Content-Length`/`Transfer-Encoding` combination, invalid transfer coding,
   and ambiguous cookie or authentication fields.
3. Parse `Connection` tokens before stripping hop-by-hop fields. Reject a
   request if a token nominates `Host`, `Cookie`, `Origin`,
   `Shiny-Shared-Secret`, `X-Rpackit-Bootstrap`, framing fields, or another
   protected end-to-end field. Strip all remaining hop-by-hop fields and their
   nominated fields before any protected header is synthesized.
4. Enforce bounded request-line and header sizes, connection and task counts,
   body rates, idle and total timeouts, and startup deadlines. Error responses
   and logs are secret-free and do not echo untrusted fields without safe
   encoding.

The exact bootstrap resource is exempt only from `P` authentication and
upstream forwarding; it still requires the one-time `B`. After common
admission it accepts only `GET` or `HEAD`, validates and consumes `B`, serves
the fixed response described above, and closes without dialing the
application. `B` on any other route is rejected and never forwarded.

Every other HTTP request must additionally satisfy all of the following:

1. Authenticate one unambiguous `P` cookie before opening an upstream
   connection or writing any upstream byte. A missing, malformed, duplicated,
   or incorrect credential fails closed.
2. Require exactly the proxy origin on every unsafe-method request. A missing,
   `null`, malformed, duplicated, or non-exact `Origin` fails closed.
3. Use one fixed upstream authority, `127.0.0.1:<upstream-port>`, captured from
   the validated native launch state. Never derive, resolve, or redirect the
   upstream target from a browser request.
4. Remove the reserved proxy-session cookie while preserving unambiguous
   application cookies. Remove all inbound `Forwarded`, `X-Forwarded-*`,
   `X-Real-IP`, and `X-Original-*` fields; synthesize only a documented
   canonical value if Shiny demonstrably requires one.
5. Reject every inbound bootstrap credential and remove every inbound
   `Shiny-Shared-Secret`; complete all hop-by-hop and protected-field
   normalization, set the fixed upstream `Host`, and only then inject exactly
   one `Shiny-Shared-Secret: S` as the final request mutation. Assert that
   exactly one protected header exists before writing upstream.
6. Stream request and response bodies with bounded buffers and backpressure.
   Do not buffer an entire upload, download, or event stream.
7. Strip response hop-by-hop fields with the same parse-before-strip rules. An
   upstream `Set-Cookie` may not set the reserved name under any
   case/encoding variant. Preserve a host-only application cookie; when an
   explicit domain exactly equals the fixed upstream host, remove the `Domain`
   attribute to make it host-only at the proxy origin. Reject every other
   domain and malformed, duplicated, or ambiguous security attribute.
8. Do not follow redirects. Rewrite `Location` only when its authority exactly
   matches the fixed upstream origin; replace that authority with the proxy
   origin. Never attach `S` to an external destination.

The implementation must use a well-maintained HTTP parser and must not attempt
to repair malformed or ambiguous requests.

## WebSocket forwarding invariants

A WebSocket handshake has the same common admission and authentication
boundary as HTTP, with additional rules:

1. Reject every upgrade other than a normalized WebSocket handshake before an
   upstream dial. Require `GET`, HTTP/1.1, exact `Upgrade: websocket` and
   `Connection: Upgrade` semantics, `Sec-WebSocket-Version: 13`, one valid
   16-byte nonce in `Sec-WebSocket-Key`, exact proxy `Origin`, and one valid
   `P`. Authentication failure must create no application session.
2. Parse and validate offered subprotocols and extensions. The baseline proxy
   strips extension offers and accepts no extension response. Forward required
   `Sec-WebSocket-*` fields in normalized form, remove the proxy cookie and all
   browser-supplied protected or forwarding headers, then inject exactly one
   `Shiny-Shared-Secret: S` last.
3. A valid upstream response is HTTP `101` with correct
   `Connection`/`Upgrade`, the exact `Sec-WebSocket-Accept` derived from the
   client key, no unoffered subprotocol, and no unrequested or invalid
   extension. Normalize the downstream response and pair upgraded streams only
   after both handshakes pass.
4. After upgrade, tunnel bytes bidirectionally with backpressure; do not parse
   or rewrite WebSocket frames. With Hyper 1, wrap `Upgraded` streams in
   `hyper_util::rt::TokioIo` before Tokio `copy_bidirectional`; pin compatible
   Hyper, hyper-util, and Tokio versions in `Cargo.lock`.
5. Propagate close and EOF in both directions and enforce idle, handshake,
   byte-rate, connection, and task limits so abandoned upgrades do not live
   forever.

Cookie delivery on a real WebView2 WebSocket handshake is an acceptance test,
not an assumption derived from ordinary HTTP cookie behavior.

## Windows ownership and lifecycle

The generated Tauri executable is the runtime owner. On Windows it must:

1. Validate the schema-1 resource bundle and protocol-2 launcher before
   starting any process.
2. Resolve the bundled portable R and application resources without consulting
   a system R installation.
3. Create a per-launch session directory with a verified DACL limited to the
   current account and `SYSTEM`; create private token and control paths there.
4. Generate independent `S`, `P`, `B`, and `N`; bind exclusive compatible
   loopback proxy listeners; prove `rpackit-<N>.localhost` resolves only to
   those loopback families; and select the upstream port under the launcher
   contract. Write `S` only to the one-time private token file.
5. Create an unnamed Job Object with a non-inheritable handle. Set
   `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` before process assignment, do not set
   `BREAKAWAY_OK` or `SILENT_BREAKAWAY_OK`, and do not duplicate or inherit the
   Job handle.
6. Start bundled `Rscript` with `CREATE_SUSPENDED`, without
   `CREATE_BREAKAWAY_FROM_JOB`, and with an explicit inherited-handle allowlist
   containing only required lifecycle pipes. Assign the still-suspended
   process to the Job; if assignment or configuration fails, terminate it
   before it can execute. Resume its primary thread only after successful
   assignment. Assignment after the process has been allowed to execute is not
   accepted.
7. Parse protocol-2 NDJSON lifecycle events and wait for the post-bind
   `listening` event. Open a create-time-aware handle for the reported runtime
   PID, verify that exact process belongs to the Job and owns the expected
   loopback listener, then perform an authenticated direct readiness request
   with `S`. PID equality alone is never trusted because PIDs can be reused.
8. Start the proxy/bootstrap/cookie/navigation sequence and show the window
   only after authenticated readiness. Monitor both wrapper and verified
   runtime handles; if either terminates unexpectedly, immediately stop
   accepting proxy traffic, invalidate the browser session, and begin bounded
   cleanup.
9. On window close, prevent immediate process exit, create the control file,
   wait a bounded graceful period, terminate the Job Object as fallback, and
   verify that wrapper and runtime descendants are gone.
10. Close the proxy listener, delete the cookie and per-launch profile, remove
    private session files, and clear in-memory launch state. Cleanup failure is
    reported and remains retryable where possible.

The Job Object handle remains owned by the Tauri process for the whole launch.
Because it is unnamed, non-inheritable, never duplicated, and has no breakaway
policy, a native-process crash closes the last handle and invokes
kill-on-job-close rather than leaving a detached R process tree.

## Generated project boundary

R users do not author or patch Rust for each application. A future generator
will validate a prepared resource bundle and render a maintained, versioned
Tauri template similar to:

```text
generated-project/
  dist/
    index.html
  src-tauri/
    Cargo.toml
    Cargo.lock
    tauri.conf.json
    rpackit-native.json
    capabilities/
      default.json
    src/
      main.rs
      lifecycle.rs
      secure_proxy.rs
      secrets.rs
      manifest.rs
    resources/
      R/
      app/
      launcher.R
      rpackit.json
```

Tauri resources preserve the portable R directory layout. Portable R is a
resource tree, not a Tauri `externalBin` sidecar. The template pins a reviewed
Tauri minor series and commits `Cargo.lock`. Its capabilities are minimal: no
JavaScript shell, filesystem, arbitrary HTTP, or secret-bearing custom command
is exposed, and the global Tauri JavaScript object remains disabled.

`rpackit-native.json` is generated build metadata, separate from the existing
schema-1 resource-bundle manifest. It records transport contract version `2`,
the template version, resource schema and launcher protocol versions, and the
pinned minimum Tauri, wry, WebView2 runtime, and supported Windows versions. A
generator refuses an unknown transport contract rather than treating it as
compatible. Generated Tauri configuration sets
`bundle.windows.minimumWebview2Version` to the tested minimum, or packages a
tested fixed WebView2 runtime. Native startup independently checks the actual
runtime and fails closed below that minimum; it never assumes the installer
check still describes the current machine.

`prepare_desktop()` continues to own preparation of portable application
resources. `desktop_app_launch_config()` remains the current trusted
development/third-party handoff, but a generated application does not call it
or serialize its returned secret. The native owner implements protocol 2
directly: it generates `S`, writes the one-time token file, and retains the
equivalent launch URL, origin, fixed upstream, and secret only in native
memory. Project generation validates prepared resources, renders the native
project, and records template/runtime versions. Native compilation and
installer production remain a later orchestration layer.

## Delivery phases and hard gates

### Phase 1: Windows transport spike

The pre-release implementation in
[`rpackit-tauri`](https://github.com/rpackit/rpackit-tauri) contains a hidden
Tauri test shell, authenticated loopback proxy, and mock upstream. Its current
development-runtime evidence exercises the one-time `B` bootstrap, host-only
`P`, document and subresource streaming, fetch/XHR, WebSocket upgrade,
redirect isolation, and negative-request behavior. This phase adds no public R
API and produces no supported installer.

Exit requires every transport and leakage gate below to pass on the minimum
supported WebView2 runtime as well as the development runtime. The passing
Tauri, wry, WebView2, Hyper, hyper-util, and Tokio minima are then pinned in
the template and native metadata, together with the tested Windows OS
baseline. A development-runtime pass alone is not Phase 1 completion. Open
work includes the reviewed fixed minimum runtime, forced-crash
profile-persistence, real browser escape-path attempts, HTTP idle/body-rate and
WebSocket byte-rate abuse, malformed streamed-body and trailer cases, and the
complete fixed-runtime rerun. Strict upstream response-head validation already
passes valid raw HTTP and WebSocket baselines plus 16 HTTP and 16 WebSocket
malformed or policy-unsafe cases over real loopback. Every rejected case
returns the exact fixed secret-free 502 response without releasing an
unvalidated head, attacker canary, or WebSocket frame downstream, and no
negative case switches the downstream connection. WebSocket activity-idle
shutdown and Windows exact-loopback routing under IPv4 wildcard, IPv6 v6-only
wildcard, and IPv6 dual-stack wildcard overlap are covered by the
development-runtime harness.
The same dual-stack contender is tested against exact IPv4 and exact IPv6.
Wildcard bind success is not itself a failure: all four traffic paths must
reach the exact proxies and every wildcard accept count must remain zero.

### Phase 2: real launcher lifecycle

Replace the mock upstream with the existing protocol-2 launcher and
hello-shiny bundle. Implement Job Object ownership, readiness, window-close,
forced termination, cleanup, and crash recovery in a development executable.

Exit requires repeated start/stop and forced-crash tests with no surviving R
wrapper or runtime process and no reusable credential.

Phase 1 owns transport, browser, cookie/profile, leakage, and mock-upstream
gates. Phase 2 owns launcher protocol 2, Job Object/process-tree, readiness,
and real-R lifecycle gates, while re-running the Phase 1 transport gates
against the real launcher. A Phase 2 lifecycle item does not retroactively
block recording a Phase 1 development-gate result, but neither phase may be
called complete until all gates assigned to it pass.

### Phase 3: generated native project

Render the maintained Tauri template around a validated resource bundle and
build the hello-shiny Windows artifact.

Exit requires installation and launch on a clean Windows machine without a
system R installation, including paths with spaces and a relocated install.

### Phase 4: release workflow

Generate the native release workflow and publish the artifact with checksum,
build provenance, dependency/runtime versions, and explicit signing status.

Exit requires reproduction from the recorded inputs and verification of every
published checksum and attestation.

### Non-negotiable Phase 1 transport and leakage gates

Phase 1, and every later phase that exercises the transport, must prove:

- missing, wrong, duplicated, malformed, or replayed `B` fails the bootstrap
  before `P` is set or the upstream observes a connection; exactly one valid
  bootstrap succeeds, and `B` is rejected on every other route;
- missing, wrong, duplicated, or malformed `P` fails for HTTP and WebSocket
  before the upstream observes a connection or application session;
- initial navigation, CSS, JavaScript, images, XHR/fetch, streaming responses,
  and WebSocket handshakes make the upstream observe exactly one correct `S`;
- `document.cookie` cannot read `P`, and browser JavaScript cannot observe
  `S`, `P`, or `B`;
- URL/history, process arguments, environment, manifests, lifecycle events,
  logs, errors, rpackit-generated crash annotations/output, and generated
  resources contain none of `S`, `P`, or `B`; automated memory-dump
  collection/upload is disabled unless independently proven secret-free;
- an external redirect/test collector receives none of `S`, `P`, `B`, or the
  proxy cookie;
- malformed bootstrap variants, `CONNECT`, `TRACE`, non-WebSocket upgrades,
  `h2c`, absolute/authority-form targets, incorrect `Host`, absent or
  cross-origin unsafe-request `Origin`, protected `Connection` tokens,
  forwarding-header spoofing, header smuggling, and oversized inputs fail
  closed;
- WebSocket version/key/accept, subprotocol, and extension negative cases fail
  before tunnelling;
- the random `.localhost` name resolves only to compatible IPv4/IPv6 loopback
  listeners, the bootstrap-to-root `SameSite=Strict` flow works, and the real
  WebView sends the HttpOnly cookie on WebSocket upgrade;
- `P` is not sent to a child hostname, is absent after WebView/profile
  recreation, and cannot be reused from disk after either clean shutdown or a
  crash;
- ordinary navigation, new-window/popup, custom-scheme, download, devtools,
  extension, and remote-debugging escape paths satisfy their independent
  deny-by-default policies;
- concurrent application instances cannot authenticate to each other's proxy,
  and listener/port takeover attempts fail closed.

### Non-negotiable Phase 2 lifecycle gates

Phase 2 and later native-runtime phases must additionally prove:

- graceful close, forced close, shell crash, launcher crash, and readiness
  timeout leave no orphan process, listener, credential, or reusable profile;
  PID reuse, port takeover, wrapper exit, failed Job assignment, inherited
  handles, and attempted breakaway also fail closed.

Failure of any gate blocks project generation from being described as secure
or release-ready. It must not be converted into a warning or an insecure
fallback.

## Official references

- [Tauri `WebviewBuilder`](https://docs.rs/tauri/latest/tauri/webview/struct.WebviewBuilder.html)
- [Tauri `Webview` cookie API](https://docs.rs/tauri/latest/tauri/webview/struct.Webview.html)
- [Tauri `WebviewBuilder::on_new_window`](https://docs.rs/tauri/latest/tauri/webview/struct.WebviewBuilder.html#method.on_new_window)
- [Tauri `WebviewBuilder::on_download`](https://docs.rs/tauri/latest/tauri/webview/struct.WebviewBuilder.html#method.on_download)
- [Tauri `WebviewBuilder` data-directory and incognito controls](https://docs.rs/tauri/latest/tauri/webview/struct.WebviewBuilder.html)
- [wry `WebViewBuilder`](https://docs.rs/wry/latest/wry/struct.WebViewBuilder.html)
- [wry `WebView`](https://docs.rs/wry/latest/wry/struct.WebView.html)
- [WebView2: custom management of network requests](https://learn.microsoft.com/en-us/microsoft-edge/webview2/how-to/webresourcerequested)
- [WebView2 resource contexts](https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/winrt/microsoft_web_webview2_core/corewebview2webresourcecontext)
- [WebView2 custom navigation with `CreateWebResourceRequest` and `NavigateWithWebResourceRequest`](https://learn.microsoft.com/en-us/microsoft-edge/webview2/how-to/webresourcerequested#constructing-a-custom-request-and-navigating-using-that-request)
- [RFC 6761 section 6.3: `.localhost` and its subdomains](https://www.rfc-editor.org/rfc/rfc6761.html#section-6.3)
- [Chromium: always treat `.localhost` as loopback](https://chromium.googlesource.com/chromium/src/+/5d131a1fd9b808c5fd08c45f8299e669b13ec393%5E%21/)
- [Microsoft: WebView2 uses the Edge/Chromium runtime](https://learn.microsoft.com/en-us/microsoft-edge/webview2/)
- [WebView2 `ICoreWebView2CookieManager`](https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/win32/icorewebview2cookiemanager)
- [WebView2 `ICoreWebView2Cookie`](https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/win32/icorewebview2cookie)
- [WebView2Feedback issue 4303: WebSocket interception](https://github.com/MicrosoftEdge/WebView2Feedback/issues/4303)
- [Tauri bundled resources](https://v2.tauri.app/develop/resources/)
- [Tauri Windows installer and minimum WebView2 version](https://v2.tauri.app/distribute/windows-installer/)
- [Hyper HTTP upgrades](https://docs.rs/hyper/latest/hyper/upgrade/index.html)
- [hyper-util `TokioIo`](https://docs.rs/hyper-util/latest/hyper_util/rt/tokio/struct.TokioIo.html)
- [Tokio `copy_bidirectional`](https://docs.rs/tokio/latest/tokio/io/fn.copy_bidirectional.html)
- [Windows Job Objects](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)
- [Windows `AssignProcessToJobObject`](https://learn.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-assignprocesstojobobject)
- [Windows `CreateJobObjectW`](https://learn.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-createjobobjectw)
- [Windows `SetInformationJobObject`](https://learn.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-setinformationjobobject)
- [Windows `CreateProcessW`](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw)
- [Windows `ResumeThread`](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-resumethread)
- [Windows `IsProcessInJob`](https://learn.microsoft.com/en-us/windows/win32/api/jobapi/nf-jobapi-isprocessinjob)
- [RFC 9110, HTTP semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9112, HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112)
- [RFC 6455, WebSocket](https://www.rfc-editor.org/rfc/rfc6455)
- [RFC 6761, `.localhost` names](https://www.rfc-editor.org/rfc/rfc6761#section-6.3)
- [RFC 6265, cookie domain and path scope](https://www.rfc-editor.org/rfc/rfc6265)
