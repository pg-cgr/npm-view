# npm-view

A CLI tool that reads a `package.json` file and reports provenance and attestation information for every dependency, querying whichever npm registry you have configured.

## Requirements

- Node.js (any modern version)
- npm (used internally to query the registry)

## Installation

Copy the script to somewhere on your `PATH`:

```
cp npm-view ~/bin/npm-view
chmod +x ~/bin/npm-view
```

## Usage

```
npm-view [--upstream] [--csv] [--output <file>] <package.json>
```

### Basic

```
npm-view package.json
```

Reads all dependencies (`dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`) from the given `package.json` and queries the registry for each one. Results stream to the terminal as each package resolves.

### Flags

| Flag | Description |
|---|---|
| `--upstream` | Force queries against `https://registry.npmjs.org`, ignoring your configured registry |
| `--csv` | Output CSV instead of a formatted table. Values are not truncated. No summary lines. |
| `--output <file>` | Write output to a file in addition to the terminal. Table output is written without ANSI color codes. |

Flags can appear in any order relative to the filename:

```
npm-view --upstream --output report.txt package.json
npm-view --csv --output report.csv package.json
npm-view package.json --upstream
```

## Output

### Table mode (default)

```
package                   version          resolved         source                  attestation url         provenance
────────────────────────  ───────────────  ───────────────  ──────────────────────  ──────────────────────  ──────────
axios                     1.15.2           1.15.2           https://registry.npm…   https://registry.npm…   yes
react                     ^18.2.0          18.3.1           https://registry.npm…   https://registry.npm…   yes
lodash                    ^4.17.21         4.18.1           https://registry.npm…   N/A                     no
../analytics-db-client    1.0.0            –                –                       –                       local
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
provenance coverage:  62%  (8 of 13 packages)
provenance sources:   registry.npmjs.org 5 (38%)  ·  libraries.cgr.dev (your registry) 3 (23%)  ·  none 5 (38%)
```

**Columns**

| Column | Description |
|---|---|
| `package` | Package name as it appears in `package.json` |
| `version` | Version spec from `package.json` (e.g. `^18.2.0`) |
| `resolved` | Actual version found on the registry. Highlighted in yellow when it differs from the spec |
| `source` | Tarball URL from the registry (`dist.tarball`) |
| `attestation url` | URL to the package's attestation data, if present |
| `provenance` | `yes` (green) if the registry reports provenance, `no` (yellow) if not |

**Color coding**

| Color | Meaning |
|---|---|
| Cyan | Package name |
| Yellow | Resolved version differs from the requested spec |
| Green | Provenance present |
| Yellow | No provenance |
| Red | Package could not be resolved |
| Dim | URLs, separators, local/skipped entries |

**Special rows**

- **local** — Package name or version spec looks like a local path (`./`, `../`, `file:`) or a git reference (`git+`, `github:`, `bitbucket:`, `gitlab:`). These are skipped and excluded from the summary counts.
- **–** (red) — Package was not found on the registry.

**Summary lines**

```
provenance coverage:  62%  (8 of 13 packages)
provenance sources:   registry.npmjs.org 5 (38%)  ·  libraries.cgr.dev (your registry) 3 (23%)  ·  none 5 (38%)
```

- **provenance coverage** — percentage of resolved packages that have provenance. Color: green (100%), yellow (≥50%), red (<50%).
- **provenance sources** — breakdown by attestation hostname. Your configured registry is labeled `(your registry)` in bold.

Local packages and unresolvable packages are excluded from both counts.

### CSV mode

```
npm-view --csv package.json
```

Outputs a standard CSV with a header row. Values are never truncated. Fields containing commas or quotes are properly escaped.

```
package,version,resolved,source,attestation url,provenance
axios,1.15.2,1.15.2,https://registry.npmjs.org/axios/-/axios-1.15.2.tgz,https://registry.npmjs.org/-/npm/v1/attestations/axios@1.15.2,yes
react,^18.2.0,18.3.1,https://registry.npmjs.org/react/-/react-18.3.1.tgz,https://registry.npmjs.org/-/npm/v1/attestations/react@18.3.1,yes
../analytics-db-client,1.0.0,,,, local
```

Unresolvable packages have `error` in the resolved/source/attestation/provenance fields. Local packages have empty fields and `local` in the provenance column. No summary lines are included.

### Writing to a file

```
npm-view --output report.txt package.json
npm-view --csv --output report.csv package.json
```

When `--output` is given, output is written to both the terminal and the specified file simultaneously. Table output in the file is stripped of ANSI color codes. CSV output is written as-is (no colors to strip).

## Registry behavior

The tool inherits npm's standard registry configuration. It does not download packages — all queries are read-only metadata lookups against the registry API (`npm view`).

Registry resolution order:

1. Project-level `.npmrc` (current directory and parent directories)
2. User-level `~/.npmrc`
3. Global npm config (`npm config get registry`)
4. npm default: `https://registry.npmjs.org`

Use `--upstream` to bypass your configured registry and query `https://registry.npmjs.org` directly. This is useful for comparing what your proxy serves versus what upstream has.

## Version resolution

Version specs like `^18.2.0` or `~4.17.0` are resolved by stripping the range operator before querying (e.g. `^18.2.0` → `18.2.0`). If that exact patch does not exist on the registry (some packages skip patch versions), the tool automatically retries with `major.minor` then `major` until a match is found. The resolved version is shown in the `resolved` column.

## Disabling color

Set `NO_COLOR=1` to disable ANSI color output (useful when piping or redirecting):

```
NO_COLOR=1 npm-view package.json
```
