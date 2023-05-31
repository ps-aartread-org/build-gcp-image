# Build GCP Image
This is a GitHub Workflow for building docker images and storing them in GCP. 

Supports both Google Container Registry and Google Artifact Registry.

# Dependencies

## GitHub Enterprise

GitHub Enterprise is required in order to call an internal/private reusable workflow. Details found [here](https://docs.github.com/en/enterprise-cloud@latest/actions/creating-actions/sharing-actions-and-workflows-with-your-enterprise)

## Third Party Actions

This workflow requires the below third party actions be available.

- actions/checkout@v3
- google-github-actions/auth@v0
- docker/setup-buildx-action@v2.2.1
- docker/build-push-action@v3.2.0

If GitHub Enterprise has restrictions on the GitHub Actions that are allowed to be used. Then it may be neccessary to clone these actions into a repository that exists within the same GitHub Enterprise. From there you can then reference them in your workflow.

For Example. A repository named github-actions has been created under myorg and the setup-buildx-action code has been copied under .github/actions/setup-buildx-action-2.2.1.
```
    - name: Set up Docker Buildx
      uses: myorg/github-actions/.github/actions/setup-buildx-action-2.2.1@main
```

## Google Cloud Authentication

There are two methods for authentication into GCP that are supported.

1. Using a GCP Service Account Key file. 

- Create the GitHub Secret **GOOGLE_CREDENTIALS** - This contaisn the contents of the json key file generated from a service account.

2. Using Workload Identity Federation.

- Pass the input workloadIdentityProvider. This is the provider endpoint configured within GCP. Example: projects/1111111111/locations/global/workloadIdentityPools/my-pool/providers/my-provider
- Pass the input gcpServiceAccount. This is the service account tied to the Workload Identity Provider with access to GCP.

The service account for both of these method must have access to push to a GCP registry.

- If Artifact Registry: `roles/artifactregistry.writer` must be granted.
- If Container Registry: `roles/storage.legacyBucketWriter` must be granted.

## Versioning Workflow

This reusable workflow has a dependency on another reusable workflow for versioning. Before versioning will work access to the reusable workflow `version-workflows` must be granted.

In addition the create-version.yaml file must be updated with the location of the version-workflows reusable workflow.

# Usage

Follow the below steps to use this workflow.

1. Create a repository in GitHub Enterprise for this workflow.
2. Copy and Commit this code into the GitHub Repository. `Note:` If the versioning workflow has been setup, this will automatically create a new v0 release.
3. Share the workflow within the Enterprise. Details found [here.](https://docs.github.com/en/enterprise-cloud@latest/actions/creating-actions/sharing-actions-and-workflows-with-your-enterprise
)
4. Reference this workflow from other repositories using the below code as a reference. Be sure to update organization and repository.

Example:
Define the workflow call in a yaml file in your repository's .github/workflows/ directory

```
name: Build GCP Image
on:
  push:
    branches:
      - main
    paths-ignore:
      - '*.md'
  workflow_dispatch:

jobs:
  build-image:
    uses: myorg/repository/.github/workflows/build-gcp-image.yaml@v0
    secrets: inherit
    with:
      buildArgs: |
        arg1=value1
        arg2=value2
      environment: development
      file: Dockerfile
      gcpAuthMethod: workload-identity
      push: true
      ref: ${{ github.sha }}
      gcpServiceAccount: serviceaccount1@projectid-1234.iam.gserviceaccount.com
      tags: gcr.io/${{ github.repository }}/app1:${{ github.sha }}
      workloadIdentityProvider: projects/1111111111/locations/global/workloadIdentityPools/my-pool/providers/my-provider
```

`Note:` docker/setup-buildx-action and docker/build-push-action have been configured with the minimal number of inputs required to build and push an image to a container registry. These third party actions can be extended with additional features if required. Please see the below for additional details.

- [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action)
- [docker/build-push-action](https://github.com/docker/build-push-action)

## Uses
- `<organization>/<repository>/.github/workflows/build-gcp-image.yaml@v0` points to the action that will be used
- Anything after the `@` specifies the version of the action to use. In the example above the specific version `v0` is used.
- Note: 
  - Using `v0` ensures that the calling workflow will continue to use any minor or patch updates.
  - Specifying the full semantic version will lock the calling workflow to a specific version, ie `v0.0.1`.

## Secrets
- Use the `inherit` keyword to pass all secrets form the calling workflow to the called workflow.
  - More info [here.](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idsecrets)

- You can pass GitHub Secrets during build time by creating a GitHub Secret called DOCKER_SECRETS containing a list of strings.


Usage Example:
```
# Set Secret from DOCKER_SECRET containing a key/value SECRET1=secret1value
 RUN --mount=type=secret,id=SECRET1 \
     SECRET1=$(cat /run/secrets/SECRET1)
```

- When using a Service Account Key for authentication to GCP. Set the GitHub Secret GOOGLE_CREDENTIALS with the contents of the JSON file.

## With
- The inputs to pass to the action when called
  - More info [here.](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepswith)

|      input           |     required    |    type      |  default | description |
| -------------------- | --------------- | ------------ | -------- | ----------- |
| `buildArgs`         |    false        | List         | -        | A multiline list of build arguments |
| `environment`   |    false        | string       | ''        | The GitHub Environment containing the secret variable DOCKER_SECRETS |
| `file`               |    false        | string       | Dockerfile        | The Dockerfile to use for builds |
| `gcpAuthMethod`               |    true        | string       |  credentials_json  | The method for authenticating with GCP. Supports workload-identity or credentials_json |
| `push`               |    false        | boolean       | true  | Push image to GHCR |
| `ref`       |    false         | string       | ''        | The specific commitId, Tag, or Branch to checkout. Will use latest if not defined |
| `gcpServiceAccount`       |    false         | string       | ''        | The Service Account used by Workload Identity Federation |
| `tags`               |    true         | string       | -        | The Image Tag |
| `workloadIdentityProvider`               |    false         | string       | -        | The Workload Identity Provider Endpoint |

## Release Creation

Will automatically create a new semantically versioned [release](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) of the workflow any time there is a push to main. The version that gets created is determined by the commit messages found in Git.

1. Major Version: Commit Messages found with (MAJOR). Will create v1.0.0 and v1.
2. Minor Version: Commit Messages found with (MINOR). Will create v0.1.0 and update v0.
3. Patch Version: All other commit messages. Will create v0.0.1 and update v0.

### Release Creation Dependencies

1. Access to the reusable workflow `version-workflows`.
2. Update the create-version.yaml file with the location of the version-workflows reusable workflow.

```
name: Version Reusable Workflow
on:
  push:
    branches:
      - main
    paths-ignore:
      - '*.md'
  workflow_dispatch:

jobs:

  version-reusable-workflow:
    uses: <organization>/<repository>/.github/workflows/version-workflow.yaml@v0
```