# Security policy

## Reporting a vulnerability

Email **security@qcomply.tech**. Please do not open a public issue for a
suspected vulnerability.

Include what you need to make it reproducible: the version (`crypto-view
--version`), the input that triggers it, and what you expected instead. If you
would like a reply encrypted, say so and we will exchange keys.

**What to expect.** We acknowledge within three working days and give an
assessment within ten. If we agree it is a vulnerability we will tell you when a
fix is planned, and we will credit you in the release notes unless you would
rather we did not. If we disagree we will say why, in enough detail that you can
argue with us.

## Scope

| In scope | |
| --- | --- |
| `QComplyAG/crypto-view-action` | The GitHub Action and its released artifacts |
| `QComplyAG/crypto-view` | The scanning engine, the `crypto-view.pyz` zipapp, and `install.sh` |
| cryptoview.qcomply.tech | The public scanning site |
| app.qcomply.tech | The QComply application and its ingest API |

Out of scope: findings against a third-party repository that Crypto-View merely
scanned, reports produced by an automated scanner with no accompanying analysis,
and issues that require an attacker to already control the machine the CLI runs
on.

## What the tool does with your code

The scan itself reads the filesystem and writes its output. It makes no external
network call, and there is no telemetry and no rule fetching.

Two things do use the network, and both are things you ask for:

- **Reporting.** Supplying a QComply token - `QCOMPLY_TOKEN`, or the
  `qcomply-token` input on the Action - is what turns reporting on. There is no
  separate switch. Without a token nothing leaves the machine.
- **`crypto-view update`.** Fetches the current release and checks it against
  the published SHA-256 before replacing anything. If the digest does not match,
  the existing copy is left exactly where it was.

## Verifying a release

Every release publishes `crypto-view.pyz` alongside `crypto-view.pyz.sha256`.
The Action verifies the checksum before running the downloaded file, and so
should you:

```bash
sha256sum -c crypto-view.pyz.sha256
```

The build is reproducible: the same source produces a byte-identical zipapp, so
two people building the same tag get the same digest.

## Supported versions

The current release is supported. Older releases receive fixes only where the
issue is severe and the upgrade path is genuinely blocked - the zipapp has no
install step and no dependencies, so upgrading is replacing one file.
