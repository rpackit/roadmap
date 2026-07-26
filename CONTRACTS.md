# Cross-repository contracts

## Portable R artifact

Artifact name:

```text
portable-r-{platform}-{arch}-{r_version}.{zip|tar.zst|tar.gz}
```

Metadata committed to the public portable-R registry must validate against
`runtime/schemas/portable-r-metadata-v1.schema.json` and contain an HTTPS
GitHub Releases URL, artifact SHA-256, archive format, and relative runtime
paths.

The package resolver accepts only registry entries whose status is
`verified`. Remote registries, metadata, and artifacts must use stable HTTPS
URLs without credentials, query strings, or fragments. A local registry may
use the same schema-v1 field set while pointing to local metadata and artifact
files for tests or air-gapped mirrors. This local transport override is an
rpackit resolver extension; such metadata is not publishable public-registry
metadata and is not accepted by the portable-R release validator. UNC/network
shares are rejected. The artifact SHA-256 is verified before extraction, and a
completed runtime is published atomically to a checksum-keyed user cache.

Automatic extraction currently supports ZIP artifacts only. Archive entries
must be ordinary files or directories, remain under the declared `r_home`,
and contain the declared `rscript` and runtime-local `library`. Other archive
formats fail closed until equivalent pre-extraction link and traversal checks
are implemented. The rpackit desktop subset requires one top-level `r_home`,
`r_home/bin/Rscript.exe` on Windows or `r_home/bin/Rscript` elsewhere, and
`r_home/library`.

Offline resolution may reuse a checksum-verified, structurally revalidated
cache entry only when its recorded registry source, platform, architecture,
and requested R version match. The cache is an executable trust boundary:
rpackit creates it with private permissions where supported and rejects
links/path escapes, but the owning user must not grant untrusted writers access
to it.

## Desktop launcher

```text
launcher.R --app <path> --port <port> --token-file <path> [--control <path>]
```

The launcher must bind to `127.0.0.1`; `0.0.0.0` is prohibited for desktop
artifacts.

The desktop resource bundle contract is version `1`:

```text
bundle/
  resources/
    R/
    app/
    launcher.R
    rpackit.json
```

`rpackit.json` must contain:

- `schema_version: "1"` and
  `bundle_type: "rpackit-desktop-resources"`;
- relative, traversal-free `app.path`, `runtime.path`, `runtime.rscript`, and
  `runtime.library` paths contained below `resources/`;
- `launcher.host: "127.0.0.1"`;
- `launcher.protocol_version: "2"` and `launcher.token: "private-file"` for
  newly prepared bundles;
- `launcher.control: "optional-argument"`;
- an NDJSON `launcher.event_stream` on standard output with the exact
  `RPACKIT_EVENT ` prefix;
- authenticated HTTP-poll readiness gated by a matching post-bind `listening`
  event;
- explicit dependency installation state and strategy;
- `launcher.network_token_enforced: true`; and
- a `launcher.authentication` descriptor with:
    - `scheme: "shiny-shared-secret"`;
    - `header: "Shiny-Shared-Secret"`;
    - `scope: ["http", "websocket"]`;
    - `token_transport: "private-file"`;
    - `token_in_url: false`; and
    - `minimum_shiny_version: "1.3.0"`.

New schema-v1 writers also record runtime provenance additively:

- `runtime.source` is `explicit` or `registry`;
- `runtime.r_version` is an exact major.minor.patch version when known and is
  required for a registry runtime;
- a registry runtime has `runtime.provenance` with non-empty `registry`,
  `metadata_source`, and `artifact_url`, a lowercase 64-character `sha256`,
  `archive_format: "zip"`, and logical `cache_hit`.

For backward compatibility, a schema-v1 bundle without `runtime.source` is
treated as an explicit-runtime bundle, and its missing `runtime.r_version` is
accepted. Registry provenance is never inferred for a legacy bundle.

