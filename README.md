# smartin

[![npm](https://img.shields.io/npm/v/smartin.svg)](https://www.npmjs.com/package/smartin)
[![Node](https://img.shields.io/badge/Node-%3E%3D22.12.0-blue?logo=nodedotjs)](https://nodejs.org)
[![Bun](https://img.shields.io/badge/Bun-%3E%3D1.3.0-blue?logo=bun)](https://bun.sh)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

A CLI for JavaScript and TypeScript projects; 3 features:

- **License**: checks, writes, and removes copyright, SPDX license, `NOTE`, and `FILE` header lines in source files.
- **Export**: bundles project files into plain text files, e.g. as LLM context.
- **Spec**: keeps numbered specs in a `SPEC.md` in sync with `@spec` references in code and test comments.

## Installation

```sh
npm i -D smartin
```

```sh
bun add -d smartin
```

- CLI only: the package cannot be imported.
- Runs on Node `>= 22.12.0` or [Bun](https://bun.sh) `>= 1.3.0`. The bin has a `node` shebang, so `npx smartin`, `bunx smartin` and `bun smartin` all run it in Node when Node is installed. To run it in Bun, add `--bun`: `bunx --bun smartin`, or `bun --bun run <script>` for a `package.json` script.
- Tested on Linux, Windows, and macOS.
- TypeScript is not required. The package has no runtime dependencies.

## How to use

Run all commands from the project root.

Every command requires the config file in the project root and the section of its feature (`license`, `export`, or `spec`). Without them, the command exits `2`. Create the config file:

```sh
npx smartin --init
```

```sh
bunx smartin --init
```

This writes `smartin.jsonc` (`--init --raw`: without comments). Fill in the `""` values or set them to `false`:

```jsonc
{
  "version": "0.1.0", // smartin version that wrote this file
  "eol": "LF", // ["LF", "CRLF"], omitted = LF
  "license": {
    "copyright": {
      "name": "", // ["Your Name", false = off, omitted = off]
      "year": "", // [2026, "2026", "2025-2026", true = current year, false = off, omitted = off]
      "email": false // ["your@mail.com", false = off, omitted = off]
    },
    "spdx": "", // ["MIT", false = off, omitted = off]
    "note": false, // ["text", false = off, omitted = off]
    "fileHeader": false, // [true = add header with path/to/file, false = off, omitted = off]
    "filter": ["*.ts"], // gitignore syntax, "!" excludes
    "packageBanner": false // [true = check package.json banner, false = off, omitted = off] - --banner="/*! Copyright <year> <name>... */"
  },
  "export": {
    "filter": ["*.ts", "package.json", "tsconfig*.json"] // gitignore syntax, "!" excludes
  },
  "spec": {
    "code": ["src/**/*.ts"], // gitignore syntax, "!" excludes
    "tests": ["tests/**/*.ts"], // gitignore syntax, "!" excludes, false = off
    "idLength": 4 // 1-9
  }
}
```

`smartin --init <dir>` (e.g. `--init packages/a`) writes a nested config into the subfolder `<dir>`: the same template without `eol` and `spec`. `<dir>` must exist and lie below the current directory. See [Nested configs](#nested-configs-license-and-export).

### Config file

- `smartin.json` (same content) works as well. Keep only one of the two per directory (else exit 2).
- Every key is required, except:
  - `eol`: omitted means `LF`.
  - the license header keys `copyright` (with `name`, `year`, `email`), `spdx`, `note`, `fileHeader`, and `packageBanner`: omitted or `false` means off.
  - `version`: written by `--init`, not read.
- The template comments list the accepted values in `[...]`. Apart from `version`, a key whose comment does not say `omitted` is required. `""` is invalid.
- A command checks the top-level keys, `eol`, and the section of its own feature. The section must be an object: a missing section, `false`, or any other value exits `2`. Sections of other features are not checked.
- Unknown keys at the top level or in the checked section exit `2`.
- Filters (`license.filter`, `export.filter`, `spec.code`, `spec.tests`) select files:
  - gitignore syntax, `!` excludes, the last matching pattern wins;
  - patterns are relative to the folder of the config file that defines them;
  - required and non-empty (`[]` exits `2`);
  - each filter must match at least one file, else exit `2`.
- `eol` sets the line ending of every line smartin writes, in all features: license headers, spec entries and `@spec` texts, and the export output. Lines of an existing file that smartin does not change keep their line endings.

### Ignored files

- `.gitignore` files are always respected by all features; this cannot be disabled. smartin reads:
  - every `.gitignore` from the project root upwards, up to and including the first directory that contains `.git` (or up to the filesystem root if there is none);
  - every `.gitignore` in subfolders, each for its own subtree.
- All features always skip `.git`, `node_modules`, and `.smartin` folders (at any depth), symbolic links, and `.env` / `.env.*` files.

### Nested configs (license and export)

For monorepos, put a config file into a subfolder. Create it from the project root with `smartin --init <dir>`.

- It governs that subfolder for `license` and `export`, until a deeper config takes over.
- The nearest config wins, nothing is inherited: restate every setting you need (e.g. `copyright.name`). Its filter patterns are relative to its own folder.
- Each nested config must contain the section of the running feature (`license` or `export`), else exit `2`. A nested config cannot turn a feature off for its subfolder.
- A nested config must not contain `eol` or `spec` (any value, also `false`), else exit `2` in every feature.
- To skip a subfolder, exclude it in the outer filter with `!dir/` (e.g. `"!packages/legacy/"`). A subfolder skipped this way or excluded by a `.gitignore` is not entered, so its config is not read.
- The report lists the config files in use (text: `⚙ Configs`, shown when a nested config exists; JSON: `configs`). Keys of the running feature that differ from the root config are shown as info (`license/config-deviation`, `export/config-deviation`). Info does not change the exit code.
- `spec` uses only the root config (see [Limitations](#limitations)).

## Commands

### General

| Command | Description |
|---------|-------------|
| `smartin --init` | creates `smartin.jsonc` template file (must not exist) |
| `smartin --init <dir>` | creates a nested config in the subfolder `<dir>`, without `eol` and `spec` |
| `smartin --init [dir] --raw` | same as `--init`, without comments |
| `--help` (`-h`) | prints the help; `smartin <feature> --help` prints the help of one feature |
| `--version` (`-v`) | prints the version |
| `--no-color` (or `NO_COLOR=1`) | disables colored output |
| `--format text\|json` | output format; `text` is the default (use it for agents), `json` is for scripts |

Exit codes: `0` clean (info and warnings only), `1` findings, `2` invalid invocation, missing or invalid config, no matched files, or file errors.

### Feature: License

| Command | Description |
|---------|-------------|
| `smartin license check` | check & report header issues (read-only) |
| `smartin license apply` | write or update own headers |
| `smartin license remove` | remove own headers |
| `smartin license remove --all` | remove all headers, including foreign ones |

The own block starts at line 1 (line 2 after a shebang). It holds up to four lines in this order, each only when enabled in the config:

```ts
// FILE: src/example.ts
// Copyright 2026 Your Name <mail@domain.com>
// SPDX-License-Identifier: MIT
// NOTE: Generated headers are managed by smartin
```

- `license.filter` selects the files.
- `check`, `apply`, and `remove` require the config file and the `license` section (else exit `2`).
- `check` and `remove` also accept configs that enable no header line. No own header is then expected: `check` reports existing header lines, `remove --all` deletes them. `apply` requires every config to enable at least one header line.
- `apply` updates an existing own block in place: it replaces values, adds missing lines, and deletes disabled lines. Without an own block, it inserts one at line 1, above any existing comments.
- A copyright line is own when its name equals `copyright.name` (case-insensitive). A copyright line with any other name is foreign (a third-party header).
- Lines outside the own block are never changed. Foreign, misplaced, or duplicate header lines are only reported.
- A file may contain only one license line, the one in the own block. Every other license line is an error.
- `check` exits `1` when `apply` would change something, so it works as a CI gate.
- `remove --all` deletes every header line at the top of the file (own and foreign). Header lines below the first non-header line (code or another comment) stay.
- `packageBanner: true` checks (never edits) the copyright banner in the `scripts` of the `package.json` next to that config file, e.g. `--banner="/*! Copyright 2026 Your Name. SPDX-License-Identifier: MIT */"`. That `package.json` must exist.
- `FILE` paths are relative to the project root, also for nested configs.
- To change `copyright.name`, run `smartin license remove` with the old config first, then change the name and run `apply`. Otherwise the old lines count as foreign.

### Feature: Export

| Command | Description |
|---------|-------------|
| `smartin export` | bundle files as `.txt` files into `.smartin/` |

- `export.filter` selects the files (gitignore syntax, `!` excludes). Gitignored files are never included.
- Each file starts with `// FILE: <path>`. Paths are relative to the project root, also for nested configs.
- Every output line ends with the root `eol`: file contents are converted, whatever line endings the source files use.
- Output in `.smartin/`:
  - `export_single/`: one `.txt` per file.
  - `export_tests/`: one `<dir>.txt` per `tests/<dir>`, `_root.txt` for files directly in `tests/`.
  - `export_packages/<pkg>/`: `src.txt` plus one `<dir>.txt` per `packages/<pkg>/tests/<dir>`.
- Every run deletes and rewrites all `export_*` folders. Other content of `.smartin/` stays.
- If any selected file cannot be read, nothing is written (exit 2).

### Feature: Spec

| Command | Description |
|---------|-------------|
| `smartin spec check` | check & report spec issues (read-only) |
| `smartin spec sync` | update test-status, texts & append missing entries (code texts overwrite SPEC.md/tests) |
| `smartin spec sync --from-spec` | same as sync; SPEC.md texts overwrite code/tests |
| `smartin spec update <id> <text>` | update text of id everywhere |
| `smartin spec update <id> --lines <line>...` | same as update, with a multi-line text (one argument per line) |
| `smartin spec lookup <id>` | list where id is used |
| `smartin spec next` | print next free spec id |

`--format json` works for every spec action except `update`.

The spec file is the first one found of `SPEC.md`, `spec.md`, `docs/SPEC.md`, `docs/spec.md`, relative to the project root. It lists one entry per spec:

```md
- 🟢 @spec 0001: The CLI prints the help
- 🔴 @spec 0002: Unknown flags exit 2
```

Code and test files reference a spec in a JSDoc comment, with or without the text:

```ts
/** @spec 0001: The CLI prints the help */
/** @spec 0002 */
```

- `spec.code` and `spec.tests` select the code and test files. A file matched by both is a test file.
- `spec.idLength` (1-9) is required: the number of digits of a spec id.
- `sync` sets the status: `🟢` a test references the id, `🔴` no test does.
- `spec.tests: false` is the only way to turn tests off; every status is then `⚪`. An omitted `spec.tests`, `[]`, or a list that matches no file is an error (exit 2).
- Every line spec adds or changes ends with the root `eol`. Other lines keep their line endings.
- `sync` appends specs that exist only in code to the spec file.
- Only JSDoc comments (`/** */`) count. `//` and `/* */` comments are ignored.
- `§` instead of a status marks a reference spec: it has an id but is not expected in code.
- `sync` and `update` write only the spec file and the text part of `@spec` statements.

## Limitations

- **Project root only:** smartin uses the current directory as the project root and never searches upwards for the config file. Run every command in the folder that contains the root config file. Started in a subfolder, smartin treats that subfolder as the project root.
- Spec only uses the root config; a nested config must not contain `spec`. Run smartin inside a sub-package to use its own spec file.
- License headers use `//` only (no `/* ... */` headers detected/updated).
- The copyright syntax is strict: `// Copyright [YYYY|YYYY-YYYY] Name [<email>]`. Lines with `(c)`, `©`, or a trailing `.` are not recognized as copyright lines: they are neither updated nor reported.
- UTF-8 only: a file with invalid UTF-8 or NUL bytes cannot be read. Any file error means nothing is analyzed or written (exit 2), in all features; only the file errors are reported.
- Each file is written in one step, but a failed write (e.g. a read-only file) does not undo files already written. It is reported as an error (exit 2).
- Filter case sensitivity follows the platform: case-insensitive on Windows and macOS, case-sensitive on Linux.

## License

Copyright 2026 Martin Winkler

Licensed under the [Apache License 2.0](LICENSE).
