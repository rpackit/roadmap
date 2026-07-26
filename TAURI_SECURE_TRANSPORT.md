# Tauri Secure Transport

**Status:** accepted architecture for the next native-desktop milestone

**Transport contract version:** `1`

**Last reviewed:** 2026-07-26

This document specifies how a generated Tauri application implements the
authenticated Shiny launcher protocol. It is a security contract and a release
gate, not an implementation claim. No Tauri application or Rust proxy is
currently delivered. The existing R-level `desktop_app_launch_config()` is a
development and third-party handoff; it is not a secret-transport mechanism for
the generated application.

## Decision

The generated desktop application will use an **authenticated native loopback
reverse proxy**. Direct WebView request-header injection is excluded from the
baseline architecture.

The browser-facing origin and the protected Shiny origin remain different:

```text
Tauri WebView
  http://rpackit-<launch-nonce>.localhost:<proxy-port>
             |
             | HttpOnly proxy-session cookie
             v
native loopback reverse proxy
  fixed upstream + injected Shiny-Shared-Secret
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

## Security boundary and secrets

Each launch creates two independent 256-bit random values and an independent
nonce of at least 128 random bits with the operating-system cryptographic
random-number generator:

- `S`, the **upstream Shiny secret**, is written through the existing
  current-account-private, one-time token-file contract. The launcher and the
  native proxy hold it. The proxy sends it upstream as exactly one
  `Shiny-Shared-Secret` header. It never enters the browser.
- `P`, the **proxy-session secret**, is held by the native process and the
  WebView cookie store. The browser sends it only as an HttpOnly session
  cookie to the exact proxy host. It is never sent upstream.
- `N`, the **launch nonce**, creates the random browser-facing hostname. It is
  never derived from `S` or `P` and is not an authentication credential.

Neither secret may appear in JavaScript, URLs, command-line arguments,
environment variables, manifests, lifecycle events, resource files, log
messages, errors, rpackit-generated crash annotations or output, or
redaction-safe diagnostics. The native process should minimize copies, compare
credentials without data-dependent early exit, and zeroize its in-memory
copies on shutdown on a best-effort basis. Release builds must disable
automatic collection or upload of memory-containing native or WebView crash
dumps unless a tested scrubber can exclude both secrets.

The preferred browser-facing host is a fresh
`rpackit-<N>.localhost` name for every launch. Cookies are not
port-scoped, so a stable `127.0.0.1` or `localhost` cookie namespace would
allow stale or concurrent desktop instances to share ambient credentials.
The actual WebView2 resolver must return only IPv4 or IPv6 loopback addresses,
and the proxy must bind compatible loopback listeners before navigation.
Resolution of the generated `.localhost` name, cookie scope and persistence,
and WebSocket cookie behavior are hard end-to-end gates. There is no silent
fallback to a stable loopback host if one of those gates fails.

## Bootstrap and HttpOnly cookie flow

The proxy exposes exactly one unauthenticated resource:
`GET` or `HEAD /__rpackit_bootstrap` with an empty query. Encoded, duplicated,
dot-segment, slash-variant, or query-bearing forms are rejected rather than
canonicalized.

That route:

- returns only a fixed, non-navigating loading document owned by rpackit;
- returns no body for `HEAD`;
- contains no application content, dynamic state, secret, or upstream data;
- never dials the upstream server;
- remains subject to the same exact-host, request-target, framing, header-size,
  connection, timeout, and secret-free-error rules as authenticated routes;
- uses a restrictive Content Security Policy, `Referrer-Policy: no-referrer`,
  `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`, and no external
  resources.

The startup sequence is:

1. Create the WebView hidden with a private, per-launch profile that is never
   reused. Incognito/private mode is preferred when the supported WebView API
   proves it behaves correctly.
2. Navigate it to the exact proxy-origin bootstrap URL.
3. After the bootstrap document is loaded, native Rust code uses the Tauri
   cookie API to install `P` for the exact generated hostname with path `/`,
   HttpOnly, `SameSite=Strict`, and no `Expires` or `Max-Age`. JavaScript is not
   involved.
4. Native code navigates the same WebView to `/`.
5. Show the window only after the authenticated application root is ready.

Setting the cookie after a same-origin bootstrap avoids relying on a
cross-site first navigation to carry a `SameSite=Strict` cookie. The proxy
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
- leakage of either secret through browser-visible state or generated
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
   `Shiny-Shared-Secret`, framing fields, or another protected end-to-end
   field. Strip all remaining hop-by-hop fields and their nominated fields
   before any protected header is synthesized.
4. Enforce bounded request-line and header sizes, connection and task counts,
   body rates, idle and total timeouts, and startup deadlines. Error responses
   and logs are secret-free and do not echo untrusted fields without safe
   encoding.

The exact bootstrap resource is exempt only from `P` authentication and
upstream forwarding. After common admission it accepts only `GET` or `HEAD`,
serves the fixed response described above, and closes without dialing the
application.

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
5. Remove every inbound `Shiny-Shared-Secret`, complete all hop-by-hop and
   protected-field normalization, set the fixed upstream `Host`, and only then
   inject exactly one `Shiny-Shared-Secret: S` as the final request mutation.
   Assert that exactly one protected header exists before writing upstream.
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
4. Generate independent `S`, `P`, and `N`; bind exclusive compatible loopback
   proxy listeners; prove `rpackit-<N>.localhost` resolves only to those
   loopback families; and select the upstream port under the launcher
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
schema-1 resource-bundle manifest. It records transport contract version `1`,
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

Build a hidden Tauri test shell and mock upstream. Prove bootstrap/cookie
authentication, document and subresource streaming, fetch/XHR, WebSocket
upgrade, redirect handling, and negative-request behavior. This phase adds no
public R API and produces no supported installer.

Exit requires every transport and leakage gate below to pass on the minimum
supported WebView2 runtime as well as the development runtime. The passing
Tauri, wry, WebView2, Hyper, hyper-util, and Tokio minima are then pinned in
the template and native metadata, together with the tested Windows OS
baseline.

### Phase 2: real launcher lifecycle

Replace the mock upstream with the existing protocol-2 launcher and
hello-shiny bundle. Implement Job Object ownership, readiness, window-close,
forced termination, cleanup, and crash recovery in a development executable.

Exit requires repeated start/stop and forced-crash tests with no surviving R
wrapper or runtime process and no reusable credential.

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

### Non-negotiable transport and leakage gates

All phases that exercise the transport must prove:

- missing, wrong, duplicated, or malformed `P` fails for HTTP and WebSocket
  before the upstream observes a connection or application session;
- initial navigation, CSS, JavaScript, images, XHR/fetch, streaming responses,
  and WebSocket handshakes make the upstream observe exactly one correct `S`;
- `document.cookie` cannot read `P`, and browser JavaScript cannot observe
  either secret;
- URL/history, process arguments, environment, manifests, lifecycle events,
  logs, errors, rpackit-generated crash annotations/output, and generated
  resources contain neither secret; automated memory-dump collection/upload
  is disabled unless independently proven secret-free;
- an external redirect/test collector receives neither secret nor the proxy
  cookie;
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
- concurrent application instances cannot authenticate to each other's proxy;
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
