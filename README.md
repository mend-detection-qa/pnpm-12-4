# multi-ecosystem-workspace

Probe for **pnpm 12.4** multi-ecosystem workspace support.

## Feature exercised

pnpm 12.4 introduces an `ecosystems:` block in `pnpm-workspace.yaml`
that declares Python and Cargo sub-projects alongside standard npm
packages. Each non-JS sub-project receives its own ecosystem-native
lockfile (`pylock.toml` for Python per PEP 751, `Cargo.lock` for
Rust), while JS packages continue to be tracked in `pnpm-lock.yaml`.
New coordinate prefixes (`pypi:`, `crate:`) and registry fields
(`cargo.indexUrl`) are part of the new manifest schema.

## Workspace layout

```
.
├── pnpm-workspace.yaml     # new ecosystems: block
├── pnpm-lock.yaml          # JS deps only (lockfileVersion: '9.0')
├── package.json            # root (private, no deps)
├── .whitesource            # Bucket A version pin
├── packages/
│   ├── api/                # JS sub-project
│   │   └── package.json    # depends on hono@^4.4.0
│   ├── ml/                 # Python sub-project
│   │   ├── pyproject.toml  # depends on requests>=2.31.0
│   │   └── pylock.toml     # PEP 751 lockfile (pnpm-generated)
│   └── cli/                # Rust/Cargo sub-project
│       ├── Cargo.toml      # depends on serde, serde_json, clap
│       └── Cargo.lock      # Cargo lockfile (v3)
```

## Expected dependency tree

The `expected-tree.json` covers **the JS pnpm sub-tree only**
(the scope of the pnpm resolver). The Python and Cargo sub-trees
are noted in `warnings[]` as separate resolver domains — Mend's
UA must route them to the Python and Cargo resolvers respectively,
not to the pnpm resolver.

JS direct dependencies (from `packages/api`):
- `hono@4.4.2` — registry, main dep
- `@hono/node-server@1.12.0` — registry, dev dep

Transitive:
- `@hono/node-server` depends on `hono` (already in tree)

## Mend detection notes

### pnpm resolver scope
The pnpm resolver (`PnpmLockCollector`) reads `pnpm-lock.yaml`.
In pnpm 12.4 the lockfile covers **only** the JS packages declared in
the `packages:` section of `pnpm-workspace.yaml`. Python and Cargo
packages do NOT appear in `pnpm-lock.yaml` and must NOT be reported
by the pnpm resolver.

### New `pnpm-workspace.yaml` `ecosystems:` block
The `ecosystems:` block is new in pnpm 12.4. The UA's YAML parser
must not throw on it. If it does, the scan produces zero deps (hard
failure mode). The block must be ignored by the pnpm resolver and
picked up by ecosystem-specific routers only.

### `pylock.toml` (PEP 751)
The Python sub-project carries `pylock.toml` — the new standardized
Python lockfile format. The Mend Python resolver must recognize this
lockfile alongside the existing `poetry.lock`, `Pipfile.lock`, etc.
If `pylock.toml` is not recognized, the Python sub-tree is silently
missed.

### `cargo.indexUrl` in workspace YAML
The `cargo.indexUrl` field in `pnpm-workspace.yaml` configures the
Cargo registry. The pnpm resolver MUST NOT interpret this as an npm
`registryUrl` or inject Cargo registry entries into the JS tree.

### Known Mend gap (as of resolver SHA 351915ae)
The upstream pnpm resolver documentation (`javascript.md`) covers
lockfile versions up to v9 and does not mention pnpm 12.4
multi-ecosystem workspaces. The `ecosystems:` block and
`pylock.toml` parsing are exploratory targets — there is no
regression baseline yet. The `expected-tree.json` encodes the
**correct** JS-only tree; divergence in the Python/Cargo sub-trees
signals a new Mend gap, not a regression.

## Mend config

Bucket A — `.whitesource` pins `pnpm: "12.4.0"` and
`node: "20.11.1"` (install-tool keys for js-pnpm). pnpm has no
dynamic version detection from the manifest; explicit pinning is
mandatory for reproducible transitive sets.

`configMode` is `"AUTO"` because no `whitesource.config` is
present in this probe.

## Resolver provenance

- Resolver file: `javascript.md`
- Upstream URL: https://raw.githubusercontent.com/whitesource/unified-agent/integration/.claude/knowledge/resolvers/javascript.md
- Fetched at: 2026-09-08T11:30:58+00:00
- Upstream SHA: 351915ae1d53c6db20db0a5d65b637fe7899d7ea
- Pattern: `multi-ecosystem-workspace` (added 2026-09-08)
- pm_version_under_test: 12.4.0
