# Platform policy

| Component | Development | Native verification |
|---|---|---|
| R package and metadata | any | CI matrix |
| Windows portable R | Windows | Windows x86_64 |
| macOS portable R | macOS | matching macOS architecture |
| Linux server bundle | any | Linux container runtime |
| Tauri artifact | any templates | target operating system |

Documentation and pure metadata changes may be made anywhere. A repository
must not claim native verification unless the relevant test ran on the target
platform.
