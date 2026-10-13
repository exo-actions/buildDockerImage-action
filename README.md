# buildDockerImage-action
Action for Docker image build, sign, attest, cosign, and multi-arch manifest merging.

## Sub-actions

| Action | Description |
|---|---|
| `build-and-push-image` | Builds and pushes Docker images with BuildKit, cache, SBOM, provenance |
| `merge-manifest` | Creates a multi-arch manifest list from single-platform images by digest |
| `mirror-image` | Copies a multi-arch image between registries (all platforms, annotations preserved) |
| `sign-image` | (Deprecated) Signs images via Docker Content Trust |
| `attest-image` | Attests images using GitHub attest-build-provenance and/or attest-sbom |
| `cosign-image` | Cosigns images via sigstore/cosign, with optional post-sign verification |
| `notation-image` | Signs images via the Notary Project (Notation) |

There is also a top-level convenience action (`exo-actions/buildDockerImage-action@v1`) that chains `build-and-push-image` → `sign-image` → `attest-image` → `cosign-image` → `notation-image`, and optionally `mirror-image` (set `mirrorTargetRegistry` to mirror the freshly built image to another registry in the same job). It accepts the union of those sub-actions' inputs. `merge-manifest` is not part of this chain — the native multi-arch flow below needs a build matrix across jobs, so it must be called as a separate job.

---

## Basic Usage (single arch)

```yaml
name: Create and publish a Docker image

on:
  push:
    tags: ['*']
    branches: ['master']
env:
  BRANCH_BUILD_TAGS: "latest"
jobs:
  parse-docker-build-env:
    name: 'Parse Docker Build Environment'
    runs-on: ubuntu-latest
    outputs:
      buildTags: ${{ steps.detect-push-event.outputs.buildTags }}
    steps:
      - name: Check if push is a tag or branch
        id: detect-push-event
        run: |
          if [[ $GITHUB_REF == refs/tags/* ]]; then
            echo "buildTags=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT
          elif [[ $GITHUB_REF == refs/heads/* ]]; then
            echo "buildTags=${{ env.BRANCH_BUILD_TAGS }}" >> $GITHUB_OUTPUT
          fi
  build:
    runs-on: ubuntu-latest
    needs: parse-docker-build-env
    steps:
      - uses: exo-actions/buildDockerImage-action/build-and-push-image@v1
        with:
          dockerImage: "exoplatform/example"
          dockerImageTag: ${{ needs.parse-docker-build-env.outputs.buildTags }}
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

---

## Native Multi-Arch Build (no QEMU emulation)

Build amd64 + arm64 in parallel on native runners, then merge into a multi-arch manifest. Zero intermediate tags, no QEMU overhead.

```yaml
jobs:
  parse:
    runs-on: ubuntu-latest
    outputs:
      buildTags: ${{ steps.parse.outputs.buildTags }}
    # ... (same tag/branch parsing as above)

  build-platforms:
    name: "Build ${{ matrix.platform }}"
    runs-on: ${{ matrix.runner }}
    needs: parse
    strategy:
      matrix:
        include:
          - platform: linux/amd64
            runner: ubuntu-latest
            arch: amd64
          - platform: linux/arm64
            runner: ubuntu-24.04-arm
            arch: arm64
    steps:
      - uses: exo-actions/buildDockerImage-action/build-and-push-image@v1
        id: build
        with:
          dockerImage: "exoplatform/example"
          dockerImageTag: ${{ needs.parse.outputs.buildTags }}
          platforms: ${{ matrix.platform }}
          qemu: "false"
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      - name: Save digest
        run: echo "${{ steps.build.outputs.digest }}" > "/tmp/${{ matrix.arch }}.digest"
      - uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.arch }}-digest
          path: /tmp/${{ matrix.arch }}.digest

  merge-manifest:
    runs-on: ubuntu-latest
    needs: [parse, build-platforms]
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: amd64-digest
          path: /tmp/digests
      - uses: actions/download-artifact@v4
        with:
          name: arm64-digest
          path: /tmp/digests
      - uses: exo-actions/buildDockerImage-action/merge-manifest@v1
        with:
          dockerImage: "exoplatform/example"
          dockerImageTag: ${{ needs.parse.outputs.buildTags }}
          digestsDir: /tmp/digests
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

