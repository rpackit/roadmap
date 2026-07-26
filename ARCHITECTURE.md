# Architecture

## Current implementation

```text
Shiny application
       |
       v
 check_app() + plan_dependencies()
       |
       v
 verified portable-R registry -> SHA-256 cache
       |
       v
 prepare_desktop()
       |
       v
 bundled R + app library + launcher + manifest
       |
       v
 start_desktop_app() -> loopback Shiny process -> stop_desktop_app()
```

The implemented Windows desktop resource contract is:

```text
bundle/
  resources/
    R/
    app/
    launcher.R
    rpackit.json
```

The lifecycle manager chooses a high loopback port, starts only the bundled
`Rscript`, waits for a protocol-2 post-bind `listening` event, and proves
readiness with an authenticated HTTP request. It uses a private control file
for graceful shutdown. A timeout falls back to asking `processx` to terminate
its known process tree. Cleanup is confirmed for the tracked wrapper and the
create-time-aware runtime handle captured from the launcher event; other
descendant membership is not independently claimed. Status distinguishes the
wrapper PID from the runtime PID because those are different processes in the
verified Windows portable-R layout.

Each default launch creates a fresh 256-bit credential from the operating
system cryptographic random source. rpackit places it in a
current-account-private one-time file and passes only that path as
`--token-file`; the launcher consumes and deletes the file before app or port
validation. Windows DACLs are restricted and verified for the current account
plus SYSTEM; POSIX permissions are verified as directory mode 0700 and file
mode 0600. The credential is never placed in the child command line, child
environment, or URL. The launcher configures
Shiny's `Shiny-Shared-Secret` header authentication for dynamic and static HTTP
as well as WebSocket traffic. New manifests describe this mechanism and record
`network_token_enforced: true`. Protocol-1 unauthenticated bundles remain
inspectable but cannot be launched.

Protocol validation requires manifest and launcher content to agree. A secure
protocol-2 launcher cannot be relabeled as legacy merely by editing its
manifest. If process termination succeeds but private-file removal does not,
the error retains a retryable managed handle. The authenticated wrapper also
preserves Shiny directory-app `DESCRIPTION` handling.

The live process handle necessarily retains the credential to perform
authenticated readiness and create `desktop_app_launch_config()`. rpackit does
not place it in manifests or generated event fields; public status, returned
logs/events, launcher error messages, and redaction-safe print methods remove
it. Raw private logs can contain arbitrary trusted app output and are not a
secrecy boundary. Confirmed cleanup clears the managed handoff and prevents a
new one, but cannot revoke an already returned launch-configuration object;
the consumer must discard every copy.

That handoff describes the current R-managed development path. A generated
Tauri executable does not receive or serialize
`desktop_app_launch_config()`; it owns protocol-2 process launch, creates the
one-time token file itself, and retains equivalent state only in native memory.

This boundary prevents unauthenticated loopback clients from entering a Shiny
session; it is not isolation from the local account or the code being run.
The threat model excludes malicious same-user processes, administrator or
debugger access, and untrusted app or package code inside the credential-
bearing R process. Loopback HTTP is not TLS. The selected Tauri transport is an
authenticated native loopback reverse proxy, not a direct WebView interceptor.
It creates independent upstream secret `S`, proxy-session secret `P`, and
one-time bootstrap secret `B`. Native WebView2 code sends `B` only on the exact
bootstrap request; the fixed HTTP response creates host-only HttpOnly cookie
`P`, and the proxy injects `S` only after `P` authenticates later HTTP or
WebSocket traffic. A bare unauthenticated proxy would collapse the existing
loopback-client boundary. The complete threat model and forwarding invariants
are transport contract version `2` in `TAURI_SECURE_TRANSPORT.md`.

The pre-release
[`rpackit-tauri`](https://github.com/rpackit/rpackit-tauri) Windows spike
implements this boundary and has current-development-runtime evidence. Cookie
scope and clean profile recreation are empirical gates, not wrapper
assumptions. A reviewed fixed minimum WebView2 runtime, forced-crash
profile-persistence result, browser escape attempts, resource-abuse and
malformed-upstream cases, and the Windows wildcard-listener overlap gate are
still required before Phase 1 can complete.

`doctor()` may report that the external Tauri toolchain is ready on a machine.
That diagnostic does not mean project generation, a native build API, or a
desktop artifact is implemented.

## Target architecture

```text
Shiny application
       |
       v
 rpackit inspection
       |
       +-- portable desktop -> Tauri + bundled R + app library
       +-- static web ------> shinylive/webR compatibility gate
       +-- dynamic server --> generated Docker build context
```

The project deliberately uses target-specific runtimes. A browser build cannot
silently fall back to a native server, and a desktop build must not depend on a
system R installation. Backend network-token enforcement is implemented. The
authenticated Tauri reverse proxy and bootstrap/cookie flow now have a
pre-release Phase 1 spike, but real launcher integration, Windows process-tree
ownership, project generation, native packaging, the static-web builder, and
the server builder remain target work rather than current exported
capabilities.
