# Crypto-View

Finds every use of cryptography in a codebase and grades it against the quantum
threat. Findings appear inline on the pull request; you decide what fails the
build.

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

Scans Java, Python, JavaScript/TypeScript and Go, plus PEM keys and
certificates, SSH keys, JOSE algorithms, TLS config and dependency manifests in
any language. The full list is at
[cryptoview.qcomply.tech/documentation](https://cryptoview.qcomply.tech/documentation).

## Failing the build

`fail-on` takes a **severity**, or a **selector** that gates on the kind of
cryptography.

```yaml
      - uses: QComplyAG/crypto-view-action@v1
        with:
          fail-on: hndl posture=vulnerable
```

That one fails on quantum-vulnerable key establishment and nothing else —
recorded traffic is decrypted retrospectively once the algorithm falls, so it is
the finding that cannot wait. A vulnerable *signature* of the same severity does
not block the build.

| Selector | Matches |
|---|---|
| `hndl` | Key establishment and public-key encryption |
| `posture=broken` | Broken today, not just by a quantum computer |
| `posture=vulnerable` | Broken by Shor's algorithm |
| `primitive=key-agree,kem` | By cryptographic primitive |
| `algorithm=RSA,ECDH` | By algorithm |
| `kind=call` | Call sites only, ignoring imports and dependencies |
| `language=java` | By language |
| `severity>=high` | That severity or worse |
| `high` | The same thing, spelled the old way |

Terms on one line must **all** match the same finding. Put alternatives on
separate lines — any one of them fails the build:

```yaml
          fail-on: |
            hndl posture=vulnerable
            posture=broken
```

### A policy file

For rules worth reviewing in a pull request, put them in a file. Each gate gets
a name and a message, and the developer sees both when it trips.

```yaml
        with:
          policy: crypto-view-policy.json
```

```json
{
  "version": 1,
  "gates": [
    {
      "name": "No quantum-vulnerable key exchange",
      "when": ["hndl", "posture=vulnerable"],
      "message": "Recorded traffic is exposed retrospectively. See ARCH-412."
    },
    { "name": "No broken primitives", "when": ["posture=broken"] }
  ],
  "score": { "min": 70, "max_drop": 5 }
}
```

`score.min` is a floor. `score.max_drop` fails when the score falls further than
this against a baseline.

### Adopting on an existing codebase

A large repository will have findings on day one. A baseline records them so
only *new* cryptography fails the build:

```bash
crypto-view . --write-baseline crypto-view-baseline.json
```

Commit that file, then:

```yaml
        with:
          baseline: crypto-view-baseline.json
          fail-on: hndl posture=vulnerable
```

Existing debt stays visible in the reports and stops blocking merges. Delete
entries from the baseline as they are fixed.

## Your own cryptographic functions

A codebase that wraps its cryptography in `my_own_crypto()` is invisible to a
scanner that only knows library APIs. Declare your own functions in
**`crypto-view.custom.json`** at the root of the scanned directory and they are
picked up automatically — no workflow change:

```json
{
  "version": 1,
  "functions": [
    {
      "id": "my-own-crypto",
      "name": "my_own_crypto",
      "language": "python",
      "quantum_safe": false,
      "primitive": "kem",
      "severity": "high",
      "symbols": ["my_own_crypto", "crypto_utils.wrap_key"],
      "description": "In-house key wrapping built on RSA-2048.",
      "remediation": "Use ML-KEM-768 through the platform crypto service."
    }
  ]
}
```

Every call site is reported like any catalogue finding, in the terminal, the
SARIF and the CBOM. Because `primitive` is `kem`, this one counts as key
establishment — so it trips `fail-on: hndl` exactly as `ECDH` would.

| Field | |
|---|---|
| `id` | Required. Becomes rule `custom.<id>` |
| `name` | Display name. Defaults to `id` |
| `symbols` | Names to match. Defaults to `name` |
| `quantum_safe` | Required unless `posture` is given. `false` → vulnerable, `true` → safe |
| `posture` | `vulnerable`, `broken`, `reduced`, `safe`, `unknown` |
| `primitive` | `kem`, `key-agree`, `signature`, `block-cipher`, `hash`, `mac`, `kdf`, … |
| `severity` | `critical`…`info`. Defaults to `high`, or `info` when safe |
| `language` | `java`, `python`, `javascript`, `go`, `any`. Defaults to `any` |
| `patterns` | Regexes, if the symbol names are not enough |
| `hndl` | Override whether this exposes recorded traffic |
| `description`, `remediation` | Shown with the finding |

Use `custom-rules:` to point at a different path.

## Inputs

| Input | Default | |
|---|---|---|
| `path` | `.` | Directory to scan |
| `fail-on` | `none` | What fails the build; see above |
| `policy` | | Path to a policy file |
| `custom-rules` | | Path to declared functions |
| `baseline` | | Only fail on findings absent from this file |
| `fail-under-score` | `0` | Fail below this readiness score |
| `include` / `exclude` | | Newline-separated globs |
| `sarif` | `crypto-view.sarif` | Where to write SARIF |
| `upload-sarif` | `true` | Upload to code scanning |
| `job-summary` | `true` | Write the inventory to the run summary |
| `cbom` | | Path to write a CycloneDX 1.7 CBOM |
| `qcomply-token` | | Report to a QComply account. Use a secret |
| `qcomply-url` | `https://app.qcomply.tech` | For a self-hosted deployment |
| `no-snippets` | `false` | Report without the matched source lines |
| `version` | `latest` | Which release to run |

## Outputs

`score`, `findings`, `actionable`, `hndl`, `sarif`, `cbom`.

```yaml
      - uses: QComplyAG/crypto-view-action@v1
        id: scan
      - run: echo "score ${{ steps.scan.outputs.score }}"
```

## Notes

Every artefact is written *before* the gate is evaluated, so a failing build
still leaves the SARIF, the summary and the CBOM behind.

The scan is local and contacts nothing unless you pass `qcomply-token`.

Uploading SARIF to the Security tab needs GitHub Advanced Security on private
repositories. Set `upload-sarif: false` if you do not have it; nothing else
changes.

Pin `@v1` for the current major version, or a full tag like `@v5.2.0`.

## Licence

Apache License 2.0 — see [LICENSE](LICENSE).