This pushes each platform to the **same final tag**, saves the digest as a GitHub artifact, then `merge-manifest` recreates the manifest list from the collected digests. No suffix tags, no registry cleanup.

---

## Multi-Registry with Mirroring

For workflows that publish to multiple registries, build once to the primary registry, then mirror to secondary registries. Avoids rebuilding the same image.

```yaml
jobs:
  # ... build-platforms and merge-dockerhub (same as above)

  mirror-to-ghcr:
    runs-on: ubuntu-latest
    needs: [parse, merge-dockerhub]
    permissions:
      packages: write
    steps:
      - uses: exo-actions/buildDockerImage-action/mirror-image@v1
        id: mirror
        with:
          sourceImage: "exoplatform/example"
          sourceTag: ${{ needs.parse.outputs.buildTags }}
          targetRegistry: ghcr.io
          SOURCE_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          SOURCE_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
          TARGET_USERNAME: ${{ secrets.GHCR_USERNAME }}
          TARGET_PASSWORD: ${{ secrets.GHCR_TOKEN }}

  cosign-ghcr:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    needs: mirror-to-ghcr
    steps:
      - uses: exo-actions/buildDockerImage-action/cosign-image@v1
        with:
          dockerImage: "exoplatform/example"
          dockerImageTag: ${{ needs.mirror-to-ghcr.outputs.tagsJson }}
          dockerImageDigest: ${{ needs.mirror-to-ghcr.outputs.digest }}
          dockerRegistry: ghcr.io
          cosignImage: true
          cosignOidcImage: true
          DOCKER_USERNAME: ${{ secrets.GHCR_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.GHCR_TOKEN }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
```

`mirror-image` uses `crane copy` to transfer the full multi-arch image (all platforms + annotations) between registries.

---

## Outputs

| Action | Output | Description |
|---|---|---|
| `build-and-push-image` | `tags` | Pushed image tags |
| `build-and-push-image` | `digest` | Pushed image digest (`sha256:...`) |
| `merge-manifest` | `digest` | Digest of the merged multi-arch manifest (first tag) |
| `mirror-image` | `digest` | Digest of the mirrored image (first tag) |
| `mirror-image` | `tagsJson` | Mirrored tags as a JSON array |

`sign-image`, `attest-image`, `cosign-image` and `notation-image` produce no outputs; feed them the `digest` from the build step.

---

## Choosing a signing mechanism

| | `cosign-image` | `notation-image` |
|---|---|---|
| Ecosystem | Sigstore | Notary Project (CNCF) |
| Keyless (OIDC) | Yes (`cosignOidcImage`) | No |
| Key material | Private key, or keyless | KMS-backed plugin, or a local key and certificate |
| Verification | Against the signing identity or public key | Against an explicit `trustpolicy.json` and trust store |
| Pick it when | You want zero key management on GitHub-hosted runners | Your consumers (e.g. Ratify, Azure/AWS admission) enforce Notary signatures |

Both sign the digest, so they can be combined on the same image. Docker Content Trust (`sign-image`) is deprecated; prefer one of the two above.

## CI

`Lint actions` runs actionlint and zizmor on every push. `Self-test composite actions` runs the sub-actions end to end against ghcr.io (manually, and on same-repo PRs touching an `action.yml`); `notation-image` is covered end to end (sign and verify) in local-key mode with a throwaway key; plugin mode needs a KMS and is not covered. All third-party actions are pinned to commit SHAs.

---

## Inputs

