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

If you set `expectedExtractor` on the Storybook addon, pass the same value here
(or via `config` / `oversight.config.json`) so `extractor-drift` can run. Omit it
to leave that rule off, matching the addon.

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
| `expected-extractor` | _(none)_                                     | Enables `extractor-drift`; same meaning as the addon's `expectedExtractor`. |
| `version`            | `latest`                                     | `oversight-lint` version to run (npm dist-tag or exact version). |
| `working-directory`  | `.`                                          | Directory to run in.                                            |

Rule overrides go through the `config` file (an `oversight.config.json`), the same
one the CLI reads directly.

## Pointing `manifest` at a ref-based manifest

`experimentalDocgenServer` emits a `v: 1` manifest, where the index holds `$ref`
pointers instead of inline docgen. `oversight-lint` has read that format since
0.5.0, and the `manifest` input takes it.

The index alone is not enough. Its sibling `services/` directory has to travel with
it, and the index has to stay in a directory named `manifests`, because a `$ref`
climbs exactly one level from there. Point at a copy of the index on its own and
every ref fails, which surfaces as `docgen-missing` for every component rather than
as a path error.

## A Storybook in a package directory

Point `working-directory` at the package and the annotations name paths from the
checkout root, so a Storybook under `storybook/` produces
`storybook/src/Avatar/Avatar.tsx`. Older versions named the path relative to the
package, and GitHub drops an annotation naming a file the repository does not have
without reporting anything. Resolving from the checkout root landed in
`oversight-lint` 0.6.0, which the default `version: latest` gives you.

Setting `working-directory` on this action alone will not get the job running. The
dependency install and the Storybook build need it too, and `actions/setup-node`
looks for the lockfile at the repository root, so a package carrying its own lockfile
needs `cache-dependency-path`:

```yaml
- uses: actions/checkout@v4
- uses: pnpm/action-setup@v4
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: pnpm
    cache-dependency-path: storybook/pnpm-lock.yaml
- run: pnpm install
  working-directory: storybook
- run: pnpm build-storybook
  working-directory: storybook
- uses: rachelslurs/oversight-lint-action@v1
  with:
    working-directory: storybook
```

## Exit behavior and annotations

The action fails (exit 1) when an error-severity rule fires, or when warnings exceed
`max-warnings`. It exits 2 (also a failure) when it could not run: a manifest that is
missing, unparseable, of a version this build does not know, or valid JSON that is not
a manifest at all. The last of those exited 0 reporting no findings before
`oversight-lint@0.6.0`, so a wrong-but-readable path used to pass green.

Findings are emitted as `::error`/`::warning`/`::notice` annotations, which GitHub
shows on the workflow run and the pull request's **Checks** tab. GitHub caps them at
~10 per type per step. The action also writes a findings table to the run's job
summary.

Each annotation names the component's own source file, so a missing description on a
component defined in `src/Avatar/Avatar.tsx` names that path. Two kinds of finding
name the stories file instead: `story-extraction-error`, which reports a failure on
one story, and any entry whose extraction produced no payload, which leaves no source
to name. `extractor-drift` is a manifest-level finding and carries no file.

Annotations printed by an action do not render beside your changed code on the
**Files changed** tab, whatever they carry. The behavior is upstream and tracked in
[issue #3](https://github.com/rachelslurs/oversight-lint-action/issues/3).

## See also

- [`oversight-lint`](https://www.npmjs.com/package/oversight-lint) — the CLI this
  action wraps, with its full options and config.
- [`storybook-addon-oversight`](https://www.npmjs.com/package/storybook-addon-oversight)
  — the live-in-Storybook panel and Docs block.

## License

MIT
