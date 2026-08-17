# iconplatforms-tools

Internal [Claude Code](https://claude.com/claude-code) plugin marketplace for Icon Platforms — git helpers and Talos/Angular workflow shortcuts, packaged as installable slash commands.

## What's in here

This repo is a Claude Code **plugin marketplace**: [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json) lists the plugins, each of which lives under [`plugins/`](plugins/) with its own commands, `README.md`, and `.claude-plugin/plugin.json`.

| Plugin | Command(s) | What it does |
| --- | --- | --- |
| [`git-tools`](plugins/git-tools/) | `/commit` | Stage all changes, commit with a message that matches your repo's style, and push to the current branch. |
| [`talos-tools`](plugins/talos-tools/) | `/talos-init`, `/talos-feature` | Bootstrap and scaffold [Talos](https://www.npmjs.com/package/@saicongames/talos-integration-angular)-backed Angular/Ionic apps. |

### `talos-tools` in a nutshell

- **`/talos-init <env-values-block>`** — one-time bootstrap: writes the `.npmrc` for the `@saicongames` private registry (referencing `${NPM_TOKEN}`), installs `@saicongames/talos-integration-angular`, writes `src/environments/environment.ts`, and wires `TalosModule.forRoot(...)` into `AppModule`. The npm auth token is read from the **`NPM_TOKEN` environment variable** (set it before running, with read access to the `@saicongames` scope) rather than passed as an argument.
- **`/talos-feature <entity> [mockup]`** — scaffolds a typed service + component + view for a Talos entity by reading the real API off `TalosService` (no invented methods). If the packages aren't installed yet, it points you at `/talos-init`.

## Installation

Add this repo as a marketplace, then install the plugins you want:

```bash
# Add the marketplace (from a local clone or the git URL)
/plugin marketplace add chamarakisg/claude-plugins

# Install a plugin
/plugin install talos-tools@iconplatforms-tools
/plugin install git-tools@iconplatforms-tools
```

Once installed, the slash commands are available in any project. See each plugin's own README for details:

- [git-tools/README.md](plugins/git-tools/README.md)
- [talos-tools/README.md](plugins/talos-tools/README.md)

## Repository layout

```
.
├── .claude-plugin/
│   └── marketplace.json        # marketplace manifest — lists the plugins
└── plugins/
    ├── git-tools/
    │   ├── .claude-plugin/plugin.json
    │   ├── commands/commit.md
    │   └── README.md
    └── talos-tools/
        ├── .claude-plugin/plugin.json
        ├── commands/talos-init.md
        ├── commands/talos-feature.md
        └── README.md
```

## Author

George Chamarakis (hamarakisg@iconplatforms.com)
