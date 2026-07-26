# Cross-repository contracts

## Portable R artifact

Artifact name:

```text
portable-r-{platform}-{arch}-{r_version}.{zip|tar.zst|tar.gz}
```

Metadata committed to the public portable-R registry must validate against
`portable-r/schemas/portable-r-metadata-v1.schema.json` and contain an HTTPS
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

A native shell or loopback proxy consuming `desktop_app_launch_config()` must
inject the header into initial navigation, every same-origin subrequest, and
every WebSocket upgrade for the exact reported loopback origin. It must keep
the credential out of browser JavaScript and must not forward it across an
external redirect. The launch configuration contains the token-free `url`,
exact `origin`, secret `headers`, `request_types` covering `http` and
`websocket`, and `follow_redirects = FALSE`. Its redaction-safe print method
does not print the credential. A stock browser cannot implement this custom
header contract. The Tauri/proxy implementation that performs the handoff is
not yet delivered.

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

## App inspection

`rpackit::check_app()` never executes application source. Target states are
`yes`, `no`, or `maybe`, with at least one human-readable reason per target.
