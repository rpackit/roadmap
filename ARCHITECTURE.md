# Architecture

```text
Shiny application
       |
       v
 rpackit check
       |
       +-- portable desktop -> Tauri + bundled R + app library
       +-- static web ------> shinylive/webR compatibility gate
       +-- dynamic server --> generated Docker build context
```

The project deliberately uses target-specific runtimes. A browser build cannot
silently fall back to a native server, and a desktop build must not depend on a
system R installation.

The Windows desktop resource contract is:

```text
resources/
  R/
  app/
  launcher.R
  rpackit.json
```

The launcher binds Shiny only to `127.0.0.1`, chooses a random port, requires a
session token, and is terminated when the desktop shell exits.