### build-and-push-image

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name (e.g. `exoplatform/exo-community`) | *(required)* |
| `dockerImageTag` | Docker image tag (comma-separated for multiple) | `latest` |
| `dockerFileContext` | Dockerfile context path | `.` |
| `dockerFile` | Path to the Dockerfile (defaults to `{context}/Dockerfile`) | `""` |
| `target` | Target build stage for a multi-stage Dockerfile | `""` |
| `buildContexts` | Additional named build contexts (multi-line, `name=path`) | `""` |
| `dockerRegistry` | Docker registry | `docker.io` |
| `platforms` | Comma-separated target platforms | `linux/amd64` |
| `generateSBOM` | Generate SBOM for the image | `true` |
| `generateProvenance` | Generate provenance for the image | `true` |
| `buildArgs` | Build-time arguments (multi-line, `KEY=VALUE`) | `""` |
| `secrets` | Build secrets (multi-line, `id=value`), exposed to `RUN --mount=type=secret,id=<id>` | `""` |
| `secretFiles` | Build secrets sourced from files (multi-line, `id=path`) | `""` |
| `cacheFrom` / `cacheTo` | Cache backend for `docker/build-push-action` (multi-line). Defaults to the GitHub Actions cache scoped to `dockerImage` | `""` (→ `type=gha`) |
| `push` | Push the image to registry (set to `false` for validation-only) | `true` |
| `qemu` | Set up QEMU for cross-platform emulation (disable for native builds) | `true` |
| `labels` | OCI labels for the image (multi-line, `key=value`) | `""` |
| `annotations` | OCI annotations for the image (multi-line, `key=value`) | `""` |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |

### merge-manifest

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name | *(required)* |
| `dockerImageTag` | Final multi-arch tag (comma-separated for multiple) | *(required)* |
| `dockerRegistry` | Docker registry | `docker.io` |
| `digestsDir` | Directory containing `*.digest` files (one per platform) | *(required)* |
| `annotations` | OCI annotations for the merged manifest (multi-line, `key=value`) | `""` |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |

**Outputs:** `digest` (digest of the merged manifest, first tag) — feed this into `attest-image`/`cosign-image` to sign or attest the multi-arch manifest itself.

### mirror-image

| Name | Description | Default |
|---|---|---|
| `sourceImage` | Source image name | *(required)* |
| `sourceTag` | Source tag(s) (comma-separated for multiple) | *(required)* |
| `sourceRegistry` | Source registry | `docker.io` |
| `targetImage` | Target image name (defaults to `sourceImage`) | `""` |
| `targetTag` | Target tag(s) (defaults to `sourceTag`) | `""` |
| `targetRegistry` | Target registry | `docker.io` |
| `SOURCE_USERNAME` | Source registry username | *(required)* |
| `SOURCE_PASSWORD` | Source registry password | *(required)* |
| `TARGET_USERNAME` | Target registry username | *(required)* |
| `TARGET_PASSWORD` | Target registry password | *(required)* |

**Outputs:** `digest` (first tag digest), `tagsJson` (JSON array of mirrored tags)

### sign-image (deprecated)

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name | *(required)* |
| `dockerImageTag` | Image tags as JSON array (e.g. `["latest","v1.0"]`) | `""` |
| `dockerRegistry` | Docker registry | `docker.io` |
| `signImage` | Enable/disable DCT signing | `true` |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |
| `DOCKER_PRIVATE_KEY_ID` | Private key ID for DCT | `""` |
| `DOCKER_PRIVATE_KEY` | Private key for DCT | `""` |
| `DOCKER_PRIVATE_KEY_PASSPHRASE` | Passphrase for the private key | `""` |

### cosign-image

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name | *(required)* |
| `dockerImageTag` | Image tags as JSON array | `""` |
| `dockerImageDigest` | Image digest to cosign | `""` |
| `dockerRegistry` | Docker registry | `docker.io` |
| `cosignImage` | Enable Cosign signing | `false` |
| `cosignOidcImage` | Enable Cosign with GitHub OIDC token | `false` |
| `verifyImage` | Run `cosign verify` against each signed tag right after signing. Opt-in: unsafe to enable if the same digest is ever signed by both `cosignImage` and `cosignOidcImage` across runs on a reused tag, since Sigstore's transparency log then carries mixed signature types and verification can false-positive-fail | `false` |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |
| `COSIGN_PRIVATE_KEY` | Cosign signing private key | `""` |
| `COSIGN_PASSWORD` | Cosign private key passphrase | `""` |
| `COSIGN_KEY_REF` | KMS-backed key reference (`awskms://`, `gcpkms://`, `azurekms://`, `hashivault://`); takes precedence over `COSIGN_PRIVATE_KEY` | `""` |