Protocol version `2` emits `starting`, post-bind `listening`, `stopping`,
`stopped`, and structured `error` events. Launcher-generated payloads contain
`protocol_version`, `event`, and a UTC timestamp; their fields are redacted and
never contain the session credential. The optional control path must not exist at startup.
Creating it requests a graceful Shiny shutdown.

The `listening` event's positive `pid` identifies the actual R runtime and
includes the loopback host, selected port, and `token_enforced: true`. It may
differ from the process tracked by `processx` when Windows uses an
`Rscript.exe` wrapper; public status records both as `runtime_pid` and `pid`.
Readiness requires a matching protocol-2 `listening` event followed by a
successful HTTP request carrying the session credential. It does not falsely
require those two PIDs to be equal.

By default, `start_desktop_app()` creates a fresh 32-byte (256-bit) credential
with the operating system cryptographic random number generator (CSPRNG). A
caller-supplied credential is accepted only as 16 to 256 URL-safe ASCII
characters, and its entropy is the caller's responsibility. rpackit writes the
credential as the sole line of a current-account-private one-time file.
Windows DACLs must be restricted and verified for the current account plus
SYSTEM; POSIX directories must verify mode 0700 and files mode 0600. Only the
file path appears in `argv`; the credential itself is absent from `argv`, the
child environment, and the URL. The launcher consumes and deletes the file
before app or port validation and fails closed if it cannot do so.

Before accepting traffic, the launcher sets Shiny's shared secret. Shiny 1.3.0
or newer then requires the exact `Shiny-Shared-Secret` header for dynamic
HTTP, static resources, and WebSocket session acceptance. Missing and
incorrect HTTP credentials receive 403. A WebSocket upgrade can return 101
before Shiny closes the unauthenticated socket; the application server
function must not start for that connection. On the managing side, the live
credential is retained in the private process handle and the explicitly requested
`desktop_app_launch_config()` handoff; the credential-bearing Shiny runtime
also necessarily holds the secret while it serves requests. The credential is
not placed by rpackit in manifests or generated lifecycle-event fields, and is
removed from launcher error messages, returned status, returned logs/events,
and redaction-safe print methods. Raw private logs can contain arbitrary app
output and are not a secrecy boundary. Confirmed cleanup clears the managed
in-memory handoff and prevents future configurations; it cannot revoke a
configuration object already returned to the caller, which must discard every
copy after use.

Legacy protocol-1 schema-v1 manifests with `token: "required-argument"`,
HTTP-poll readiness gated by `starting`, no authentication descriptor, and
`network_token_enforced: false` remain valid for inspection. They are
deliberately refused by `start_desktop_app()` and must be rebuilt before
launch. Validation also requires the protocol-1 manifest to match genuine
protocol-1 launcher content; changing only a protocol-2 manifest is rejected.

`desktop_app_launch_config()` remains the current development and trusted
third-party native-shell handoff. It contains the token-free `url`, exact
`origin`, secret `headers`, `request_types` covering `http` and `websocket`,
and `follow_redirects = FALSE`; its redaction-safe print method does not print
the credential. A consumer of that R-level object owns every copy and must
discard it after use.

A generated Tauri application does not call this function or serialize its
returned Shiny secret across an R/native IPC boundary. The generated
executable is the runtime owner: it implements launcher protocol 2 directly,
creates independent upstream Shiny secret `S`, proxy-session secret `P`, and
one-time bootstrap secret `B`, writes `S` only through the one-time private
token file, and retains equivalent launch state only in native memory.

Its browser transport is an authenticated native loopback reverse proxy.
Native WebView2 code constructs exactly one request to the fixed bootstrap
route with `X-Rpackit-Bootstrap: B`. The proxy validates and atomically
consumes `B`, never dials upstream, and returns a fixed document with
`Set-Cookie: rpackit_proxy_v1=P; Path=/; HttpOnly; SameSite=Strict` and no
`Domain`, `Expires`, or `Max-Age`. The HTTP stack therefore creates a host-only
session cookie under the mandatory isolated per-launch profile. Missing,
wrong, duplicated, malformed, or replayed `B` sets no cookie and fails closed.
Cookie scope and non-persistence are acceptance gates rather than assumptions
about a wrapper API.

