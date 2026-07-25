# Roadmap

## Foundation

- [x] Create organization profile
- [x] Establish roadmap and contracts
- [x] Create R package, runtime index, Windows builder, and examples repositories

## App checker

- [x] Implement `doctor()`
- [x] Implement `check_app()`
- [x] Implement non-executing dependency plans from R syntax, DESCRIPTION, and
      renv.lock
- [x] Detect Shiny layouts, packages, renv, system calls, and static-web risks
- [x] Require the actual Tauri CLI and native prerequisites before `doctor()`
      reports desktop-build support

## Portable R Windows

- [x] Build from the official CRAN installer
- [x] Configure a runtime-local package library
- [x] Verify a path containing spaces and relocation
- [x] Generate ZIP, SHA-256, and metadata sidecar
- [x] Publish the first explicitly unsigned GitHub prerelease

## Next

- [x] Prepare atomic, versioned desktop resource bundles with portable R
- [x] Restore hello-shiny dependencies and smoke-test its bundled launcher over
      HTTP without using system R
- [ ] Enforce the session token for HTTP and WebSocket traffic
- [ ] Generate Tauri desktop projects
- [ ] Package hello-shiny as a native desktop executable
- [ ] Add shinylive and Docker targets
- [ ] Add macOS portable runtime builders
