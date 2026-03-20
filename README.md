# gitops-manifest-example
Example repository hosting manifests for GitOps deployments

## GenAI Usage

This repository made use of GenAI through Anthropic's Claude.

## Setup

See [SETUP.md](SETUP.md) for instructions on creating and configuring the
GitHub App used for cross-repo dispatch (replacing individual developer PATs).

The `gitops-aws-example` repository contains Terraform code to set up the AWS infrastructure used in this example, including ECR repositories and IAM permissions. These steps can be performed manually, but the ideal is to script as much as possible to avoid human error and ensure consistency across environments.

## GitOps Flow

- `gitops-base-image-example` contains Containerfiles for hardened base images as well as a GitHub Actions workflow that builds and pushes these images to a container registry.
  - Scans with Syft and Grype.
  - SBOMs and Cosign attestations are stored in the container registry (AWS ECR in the initial example).
  - So long as the new image build has reduced the number of vulnerabilities, the workflow will trigger a cross-repo dispatch to `gitops-application-example` to update the image tag in the application's Containerfile.
  - This update generates a pull request in `gitops-application-example` which must be reviewed and merged by a human.
- `gitops-application-example` contains a Containerfile for an application image that uses the hardened base image from `gitops-base-image-example`.
  - Each Pull Request runs comprehensive tests before permitting a merge to `main`.
  - Each Pull Request also runs Syft and Grype scans to ensure no new vulnerabilities are introduced by the change.
    - Vulnerabilities already in the base image do not block the PR. Those are to be addressed in the `gitops-base-image-example` repository.
    - Vulnerabilities already in `main` do not block the PR. Only new vulnerabilities introduced by the PR will block it.
    - Vulnerabilities can be ignored by committing to the list of accepted risks.
  - Once merged to `main, a workflow builds a new image and pushes it to the container registry, storing SBOMs and Cosign attestations.
  - This then triggers a cross-repo dispatch to `gitops-manifest-example` to update the image tag in the Kubernetes manifests managed by Argo CD.
- `gitops-manifest-example` contains Kubernetes manifests for Argo CD to deploy the application.
  - A `new-tag` event triggers a workflow that creates a branch and PR with the updated image tag applied to the Kubernetes manifests.
  - Once the PR is auto-merged, Argo CD detects the change and deploys the new image to the cluster.
  - A promotion PR is then created to promote the change to the next environment.
  - Promotion PRs run smoke tests against the previous environment to ensure end-to-end functionality is maintained before allowing the merge to the next environment.