### attest-image

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name | *(required)* |
| `dockerImageDigest` | Digest of the image to attest | `""` |
| `dockerRegistry` | Docker registry | `docker.io` |
| `attestImage` | Enable build-provenance attestation | `false` |
| `attestSBOM` | Generate an SBOM for the image and attest it with GitHub (needs `attestations`/`id-token` write permissions) | `false` |
| `attestImageRegistry` | Registry for attestation | `docker.io` |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |

### notation-image

Signs/verifies via the [Notary Project](https://notaryproject.dev)'s `notation` CLI, a separate signing mechanism alongside `cosign-image`. Two signing modes; one signature per digest covers every tag pointing at it, so there's no per-tag loop.

- **Plugin mode:** sign through a KMS-backed plugin (Azure Key Vault, AWS Signer, HashiCorp Vault, ...) with `pluginName`, `pluginUrl`, `pluginChecksum` and `keyId`.
- **Local-key mode (free, no service):** set `signingKey` and `signingCert` (PEM, from secrets). The key is imported into the runner's Notation config for the job and deleted afterwards. The key must be unencrypted and the certificate needs a code-signing EKU and digital signature key usage.

```yaml
- uses: exo-actions/buildDockerImage-action/notation-image@v1
  with:
    dockerImage: exoplatform/example
    dockerImageDigest: ${{ needs.build.outputs.digest }}
    notationImage: "true"
    signingKey: ${{ secrets.NOTATION_SIGNING_KEY }}
    signingCert: ${{ secrets.NOTATION_SIGNING_CERT }}
    DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

Generate a key and certificate once with `openssl req -x509 -newkey rsa:3072 -nodes -days 365 -keyout key.pem -out cert.pem -subj "/CN=myorg-signing/O=myorg" -addext basicConstraints=critical,CA:FALSE -addext keyUsage=critical,digitalSignature -addext extendedKeyUsage=codeSigning`, store both as secrets (ideally on a protected environment), and give consumers `cert.pem` for their trust store. Rotate before the certificate expires.

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name | *(required)* |
| `dockerImageDigest` | Image digest to sign/verify | *(required)* |
| `dockerRegistry` | Docker registry | `docker.io` |
| `notationImage` | Enable Notation signing | `false` |
| `notationVersion` | Version of the Notation CLI to install | `1.3.1` |
| `pluginName` | Signing plugin name (e.g. `azure-kv`, `aws-signer`, `notation-hashicorp-vault`) | `""` |
| `pluginUrl` / `pluginChecksum` | Download URL and SHA256 of the signing plugin | `""` |
| `keyId` | KMS key identifier/ARN/URI for the signing key pair | `""` |
| `signingKey` / `signingCert` | Local-key mode: PEM private key and certificate (chain). Replaces the plugin inputs | `""` |
| `signingKeyName` | Local-key mode: name the key is registered under | `ci-signing-key` |
| `signatureFormat` | Signature envelope format: `jws` or `cose` | `cose` |
| `pluginConfig` | Plugin-defined config (multi-line, `key=value`) | `""` |
| `timestampUrl` / `timestampRootCert` | RFC 3161 timestamping authority URL and root cert | `""` |
| `verifyImage` | Run `notation verify` against the signed digest right after signing (requires `trustPolicy`/`trustStore`) | `false` |
| `trustPolicy` | File path to a Notation `trustpolicy.json` | `""` |
| `trustStore` | Directory path to a Notation trust store | `""` |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |
