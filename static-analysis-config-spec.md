# Static Analysis Repository Configuration Specification

## 1. Purpose

This document defines the TOML configuration used by a reporting agent to identify:

- local source-code repositories to analyze;
- static-analysis tools to run for each repository; and
- optional exceptions that suppress matching findings from the agent's report.

An exception means only that a matching finding is omitted from the report. It does not change the static-analysis tool itself, modify source code, or imply that the underlying finding is invalid.

## 2. Document structure

The document MUST be valid TOML.

The top-level document contains a schema version and one or more `repo` tables.

```toml
version = 1

[[repo]]
name = "example-repo"
path = "C:\\dev\\example-repo"
```

### `version`

- REQUIRED.
- Integer.
- Currently `1`.

## 3. Repository tables

Each `[[repo]]` table describes one local repository.

Required fields:

- `name`: A concise repository name.
- `path`: The local filesystem path to the repository.

Repositories are identified only by local name and path. Repository URLs are not used.

Example:

```toml
[[repo]]
name = "sbom-tool"
path = "C:\\dev\\sbom-tool"
```

## 4. Static-analysis tools

Each repository contains one or more `[[repo.tool]]` tables describing the static-analysis tools the agent should run.

Required fields:

- `name`: Name of the tool, such as `clippy`, `cargo-audit`, `eslint`, or `semgrep`.
- `command`: Exact command used to invoke the tool for this repository.

Example:

```toml
[[repo.tool]]
name = "clippy"
command = "cargo clippy --all-targets --all-features"

[[repo.tool]]
name = "cargo-audit"
command = "cargo audit"
```

Tool names need only be meaningful and consistent within the configuration.

## 5. Exceptions

A repository MAY contain zero or more `[[repo.exception]]` tables.

An exception suppresses a matching static-analysis finding from the final report.

Every exception MUST contain:

- `name`
- `tool`
- `rule`
- `reason`
- `expires`
- `hash`
- a `repo.exception.location` table

An exception MAY additionally contain:

- `build`

Example:

```toml
[[repo.exception]]
name = "vendor-api-has-many-arguments"
tool = "clippy"
rule = "clippy::too_many_arguments"
reason = "Signature intentionally mirrors the upstream vendor API."
expires = 2027-04-30
hash = "sha256:<hash>"

[repo.exception.location]
path = "src/vendor/api.rs"
symbol = "VendorClient::execute"
```

### `name`

- REQUIRED.
- MUST be kebab-case.
- MUST be a short, meaningful description of the exception.
- SHOULD communicate approximately the same justification as `reason`, but at a glance.

Good examples:

```text
vendor-api-has-many-arguments
test-preconditions-may-unwrap
validated-payload-length-cast
debug-diagnostics-may-unwrap
```

### `tool`

- REQUIRED.
- Identifies the static-analysis tool whose finding may be suppressed.
- MUST correspond to the tool that produced the finding.

### `rule`

- REQUIRED.
- Identifies the analyzer rule, lint, advisory, diagnostic, or equivalent finding identifier.

Examples:

```toml
rule = "clippy::unwrap_used"
rule = "clippy::too_many_arguments"
rule = "@typescript-eslint/no-explicit-any"
```

### `reason`

- REQUIRED.
- MUST briefly explain why the finding is intentionally omitted from the report.
- SHOULD be factual and specific.

### `expires`

- REQUIRED.
- TOML local date in `YYYY-MM-DD` form.
- An exception MUST NOT suppress a finding after its expiration date.

Example:

```toml
expires = 2027-04-30
```

### `build`

- OPTIONAL.
- Restricts the exception to a repository-specific build configuration.
- The value is intentionally not standardized globally. It MUST make sense for the repository being analyzed, for example `debug`, `release`, `Debug`, or `development`.
- If `build` is absent, the exception applies regardless of build configuration.
- If `build` is present, the exception applies ONLY when the agent knows the analyzed build matches this value.
- If the analyzed build is unknown, a build-specific exception MUST NOT be applied.

Example:

```toml
build = "debug"
```

## 6. Exception locations

Every exception MUST contain a `[repo.exception.location]` table specifying where the exception applies.

Supported fields are:

- `path`
- `symbol`
- `pattern`

### `path`

- REQUIRED unless the exception deliberately applies to the entire repository.
- May identify a file, directory, or glob-like path.
- Paths are relative to the repository root.

Examples:

```toml
path = "src/vendor/api.rs"
path = "tests/**"
path = "src/generated/**"
```

For a repository-wide exception, use:

```toml
[repo.exception.location]
scope = "repository"
```

`scope = "repository"` and `path` are mutually exclusive.

### `symbol`

- OPTIONAL.
- Identifies a named code element associated with the finding.
- Examples include a function, method, type, constant, or variable/binding.
- It further narrows the location identified by `path`.

Examples:

```toml
symbol = "Header"
symbol = "Header::encode"
symbol = "parse_header"
symbol = "MAX_PACKET_SIZE"
```

### `pattern`

- OPTIONAL.
- Identifies literal source text associated with the intended finding.
- It is used when `path` and optionally `symbol` are not sufficiently specific to distinguish the intended occurrence.

Example:

```toml
pattern = "payload.len() as u16"
```

