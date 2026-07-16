<p align="center">
  <img src="https://raw.githubusercontent.com/rachelslurs/storybook-oversight/main/assets/oversight-icon-128.png" alt="Oversight" width="96" height="96" />
</p>

<h1 align="center">oversight-lint-action</h1>

Run [`oversight-lint`](https://www.npmjs.com/package/oversight-lint) in GitHub
Actions to lint your Storybook MCP components manifest, and surface findings as
annotations on the pull request.

Your coding agent reads your components from the manifest Storybook's MCP server
generates. When a description never reaches that manifest, the agent sees a
component with no docs. This action fails the build when that happens, so the
regression stops at CI. It runs the same rules as
[`storybook-addon-oversight`](https://www.npmjs.com/package/storybook-addon-oversight),
which surfaces them live in Storybook while you work.

## Quick start

The action lints an already-built manifest, so build Storybook first:

```yaml
name: Docs coverage
on: pull_request
jobs:
  oversight:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install
      # Needs @storybook/addon-mcp; writes storybook-static/manifests/components.json.
      - run: pnpm build-storybook
      - uses: rachelslurs/oversight-lint-action@v1
        with:
          max-warnings: 0
```

## Prerequisites

- **A built manifest.** Run `storybook build` with
  [`@storybook/addon-mcp`](https://www.npmjs.com/package/@storybook/addon-mcp)
  enabled; it writes `storybook-static/manifests/components.json`. The action does
  not build Storybook for you.
- **Node 20.19+** on the runner (GitHub-hosted runners provide this).

## Inputs

| Input                | Default                                      | Description                                                       |
| -------------------- | -------------------------------------------- | ---------------------------------------------------------------- |
| `manifest`           | `storybook-static/manifests/components.json` | Path to the built `components.json`.                             |
| `max-warnings`       | _(no limit)_                                 | Fail if warnings exceed this count.                              |
| `config`             | _(none)_                                     | Path to an `oversight.config.json` (rules, `expectedExtractor`). |
| `expected-extractor` | `react-docgen-typescript`                    | Extractor the manifest should have used.                        |
| `version`            | `latest`                                     | `oversight-lint` version to run (npm dist-tag or exact version). |
| `working-directory`  | `.`                                          | Directory to run in.                                            |

Rule overrides go through the `config` file (an `oversight.config.json`), the same
one the CLI reads directly.

## Exit behavior and annotations

The action fails (exit 1) when an error-severity rule fires, or when warnings exceed
`max-warnings`. It exits 2 (also a failure) when it could not run: a missing,
unparseable, or unsupported manifest.

Findings are emitted as `::error`/`::warning`/`::notice` annotations. GitHub lists
them on the workflow run and the pull request's **Checks** tab, and shows them inline
on the **Files-changed** tab when the anchored line is part of the diff. Findings
carry no line numbers, so annotations anchor to the top of the component's stories
file, and GitHub caps them at ~10 per type per step.

## See also

- [`oversight-lint`](https://www.npmjs.com/package/oversight-lint) — the CLI this
  action wraps, with its full options and config.
- [`storybook-addon-oversight`](https://www.npmjs.com/package/storybook-addon-oversight)
  — the live-in-Storybook panel and Docs block.

## License

MIT
