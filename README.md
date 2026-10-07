# Yarn: The Complete Engineer's Guide (and How It Compares to npm)

A practical, in-depth reference for the Yarn package manager: what it is, how it works internally, how to use it day to day, how to run it in CI and monorepos, and how it stacks up against npm (with a short note on pnpm).

> **Version note:** Yarn's feature set differs significantly between **Yarn Classic (1.x)** and **Yarn Berry (2.x, 3.x, 4.x)**. Commands and behaviors below are labeled where they differ. Always check the [official docs](https://yarnpkg.com) for the release you're using, since details change between minor versions.

---

## Table of Contents

1. [What Is Yarn?](#1-what-is-yarn)
2. [History and Versions](#2-history-and-versions)
3. [Installation and Version Management](#3-installation-and-version-management)
4. [Core Concepts](#4-core-concepts)
5. [Command Cheat Sheet (Yarn vs npm)](#5-command-cheat-sheet-yarn-vs-npm)
6. [Install Strategies: node_modules, PnP, pnpm Linker](#6-install-strategies)
7. [Plug'n'Play (PnP) in Depth](#7-plugnplay-pnp-in-depth)
8. [Zero-Installs and the Offline Cache](#8-zero-installs-and-the-offline-cache)
9. [The Lockfile](#9-the-lockfile)
10. [Configuration (`.yarnrc.yml`)](#10-configuration-yarnrcyml)
11. [Dependency Protocols and Resolution Control](#11-dependency-protocols-and-resolution-control)
12. [Workspaces and Monorepos](#12-workspaces-and-monorepos)
13. [Scripts, Binaries, and Running Tools](#13-scripts-binaries-and-running-tools)
14. [Plugins and Constraints](#14-plugins-and-constraints)
15. [Security](#15-security)
16. [CI/CD Usage](#16-cicd-usage)
17. [Publishing Packages](#17-publishing-packages)
18. [Yarn vs npm: Detailed Comparison](#18-yarn-vs-npm-detailed-comparison)
19. [Yarn vs npm vs pnpm (Quick View)](#19-yarn-vs-npm-vs-pnpm-quick-view)
20. [Migration Guides](#20-migration-guides)
21. [Best Practices](#21-best-practices)
22. [Troubleshooting](#22-troubleshooting)
23. [Decision Guide](#23-decision-guide)
24. [Resources](#24-resources)

---

## 1. What Is Yarn?

Yarn is a package manager for JavaScript and TypeScript projects. It installs, updates, and removes dependencies declared in `package.json`, resolves the full dependency tree, records exact resolved versions in a lockfile, and runs project scripts. It is compatible with the **npm registry** (`registry.npmjs.org`), so every package published to npm is installable with Yarn.

Core promises:

- **Deterministic installs:** the same lockfile yields the same dependency tree on every machine.
- **Speed:** parallel downloads, a global/offline cache, and (in modern Yarn) installs that skip creating `node_modules` entirely.
- **Correctness:** strict dependency resolution (especially with PnP) that prevents code from importing packages it never declared ("phantom dependencies").
- **First-class monorepo support** through workspaces.

---

## 2. History and Versions

| Era | Name | Version | Status |
|---|---|---|---|
| 2016 | **Yarn Classic** | 1.x (latest 1.22.x) | Maintenance mode. Receives only critical fixes. |
| 2020 | **Yarn Berry** | 2.x | Rewrite in TypeScript; introduced PnP by default, plugins, constraints. |
| 2021 | Yarn Berry | 3.x | Performance and ergonomics improvements. |
| 2023+ | Yarn Berry | **4.x** | Current line. Requires a modern Node.js (18+). |

Background:

- Yarn was created at Facebook (now Meta) with Google, Exponent, and Tilde to fix npm's problems at the time: non-deterministic installs, slow installs, and weak offline support. It pushed npm to adopt lockfiles (npm 5, 2017) and many other improvements.
- "Berry" is the codename for Yarn 2+. It is a ground-up rewrite, **not** a drop-in upgrade from 1.x. The `yarn` binary you invoke is a thin shim that loads the version pinned by the project.
- Classic and Berry have different configuration files (`.yarnrc` vs `.yarnrc.yml`), lockfile formats, and many command names/flags.

**Rule of thumb:** for new projects choose Yarn 4 (Berry) or npm/pnpm. Avoid starting new projects on Yarn 1.x unless a constraint forces it.

---

## 3. Installation and Version Management

### Recommended: Corepack

[Corepack](https://nodejs.org/api/corepack.html) is a tool that manages package manager versions per project. It shipped with Node.js for many releases. Newer Node.js releases (starting with Node 25) no longer bundle it, so you may need to install it yourself.

```bash
# Install Corepack if your Node.js doesn't bundle it
npm install -g corepack

# Enable shims for yarn/pnpm
corepack enable

# Pin Yarn for a project (writes the packageManager field in package.json)
corepack use yarn@stable
# or a specific version
corepack use yarn@4.9.2
```

Pinning happens through the `packageManager` field:

```json
{
  "name": "my-app",
  "packageManager": "yarn@4.9.2"
}
```

Anyone running `yarn` in the project then automatically gets that exact version, with no global install and no version drift between developers and CI.

### Alternative: `yarn set version`

From within a project (with any Yarn available):

```bash
yarn set version stable      # latest Yarn Berry
yarn set version 4.9.2       # specific version
yarn set version classic     # Yarn 1.x
```

This downloads the release into `.yarn/releases/` and records it as `yarnPath` in `.yarnrc.yml`. Commit that file so the whole team uses the same version.

### Other options

- `npm install -g yarn` installs Yarn Classic 1.x (not Berry). Fine for bootstrapping; use `yarn set version` afterward.
- System packages (Homebrew, apt, Chocolatey) tend to provide Classic and lag behind.

### Verify

```bash
yarn --version
```

---

## 4. Core Concepts

- **`package.json`**: declares the project, dependencies (`dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`), and scripts.
- **Lockfile (`yarn.lock`)**: records the exact resolved version, checksum, and dependency graph.
- **Cache**: compressed archives of every package version. In Berry it is a first-class, optionally committed artifact (`.yarn/cache`).
- **Linker**: the strategy for making installed packages available to Node.js (PnP, `node_modules`, or pnpm-style).
- **Workspaces**: multiple packages in one repository sharing one lockfile.
- **Resolution vs. fetch vs. link**: Yarn's install is three phases. *Resolution* computes the dependency graph and updates the lockfile; *fetch* downloads archives into the cache; *link* makes packages importable.
- **Descriptors vs. locators**: Berry terminology. A *descriptor* is a request (`lodash@^4.17.0`); a *locator* is a concrete resolution (`lodash@npm:4.17.21`). You'll see both in `yarn.lock` and error messages.

---

## 5. Command Cheat Sheet (Yarn vs npm)

### Everyday commands

| Task | Yarn 4 (Berry) | Yarn 1 (Classic) | npm |
|---|---|---|---|
| Install all dependencies | `yarn` / `yarn install` | `yarn` | `npm install` |
| Reproducible CI install | `yarn install --immutable` | `yarn install --frozen-lockfile` | `npm ci` |
| Add a dependency | `yarn add pkg` | `yarn add pkg` | `npm install pkg` |
| Add a dev dependency | `yarn add -D pkg` | `yarn add -D pkg` | `npm install -D pkg` |
| Add a peer dependency | `yarn add -P pkg` | `yarn add -P pkg` | `npm install --save-peer pkg` |
| Add exact version | `yarn add -E pkg` | `yarn add -E pkg` | `npm install -E pkg` |
| Remove a dependency | `yarn remove pkg` | `yarn remove pkg` | `npm uninstall pkg` |
| Upgrade a package | `yarn up pkg` | `yarn upgrade pkg` | `npm update pkg` |
| Upgrade interactively | `yarn upgrade-interactive` | `yarn upgrade-interactive` | (none built-in; use `npm-check-updates`) |
| Show outdated packages | `yarn upgrade-interactive` (shows outdated) | `yarn outdated` | `npm outdated` |
| Why is X installed? | `yarn why pkg` | `yarn why pkg` | `npm explain pkg` (alias `npm why`) |
| List installed packages | `yarn info -R` / `yarn info --all` | `yarn list` | `npm ls` |
| Package info from registry | `yarn npm info pkg` | `yarn info pkg` | `npm view pkg` |
| Deduplicate | `yarn dedupe` | (manual / `yarn-deduplicate`) | `npm dedupe` |
| Run a script | `yarn run build` or `yarn build` | `yarn build` | `npm run build` |
| Run a binary from deps | `yarn exec tool` / `yarn tool` | `yarn tool` | `npm exec tool` / `npx tool` |
| One-off remote package | `yarn dlx pkg` | (not available; use `npx`) | `npx pkg` / `npm exec` |
| Security audit | `yarn npm audit` | `yarn audit` | `npm audit` |
| Publish | `yarn npm publish` | `yarn publish` | `npm publish` |
| Log in to registry | `yarn npm login` | `yarn login` | `npm login` |
| Clean cache | `yarn cache clean` | `yarn cache clean` | `npm cache clean --force` |
| Get/set config | `yarn config get/set` | `yarn config get/set` | `npm config get/set` |
| Init project | `yarn init` | `yarn init` | `npm init` |
| Rebuild native deps | `yarn rebuild` | `yarn install --force` | `npm rebuild` |
| Print package manager version | `yarn --version` | `yarn --version` | `npm --version` |

### Yarn-specific commands (Berry)

| Command | Purpose |
|---|---|
| `yarn set version <v>` | Pin the Yarn version for the project |
| `yarn plugin import <name>` | Install a plugin |
| `yarn workspaces list` | List all workspaces |
| `yarn workspaces foreach ...` | Run a command across workspaces |
| `yarn workspaces focus <ws>` | Install only the deps needed by given workspaces (great for Docker/CI) |
| `yarn patch <pkg>` / `yarn patch-commit` | Patch a dependency in place (built-in `patch-package` equivalent) |
| `yarn unplug <pkg>` | Extract a package from the zip cache to disk (PnP) |
| `yarn constraints` | Enforce rules across workspaces |
| `yarn explain peer-requirements` | Debug peer dependency warnings |
| `yarn sdks` | Generate editor SDKs for PnP (VS Code, etc.) |
| `yarn node` | Run Node with the PnP runtime loaded |
| `yarn bin` | Print the path to a binary |
| `yarn version` | Release/versioning workflow for (monorepo) packages |

> Note: In Berry, `yarn global` was removed. For global-style tools use `yarn dlx`, or install a tool per project. Classic's `yarn global add` still exists in 1.x.

---

## 6. Install Strategies

Yarn's `nodeLinker` setting (in `.yarnrc.yml`) controls how installed packages are exposed to Node.js:

| `nodeLinker` | What it does | Pros | Cons |
|---|---|---|---|
| `pnp` (default in Berry) | No `node_modules`. A `.pnp.cjs` map tells Node where each package lives (usually inside zip archives). | Fastest installs, strict correctness, tiny disk footprint, enables zero-installs | Tooling compatibility issues, needs editor SDKs, some packages need patches/unplugging |
| `node-modules` | Classic hoisted `node_modules` layout. | Maximum compatibility with existing tools | Slower, bigger, allows phantom dependencies |
| `pnpm` | `node_modules` with symlinks to a content-addressable store (like pnpm). | Strict-ish, saves disk, high compatibility | Less battle-tested than the other two within Yarn |

```yaml
# .yarnrc.yml
nodeLinker: node-modules
```

**Practical advice:** many teams (especially with React Native, certain Jest/webpack plugin ecosystems, or legacy tooling) start with `nodeLinker: node-modules` for a smooth migration, then evaluate PnP later.

---

## 7. Plug'n'Play (PnP) in Depth

### How `node_modules` works (and why it's problematic)

Node resolves `require('x')` by walking up the directory tree looking for `node_modules/x`. This implies:

- Huge numbers of files and directories to write on every install.
- Hoisting makes undeclared packages accidentally importable (**phantom dependencies**), causing "works on my machine" bugs.
- Resolution is a filesystem search at runtime.

### How PnP works

At install time Yarn already knows the entire graph, so it generates a single file, `.pnp.cjs`, containing a static lookup table: *"package A may require package B, located at path P."* Node is patched at runtime (via `--require .pnp.cjs`, handled when you launch through `yarn`) so resolution uses the table.

Benefits:

- **Strictness:** a package can only import what it declared. Undeclared imports fail loudly.
- **Speed:** installs write mostly just the lockfile, the cache, and `.pnp.cjs`. Startup resolution avoids filesystem walking.
- **Smaller footprint:** packages remain as zip files in the cache.

### Practical considerations

- **Run things through Yarn:** `yarn node script.js`, `yarn tsc`, `yarn jest`. Running bare `node` won't know about PnP.
- **Editor support:** run `yarn dlx @yarnpkg/sdks vscode` (or `yarn sdks`) so TypeScript/ESLint/Prettier work in your editor.
- **Packages that need real files:** use `yarn unplug <pkg>` or declare in `package.json`:

  ```json
  {
    "dependenciesMeta": {
      "some-native-pkg": { "unplugged": true }
    }
  }
  ```
- **Broken packages:** fix missing dependency declarations via `packageExtensions` in `.yarnrc.yml` rather than forking:

  ```yaml
  packageExtensions:
    "some-pkg@*":
      dependencies:
        missing-dep: "*"
  ```
- **ESM and loaders:** modern Yarn supports ESM under PnP via loader hooks; occasionally edge cases appear with experimental Node flags.
- **Fallback mode:** `pnpMode: loose` lets packages access hoisted undeclared dependencies as a migration crutch (not recommended long-term).

---

## 8. Zero-Installs and the Offline Cache

Yarn Berry stores every package as a compressed archive in the cache (`.yarn/cache` by default, configurable via `cacheFolder`; or a global cache when `enableGlobalCache: true`).

**Zero-installs** means committing `.yarn/cache` and `.pnp.cjs` to version control. After `git clone` or `git pull`, the project runs **without running `yarn install`**.

| Pros | Cons |
|---|---|
| No install step in CI or after pulling branches | Repository grows (often tens to hundreds of MB; use Git LFS cautiously) |
| Fully offline, reproducible builds | Noisy diffs when upgrading many packages |
| Dependency changes are visible/reviewable in PRs | Needs discipline: always commit cache changes together with the lockfile |

Not everyone should use zero-installs. A common middle path: **don't commit the cache**, but cache `.yarn/cache` (or the global cache) in CI between runs.

Typical `.gitignore` for a Yarn 4 project that does **not** use zero-installs:

```gitignore
.yarn/*
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
.pnp.*
```

For zero-installs, also keep `.yarn/cache` and `.pnp.*` tracked (remove them from the ignore list).

---

## 9. The Lockfile

`yarn.lock` is committed to version control.

### Yarn 1 format (custom, indentation-based)

```
lodash@^4.17.21:
  version "4.17.21"
  resolved "https://registry.yarnpkg.com/lodash/-/lodash-4.17.21.tgz#..."
  integrity sha512-...
```

### Yarn Berry format (YAML)

```yaml
__metadata:
  version: 8
  cacheKey: 10c0

"lodash@npm:^4.17.21":
  version: 4.17.21
  resolution: "lodash@npm:4.17.21"
  checksum: 10c0/...
  languageName: node
  linkType: hard
```

Key points:

- Multiple ranges that resolve to the same version are merged into one entry.
- Berry records **checksums of the archive** (not just the registry's integrity hash), verified at install time.
- Berry includes workspace entries (`languageName`, `linkType: soft` for workspaces/links).
- **Never hand-edit the lockfile** except for resolving merge conflicts; after merging, run `yarn install` to normalize it. Yarn can resolve most lockfile merge conflicts itself in Berry.
- `--immutable` fails the install if the lockfile would change. Use it in CI.
- Lockfile formats are **not** interchangeable with `package-lock.json`. Yarn has no built-in command to convert from npm's lockfile, but running `yarn install` in a project that has only `package-lock.json` resolves fresh (and Berry can import some data from an existing `package-lock.json` or Classic `yarn.lock`).

---

## 10. Configuration (`.yarnrc.yml`)

Yarn Berry uses `.yarnrc.yml` (project-level, plus an optional user-level `~/.yarnrc.yml`). Yarn 1 used `.yarnrc` and also read `.npmrc`. **Yarn Berry does not read `.npmrc`** for its own settings (it has its own `npm*` settings).

Example:

```yaml
# .yarnrc.yml
yarnPath: .yarn/releases/yarn-4.9.2.cjs   # set by `yarn set version`
nodeLinker: node-modules                  # pnp | node-modules | pnpm
enableGlobalCache: false                  # store cache in project vs ~/.yarn/berry/cache
enableTelemetry: false
enableImmutableInstalls: true             # (also default true on CI environments)
compressionLevel: mixed
defaultSemverRangePrefix: "^"             # "" for exact versions by default

npmRegistryServer: "https://registry.npmjs.org"

# Scoped / private registries
npmScopes:
  mycompany:
    npmRegistryServer: "https://npm.pkg.github.com"
    npmAlwaysAuth: true
    npmAuthToken: "${GITHUB_TOKEN}"       # use env vars, never commit secrets

# Per-registry settings (e.g. proxies, Artifactory, Verdaccio)
npmRegistries:
  "//registry.internal.example.com":
    npmAuthToken: "${INTERNAL_TOKEN}"
    npmAlwaysAuth: true

# Safety
enableScripts: true          # set false to ignore lifecycle scripts for all deps
enableStrictSsl: true

# Fixes for misbehaving packages
packageExtensions:
  "some-pkg@*":
    peerDependencies:
      react: "*"

# Plugins
plugins:
  - path: .yarn/plugins/@yarnpkg/plugin-workspace-tools.cjs
```

Notes:

- Environment variables can be referenced as `${VAR}`; use `${VAR:-default}` for fallbacks.
- Any setting can also be provided as an environment variable `YARN_<SETTING_NAME_IN_SNAKE_CASE>` (e.g. `YARN_NODE_LINKER=node-modules`).
- `yarn config` prints/reads resolved values (`yarn config get nodeLinker`, `yarn config -v` for descriptions).
- Recent 4.x releases added supply-chain-oriented settings (for example, hardened mode and a minimum package age gate, `npmMinimalAgeGate`). Check `yarn config -v` or the docs for what your version supports.

---

## 11. Dependency Protocols and Resolution Control

### Protocols (Berry)

| Protocol | Example | Meaning |
|---|---|---|
| `npm:` | `"react": "npm:^18.2.0"` / alias `"my-lodash": "npm:lodash@^4"` | Registry package (default); can alias |
| `workspace:` | `"@acme/ui": "workspace:^"` | Link to a workspace in this repo |
| `file:` | `"lib": "file:../lib"` | Copy from a local path |
| `link:` | `"lib": "link:../lib"` | Symlink local folder (its deps aren't resolved) |
| `portal:` | `"lib": "portal:../lib"` | Symlink local folder **and** resolve its dependencies |
| `patch:` | `"pkg": "patch:pkg@npm:1.0.0#./.yarn/patches/pkg.patch"` | Apply a patch to a dependency |
| `git:` / `github:` | `"pkg": "github:user/repo#semver:^1.2.0"` | Git dependency |
| `exec:` | `"gen": "exec:./gen.js"` | Generated at install time by a script |

### `workspace:` ranges

- `workspace:*` always links to the local workspace.
- `workspace:^` / `workspace:~` get replaced with a real semver range (`^1.2.3`) at **publish/pack** time, so published packages are valid for consumers.

### Forcing versions: `resolutions`

```json
{
  "resolutions": {
    "minimist": "^1.2.8",
    "**/lodash": "4.17.21",
    "foo/bar": "2.0.0"
  }
}
```

Use to patch vulnerable transitive dependencies. npm's analogue is `overrides`; pnpm uses `pnpm.overrides`. `resolutions` is supported in both Yarn Classic and Berry (syntax is slightly different).

### Patching dependencies

```bash
yarn patch lodash          # prints a temp folder; edit files there
yarn patch-commit -s /tmp/xyz   # writes .yarn/patches/*.patch and updates resolutions
```

This is built in, so `patch-package` isn't needed.

### Peer dependencies

Yarn Berry is strict: a package with unmet peer dependencies will warn, and under PnP an undeclared peer is genuinely unavailable. Use `yarn explain peer-requirements <hash>` to debug.

---

## 12. Workspaces and Monorepos

Declare workspaces in the root `package.json`:

```json
{
  "name": "monorepo",
  "private": true,
  "packageManager": "yarn@4.9.2",
  "workspaces": ["packages/*", "apps/*"]
}
```

Directory layout:

```
monorepo/
├── package.json
├── yarn.lock
├── .yarnrc.yml
├── apps/
│   └── web/package.json
└── packages/
    ├── ui/package.json
    └── utils/package.json
```

All workspaces share a **single lockfile** and a deduplicated install.

### Referencing sibling packages

```json
{
  "name": "@acme/web",
  "dependencies": {
    "@acme/ui": "workspace:^",
    "@acme/utils": "workspace:*"
  }
}
```

### Common workspace commands (Berry)

```bash
yarn workspaces list --json                         # discover workspaces
yarn workspace @acme/web add react                  # run a command in one workspace
yarn workspace @acme/web run dev
yarn workspaces foreach -A run build                # all workspaces (-A, --all)
yarn workspaces foreach -At run build               # -t: topological (dependencies first)
yarn workspaces foreach -Ap -j 4 run test           # -p: parallel, -j: concurrency
yarn workspaces foreach --since=origin/main run lint  # only changed workspaces (+ deps/dependents with flags)
yarn workspaces focus @acme/web                     # install only what @acme/web needs
yarn workspaces focus --production                  # production deps only (e.g. in Docker)
```

Related tools often paired with Yarn workspaces: **Turborepo**, **Nx**, **Lerna** (now maintained by Nrwl/Nx), and **Changesets** for versioning. Yarn alone gives you dependency management, running scripts across workspaces, and version bump workflows (`yarn version`), but not remote caching or sophisticated task pipelines.

### npm equivalents

npm supports workspaces (7+): `npm install -w packages/ui`, `npm run build --workspaces`, `--workspace=...`, `--include-workspace-root`. It lacks Yarn's `foreach` topological/parallel orchestration, `focus`, and constraints, so teams typically add Turborepo/Nx.

---

## 13. Scripts, Binaries, and Running Tools

```json
{
  "scripts": {
    "build": "tsc -b",
    "test": "jest",
    "lint": "eslint .",
    "prepare": "husky",
    "postinstall": "node scripts/setup.js"
  }
}
```

- `yarn build` and `yarn run build` are equivalent.
- **Binaries from dependencies** are available directly: `yarn tsc`, `yarn eslint`. Under PnP these resolve correctly because they run inside Yarn's environment.
- **Lifecycle scripts:** `preinstall`, `install`, `postinstall` (for dependencies, run at install time), and `prepare`/`prepack`/`postpack` (for the project itself).
  - Berry runs dependency lifecycle scripts only for packages that need them and with `enableScripts` controlling the global switch. You can disable scripts for specific packages through `dependenciesMeta`:

    ```json
    { "dependenciesMeta": { "esbuild": { "built": false } } }
    ```
- **Pre/post hooks:** In Berry, automatic `pre`/`post` scripts for user-defined scripts (e.g. `prebuild`) are **not** run (only the standard lifecycle ones); Yarn Classic and npm do run them. Call the steps explicitly instead.
- **Argument passing:** `yarn build --watch` forwards args to the script. Berry does not require `--` separators.
- **`yarn dlx <pkg>`:** downloads and runs a package in a temporary environment, like `npx` (does not touch the project's dependencies).
- **Environment:** Yarn exposes env vars such as `npm_package_name`, `INIT_CWD`, and `PROJECT_CWD` in scripts. Some legacy `npm_package_*` vars are only partially supported in Berry.
- **Cross-platform shell:** Berry ships a portable shell interpreter for scripts (supports `&&`, `||`, pipes, env var assignment like `FOO=bar cmd`, globs) so scripts behave identically on Windows and Unix. Exotic bashisms may fail; call a real script file for complex logic.

---

## 14. Plugins and Constraints

### Plugins

Berry is built around plugins; many core features (e.g., `workspace-tools`, `interactive-tools`, `typescript`) are bundled or optional.

```bash
yarn plugin list
yarn plugin import https://example.com/my-plugin.js
yarn plugin import typescript          # official plugin: auto-adds @types/*
yarn plugin remove @yarnpkg/plugin-typescript
```

The TypeScript plugin automatically adds corresponding `@types/*` packages when you `yarn add` a package that lacks bundled types. Plugins live in `.yarn/plugins/` and are referenced from `.yarnrc.yml`. Commit them.

### Constraints

Define repo-wide rules in `yarn.config.cjs` (JavaScript API) or `constraints.pro` (Prolog, legacy). Examples: "all workspaces must use the same React version," "every package must have a `license` field," "no workspace may depend on package X."

```js
// yarn.config.cjs
/** @type {import('@yarnpkg/types').Yarn.Config} */
module.exports = {
  async constraints({ Yarn }) {
    for (const dep of Yarn.dependencies({ ident: "react" })) {
      dep.update("^18.3.1");
    }
  },
};
```

Run `yarn constraints` to check and `yarn constraints --fix` to apply fixes. Neither npm nor pnpm provides an equivalent built-in system.

---

## 15. Security

### Supply-chain practices

- **Lockfile + `--immutable`** in CI to prevent unexpected version drift.
- **Checksums:** Berry verifies archive checksums on install (`--check-cache` re-verifies the cache against the registry).
- **Lifecycle script control:** set `enableScripts: false` globally and opt in for specific packages that truly need build steps, since install scripts are the most common supply-chain attack vector. (With scripts disabled, packages that require a build step, such as native addons, won't be compiled.)
- **Auditing:**
  ```bash
  yarn npm audit                      # Berry
  yarn npm audit --all --recursive    # include all workspaces
  yarn npm audit --severity high
  yarn audit                          # Classic
  ```
- **Provenance and trusted publishing:** publishing with provenance/OIDC is mainly an npm registry feature; Yarn Berry supports provenance on `yarn npm publish` via `--provenance` in recent versions. Verify support for your version.
- **Pin and review:** consider `defaultSemverRangePrefix: ""` (exact versions) and use Renovate or Dependabot for controlled updates.
- **Strictness helps:** PnP and `pnpm`-style linkers block phantom dependency access, which reduces accidental reliance on transitive packages that could be swapped.
- **Registry authentication:** store tokens in environment variables or a user-level `~/.yarnrc.yml`, never in committed files.
- **Malicious/typosquatted packages:** neither npm nor Yarn can fully prevent them. Review new dependencies, use scoped private registries, and consider package-age gates and allow-lists where supported.

---

## 16. CI/CD Usage

### GitHub Actions (Yarn 4, no zero-installs)

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: yarn            # caches Yarn's cache folder

      - run: corepack enable     # use the version from "packageManager"

      - run: yarn install --immutable

      - run: yarn build
      - run: yarn test
```

Tips:

- Run `corepack enable` **before** `setup-node`'s `cache: yarn` step in some setups, as the cache step calls `yarn` to locate the cache directory. If you see errors, reorder or enable Corepack first.
- `--immutable` is the default in CI environments (Berry detects `CI`). Be explicit anyway.
- Use `--immutable-cache` with zero-installs to ensure no cache changes are needed (`yarn install --immutable --immutable-cache --check-cache`).
- Cache `.yarn/cache` (or the global cache) keyed on `yarn.lock`.
- For monorepos use `yarn workspaces focus` and `yarn workspaces foreach --since` to avoid doing unnecessary work.

### Docker

```dockerfile
FROM node:22-slim AS base
WORKDIR /app
RUN corepack enable

# Install dependencies with good layer caching
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn
RUN yarn install --immutable

COPY . .
RUN yarn build

# Production image
FROM node:22-slim
WORKDIR /app
RUN corepack enable
COPY --from=base /app ./
# For monorepos: yarn workspaces focus --production @acme/api
CMD ["yarn", "node", "dist/server.js"]
```

Consider multi-stage builds, `yarn workspaces focus --production`, and keeping the Yarn release/cache in the build context only when needed.

---

## 17. Publishing Packages

```bash
yarn npm login
yarn npm publish --access public
yarn npm publish --tag next
yarn pack                          # create a tarball locally to inspect it
yarn npm whoami
yarn npm tag add pkg@1.2.3 latest
```

Notes:

- `workspace:` ranges are rewritten to concrete versions in the published tarball.
- Control published files with the `files` field, `.npmignore`, or `publishConfig`.
- Use `publishConfig` to alter `main`, `types`, `exports`, or `registry` for the published manifest (for example, to point `main` at `dist/` while developing against `src/`).
- Versioning workflow: `yarn version patch|minor|major` for single packages. For monorepos, use `yarn version check` with the "deferred versioning" workflow (`yarn version <bump> --deferred`, `yarn version apply --all`), or Changesets.

---

## 18. Yarn vs npm: Detailed Comparison

### 18.1 Summary Table

| Area | **npm** (v10/11) | **Yarn Classic** (1.x) | **Yarn Berry** (4.x) |
|---|---|---|---|
| Ships with Node.js | **Yes** | No | No (use Corepack or `yarn set version`) |
| Maintained by | npm (GitHub / Microsoft) | Community, maintenance only | Yarn core team (community-driven) |
| Lockfile | `package-lock.json` (JSON) | `yarn.lock` (custom) | `yarn.lock` (YAML) |
| Default install layout | Hoisted `node_modules` | Hoisted `node_modules` | **PnP** (or `node_modules` / `pnpm` linker) |
| Phantom dependency protection | No (hoisting exposes them) | No | **Yes** under PnP / pnpm linker |
| Offline install | Partial (cache, `--offline`) | Good (offline mirror) | **Excellent** (committed cache, zero-installs) |
| Install speed | Good, much improved | Faster than old npm | Typically fastest of the three, especially with PnP and a warm cache |
| Workspaces | Yes (7+), basic | Yes | **Yes, advanced** (`foreach`, `focus`, `workspace:` protocol, constraints) |
| Workspace protocol | No (`*` or version, resolved via workspace link) | No | **`workspace:`** |
| Dependency overrides | `overrides` | `resolutions` | `resolutions` |
| Built-in patching | No (use `patch-package`) | No | **Yes** (`yarn patch`) |
| Plugin system | No | No | **Yes** |
| Constraints / policy | No | No | **Yes** |
| Peer dependency handling | Auto-installs peers (npm 7+) | Warns only | Strict, warns; PnP enforces |
| Config file | `.npmrc` | `.yarnrc` + reads `.npmrc` | `.yarnrc.yml` (ignores `.npmrc`) |
| One-off execution | `npx` / `npm exec` | (use `npx`) | `yarn dlx` |
| Global installs | `npm i -g` | `yarn global` | Not supported (use `dlx` or per-project tools) |
| Auto `pre/post` scripts | Yes | Yes | No (only lifecycle scripts) |
| Audit | `npm audit`, `npm audit fix` | `yarn audit` (no auto-fix) | `yarn npm audit` (no auto-fix) |
| Provenance / trusted publishing | **First-class** support | No | Limited / version-dependent |
| Ecosystem/tooling compat | **Best** (reference implementation) | Very good | Good, with PnP caveats |
| Learning curve | Lowest | Low | Moderate |
| Windows/Unix script portability | Depends on shell (use `cross-env` etc.) | Depends on shell | **Built-in portable shell** |

### 18.2 Architecture Differences

**npm** builds a hoisted tree in `node_modules` (a "flattened" layout where shared deps are moved up as far as possible). It's simple and universal but allows undeclared imports and has dedupe edge cases.

**Yarn Classic** uses the same hoisted layout but with a faster, more deterministic algorithm and an offline mirror feature.

**Yarn Berry** models the install as `resolution → fetch → link` with swappable linkers. PnP removes `node_modules` entirely.

### 18.3 Determinism

- Both npm (since v5) and Yarn use lockfiles. `npm ci` and `yarn install --immutable` are the "install exactly this" commands.
- Berry's lockfile includes archive checksums; npm's includes the registry's `integrity` hash. Both detect tampering of tarballs.
- Historically, lockfile drift between package manager versions caused spurious diffs; pinning the package manager (`packageManager` + Corepack) avoids this for both.

### 18.4 Performance

Real-world numbers depend on project, network, disk, and cache state. Directionally:

- **Cold install, no cache:** the three are in the same ballpark; Yarn Berry (PnP) and pnpm often edge out npm because they write far fewer files.
- **Warm cache / CI with cache:** Yarn Berry with PnP or zero-installs is typically fastest; npm has improved considerably.
- **Disk usage:** PnP stores zipped archives in one cache; hoisted `node_modules` duplicates package contents per project. pnpm also uses a global content-addressable store with hard links.

Benchmark in your own repo before deciding; published benchmarks vary by methodology.

### 18.5 Developer Experience

| Aspect | npm | Yarn (Berry) |
|---|---|---|
| Setup friction | None (bundled) | Needs Corepack or version pin |
| Error messages | Improving | Detailed, with explain tooling (`yarn explain`) |
| Editor integration | Works everywhere | Requires SDK setup for PnP |
| Documentation | Large | Comprehensive but version-split (Classic vs Berry docs) |
| Interactive upgrade | No built-in | `yarn upgrade-interactive` |
| Debugging deps | `npm ls`, `npm explain` | `yarn why`, `yarn info`, `yarn explain` |

### 18.6 Compatibility Gotchas with Yarn Berry/PnP

- Tools that read `node_modules` directly (some bundlers, native module loaders, Jest plugins, old Storybook/React Native tooling) may break. Use `nodeLinker: node-modules` or `unplug`.
- Packages with undeclared dependencies need `packageExtensions`.
- Node's own `--require` / loader behavior interacts with PnP; start scripts through `yarn`.
- Docker/serverless environments need `.pnp.cjs` and the cache present at runtime (or use `node-modules` linker for deploy artifacts).
- IDEs need `yarn sdks`.

### 18.7 Where npm Is Better

- Zero setup, universally documented, default in CI templates.
- Reference implementation of registry features (provenance, trusted publishing, `npm audit fix`, `npm pkg`, `npm query`).
- Max compatibility with tools and tutorials.
- Simple projects don't benefit from Yarn's complexity.

### 18.8 Where Yarn Is Better

- Monorepos (workspace tooling, `foreach`, `focus`, constraints, `workspace:` protocol).
- Strict dependency correctness (PnP).
- Offline/zero-install workflows and fast CI.
- Built-in patching, plugins, and portable scripts.
- Fine-grained configuration for registries and scopes.

---

## 19. Yarn vs npm vs pnpm (Quick View)

| | npm | Yarn 4 | pnpm |
|---|---|---|---|
| Storage | Per-project copies in `node_modules` | Zip cache + PnP map (default) | Global content-addressable store + symlinked `node_modules` |
| Strictness | Loose | Strict (PnP) | Strict (non-flat `node_modules`) |
| Compatibility | Highest | Good (PnP caveats) | Very high |
| Monorepo features | Basic | Rich | Rich (`--filter`, `catalog:`) |
| Unique strengths | Bundled, registry features | PnP, zero-installs, constraints, plugins | Disk efficiency, speed, strictness with fewer compat problems |

If you want Yarn's strictness without PnP's tooling friction, evaluate `nodeLinker: pnpm` or pnpm itself.

---

## 20. Migration Guides

### 20.1 npm → Yarn 4

```bash
# 1. Pin Yarn
corepack enable
corepack use yarn@stable        # or: yarn set version stable

# 2. Choose a linker (safe start)
echo "nodeLinker: node-modules" >> .yarnrc.yml

# 3. Remove npm artifacts
rm -rf node_modules package-lock.json

# 4. Install and generate yarn.lock
yarn install

# 5. Update scripts/docs/CI: npm ci -> yarn install --immutable, npx -> yarn dlx, etc.
```

Checklist:

- Replace `.npmrc` registry/auth settings with `.yarnrc.yml` equivalents (`npmRegistryServer`, `npmScopes`, `npmAuthToken`).
- Replace `overrides` with `resolutions`.
- Replace `npm run a && npm run b` in scripts with `yarn a && yarn b` (or keep `npm run` only if you truly want npm).
- Replace `pre*/post*` custom hook scripts with explicit calls.
- Update CI cache keys from `package-lock.json` to `yarn.lock`.
- Update `.gitignore` as in [section 8](#8-zero-installs-and-the-offline-cache).
- Run tests with `nodeLinker: node-modules` first, then try PnP.

### 20.2 Yarn Classic → Yarn Berry

```bash
yarn set version stable
yarn install                    # upgrades the lockfile format
```

Differences to handle:

- `.yarnrc` → `.yarnrc.yml`; `.npmrc`-based auth must be re-declared.
- `yarn install --frozen-lockfile` → `yarn install --immutable`.
- `yarn upgrade` → `yarn up`.
- `yarn audit` → `yarn npm audit`; `yarn publish` → `yarn npm publish`; `yarn info` → `yarn npm info`.
- `yarn global` is gone.
- `yarn licenses`, `yarn outdated`, `yarn list`: use plugins or alternatives (`yarn info`, `yarn upgrade-interactive`, third-party tools).
- `resolutions` syntax mostly carries over (use `**/pkg` globs carefully).
- Start with `nodeLinker: node-modules`. The official migration guide recommends this as the incremental path.

### 20.3 Yarn → npm

```bash
rm -rf node_modules yarn.lock .yarn .pnp.* .yarnrc.yml
# remove "packageManager": "yarn@..." from package.json (or set npm@...)
npm install                      # generates package-lock.json
```

Convert `resolutions` → `overrides`, `workspace:` ranges → plain version ranges or `*`, and Yarn-only commands in scripts/CI.

---

## 21. Best Practices

1. **Pin the package manager** with the `packageManager` field and Corepack.
2. **Commit the lockfile**, always. Never `.gitignore` it for applications.
3. **Use `--immutable`** (Yarn) / `npm ci` in CI.
4. **Declare every dependency you import.** Treat PnP strictness as a feature.
5. **Prefer `nodeLinker: node-modules` initially** when migrating, then reassess PnP.
6. **Use `yarn dedupe`** periodically (and in CI with `--check`) to reduce duplicate versions.
7. **Keep dependencies updated** with Renovate/Dependabot or `yarn upgrade-interactive`.
8. **Use `resolutions` sparingly** and document why each exists; remove when upstream fixes land.
9. **Restrict install scripts** (`enableScripts: false` with explicit allowances) for security-conscious projects.
10. **Never commit auth tokens.** Use env vars in `.yarnrc.yml`.
11. **Use `workspace:^`** for internal deps so published packages have correct ranges.
12. **Run tools via `yarn`** (`yarn node`, `yarn tsc`) when using PnP.
13. **Use `yarn workspaces focus`** in Docker/CI to limit install scope.
14. **Keep one package manager per repo.** Add a guard (e.g., `only-allow`, or a CI check that rejects foreign lockfiles) to prevent mixed lockfiles.
15. **Document the workflow** in your repo's README: install, test, and release commands.
16. **Decide on zero-installs deliberately**, weighing repo size against install-free workflows.
17. **Review lockfile diffs** in PRs; unexpected changes can indicate supply-chain issues.

---

## 22. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot find module 'x'` under PnP | Undeclared dependency or tool not run via Yarn | Add the dependency, add `packageExtensions`, run via `yarn`, or use `nodeLinker: node-modules` |
| `The lockfile would have been modified by this install, which is explicitly forbidden` | `--immutable` and lockfile out of date | Run `yarn install` locally and commit `yarn.lock` |
| `Usage Error: This project's package.json defines "packageManager": "yarn@x" ... but the current global is ...` | Corepack not enabled or a global Yarn conflicts | Run `corepack enable`; remove/uninstall conflicting global Yarn |
| Editor can't resolve imports / TS errors | PnP SDK not configured | `yarn dlx @yarnpkg/sdks vscode`, select workspace TypeScript version |
| Native module fails (`node-gyp`, missing binary) | Scripts disabled or package zipped under PnP | Enable scripts for that package, `yarn unplug pkg`, or use `nodeLinker: node-modules` |
| Peer dependency warnings flood output | Packages with undeclared/mismatched peers | `yarn explain peer-requirements`, add the peers, or use `packageExtensions` |
| Different results on CI vs local | Different Yarn/Node versions or stale cache | Pin versions, clear cache (`yarn cache clean`), reinstall with `--check-cache` |
| `YN0028` / lockfile merge conflicts | Git merge conflict in `yarn.lock` | Resolve conflict markers if possible, then run `yarn install`; Berry can auto-resolve many conflicts |
| Jest/webpack/Metro can't find modules | Tool incompatible with PnP | Use a PnP-aware plugin/config or switch linker to `node-modules` |
| Slow installs | Cold cache, network, antivirus scanning many files | Use a CI cache, `enableGlobalCache`, PnP, or a private registry mirror |
| Auth errors for private registry | Config in `.npmrc` ignored by Berry | Move to `.yarnrc.yml` (`npmScopes` / `npmRegistries`) |
| `yarn global` not found | Removed in Berry | Use `yarn dlx` or project-local tools |

Useful diagnostics:

```bash
yarn --version
yarn config            # effective configuration
yarn why <pkg>         # who depends on it
yarn info <pkg> -R     # dependency details
yarn install --inline-builds   # show build script output
YARN_ENABLE_INLINE_BUILDS=1 yarn install
```

---

## 23. Decision Guide

- **Small app or library, want zero friction:** **npm**.
- **Monorepo with many packages and a need for strong tooling:** **Yarn 4** (or pnpm).
- **Strict dependency hygiene, fast CI, offline builds:** **Yarn 4 with PnP**.
- **Need maximum tool compatibility but better efficiency than npm:** **pnpm**, or **Yarn 4 with `nodeLinker: node-modules`/`pnpm`**.
- **Existing Yarn 1 project:** upgrade to Yarn 4 when you have time to test; there's no urgency if it works, but Classic is no longer evolving.
- **React Native or other ecosystems with heavy native tooling:** start with `node-modules` linker.
- **You need npm provenance/trusted publishing flows:** npm CLI for publishing is the safest path (you can still develop with Yarn).

---

## 24. Resources

- Yarn documentation: <https://yarnpkg.com>
- Yarn Berry source: <https://github.com/yarnpkg/berry>
- Yarn Classic docs (legacy): <https://classic.yarnpkg.com>
- Yarn migration guide: <https://yarnpkg.com/migration/overview>
- Corepack: <https://nodejs.org/api/corepack.html>
- npm docs: <https://docs.npmjs.com>
- pnpm docs: <https://pnpm.io>
- Package compatibility under PnP: <https://yarnpkg.com/features/pnp>

---

*This guide reflects Yarn 4.x, Yarn 1.22.x, and npm 10/11 behaviors as generally understood. Flags, defaults, and plugin availability evolve, so confirm against `yarn --help`, `yarn config -v`, and the official documentation for your specific versions.*
