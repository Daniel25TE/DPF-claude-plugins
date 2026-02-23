# DPF Claude Plugins

Internal Claude Code plugin marketplace for the DPF team.

## Available Plugins

| Plugin | Description |
|---|---|
| `ship-it` | Full PR workflow: branch, commit, rebase, push, open PR, post to Slack |

## Setup (one-time per machine)

**1. Register this marketplace:**

```bash
claude plugin marketplace add DPF-claude-plugins git@github.com:Daniel25TE/DPF-claude-plugins.git
```

**2. Install plugins:**

```bash
claude plugin install ship-it@DPF-claude-plugins
```

**3. Restart Claude Code** to load the new plugin.

## Using a plugin

```
/ship-it
```

## Adding new plugins

See `plugins/` for examples. Each plugin needs at minimum:

```
plugins/your-plugin/
├── .claude-plugin/
│   └── plugin.json
└── README.md
```
