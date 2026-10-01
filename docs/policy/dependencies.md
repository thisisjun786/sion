# Dependencies

This policy covers code that SION takes from outside the repository: the upstream ClawSweeper tree and the npm packages it locks, the Node.js and pnpm versions that install and run them, npm packages that only SION's own code uses, LINA's protocol schema and conformance fixtures, CI tools, and GitHub Actions. SION runs with a GitHub App key and a model API key in its operator repositories, so outside code is pinned, reviewed like any other code, and verified in CI before it runs. The [CI policy](ci.md) owns path mapping and job layout.

## Upstream tree

The upstream tree under `upstream/` carries its own package manifest, lockfile and pnpm workspace settings ([source layout](../design/sion.md#source-layout)).

- SION uses `upstream/package.json`, `upstream/pnpm-lock.yaml` and `upstream/pnpm-workspace.yaml` as upstream ships them, including upstream's own install settings. They change only through an upstream sync.
- The Node.js major version comes from upstream's `engines` field and the exact pnpm version from its `packageManager` field. Corepack activates that pnpm.
- CI installs the tree with `pnpm install --frozen-lockfile` in `upstream/`, which fails when the lockfile and the manifest disagree. The install job has no secrets or write tokens.
- An upstream sync PR lists the packages that the sync adds, removes or updates in upstream's lockfile, with any license change, so reviewers assess them with the rest of the sync.
- Upstream's workflows and composite actions keep the action and tool pins upstream gives them.

## SION's own packages

A package that only SION's own code uses, such as the generator of the protocol TypeScript types, is a SION addition. It never goes into upstream's manifest or lockfile, and it follows these rules:

- SION's own packages are declared in SION's own `package.json` and `pnpm-lock.yaml` and installed with the same pnpm version as the upstream tree.
- Every entry names one exact version from the public npm registry. Ranges, dist-tags, `*`, Git URLs, tarball URLs and `file:` specs are rejected. Every lockfile entry resolves to `https://registry.npmjs.org/` and carries an integrity value.
- The lockfile is committed and changes only through pnpm, in the same PR as the manifest change that causes it. CI installs with `pnpm install --frozen-lockfile`.
- No dependency runs a lifecycle script. SION's pnpm settings allow no package to run install scripts, and SION's own `package.json` defines no install lifecycle script.
- Licenses allow distribution with SION under MIT when their notices are kept, such as MIT, Apache-2.0, BSD and ISC.
- A new release may be adopted as soon as it is published. No minimum release age applies to SION's own packages.
- Use Node.js built-in modules when they cover the need. The repository tooling under `.github/scripts/` uses built-in modules only.

## Protocol schema and fixtures

SION pins LINA's protocol JSON Schema and conformance fixtures by LINA release tag and content digest, and generates its TypeScript types from the pinned schema ([types](../design/sion.md#types)). Generated types are never edited by hand.

## Tools and actions

- Pin GitHub Actions in SION's own workflows by full commit SHA, with the release tag in a comment.
- Pin every binary that SION's CI downloads, such as actionlint, by version and SHA-256 digest. Verify each download against its digest before extraction, every time, including after a cache hit.

## CI enforcement

The `upstream` job of the [CI policy](ci.md#upstream-tree) installs the upstream tree from its frozen lockfile. The job that installs SION's own packages is registered with the first of them. It fails when an entry is not an exact registry version, when a lockfile entry resolves outside the npm registry or lacks an integrity value, or when SION's pnpm settings or `package.json` let an install script run. Either job's failure blocks `foundation`.
