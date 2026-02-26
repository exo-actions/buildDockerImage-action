

# Enhanced README — `buildDockerImage-action`

```markdown
# 🐳 buildDockerImage-action

A comprehensive GitHub Action for building, publishing, signing, and attesting Docker container images with full supply-chain security support.

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Build%20Docker%20Image-blue?logo=github)](https://github.com/marketplace/actions/build-publish-sign-attest-docker-image)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🏗️ **Build & Push** | Multi-platform builds with Docker Buildx, layer caching, and full OCI metadata |
| 📦 **SBOM** | Automatic Software Bill of Materials generation |
| 🛡️ **SLSA Provenance** | Build provenance attestation for supply-chain integrity |
| 🔏 **Cosign Signing** | Sigstore-based image signing (key-pair or keyless OIDC) |
| 📝 **GitHub Attestation** | Native GitHub build-provenance and SBOM attestation |
| ✍️ **DCT Signing** | ⚠️ Deprecated — Docker Content Trust (Notary v1) |
| 🚀 **Registry Caching** | Registry-backed layer caching for faster builds |
| 📊 **Job Summary** | Rich Markdown summaries with verification commands |

---

## 📋 Table of Contents

- [Quick Start](#-quick-start)
- [Architecture](#-architecture)
- [Usage Examples](#-usage-examples)
  - [Simple Build & Push](#1-simple-build--push)
  - [Build with Cosign OIDC Signing](#2-build-with-cosign-oidc-keyless-signing-recommended)
  - [Build with Cosign Key-pair Signing](#3-build-with-cosign-key-pair-signing)
  - [Build with GitHub Attestation](#4-build-with-github-attestation)
  - [Full Pipeline (All Features)](#5-full-pipeline-all-features)
  - [Multi-platform Build](#6-multi-platform-build)
  - [Using Individual Sub-actions](#7-using-individual-sub-actions)
  - [Migrating from DCT to Cosign](#8-migrating-from-dct-to-cosign)
- [Inputs Reference](#-inputs-reference)
  - [Main Action](#main-action-buildDockerImage-action)
  - [Build Sub-action](#build-sub-action-build-and-push-image)
  - [Cosign Sub-action](#cosign-sub-action-cosign-image)
  - [Attest Sub-action](#attest-sub-action-attest-image)
  - [Sign Sub-action (Deprecated)](#sign-sub-action-deprecated-sign-image)
- [Outputs Reference](#-outputs-reference)
- [Permissions Reference](#-permissions-reference)
- [Verification Guide](#-verification-guide)
- [Migration Guide](#-migration-guide)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)

---

## 🚀 Quick Start

```yaml
name: Build and publish Docker image

on:
  push:
    tags: ['v*']
    branches: ['main']

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write        # Required for Cosign OIDC & attestation
      attestations: write    # Required for GitHub attestation
    steps:
      - uses: exo-actions/buildDockerImage-action@v2
        with:
          dockerImage: myorg/myapp
          dockerImageTag: latest,v1.0.0
          cosignOidcImage: 'true'
          attestImage: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

---

## 🏛️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    buildDockerImage-action                       │
│                      (Main Orchestrator)                         │
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐ │
│  │  Build &  │──▶│  Cosign  │──▶│  Attest  │──▶│ DCT Sign     │ │
│  │   Push    │   │ Signing  │   │          │   │ (deprecated) │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────────┘ │
│       │              │              │               │            │
│       ▼              ▼              ▼               ▼            │
│   [digest]     [signatures]   [attestations]   [trust data]     │
│   [tags]       [transparency]  [bundle IDs]    [notary]         │
│   [metadata]   [rekor log]                                      │
└─────────────────────────────────────────────────────────────────┘
```

**Sub-actions can also be used independently** — see
[Using Individual Sub-actions](#7-using-individual-sub-actions).

| Sub-action | Path | Purpose |
|---|---|---|
| `build-and-push-image` | `exo-actions/buildDockerImage-action/build-and-push-image@v2` | Build, tag, and push |
| `cosign-image` | `exo-actions/buildDockerImage-action/cosign-image@v2` | Cosign signing (key-pair or OIDC) |
| `attest-image` | `exo-actions/buildDockerImage-action/attest-image@v2` | GitHub attestation (provenance & SBOM) |
| `sign-image` | `exo-actions/buildDockerImage-action/sign-image@v2` | ⚠️ DCT signing (deprecated) |

---

## 📖 Usage Examples

### 1. Simple Build & Push

The minimal configuration — build and push with SBOM and provenance enabled by default.

```yaml
name: Build Docker Image