Line numbers SHOULD NOT be used as exception locations because they are unstable as source files change.

Location fields narrow progressively. For example:

```text
path -> symbol -> pattern
```

All location constraints that are present MUST match before the exception is applied.

## 7. Exception matching

A finding may be omitted from the report only when ALL applicable constraints match.

At minimum:

```text
tool matches
AND rule matches
AND location matches
AND exception has not expired
AND (build is absent OR build matches the known analyzed build)
```

If any required match is uncertain, the exception SHOULD NOT be applied. The safe behavior is to report the finding.

## 8. Exception hashes

Every exception MUST contain a SHA-256 hash that represents the complete exception definition.

The hash protects the identity of the exception and makes any substantive modification detectable.

### Fields included

The hash input MUST include the values of every field in `repo.exception` and all descendant tables, including `repo.exception.location`, EXCEPT for the `hash` field itself.

Therefore changes to any of the following invalidate the hash when the field is present:

- `name`
- `tool`
- `rule`
- `reason`
- `expires`
- `build`
- `location.scope`
- `location.path`
- `location.symbol`
- `location.pattern`
- any future descendant field added to the exception schema

Fields outside the exception, such as repository name, repository path, and tool commands, are NOT part of the exception hash.

### Canonicalization

Implementations MUST construct the hash input deterministically.

1. Exclude the `hash` field itself.
2. Process fields in a defined canonical order, including descendant fields.
3. Include only fields that actually exist. Do not insert placeholder values for absent optional fields.
4. Represent dates as `YYYY-MM-DD`.
5. Preserve string values exactly, including case, after normalizing line endings in multiline strings to LF (`\n`).
6. Encode each canonical value as UTF-8.
7. Separate consecutive field values with a single NUL byte (`0x00`) so field boundaries are unambiguous.
8. Compute SHA-256 over the resulting byte sequence.
9. Encode the digest as lowercase hexadecimal.
10. Store it prefixed with `sha256:`.

Conceptually:

```text
value1 + NUL + value2 + NUL + value3 + ...
```

The separator is necessary because direct concatenation would make different sequences such as `ab` + `c` and `a` + `bc` indistinguishable.

Example stored value:

```toml
hash = "sha256:670083c6..."
```

An exception whose stored hash does not equal its calculated hash MUST be treated as invalid and MUST NOT suppress a finding.

## 9. Complete example

```toml
version = 1

[[repo]]
name = "sbom-tool"
path = "C:\\dev\\sbom-tool"

[[repo.tool]]
name = "clippy"
command = "cargo clippy --all-targets --all-features"

[[repo.tool]]
name = "cargo-audit"
command = "cargo audit"

[[repo.exception]]
name = "vendor-api-has-many-arguments"
tool = "clippy"
rule = "clippy::too_many_arguments"
reason = "Signature intentionally mirrors the upstream vendor API."
expires = 2027-04-30
hash = "sha256:<generated-hash>"

[repo.exception.location]
path = "src/vendor/api.rs"
symbol = "VendorClient::execute"

[[repo.exception]]
name = "test-preconditions-may-unwrap"
tool = "clippy"
rule = "clippy::unwrap_used"
reason = "Panicking on failed preconditions is intentional in integration tests."
expires = 2027-04-30
hash = "sha256:<generated-hash>"

[repo.exception.location]
path = "tests/**"

[[repo.exception]]
name = "validated-payload-length-cast"
tool = "clippy"
rule = "clippy::cast_possible_truncation"
reason = "Protocol validates the payload length is within u16 range before this cast."
expires = 2027-01-31
hash = "sha256:<generated-hash>"

[repo.exception.location]
path = "src/protocol/header.rs"
symbol = "Header::encode"
pattern = "payload.len() as u16"

[[repo.exception]]
name = "debug-diagnostics-may-unwrap"
tool = "clippy"
rule = "clippy::unwrap_used"
reason = "Intentional in the debug-only diagnostic initialization path."
expires = 2027-04-30
build = "debug"
hash = "sha256:<generated-hash>"

[repo.exception.location]
path = "src/debug/diagnostics.rs"
symbol = "Diagnostics::initialize"


[[repo]]
name = "log-viewer"
path = "C:\\dev\\log-viewer"

[[repo.tool]]
name = "eslint"
command = "npm run lint"

[[repo.tool]]
name = "semgrep"
command = "semgrep scan --config auto ."

[[repo.exception]]
name = "legacy-import-allows-dynamic-properties"
tool = "eslint"
rule = "@typescript-eslint/no-explicit-any"
reason = "External legacy log records may contain arbitrary structured properties."
expires = 2027-06-30
hash = "sha256:<generated-hash>"

[repo.exception.location]
path = "src/import/legacy/**"
```

## 10. Summary of required behavior

A conforming reporting agent MUST:

1. Analyze each configured local repository using its configured static-analysis tools.
2. Treat exceptions solely as report suppressions.
3. Validate an exception's hash before using it.
4. Ignore expired exceptions.
5. Require all supplied location constraints to match.
6. Apply a build-specific exception only when the analyzed build is known and matches.
7. Report a finding whenever applicability of an exception is uncertain.
8. Never infer a new exception merely because an existing exception is similar.
