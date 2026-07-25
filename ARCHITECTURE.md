# Architecture

## Current implementation

```text
Shiny application
       |
       v
 check_app() + plan_dependencies()
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
`Rscript`, waits for both a versioned launcher event and an HTTP response, and
uses a private control file for graceful shutdown. A timeout falls back to
asking `processx` to terminate its known process tree. Cleanup is confirmed for
the tracked wrapper and the create-time-aware runtime handle captured from the
launcher event; other descendant membership is not independently claimed.
Status distinguishes the wrapper PID from the runtime PID because those are
different processes in the verified Windows portable-R layout.

The session token is currently a correlation/bootstrap value passed to the
application. It is not HTTP or WebSocket authentication. Public status objects
contain only the token-free loopback URL, and the manifest records
`network_token_enforced: false`.

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
system R installation. The Tauri shell, network token enforcement,
static-web builder, and server builder remain target work rather than current
exported capabilities.