on:
  push:
    branches: ['main']

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Build and push
        uses: exo-actions/buildDockerImage-action@v2
        with:
          dockerImage: exoplatform/exo-community
          dockerImageTag: latest
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### 2. Build with Cosign OIDC Keyless Signing (Recommended)

No signing keys needed — uses GitHub's OIDC token for identity-based signing via Sigstore.

```yaml
name: Build, Push & Sign (Keyless)

on:
  push:
    tags: ['v*']
    branches: ['main']

env:
  IMAGE_NAME: exoplatform/exo-community

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write    # Required for OIDC keyless signing
    steps:
      - name: Determine tags
        id: tags
        run: |
          if [[ "$GITHUB_REF" == refs/tags/* ]]; then
            echo "tags=${GITHUB_REF#refs/tags/}" >> "$GITHUB_OUTPUT"
          else
            echo "tags=latest" >> "$GITHUB_OUTPUT"
          fi

      - name: Build, push & sign
        uses: exo-actions/buildDockerImage-action@v2
        with:
          dockerImage: ${{ env.IMAGE_NAME }}
          dockerImageTag: ${{ steps.tags.outputs.tags }}
          cosignOidcImage: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### 3. Build with Cosign Key-pair Signing

Use your own Cosign key-pair for environments where keyless signing isn't available.

```yaml
name: Build, Push & Sign (Key-pair)

on:
  push:
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Build, push & sign
        uses: exo-actions/buildDockerImage-action@v2
        with:
          dockerImage: exoplatform/exo-community
          dockerImageTag: ${{ github.ref_name }}
          cosignImage: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

### 4. Build with GitHub Attestation

Generate and push SLSA build-provenance and SBOM attestations using GitHub's native attestation infrastructure.

```yaml
name: Build, Push & Attest

on:
  push:
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write       # Required for attestation
      attestations: write   # Required for attestation
    steps:
      - name: Build, push & attest
        uses: exo-actions/buildDockerImage-action@v2
        with:
          dockerImage: exoplatform/exo-community
          dockerImageTag: ${{ github.ref_name }}
          attestImage: 'true'
          attestSBOM: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### 5. Full Pipeline (All Features)

Complete example with dynamic tagging, Cosign OIDC signing, GitHub attestation, caching, and multi-platform builds.

```yaml
name: Docker Image Pipeline

on:
  push:
    tags: ['v*']
    branches: ['main', 'develop']
  pull_request:
    branches: ['main']

env:
  IMAGE_NAME: exoplatform/exo-community
  REGISTRY: docker.io

