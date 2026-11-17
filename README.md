# setup-roadcase

A GitHub Action that installs [roadcase](https://github.com/transcodeworks/roadcase), the
tool that packages, signs and updates desktop applications written in Rust, and the tools
its releases need on the runner.

- Downloads the `roadcase` program built for the runner (Linux and macOS on x86_64 and
  aarch64, Windows on x86_64) from roadcase's GitHub release. Nothing is compiled.
- Verifies it twice before running it: its SHA-256 against the release's, and its
  [build provenance attestation](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations)
  against the roadcase repository.
- Installs what `roadcase release` runs on that system, each tool pinned to an exact
  version and checksum: `appimagetool` and the AppImage runtime on Linux, NSIS on Windows.
- Puts `roadcase` on `PATH` for the steps that follow.

## Quick start

One job, every system: each runner packages its own release, since roadcase does not
cross-build, and `roadcase release` takes the system it runs on (roadcase 0.1.1 or later).

```yaml
jobs:
  release:
    strategy:
      fail-fast: false
      matrix:
        runner:
          - macos-15         # Apple silicon
          - macos-15-intel   # Intel
          - windows-2025
          - ubuntu-24.04     # x86_64
          - ubuntu-24.04-arm # aarch64
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v7
      - uses: dtolnay/rust-toolchain@stable
      - uses: transcodeworks/setup-roadcase@v1
      - run: roadcase release
        env:
          # The key that signs the update feed (`roadcase keygen`).
          ROADCASE_UPDATE_SECRET_KEY: ${{ secrets.ROADCASE_UPDATE_SECRET_KEY }}
          # macOS: your Developer ID Application certificate, a .p12 in base64,
          # and the Apple ID that notarizes, with its app-specific password.
          CODESIGN_CERTIFICATE: ${{ secrets.CODESIGN_CERTIFICATE }}
          CODESIGN_CERTIFICATE_PASSWORD: ${{ secrets.CODESIGN_CERTIFICATE_PASSWORD }}
          NOTARY_APPLE_ID: ${{ secrets.NOTARY_APPLE_ID }}
          NOTARY_PASSWORD: ${{ secrets.NOTARY_PASSWORD }}
          NOTARY_TEAM_ID: ${{ secrets.NOTARY_TEAM_ID }}
          # Windows: whatever the command in your [windows.sign] table reads.
```

The secrets are yours; the action reads none of them, and each system uses only its own.
With none set, roadcase still packages (ad hoc signed on macOS, unsigned elsewhere), which
is enough to try a workflow out. The
[roadcase README](https://github.com/transcodeworks/roadcase#readme) describes
`roadcase.toml` and every variable.

A system can still be named, and Linux can be limited to some formats:

```yaml
      - run: roadcase release linux --format appimage --format deb
```

## Inputs

| Input | Default | What it is |
| --- | --- | --- |
| `version` | `latest` | The roadcase version: `latest`, or a version such as `0.1.0` or `v0.1.0`. Name one for builds that must not change under you. |
| `token` | `${{ github.token }}` | A GitHub token, used only to ask GitHub for the latest release and to verify the attestation. The workflow's own token is enough, with no permission beyond the default. It is never printed, and the downloads themselves are anonymous. |
| `verify-attestation` | `true` | Verify the archive's build provenance attestation (`gh attestation verify <archive> --repo transcodeworks/roadcase`). See [Security](#security). |
| `tools` | `true` | Also install what `roadcase release` needs on this system; see the next section. `false` installs roadcase alone. |

## Outputs

| Output | What it is |
| --- | --- |
| `version` | The version installed, without a leading `v`: `0.1.0`. |
| `path` | The folder that holds the `roadcase` program, which is added to `PATH`. |

## What `tools` installs

| System | Installed | Left to you |
| --- | --- | --- |
| Linux | `appimagetool` 1.9.1 and the AppImage type-2 runtime 20251108 for the runner's architecture, each checked against a SHA-256 kept in this repository. Sets `APPIMAGETOOL` and `APPIMAGE_RUNTIME`, which roadcase reads, puts `appimagetool` on `PATH`, and sets `APPIMAGE_EXTRACT_AND_RUN=1`. | See below. |
| Windows | Nothing where `makensis` is found: on `PATH`, or in NSIS's own folder (which is then added to `PATH`), as on GitHub's runners. Otherwise NSIS 3.11, checked against a SHA-256 kept in this repository. | The signing tool your `[windows.sign]` command names. |
| macOS | Nothing: `codesign`, `notarytool` and `hdiutil` come with the runner. | |

`APPIMAGE_EXTRACT_AND_RUN=1` makes every AppImage a later step runs unpack itself
rather than mount itself, since GitHub's runners and containers have no FUSE 2.
`appimagetool` is an AppImage, so it needs this to run there. roadcase gives
`appimagetool` the pinned runtime, so nothing is downloaded while packaging.

The action never installs system packages. On Linux, GitHub's Ubuntu runners already
have the rest; a container or your own runner may not:

| For | You need | On Debian or Ubuntu |
| --- | --- | --- |
| An AppImage | `file`, which `appimagetool` runs; `desktop-file-validate`, which it runs when it is there | `apt-get install file desktop-file-utils` |
| Symbols kept apart (`symbols = true`) | `objcopy` and `strip` | `apt-get install binutils` |
| A `.deb` | `dpkg-deb`, and `dpkg-shlibdeps` to find the library dependencies | `apt-get install dpkg-dev` |
| A pacman package | `makepkg`, on Arch Linux, as someone other than root: run the job in an `archlinux` container | `pacman -S --needed base-devel` |

In a container the action itself needs `bash`, `curl` and `tar`, and `gh` for the
attestation.

## Security

- **Checksum.** The archive's SHA-256 must be the one in the release's `.sha256` file, or
  nothing is installed.
- **Attestation.** With `verify-attestation: true`, the GitHub CLI must confirm that the
  archive was built by a workflow of `transcodeworks/roadcase`. A failed verification
  stops the action: nothing is installed. If `gh` is not installed (it is on every
  GitHub-hosted runner; a container or your own runner may lack it), or is older than
  2.49, the action writes a warning and continues on the checksum alone. The checksum
  and the archive come from the same release, so the checksum alone guards against a
  damaged download, not against a release that was tampered with: install `gh` where
  that matters to you.
- **Pinned tools.** `appimagetool`, the AppImage runtime and NSIS are downloaded from an
  exact release, never a moving one, and checked against SHA-256 values in
  [`install.sh`](install.sh).
- **Pin the action.** `@v1` follows this repository's releases. To decide yourself when
  the action changes, name a commit, and let Dependabot or Renovate move it:

  ```yaml
  - uses: transcodeworks/setup-roadcase@<full commit SHA> # v1.0.0
  ```

- **Secrets.** The action takes no secret but `token`, sends it only to `api.github.com`
  and to `gh`, and never prints it. Your signing secrets go to the `roadcase` step, not
  to this one.

To report a vulnerability, use the repository's
[private vulnerability reporting](https://github.com/transcodeworks/setup-roadcase/security/advisories/new).

## Versioning

The action follows [Semantic Versioning](https://semver.org). `v1` is a tag that moves to
each `v1.x.y` release; `v1.x.y` tags never move. The action's version is its own: it says
nothing about the roadcase version, which is the `version` input. A change to the pinned
tools is a minor release.

## Licence

MIT; see [LICENSE](LICENSE).
