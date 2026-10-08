# SOPS with STACKIT KMS

This repository is a fork of [getsops/sops](https://github.com/getsops/sops)
that adds STACKIT KMS as a key source. It exists until
[getsops/sops#2094](https://github.com/getsops/sops/pull/2094) is merged.
Files encrypted with this fork use the same metadata format as that pull
request, so upstream SOPS releases that include it can read them.

## Versions

Each release is an upstream release plus the STACKIT KMS change. Tags have the
form `vX.Y.Z-stackit.N`, where `vX.Y.Z` is the upstream release and `N` counts
fork releases on top of it.

## Installation

### Homebrew (macOS, Linux)

```shell
brew install stackitcloud/tap/sops
```

The formula is named `sops`, like the one in homebrew-core. If that one is
installed, remove it first with `brew uninstall sops`.

### mise

```shell
mise use -g github:stackitcloud/sops
```

### Binary (macOS, Linux)

```shell
VERSION=$(curl -fsSL https://api.github.com/repos/stackitcloud/sops/releases/latest | grep -o '"tag_name": *"[^"]*"' | cut -d'"' -f4)
OS=linux     # or darwin
ARCH=amd64   # or arm64
curl -fLo sops "https://github.com/stackitcloud/sops/releases/download/${VERSION}/sops-${VERSION}.${OS}.${ARCH}"
chmod +x sops
sudo mv sops /usr/local/bin/sops
```

Windows binaries (`sops-vX.Y.Z-stackit.N.amd64.exe`) are on the
[releases](https://github.com/stackitcloud/sops/releases) page.

### Debian, Ubuntu (`.deb`)

```shell
ARCH=amd64   # or arm64
curl -fsSL https://api.github.com/repos/stackitcloud/sops/releases/latest | grep -o "https://[^\"]*_${ARCH}\.deb" | xargs curl -fLO
sudo dpkg -i sops_*_${ARCH}.deb
```

### RHEL, Fedora, openSUSE (`.rpm`)

```shell
ARCH=x86_64   # or aarch64
curl -fsSL https://api.github.com/repos/stackitcloud/sops/releases/latest | grep -o "https://[^\"]*\.${ARCH}\.rpm" | xargs curl -fLO
sudo rpm -Uvh sops-*.${ARCH}.rpm
```

### Container image

```shell
docker run --rm ghcr.io/stackitcloud/sops:v3.13.3-stackit.1 --version
```

Images are tagged `vX.Y.Z-stackit.N` and `vX.Y.Z-stackit.N-alpine`. There is
no `latest` tag.

### Go

`go install` does not work, because the Go module path stays
`github.com/getsops/sops/v3`.

## Usage

Set the STACKIT KMS key in `.sops.yaml`:

```yaml
creation_rules:
  - stackit_kms: projects/<projectId>/regions/<regionId>/keyRings/<keyRingId>/keys/<keyId>/versions/<versionNumber>
```

On the command line, use `--stackit-kms` or `SOPS_STACKIT_KMS_IDS`.
`sops updatekeys`, `--add-stackit-kms` and `--rm-stackit-kms` work like the
other key sources.

Authentication uses the default
[STACKIT SDK credentials](https://github.com/stackitcloud/stackit-sdk-go#authentication).

## Releasing

The `stackit` branch is the latest upstream release tag plus the fork commits.
`main` is a copy of upstream `main` and gets no fork commits.

To release on top of a new upstream tag `vNEW`, where `vOLD` is the current
base:

```shell
git fetch upstream --tags
git checkout stackit
git rebase --onto vNEW vOLD stackit
```

If `go.mod` or `go.sum` conflict, take the upstream files and add the STACKIT
SDK again:

```shell
git checkout --ours go.mod go.sum
go get github.com/stackitcloud/stackit-sdk-go/core@latest github.com/stackitcloud/stackit-sdk-go/services/kms@latest
go mod tidy
git add go.mod go.sum
git rebase --continue
```

Then push the upstream tag, the branch and the release tag. The release notes
are generated from the previous tag, so the upstream tag must exist in this
repository. Wait for the CLI workflow to pass on `stackit` before pushing the
release tag.

```shell
git push origin refs/tags/vNEW
git push --force-with-lease origin stackit
git tag vNEW-stackit.1
git push origin vNEW-stackit.1
```

The release workflow publishes binaries, packages and images. It updates the
Homebrew formula in
[stackitcloud/homebrew-tap](https://github.com/stackitcloud/homebrew-tap) when
the `HOMEBREW_TAP_TOKEN` secret holds a token with write access to that
repository. Without the secret, the release skips the formula.