The proxy authenticates every non-bootstrap HTTP request and WebSocket upgrade
with `P` before dialing the fixed upstream, removes browser-supplied protected
and forwarding headers, and injects exactly one `Shiny-Shared-Secret: S` as the
final request mutation. `S` never enters the WebView, `P` is never sent
upstream, and `B` is accepted only for the one bootstrap request. A bare
unauthenticated proxy and a direct request interceptor are not accepted
implementations.

The native proxy never follows redirects, never forwards the protected header
externally, and fails closed on missing or ambiguous authentication, host,
origin, framing, upgrade, or resource-limit state. Transport contract version
`2`, including bootstrap, HTTP/WebSocket, leakage, lifecycle, and acceptance
rules, is specified in `TAURI_SECURE_TRANSPORT.md`. Future generated native
metadata records that contract version plus the template and pinned
Tauri/wry/WebView2 minima. The pre-release
[`rpackit-tauri`](https://github.com/rpackit/rpackit-tauri) Phase 1 spike
implements this boundary and has both development- and fixed-runtime evidence.
The active browser-escape, resource-abuse, malformed-upstream,
listener-overlap, and cross-process forced-crash profile gates pass on the
development runtime and the exact reviewed x64 Fixed Version Runtime
`149.0.4022.98`, in Debug and Release. Fixed startup verifies the pinned
Microsoft package and complete expanded tree before Tauri selects it; the
reports require that exact loaded version, contain no unproven gate, and set
`phase1_release_ready` to true. Phase 1 is complete. The spike is not a
generated application, supported installer, or production release; those
claims require later phases.

Directory-app `DisplayMode` and `IncludeWWW` `DESCRIPTION` semantics remain
active under the authenticated wrapper. Any incomplete private-file cleanup
must return a retryable managed process handle rather than reporting success.

Prepared output is atomic and non-overwriting: an existing output directory is
never replaced.

The launcher runtime dependency set is explicit: `jsonlite`, `later`, and
`shiny` are included in both the install plan and bundle manifest.

## Dependency plan

`rpackit::plan_dependencies()` never evaluates application source. Its
precedence is:

1. `renv.lock` for exact versions and sources;
2. `DESCRIPTION` for roles and version constraints;
3. parsed R calls for undeclared direct use.

The result retains normalized references and provenance. Read failures,
invalid R syntax, malformed DCF, and malformed lock records are errors rather
than silently incomplete plans.

The plan also validates installation determinism. When `renv.lock` exists,
every non-standard package required by parsed source or required DESCRIPTION
roles must have a lock record, and every locked version must satisfy its
DESCRIPTION constraints. `DESCRIPTION Remotes` without an exact lockfile is an
error rather than permission to install a same-named repository package.
Returned plan objects expose only the presence and count of remote
specifications, not their possibly credential-bearing text. URL credentials,
queries, and fragments in lockfile remote provenance are redacted before the
plan is returned or printed.

`prepare_desktop(install_packages = TRUE)` rejects dependency-plan errors
before resolving or copying a runtime. After `renv::restore()` or
`install.packages()`, every required DESCRIPTION package constraint is checked
against the copied runtime library before atomic publication. The manifest
records each constraint and whether installation verified it. Setting
`install_packages = FALSE` may produce an explicitly uninstalled inspection
bundle; it does not turn failed dependency evidence into a successful install.

Bundle validation reparses the copied application without evaluating it. New
manifests must contain exactly the required package set and DESCRIPTION
constraints derived from those copied files. Legacy schema-v1 manifests that
predate additive constraint fields remain readable. When runtime verification
is requested, package presence and all derived constraints are checked again
inside bundled R.

## App inspection

`rpackit::check_app()` never executes application source. Target states are
`yes`, `no`, or `maybe`, with at least one human-readable reason per target.
