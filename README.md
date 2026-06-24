# buildDockerImage-action
Action for Docker image build, sign, attest, cosign, and multi-arch manifest merging.

## Sub-actions

| Action | Description |
|---|---|
| `build-and-push-image` | Builds and pushes Docker images with BuildKit, cache, SBOM, provenance |
| `merge-manifest` | Creates a multi-arch manifest list from single-platform images by digest |
| `sign-image` | (Deprecated) Signs images via Docker Content Trust |
| `attest-image` | Attests images using GitHub attest-build-provenance |
| `cosign-image` | Cosigns images via sigstore/cosign |

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

## Inputs

### build-and-push-image

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name (e.g. `exoplatform/exo-community`) | *(required)* |
| `dockerImageTag` | Docker image tag (comma-separated for multiple) | `latest` |
| `dockerFileContext` | Dockerfile context path | `.` |
| `dockerRegistry` | Docker registry | `docker.io` |
| `platforms` | Comma-separated target platforms | `linux/amd64` |
| `generateSBOM` | Generate SBOM for the image | `true` |
| `generateProvenance` | Generate provenance for the image | `true` |
| `buildArgs` | Build-time arguments (multi-line, `KEY=VALUE`) | `""` |
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
| `dockerImageTag` | Final multi-arch tag | *(required)* |
| `dockerRegistry` | Docker registry | `docker.io` |
| `digestsDir` | Directory containing `*.digest` files (one per platform) | *(required)* |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |

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
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |
| `COSIGN_PRIVATE_KEY` | Cosign signing private key | `""` |
| `COSIGN_PASSWORD` | Cosign private key passphrase | `""` |

### attest-image

| Name | Description | Default |
|---|---|---|
| `dockerImage` | Docker image name | *(required)* |
| `dockerImageDigest` | Digest of the image to attest | `""` |
| `dockerRegistry` | Docker registry | `docker.io` |
| `attestImage` | Enable attestation | `false` |
| `attestImageRegistry` | Registry for attestation | `docker.io` |
| `DOCKER_USERNAME` | Registry username | *(required)* |
| `DOCKER_PASSWORD` | Registry password | *(required)* |
