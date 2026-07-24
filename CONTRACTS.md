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
launcher.R --app <path> --port <port> --token <token>
```

The launcher must bind to `127.0.0.1`; `0.0.0.0` is prohibited for desktop
artifacts.

## App inspection

`rpackit::check_app()` never executes application source. Target states are
`yes`, `no`, or `maybe`, with at least one human-readable reason per target.
