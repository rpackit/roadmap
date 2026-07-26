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
launcher.R --app <path> --port <port> --token <token> [--control <path>]
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
- `launcher.protocol_version: "1"`;
- `launcher.control: "optional-argument"`;
- an NDJSON `launcher.event_stream` on standard output with the exact
  `RPACKIT_EVENT ` prefix;
- HTTP-poll readiness gated by a matching `starting` event;
- explicit dependency installation state and strategy;
- explicit `launcher.network_token_enforced`.

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

Protocol version `1` emits `starting`, `stopping`, `stopped`, and structured
`error` events. Event payloads contain `protocol_version`, `event`, and a UTC
timestamp; they never contain the session token. The optional control path
must not exist at startup. Creating it requests a graceful Shiny shutdown.
The `starting` event's positive `pid` identifies the actual R runtime. It may
differ from the process tracked by `processx` when Windows uses an
`Rscript.exe` wrapper; public status records both as `runtime_pid` and `pid`.
Readiness therefore validates the event protocol, host, port, and token
enforcement state without falsely requiring those two PIDs to be equal.

The resource-stage launcher requires the token argument and exports it as
`RPACKIT_SESSION_TOKEN`, but records
`launcher.network_token_enforced: false`. A production desktop shell must add
HTTP and WebSocket token enforcement before this field can become `true`.
Consumers must not infer network authentication merely because the launcher
accepts `--token`.

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
