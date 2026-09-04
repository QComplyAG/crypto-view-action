# Crypto-View — GitHub Action

Find every use of cryptography in a codebase and grade it against the quantum
threat. Findings appear inline on the pull request through code scanning, an
inventory is written to the run summary, and the full result is available as a
[CycloneDX 1.7](https://cyclonedx.org/docs/1.7/json/) Cryptographic Bill of
Materials.

```yaml
permissions:
  contents: read
  security-events: write

jobs:
  crypto-view:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: QComplyAG/crypto-view-action@v1
```

The scan is entirely local and contacts nothing unless you give it a QComply
token. There is no separate switch for reporting — supplying the token is what
turns it on, so there is nothing to forget:

```yaml
      - uses: QComplyAG/crypto-view-action@v1
        with:
          qcomply-token: ${{ secrets.QCOMPLY_TOKEN }}
```

## What it detects

Java, Python, JavaScript/TypeScript and Go, plus formats that carry
cryptography in any language: PEM certificates and keys, SSH keys, JOSE
algorithms, TLS and SSH configuration, and dependency manifests.

Renamed imports are resolved before matching, so `from lib import encrypt as
enc` followed by `enc(...)` is found. Imports are reported separately from call
sites: importing a library shows what a file can reach, not that anything is
called.

The current list is published at
<https://cryptoview.qcomply.tech/documentation>.

## Failing the build

`fail-on` gates the job on severity. Every artifact — the SARIF, the summary,
the CBOM, the QComply report — is written before the gate is evaluated, so a
failing gate still leaves them behind, which is exactly when they are wanted.

```yaml
      - uses: QComplyAG/crypto-view-action@v1
        with:
          fail-on: high
```

## Inputs

| Input | Default | |
|---|---|---|
| `path` | `.` | Directory to scan |
| `fail-on` | `none` | `critical`, `high`, `medium`, `low`, `info` |
| `include` | | Newline-separated globs; only matching paths are scanned |
| `exclude` | | Newline-separated globs to skip |
| `sarif` | `crypto-view.sarif` | Where to write SARIF |
| `upload-sarif` | `true` | Upload to code scanning; needs `security-events: write` |
| `job-summary` | `true` | Write the inventory to the run summary |
| `cbom` | | Path to write a CycloneDX 1.7 CBOM to |
| `qcomply-token` | | Report to a QComply account. Pass a secret, never a literal |
| `qcomply-url` | `https://app.qcomply.tech` | For a self-hosted deployment |
| `no-snippets` | `false` | Report without the matched source lines |
| `version` | `latest` | Which release to run |

## Outputs

| Output | |
|---|---|
| `score` | Readiness score out of 100 |
| `findings` | Total number of findings |
| `actionable` | Findings that need a decision, excluding inventory |
| `hndl` | Findings that are key establishment, so exposed retrospectively |
| `sarif` | Path to the SARIF file |
| `cbom` | Path to the CBOM, if one was requested |

```yaml
      - uses: QComplyAG/crypto-view-action@v1
        id: crypto-view
      - run: echo "score ${{ steps.crypto-view.outputs.score }}"
```

## Private repositories

The scan, the run summary, the CBOM and reporting to QComply all work on
private repositories. Uploading SARIF to the Security tab does not — GitHub
gates code scanning on Advanced Security. Set `upload-sarif: false` if you do
not have it; nothing else changes.

## About this repository

This repository holds the action definition. Each release carries
`crypto-view.pyz` and its `.sha256`; the action downloads that pair by tag and
verifies the checksum before running it.

Pin `@v1` to track the current major version, or a full tag such as `@v5.1.0`
to pin exactly.

## Licence

Apache License 2.0 — see [LICENSE](LICENSE).
