# buildDockerImage-action
Action for Docker image build, sign, attest, and Cosign.

## Basic Usage example:

```yaml
name: Create and publish a Docker image v1

on:
  push:
    tags:
      - '*'
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
            echo "This is a tag push (${GITHUB_REF#refs/tags/})"
            echo "Building docker tag: ${GITHUB_REF#refs/tags/}"
            echo "buildTags=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT
          elif [[ $GITHUB_REF == refs/heads/* ]]; then
            echo "This is a branch push (${GITHUB_REF#refs/heads/})"
            echo "Building docker tags: ${{ env.BRANCH_BUILD_TAGS }}"
            echo "buildTags=${{ env.BRANCH_BUILD_TAGS }}" >> $GITHUB_OUTPUT
          else
            echo "Unknown push type"
            exit 1

  build-dockerhub-image:
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    name: "Build Docker Images and push them to DockerHub Registry"
    runs-on: ubuntu-latest
    outputs:
      tags: ${{ steps.build-docker-image.outputs.tags }}
      digest: ${{ steps.build-docker-image.outputs.digest }}
    timeout-minutes: 120
    needs: parse-docker-build-env
    steps:
      - name: build docker image
        uses: exo-actions/buildDockerImage-action/build-and-push-image@v1 
        id: build-docker-image
        with:
          dockerImage: "exoplatform/exo-community"
          dockerImageTag: ${{ needs.parse-docker-build-env.outputs.buildTags }} 
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

  sign-dockerhub-image:
    permissions:
      contents: read
      packages: write
      id-token: write
    strategy: 
      fail-fast: false
      max-parallel: 1
      matrix: 
        tags: ${{ fromJson(needs.build-dockerhub-image.outputs.tags) }}        
    name: "Sign Docker Images"
    runs-on: ubuntu-latest
    timeout-minutes: 120
    needs: build-dockerhub-image
    steps:
      - name: sign docker image
        uses: exo-actions/buildDockerImage-action/sign-image@v1 
        id: sign-docker-image
        with:
          dockerImage: "exoplatform/exo-community"          
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
          DOCKER_PRIVATE_KEY_ID: ${{ secrets.DOCKER_PRIVATE_KEY_ID }}
          DOCKER_PRIVATE_KEY: ${{ secrets.DOCKER_PRIVATE_KEY }}
          DOCKER_PRIVATE_KEY_PASSPHRASE: ${{ secrets.DOCKER_PRIVATE_KEY_PASSPHRASE }}

  attest-dockerhub-image:
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    name: "Attest Docker Images"
    runs-on: ubuntu-latest
    timeout-minutes: 120
    needs: build-dockerhub-image
    steps:
      - name: attest docker image
        uses: exo-actions/buildDockerImage-action/attest-image@v1 
        id: attest-docker-image
        with:
          dockerImage: "exoplatform/exo-community"
          dockerImageDigest: ${{ needs.build-dockerhub-image.outputs.digest }} 
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
          attestImage: "true"

  cosign-dockerhub-image:
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    name: "Cosign Docker Images"
    runs-on: ubuntu-latest
    timeout-minutes: 120
    needs: build-dockerhub-image
    steps:
      - name: cosign docker image
        uses: exo-actions/buildDockerImage-action/cosign-image@v1 
        id: cosign-docker-image
        with:
          dockerImage: "exoplatform/exo-community"
          dockerImageTag: ${{ needs.build-dockerhub-image.outputs.tags }} 
          dockerImageDigest: ${{ needs.build-dockerhub-image.outputs.digest }} 
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
          cosignImage: "true"
          cosignOidcImage: "true"
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

## Inputs Reference

### build-dockerhub-image

| Name               | Description                                                        | Default Value       |
|-------------------|--------------------------------------------------------------------|-------------------|
| dockerImage        | Targeted Docker image (ex: `exoplatform/exo-community`)           | ``                 |
| dockerImageTag     | Docker Image tag(s), comma-separated for multiple                  | `latest`           |
| dockerFileContext  | Dockerfile Context (path)                                         | `.`                |
| dockerRegistry     | Docker registry (Default)                                         | `docker.io`        |
| DOCKER_USERNAME    | Username allowed to push to Docker registry                        | ``                 |
| DOCKER_PASSWORD    | Password for the above username                                     | ``                 |
| generateSBOM       | Flag to generate SBOM for the image                                | `true`             |
| generateProvenance | Flag to generate provenance for the image                           | `true`             |

### sign-dockerhub-image

| Name                     | Description                                               | Default Value |
|---------------------------|-----------------------------------------------------------|---------------|
| dockerImage              | Targeted Docker image                                      | ``            |
| dockerImageTag           | Targeted Docker image tag(s)                               | ``            |
| dockerRegistry           | Docker registry (Default)                                  | `docker.io`   |
| signImage                | Flag to enable/disable DCT signing (Deprecated)           | `true`        |
| DOCKER_USERNAME          | Username for Docker registry                                | ``            |
| DOCKER_PASSWORD          | Password for Docker registry                                | ``            |
| DOCKER_PRIVATE_KEY_ID    | Private key ID used for signature                           | ``            |
| DOCKER_PRIVATE_KEY       | Private key used for signing                                | ``            |
| DOCKER_PRIVATE_KEY_PASSPHRASE | Password of private key                                  | ``            |

### attest-dockerhub-image

| Name                  | Description                                                | Default Value |
|-----------------------|------------------------------------------------------------|---------------|
| dockerImage           | Targeted Docker image                                      | ``            |
| dockerImageDigest     | Digest of the Docker image to attest                        | ``            |
| dockerRegistry        | Docker registry (Default)                                  | `docker.io`   |
| DOCKER_USERNAME       | Username for Docker registry                                | ``            |
| DOCKER_PASSWORD       | Password for Docker registry                                | ``            |
| attestImageRegistry   | Registry for GitHub attestations                            | `docker.io`   |
| attestImage           | Enable GitHub attestation                                    | `false`       |

### cosign-dockerhub-image

| Name                  | Description                                                | Default Value |
|-----------------------|------------------------------------------------------------|---------------|
| dockerImage           | Targeted Docker image                                      | ``            |
| dockerImageTag        | Docker Image tag(s)                                        | ``            |
| dockerImageDigest     | Digest of the Docker image to sign                          | ``            |
| dockerRegistry        | Docker registry (Default)                                   | `docker.io`   |
| cosignImage           | Enable Cosign signing                                       | `false`       |
| cosignOidcImage       | Enable Cosign OIDC signing (requires cosignImage=true)      | `false`       |
| DOCKER_USERNAME       | Username for Docker registry                                | ``            |
| DOCKER_PASSWORD       | Password for Docker registry                                | ``            |
| COSIGN_PRIVATE_KEY    | Cosign private key                                          | ``            |
| COSIGN_PASSWORD       | Cosign private key passphrase                               | ``            |
