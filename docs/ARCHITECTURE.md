# Architecture

## Layers

```
niri/loom-niri
  core: DRM/KMS, renderers, layout, input, IPC, config, capture
  built-in modules: effects, navigation, cursor, agent, kde, vulkan
        │  niri IPC (unix socket, JSON)
        ▼
loom-companion ── one daemon, internal modules
  parking / rules / launch / swallow / marks / hooks / idle / rotate
        │
        ├── loom-session   (contexts, restore order)
        ├── loom-monitor   (output profiles, rollback)
        ├── loom-shot      (capture, annotate, record, pins)
        ├── loom-audio     (PipeWire routing)
        ├── loom-wallpaper (scenes, native layer surfaces)
        ├── loom-sidepanels(layer-shell panels and stacks)
        └── loom-portal    (XDG portal backend)
                │
                ▼
             Noctalia widgets (Luau, public CLI only)
```

`better` is a dispatcher, not a layer above these processes. It resolves the
requested binary and exits with its status.

## Boundaries

- **Compositor owns privileged state.** Seats, outputs, scanout, protocols,
  surfaces and the render loop stay inside niri.
- **Companion owns automation.** One IPC connection, one event stream and one
  state manager; modules may not open a second socket.
- **Tools own their domain.** A tool may call another tool's public CLI, but
  never links its internals.
- **Shell owns presentation.** Noctalia plugins call CLIs and read JSON; they do
  not hold business state.
- **Optional adapters.** A missing tool is reported as `missing`/`unavailable`,
  never turned into a fake success.

## Data and configuration

- niri: KDL at `$XDG_CONFIG_HOME/niri/config.kdl`; IPC socket from
  `$NIRI_SOCKET`.
- companion: optional config plus JSON state under
  `$XDG_STATE_HOME/loom-companion/`; UDS API with newline-delimited JSON.
- monitor: profiles in `$XDG_STATE_HOME/loom-monitor/`.
- session: contexts in `$XDG_STATE_HOME/loom-session/`.
- shot: history and pins in `$XDG_STATE_HOME/loom-shot/`.
- wallpaper: scene config and state in `$XDG_CONFIG_HOME` /
  `$XDG_STATE_HOME/loom-wallpaper/`.
- sidepanels: `$XDG_CONFIG_HOME/loom-sidepanels/config.json`; the stack
  manager has its own TOML.

Every state file is written to a temporary file and renamed, so a crash leaves
the previous version intact.

## Extension points

- niri actions and IPC events are additive and versioned.
- Companion actions are namespaced (`window.sticky.*`, `rules.*`, …) and owned
  by exactly one module; the registry rejects duplicate prefixes.
- Tools expose CLI commands with JSON output for machine use and a human output
  by default.
- No dynamic `.so` plugins. Rust has no stable ABI and a compositor cannot
  serialize its state per frame. Reuse happens at the IPC/CLI boundary or by
  compiling the module into the tree.

## Failure model

A companion failure must not take down the session. Each tool:
- reconnects to niri with backoff;
- reports partial restore steps (`applied`, `skipped`, `missing`, `failed`);
- keeps its previous state file when validation fails;
- logs the reason and exits non-zero rather than pretending to have applied a
  setting.
