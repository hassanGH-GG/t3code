# Hexfy bootstrap

How to get from a fresh clone to a verified build. Every command here was run and its result recorded
on 2026-08-17 against commit `13458e65`.

This is Phase A slice 0.1. It exists because the upstream contributing docs install the build
toolchain by piping a remote script into a shell, which is not reproducible in every environment. It
turns out not to be necessary.

## Prerequisites

| Tool    | Version used | Notes                                                                                                                                                                                                                                             |
| ------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Node.js | 22.22.2      | The workspace declares `engines: { node: "^24.13.1" }`. pnpm warns and proceeds. Install, targeted tests and targeted typecheck all pass on 22.22.2. Bundling, desktop and mobile builds have **not** been verified on 22 — see "Open questions". |
| pnpm    | 11.10.0      | Pinned by `packageManager` in the root `package.json`.                                                                                                                                                                                            |

## Steps

```sh
git clone <hexfy remote> hexfy
cd hexfy
pnpm install
```

That is the whole bootstrap. `pnpm install` took 1m52s and exited 0.

### You do not need the remote install script

The upstream README instructs you to install the `vp` (Vite+) command globally with
`curl -fsSL https://vite.plus | bash`. Do not bother, and do not depend on it:

- `vite-plus` is an ordinary devDependency in the root `package.json`, pinned to `0.2.2` by the
  catalog in `pnpm-workspace.yaml`.
- That package ships the binaries directly: `bin: { vp, vpr, oxlint, oxfmt }`.
- After `pnpm install` they are present in `node_modules/.bin/`, alongside `tsgo`, `tsgolint` and
  `effect-tsgo`.

So the toolchain comes from the npm registry like any other dependency. In an environment where
`vite.plus` is unreachable (a network policy that denies the host, for example), the bootstrap still
works. Use `./node_modules/.bin/vp` or a package-manager-provided shim rather than a global `vp`.

The root `prepare` script does run `vp config --no-agent`, and it succeeds, because by the time
`prepare` executes `node_modules/.bin` is already on `PATH`.

## Verifying the install

Run **targeted** checks only. Repo-wide checks (`vp check`, `vp run -r test`, `vp run -r typecheck`)
are owned by CI and are explicitly discouraged locally.

Targeted test:

```sh
./node_modules/.bin/vp test run packages/contracts/src/git.test.ts
```

Recorded result: `Test Files 1 passed (1)`, `Tests 6 passed (6)`, 662ms. The test runner is Vitest 4
under the hood, and test files import from `vite-plus/test`.

Targeted typecheck, from the package you changed:

```sh
cd packages/contracts && ../../node_modules/.bin/tsgo --noEmit
```

Recorded result: exit 0, no diagnostics.

Server entry point:

```sh
node apps/server/src/bin.ts --help
```

Recorded result: prints usage. Subcommands today are `start`, `serve`, `pair`, `auth`, `project`,
`service`, `service-preflight`, and `connect` (which hides itself when the build carries no cloud
configuration).

## A vp-free path, if you ever want one

Useful to know, because it bounds how much the fork depends on Vite+:

- `typecheck` in every package is plain `tsgo --noEmit`. No Vite+ involvement.
- `test` is `vp test run`, but the tests themselves import from `vite-plus/test`, so the dependency is
  the package rather than a global binary.
- Only `build:bundle` genuinely needs Vite+ tooling: `vp pack`.

## Open questions

These are unverified and should be resolved before they can bite:

1. **Node 24 vs 22.** The workspace wants `^24.13.1`. Everything exercised above works on 22.22.2, but
   `vp pack`, the Electron desktop build and the React Native mobile build were not run. Pin a Node
   version in the fork once one of those has been exercised.
2. **Dependency drift.** `pnpm install` reported newer versions available for `@types/node`
   (24.12.4 installed, 26.2.0 available), `@typescript/native-preview` and `vite-plus` (0.2.2
   installed, 0.2.9 available). The catalog pins these deliberately; do not bump them casually,
   because `prepare` also runs `effect-tsgo patch` against a beta build of Effect
   (`effect@4.0.0-beta.103`).
3. **Disk.** `node_modules` is 3.1 GB across 16 workspace projects. Budget accordingly in CI and in
   any container.
