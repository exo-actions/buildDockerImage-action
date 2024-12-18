# buildDockerImage-action
Action for Docker image build and sign


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
          fi
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
    name: "sign-docker-image"
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
          DOCKER_PRIVATE_KEY_ID: ${{secrets.DOCKER_PRIVATE_KEY_ID}}
          DOCKER_PRIVATE_KEY: ${{secrets.DOCKER_PRIVATE_KEY}}
          DOCKER_PRIVATE_KEY_PASSPHRASE: ${{secrets.DOCKER_PRIVATE_KEY_PASSPHRASE}}

  attest-dockerhub-image:
    permissions:
        contents: read
        packages: write
        id-token: write
        attestations: write
    name: "attest-docker-image"
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
    name: "cosign-docker-image"
    runs-on: ubuntu-latest
    timeout-minutes: 120
    needs: build-dockerhub-image
    steps:
      - name: attest docker image
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
          COSIGN_PRIVATE_KEY: ${{secrets.COSIGN_PRIVATE_KEY}}
          COSIGN_PASSWORD: ${{secrets.COSIGN_PASSWORD}}
          
    
```

## Inputs

### build-dockerhub-image

| Name                 | Description                                                                                          | Default value                    |
|----------------------|------------------------------------------------------------------------------------------------------|----------------------------------|
| dockerImage          | targeted docker image (exoplatform/exo-community,...)                                                | ``      (empty)                  |
| dockerImageTag       | Docker Image tag (comma separated for multiple)                                                      | `latest`                         | 
| dockerFileContext    | Dockerfile Context (Dockerfile location)                                                             | `.`                              |
| dockerRegistry       | Docker registry (Default )                                                                           | `docker.io`                      |
| DOCKER_USERNAME      | Username allowed to access and push on docker registry                                               | ``                               |
| DOCKER_PASSWORD      | Password for the previous username                                                                   | ``                               |

### sign-dockerhub-image

| Name                 | Description                                                                                          | Default value                    |
|----------------------|------------------------------------------------------------------------------------------------------|----------------------------------|
| dockerImage          | targeted docker image (exoplatform/exo-community,...)                                                | ``      (empty)                  |
| matrix.tags          | targeted docker image versions (["latest","v1.0"])                                                   | ``      (empty)                  |
| dockerRegistry       | Docker registry (Default )                                                                           | `docker.io`                      |
| signImage            | flag to enable/disable signature                                                                     | `true`                           |
| DOCKER_USERNAME      | Username allowed to access and push on docker registry                                               | ``                               |
| DOCKER_PASSWORD      | Password for the previous username                                                                   | ``                               |
| DOCKER_PRIVATE_KEY_ID| Id of the used private key for signature                                                             | ``                               |
| DOCKER_PRIVATE_KEY   | private key used for signature                                                                       | ``                               |
| DOCKER_PRIVATE_KEY_PASSPHRASE   | Password of the used private key for signature                                            | ``                               |

### cosign-dockerhub-image

| Name                 | Description                                                                                          | Default value                    |
|----------------------|------------------------------------------------------------------------------------------------------|----------------------------------|
| dockerImage          | targeted docker image (exoplatform/exo-community,...)                                                | ``      (empty)                  |
| matrix.tags          | targeted docker image versions (["latest","v1.0"])                                                   | ``      (empty)                  |
| dockerRegistry       | Docker registry (Default )                                                                           | `docker.io`                      |
| signImage            | flag to enable/disable signature                                                                     | `true`                           |
| DOCKER_USERNAME      | Username allowed to access and push on docker registry                                               | ``                               |
| DOCKER_PASSWORD      | Password for the previous username                                                                   | ``                               |
| DOCKER_PRIVATE_KEY_ID| Id of the used private key for signature                                                             | ``                               |
| DOCKER_PRIVATE_KEY   | private key used for signature                                                                       | ``                               |
| DOCKER_PRIVATE_KEY_PASSPHRASE   | Password of the used private key for signature                                            | ``                               |

### attest-dockerhub-image

| Name                 | Description                                                                                          | Default value                    |
|----------------------|------------------------------------------------------------------------------------------------------|----------------------------------|
| dockerImage          | targeted docker image (exoplatform/exo-community,...)                                                | ``      (empty)                  |
| dockerImageTag       | Docker Image tags (matrix)                                                                           | `"[latest]"`                     |
| dockerImageDigest    | Digest of the docker image to attest (example: sha256:15695b16b1dc0bec528e2111c678d21194651613caccc8b452253c530b204459)|`` (empty)      |
| dockerRegistry       | Docker registry (Default )                                                                           | `docker.io`                      |
| DOCKER_USERNAME      | Username allowed to access and push on docker registry                                               | ``                               |
| DOCKER_PASSWORD      | Password for the previous username                                                                   | ``                               |
| cosignImage          | Enable Docker Image Signing (Cosign)                                                                 | `false`                          |
| cosignOidcImage      | Enable Docker Image Signing (Cosign) with Github OIDC Token                                          | `false`                          |
| COSIGN_PRIVATE_KEY   | Cosign Signing Private Key                                                                           | ``(empty)                        |
| COSIGN_PASSWORD      | Cosign Signing Private Key Passphrase                                                                | ``(empty)                        |