jobs:
  # ─── Determine Build Tags ─────────────────────────────────────
  prepare:
    name: Prepare Build Environment
    runs-on: ubuntu-latest
    outputs:
      tags: ${{ steps.meta.outputs.tags }}
      push: ${{ steps.meta.outputs.push }}
    steps:
      - name: Compute tags and push decision
        id: meta
        run: |
          set -euo pipefail

          PUSH="true"
          TAGS=""

          case "$GITHUB_REF" in
            refs/tags/v*)
              VERSION="${GITHUB_REF#refs/tags/v}"
              MAJOR=$(echo "$VERSION" | cut -d. -f1)
              MINOR=$(echo "$VERSION" | cut -d. -f1-2)
              TAGS="${VERSION},${MINOR},${MAJOR},latest"
              echo "📋 Tag push: ${TAGS}"
              ;;
            refs/heads/main)
              TAGS="latest,main-${GITHUB_SHA:0:7}"
              echo "📋 Main branch push: ${TAGS}"
              ;;
            refs/heads/develop)
              TAGS="develop,develop-${GITHUB_SHA:0:7}"
              echo "📋 Develop branch push: ${TAGS}"
              ;;
            refs/pull/*)
              TAGS="pr-${{ github.event.number }}"
              PUSH="false"
              echo "📋 PR build (no push): ${TAGS}"
              ;;
            *)
              echo "::error::Unrecognized ref: ${GITHUB_REF}"
              exit 1
              ;;
          esac

          echo "tags=${TAGS}" >> "$GITHUB_OUTPUT"
          echo "push=${PUSH}" >> "$GITHUB_OUTPUT"

  # ─── Build, Sign & Attest ─────────────────────────────────────
  docker:
    name: Build, Sign & Attest
    runs-on: ubuntu-latest
    needs: prepare
    if: ${{ needs.prepare.outputs.push == 'true' }}
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    outputs:
      tags: ${{ steps.pipeline.outputs.tags }}
      digest: ${{ steps.pipeline.outputs.digest }}
      fullImageRef: ${{ steps.pipeline.outputs.fullImageRef }}
    timeout-minutes: 120
    steps:
      - name: Run Docker pipeline
        id: pipeline
        uses: exo-actions/buildDockerImage-action@v2
        with:
          # Image
          dockerImage: ${{ env.IMAGE_NAME }}
          dockerImageTag: ${{ needs.prepare.outputs.tags }}
          dockerRegistry: ${{ env.REGISTRY }}

          # Build
          platforms: linux/amd64,linux/arm64
          generateSBOM: 'true'
          generateProvenance: 'true'
          cacheEnabled: 'true'

          # Registry auth
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

          # Cosign (keyless OIDC)
          cosignOidcImage: 'true'
          cosignAnnotations: >-
            repo=${{ github.repository }},
            sha=${{ github.sha }},
            ref=${{ github.ref }},
            run=${{ github.run_id }}

          # GitHub attestation
          attestImage: 'true'
          attestSBOM: 'true'

          # Verification
          verifySignatures: 'true'

  # ─── Post-pipeline Smoke Test ──────────────────────────────────
  verify:
    name: Verify Published Image
    runs-on: ubuntu-latest
    needs: docker
    if: ${{ needs.docker.result == 'success' }}
    steps:
      - name: Pull and verify
        run: |
          set -euo pipefail
          echo "📥 Pulling ${{ needs.docker.outputs.fullImageRef }}..."
          docker pull "${{ needs.docker.outputs.fullImageRef }}"
          docker image inspect "${{ needs.docker.outputs.fullImageRef }}"
          echo "✅ Image verified"
```

### 6. Multi-platform Build

Build for multiple architectures with QEMU emulation.

```yaml
name: Multi-platform Build

on:
  push:
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    steps:
      - uses: exo-actions/buildDockerImage-action@v2
        with:
          dockerImage: exoplatform/exo-community
          dockerImageTag: ${{ github.ref_name }}
          platforms: linux/amd64,linux/arm64,linux/arm/v7
          cosignOidcImage: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### 7. Using Individual Sub-actions

Use sub-actions independently for maximum flexibility.

```yaml
name: Custom Pipeline with Sub-actions

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      tags: ${{ steps.build.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - name: Build & push
        id: build
        uses: exo-actions/buildDockerImage-action/build-and-push-image@v2
        with:
          dockerImage: exoplatform/exo-community
          dockerImageTag: ${{ github.ref_name }},latest
          cacheEnabled: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

  sign:
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Cosign (keyless)
        uses: exo-actions/buildDockerImage-action/cosign-image@v2
        with:
          dockerImage: exoplatform/exo-community
          dockerImageTag: ${{ needs.build.outputs.tags }}
          dockerImageDigest: ${{ needs.build.outputs.digest }}
          cosignOidcImage: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

  attest:
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: read
      id-token: write
      attestations: write
    steps:
      - name: Attest
        uses: exo-actions/buildDockerImage-action/attest-image@v2
        with:
          dockerImage: exoplatform/exo-community
          dockerImageDigest: ${{ needs.build.outputs.digest }}
          attestImage: 'true'
          attestSBOM: 'true'
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### 8. Migrating from DCT to Cosign

```yaml
# ❌ BEFORE (DCT — deprecated)
- uses: exo-actions/buildDockerImage-action@v1
  with:
    dockerImage: exoplatform/exo-community
    dockerImageTag: latest
    signImage: 'true'
    DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
    DOCKER_PRIVATE_KEY_ID: ${{ secrets.DCT_KEY_ID }}
    DOCKER_PRIVATE_KEY: ${{ secrets.DCT_KEY }}
    DOCKER_PRIVATE_KEY_PASSPHRASE: ${{ secrets.DCT_PASSPHRASE }}

# ✅ AFTER (Cosign keyless — recommended, no secrets needed)
- uses: exo-actions/buildDockerImage-action@v2
  with:
    dockerImage: exoplatform/exo-community
    dockerImageTag: latest
    cosignOidcImage: 'true'
    DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

# ✅ AFTER (Cosign key-pair — if keyless isn't available)
- uses: exo-actions/buildDockerImage-action@v2
  with:
    dockerImage: exoplatform/exo-community
    dockerImageTag: latest
    cosignImage: 'true'
    DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

---

## 📥 Inputs Reference

### Main Action (`buildDockerImage-action`)

#### Image Configuration

| Input | Description | Required | Default |
|---|---|---|---|
| `dockerImage` | Docker image name (e.g. `exoplatform/exo-community`) | ✅ | — |
| `dockerImageTag` | Comma-separated tags (e.g. `latest,v1.0.0`) | ❌ | `latest` |
| `dockerRegistry` | Container registry host | ❌ | `docker.io` |

#### Build Configuration

| Input | Description | Required | Default |
|---|---|---|---|
| `dockerFileContext` | Build context path | ❌ | `.` |
| `dockerFilePath` | Path to Dockerfile (relative to context) | ❌ | — |
| `target` | Multi-stage build target stage | ❌ | — |
| `buildArgs` | Newline-separated build-time variables | ❌ | — |
| `buildSecrets` | Newline-separated build secrets | ❌ | — |
| `platforms` | Comma-separated target platforms | ❌ | `linux/amd64` |

#### Supply-chain Security

| Input | Description | Required | Default |
|---|---|---|---|
| `generateSBOM` | Attach SBOM during build | ❌ | `true` |
| `generateProvenance` | Attach SLSA provenance during build | ❌ | `true` |

#### Caching

| Input | Description | Required | Default |
|---|---|---|---|
| `cacheEnabled` | Enable Docker layer caching | ❌ | `true` |
| `cacheFrom` | Custom cache source | ❌ | Auto (registry) |
| `cacheTo` | Custom cache destination | ❌ | Auto (registry) |

#### Registry Authentication

| Input | Description | Required | Default |
|---|---|---|---|
| `DOCKER_USERNAME` | Registry username | ✅ | — |
| `DOCKER_PASSWORD` | Registry password / token | ✅ | — |

#### Cosign Signing

| Input | Description | Required | Default |
|---|---|---|---|
| `cosignImage` | Enable Cosign key-pair signing | ❌ | `false` |
| `cosignOidcImage` | Enable Cosign keyless OIDC signing | ❌ | `false` |
| `COSIGN_PRIVATE_KEY` | Cosign private key (PEM or KMS URI) | When `cosignImage: true` | — |
| `COSIGN_PASSWORD` | Cosign private key passphrase | ❌ | — |
| `COSIGN_PUBLIC_KEY` | Cosign public key (for verification) | ❌ | — |
| `cosignAnnotations` | Comma-separated `key=value` annotations | ❌ | — |
| `cosignVersion` | Cosign version to install | ❌ | `v2.4.1` |

#### GitHub Attestation

| Input | Description | Required | Default |
|---|---|---|---|
| `attestImage` | Enable GitHub attestation | ❌ | `false` |
| `attestImageRegistry` | Registry for attestation subject-name | ❌ | Same as `dockerRegistry` |
| `attestSBOM` | Enable SBOM attestation | ❌ | `false` |
| `sbomFilePath` | Path to pre-generated SBOM file | ❌ | Auto-generated |
| `githubToken` | GitHub token for attestation | ❌ | `${{ github.token }}` |

#### DCT Signing (⚠️ Deprecated)

| Input | Description | Required | Default |
|---|---|---|---|
| `signImage` | ⚠️ Enable DCT signing | ❌ | `false` |
| `DOCKER_PRIVATE_KEY_ID` | ⚠️ DCT key ID | When `signImage: true` | — |
| `DOCKER_PRIVATE_KEY` | ⚠️ DCT private key | When `signImage: true` | — |
| `DOCKER_PRIVATE_KEY_PASSPHRASE` | ⚠️ DCT key passphrase | ❌ | — |

#### Advanced

| Input | Description | Required | Default |
|---|---|---|---|
| `verifySignatures` | Verify all signatures/attestations after creation | ❌ | `true` |

---

### Build Sub-action (`build-and-push-image`)

```yaml
uses: exo-actions/buildDockerImage-action/build-and-push-image@v2
```

| Input | Description | Required | Default |
|---|---|---|---|
| `dockerImage` | Image name | ✅ | — |
| `dockerImageTag` | Comma-separated tags | ❌ | `latest` |
| `dockerRegistry` | Registry host | ❌ | `docker.io` |
| `dockerFileContext` | Build context path | ❌ | `.` |
| `dockerFilePath` | Dockerfile path | ❌ | — |
| `target` | Multi-stage target | ❌ | — |
| `buildArgs` | Build-time variables | ❌ | — |
| `buildSecrets` | Build secrets | ❌ | — |
| `platforms` | Target platforms | ❌ | `linux/amd64` |
| `generateSBOM` | Attach SBOM | ❌ | `true` |
| `generateProvenance` | Attach provenance | ❌ | `true` |
| `cacheEnabled` | Enable caching | ❌ | `true` |
| `cacheFrom` | Cache source | ❌ | Auto |
| `cacheTo` | Cache destination | ❌ | Auto |
| `pushImage` | Push after build | ❌ | `true` |
| `loadImage` | Load into local daemon | ❌ | `false` |
| `noCache` | Disable all caching | ❌ | `false` |
| `pullBase` | Always pull base images | ❌ | `false` |
| `extraLabels` | Additional OCI labels | ❌ | — |
| `DOCKER_USERNAME` | Registry username | ✅ | — |
| `DOCKER_PASSWORD` | Registry password | ✅ | — |

---

### Cosign Sub-action (`cosign-image`)

```yaml
uses: exo-actions/buildDockerImage-action/cosign-image@v2
```

| Input | Description | Required | Default |
|---|---|---|---|
| `dockerImage` | Image name | ✅ | — |
| `dockerImageTag` | JSON array of tags | ❌ | `["latest"]` |
| `dockerImageDigest` | Image digest (`sha256:…`) | ❌ | — |
| `dockerRegistry` | Registry host | ❌ | `docker.io` |
| `cosignImage` | Enable key-pair signing | ❌ | `false` |
| `cosignOidcImage` | Enable OIDC keyless signing | ❌ | `false` |
| `COSIGN_PRIVATE_KEY` | Private key | When `cosignImage: true` | — |
| `COSIGN_PASSWORD` | Key passphrase | ❌ | — |
| `COSIGN_PUBLIC_KEY` | Public key (verification) | ❌ | — |
| `cosignVersion` | Cosign version | ❌ | `v2.4.1` |
| `annotations` | Signature annotations | ❌ | — |
| `verifySigning` | Verify after signing | ❌ | `true` |
| `force` | Overwrite existing signatures | ❌ | `false` |
| `recursiveSigning` | Sign manifest list recursively | ❌ | `false` |
| `DOCKER_USERNAME` | Registry username | ✅ | — |
| `DOCKER_PASSWORD` | Registry password | ✅ | — |

---

### Attest Sub-action (`attest-image`)

```yaml
uses: exo-actions/buildDockerImage-action/attest-image@v2
```

| Input | Description | Required | Default |
|---|---|---|---|
| `dockerImage` | Image name | ✅ | — |
| `dockerImageDigest` | Image digest (`sha256:…`) | ✅ | — |
| `dockerRegistry` | Registry host | ❌ | `docker.io` |
| `attestImage` | Enable attestation | ❌ | `false` |
| `attestImageRegistry` | Attestation subject registry | ❌ | Same as `dockerRegistry` |
| `attestProvenance` | Enable provenance attestation | ❌ | `true` |
| `attestSBOM` | Enable SBOM attestation | ❌ | `false` |
| `sbomFilePath` | Pre-generated SBOM path | ❌ | Auto-generated |
| `pushProvenanceToRegistry` | Push provenance bundle | ❌ | `true` |
| `pushSBOMToRegistry` | Push SBOM bundle | ❌ | `true` |
| `verifyAttestation` | Verify after attesting | ❌ | `true` |
| `githubToken` | GitHub token | ❌ | `${{ github.token }}` |
| `DOCKER_USERNAME` | Registry username | ✅ | — |
| `DOCKER_PASSWORD` | Registry password | ✅ | — |

---

### Sign Sub-action (⚠️ Deprecated — `sign-image`)

> **⚠️ This sub-action uses Docker Content Trust (DCT / Notary v1) and is deprecated.**
> Migrate to the [Cosign sub-action](#cosign-sub-action-cosign-image) or use
> `cosignOidcImage: true` on the main action.

```yaml
uses: exo-actions/buildDockerImage-action/sign-image@v2
```

| Input | Description | Required | Default |
|---|---|---|---|
| `dockerImage` | Image name | ✅ | — |
| `dockerImageTag` | Single tag to sign | ✅ | — |
| `dockerRegistry` | Registry host | ❌ | `docker.io` |
| `signImage` | Enable DCT signing | ❌ | `true` |
| `DOCKER_USERNAME` | Registry username | ✅ | — |
| `DOCKER_PASSWORD` | Registry password | ✅ | — |
| `DOCKER_PRIVATE_KEY_ID` | DCT key ID | When `signImage: true` | — |
| `DOCKER_PRIVATE_KEY` | DCT private key | When `signImage: true` | — |
| `DOCKER_PRIVATE_KEY_PASSPHRASE` | DCT key passphrase | ❌ | — |
| `verifySignature` | Verify after signing | ❌ | `true` |
| `pullRetries` | Pull retry attempts | ❌ | `3` |
| `pullRetryDelay` | Initial retry delay (seconds) | ❌ | `5` |

---

## 📤 Outputs Reference

### Main Action

| Output | Description |
|---|---|
| `tags` | JSON array of pushed image tags (e.g. `["latest","v1.0.0"]`) |
| `digest` | Image digest (`sha256:…`) |
| `metadata` | Full build metadata JSON |
| `imageid` | Image ID |
| `fullImageRef` | Primary `registry/image:tag@digest` reference |
| `cosignSignedImages` | Space-separated Cosign-signed references |
| `dctSignedImageRef` | DCT-signed reference (deprecated) |
| `provenanceBundleId` | Provenance attestation bundle ID |
| `sbomBundleId` | SBOM attestation bundle ID |
| `pipelineStatus` | JSON: `{build, dct_signing, cosign_signing, attestation}` |

### Build Sub-action

| Output | Description |
|---|---|
| `tags` | JSON array of tags |
| `digest` | Image digest |
| `metadata` | Build metadata JSON |
| `imageid` | Image ID |
| `fullImageRef` | Primary image reference with digest |

### Cosign Sub-action

| Output | Description |
|---|---|
| `signedImages` | Space-separated signed references |
| `verificationStatus` | `success` or `skipped` |
| `cosignVersion` | Installed Cosign version |

### Attest Sub-action

| Output | Description |
|---|---|
| `provenanceBundleId` | Provenance bundle ID |
| `sbomBundleId` | SBOM bundle ID |
| `attestedImage` | Attested image reference |
| `verificationStatus` | `success` or `skipped` |

---

## 🔐 Permissions Reference

| Feature | `contents` | `packages` | `id-token` | `attestations` |
|---|---|---|---|---|
| Build & Push | `read` | `write` | — | — |
| Cosign Key-pair | `read` | `write` | — | — |
| Cosign OIDC (Keyless) | `read` | `write` | **`write`** | — |
| GitHub Attestation | `read` | `write` | **`write`** | **`write`** |
| DCT Signing (deprecated) | `read` | `write` | — | — |
| **All Features** | `read` | `write` | **`write`** | **`write`** |

**Example (all features):**

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write
```

---

## 🔍 Verification Guide

### Verify Cosign Signature (Keyless OIDC)

```bash
# Install cosign: https://docs.sigstore.dev/system_config/installation/

cosign verify \
  --certificate-identity-regexp 'https://github.com/exo-actions/.*' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  docker.io/exoplatform/exo-community:latest
```

### Verify Cosign Signature (Key-pair)

```bash
cosign verify \
  --key cosign.pub \
  docker.io/exoplatform/exo-community:latest
```

### Verify GitHub Attestation

```bash
# Requires GitHub CLI: https://cli.github.com/

gh attestation verify \
  oci://docker.io/exoplatform/exo-community:latest \
  --owner exoplatform
```

### Verify Docker Content Trust (Deprecated)

```bash
DOCKER_CONTENT_TRUST=1 docker pull docker.io/exoplatform/exo-community:latest
docker trust inspect --pretty docker.io/exoplatform/exo-community:latest
```

---

## 🔄 Migration Guide

### v1 → v2

| Change | v1 | v2 |
|---|---|---|
| **Action version** | `@v1` | `@v2` |
| **COSIGN_PASSWORD default** | `"false"` (bug) | `""` (empty string) |
| **DCT signing** | `signImage: true` | `signImage: true` (deprecated warning emitted) |
| **Cosign signing** | Basic | + annotations, verification, version pinning |
| **Attestation** | Provenance only | + SBOM attestation, verification |
| **Build** | Basic | + caching, multi-stage targets, build args/secrets |
| **Outputs** | `tags`, `digest` | + `metadata`, `imageid`, `fullImageRef`, `pipelineStatus`, bundle IDs |
| **Validation** | None | Comprehensive input validation with actionable errors |
| **Job summary** | None | Rich Markdown summary with verification commands |

### DCT → Cosign

See [Migration Example](#8-migrating-from-dct-to-cosign) above.

---

## 🔧 Troubleshooting

<details>
<summary><b>❌ "COSIGN_PRIVATE_KEY is required when cosignImage is enabled"</b></summary>

You enabled `cosignImage: true` but didn't provide the key. Either:
1. Provide `COSIGN_PRIVATE_KEY` and `COSIGN_PASSWORD` secrets, or
2. Switch to keyless: `cosignOidcImage: true` (no keys needed)

</details>

<details>
<summary><b>❌ "cosignImage and cosignOidcImage are mutually exclusive"</b></summary>

Only one Cosign signing mode can be active. Choose either:
- `cosignImage: true` — key-pair signing
- `cosignOidcImage: true` — keyless OIDC signing

</details>

<details>
<summary><b>❌ "Error: buildx failed … permission denied"</b></summary>

Ensure your workflow has the correct permissions:
```yaml
permissions:
  contents: read
  packages: write
```

</details>

<details>
<summary><b>❌ "Error: unable to get OIDC token"</b></summary>

Add `id-token: write` to your job permissions:
```yaml
permissions:
  id-token: write
```

This is required for Cosign OIDC keyless signing and GitHub attestation.

</details>

<details>
<summary><b>❌ "Error: Resource not accessible by integration" (attestation)</b></summary>

Add `attestations: write` to your job permissions:
```yaml
permissions:
  attestations: write
```

</details>

<details>
<summary><b>⚠️ "DCT signing only supports a single tag"</b></summary>

Docker Content Trust can only sign one tag at a time. The action will sign only the first tag. Migrate to Cosign for multi-tag signing support.

</details>

<details>
<summary><b>⚠️ Build is slow (no caching)</b></summary>

Ensure caching is enabled:
```yaml
with:
  cacheEnabled: 'true'
```

The action uses registry-backed caching by default. For GitHub Actions cache:
```yaml
with:
  cacheFrom: type=gha
  cacheTo: type=gha,mode=max
```

</details>

<details>
<summary><b>❌ "loadImage is only supported for single-platform builds"</b></summary>

Docker cannot load multi-platform images into the local daemon. Either:
1. Use a single platform: `platforms: linux/amd64`
2. Disable load: `loadImage: false` (default)

</details>

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-enhancement`
3. Make your changes and add tests
4. Submit a pull request

Please ensure all sub-actions maintain backward compatibility with existing workflows.

---

## 📄 License

A-GPL3 — see [LICENSE](LICENSE) for details.

---

<div align="center">

**Made with ❤️ by [eXo Platform](https://github.com/exoplatform)**

</div>
```

---

## Summary of README Enhancements

### 📐 Structure & Navigation
- **Table of contents** with anchor links to every section
- **Architecture diagram** showing the pipeline flow and sub-action relationships
- **Consistent section hierarchy** with emoji-prefixed headers

### 📖 Usage Examples (8 Complete Workflows)
| # | Example | What It Demonstrates |
|---|---|---|
| 1 | Simple Build & Push | Minimal configuration |
| 2 | Cosign OIDC Keyless | Recommended signing approach, no secrets needed |
| 3 | Cosign Key-pair | Traditional key-based signing |
| 4 | GitHub Attestation | Provenance + SBOM attestation |
| 5 | Full Pipeline | All features: dynamic tags, multi-platform, signing, attestation, caching, verification |
| 6 | Multi-platform | ARM64 + ARMv7 cross-compilation |
| 7 | Individual Sub-actions | Build/sign/attest as separate jobs |
| 8 | DCT → Cosign Migration | Before/after comparison |

### 📥 Inputs Reference
- **Complete tables** for every sub-action with Required/Default columns
- **Grouped by category** (image, build, security, caching, auth, advanced)
- **Deprecated inputs clearly marked** with ⚠️

### 📤 Outputs Reference
- **Separate tables** for main action and each sub-action
- **Descriptions** include example formats

### 🔐 Permissions Matrix
- **Feature × permission grid** showing exactly what each feature requires
- **Copy-paste permission block** for all features

### 🔍 Verification Guide
- **Four verification methods** with copy-paste commands (Cosign OIDC, Cosign key-pair, GitHub attestation, DCT)

### 🔄 Migration Guide
- **v1 → v2 changelog table** with every breaking/additive change
- **DCT → Cosign** cross-reference to code example

### 🔧 Troubleshooting
- **8 common errors** in collapsible `<details>` blocks
- Each with **cause explanation** and **fix with code snippet**