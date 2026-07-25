# Cross-repository contracts

## Portable R artifact

Artifact name:

```text
portable-r-{platform}-{arch}-{r_version}.{zip|tar.zst|tar.gz}
```

The metadata sidecar must validate against
`portable-r/schemas/portable-r-metadata-v1.schema.json` and contain a release
URL, artifact SHA-256, archive format, and relative runtime paths.

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
