# devkit-plugins

A Claude Code plugin marketplace with **devkit** — a developer toolkit.

Current skills:

| Skill | Command | What it does |
| ----- | ------- | ------------ |
| review-sps | `/devkit:review-sps` | Reviews a feature/change with priorities **Security > Performance > Simplicity**, then applies fixes only when you approve them. |

> More skills (e.g. a code generator) will live under the same `devkit` namespace: `/devkit:gen`, etc.

## Install

From GitHub (after you push this repo):

```
/plugin marketplace add julles/devkit-plugins
/plugin install devkit@devkit-plugins
```

Or from a local clone:

```
/plugin marketplace add /path/to/devkit-plugins
/plugin install devkit@devkit-plugins
```

Then restart Claude Code (or run `/reload-plugins`).

### Try it without installing

```
claude --plugin-dir ./plugins/devkit
```

## Usage: review-sps

Point it at a diff, files, or a named feature:

```
/devkit:review-sps                    # reviews your uncommitted git diff
/devkit:review-sps internal/auth/     # reviews a folder
/devkit:review-sps login flow         # finds the feature's files and reviews them
```

Flow:

1. It reads the target repo's `CLAUDE.md` for context (language, conventions).
2. Reviews against the SPS ruleset and lists **numbered findings**, grouped
   Security → Performance → Simplicity, each with a severity
   (Critical / High / Medium / Low), `file:line`, the problem, and a suggested fix.
3. **Nothing is changed yet.** You decide what to apply:

   ```
   apply #1 #3
   apply all security
   skip
   ```

It only edits the files for findings you approve.

## Repo layout

```
devkit-plugins/
├── .claude-plugin/
│   └── marketplace.json          # marketplace catalog
├── plugins/
│   └── devkit/
│       ├── .claude-plugin/
│       │   └── plugin.json        # plugin manifest
│       └── skills/
│           └── review-sps/
│               └── SKILL.md       # the review-sps skill
└── README.md
```

## License

MIT — see [LICENSE](LICENSE).
