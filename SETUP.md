# Setup Guide: GitHub App for Cross-Repo Dispatch

This guide walks through creating a GitHub App that replaces individual developer
PATs for `repository_dispatch` and private repo access across a multi-repo GitOps
pipeline in GitHub.

## Overview

The pipeline has two roles:

- **Sender repos** (e.g. `gitops-aws-example`, `gitops-base-image-example`) —
  build container images and dispatch a `new-tag` event to the manifest repo
- **Manifest repo** (`gitops-manifest-example`) — receives dispatches and
  updates deployment manifests

A GitHub App generates short-lived, scoped installation tokens that replace
long-lived PATs tied to individual developer accounts.

## 1. Create the GitHub App

1. Go to your org's GitHub App settings:
   **Organization Settings → Developer settings → GitHub Apps → New GitHub App**

2. Fill in the basic fields:

   | Field               | Value                                                               |
   | ------------------- | ------------------------------------------------------------------- |
   | **GitHub App name** | Something descriptive and globally unique (e.g. `myorg-deploy-bot`) |
   | **Homepage URL**    | Your org's GitHub URL (e.g. `https://github.com/myorg`)             |

3. Under **Webhook**, uncheck **Active**. This app generates tokens only — it
   does not need to receive webhook events.

4. Under **Permissions → Repository permissions**, set:

   | Permission            | Access           | Why                                                                                     |
   | --------------------- | ---------------- | --------------------------------------------------------------------------------------- |
   | **Contents**/**Code** | **Read & write** | Required to trigger `repository_dispatch` on the target repo and to clone private repos |
   | **Deployments**       | **Read & write** | **FUTURE** Manage GitHub's display of deployments                                       |
   | **Environments**      | **Read & write** | **FUTURE** Manage GitHub's per-environment settings                                     |
   | **Issues**            | **Read & write** | **FUTURE** Create Issues when a pipeline fails and needs attention                      |
   | **Pull requests**     | **Read & write** | **FUTURE** Create and update Pull Requests for change visibility                        |
   | **Metadata**          | **Read-only**    | Automatically granted, cannot be changed                                                |

   Leave all other permissions at "No access".

5. Under **Where can this GitHub App be installed?**, select
   **Only on this account**.

6. Click **Create GitHub App**.

7. On the app's settings page, note the **App ID** (numeric value near the top
   of the page). You will need this as a secret.

## 2. Generate and store a Private Key

1. On the app's settings page, scroll down to **Private keys**.
2. Click **Generate a private key**.
3. Your browser downloads a `.pem` file. Keep this file — the full contents
   (including the `-----BEGIN RSA PRIVATE KEY-----` and
   `-----END RSA PRIVATE KEY-----` lines) will be stored as a secret.
4. Go to **Organization Settings → Secrets and variables → Actions → New organization secret**.
5. Store the contents of the `.pem` file as an org-level secret, along with the App ID.

   | Secret name              | Value                                        | Repository access                |
   | ------------------------ | -------------------------------------------- | -------------------------------- |
   | `DEPLOY_APP_ID`          | The numeric App ID from step 1.7             | Grant access to all sender repos |
   | `DEPLOY_APP_PRIVATE_KEY` | Full contents of the `.pem` file from step 2 | Same repos as above              |

   Under **Repository access**, select **Selected repositories** and add each
   repo that has a workflow needing to mint tokens (the sender repos). The
   manifest repo itself only needs these secrets if it also mints tokens (e.g.
   to clone a private repo during smoke tests).


## 3. Install the App on Target Repos

The app only needs to be installed on repos where tokens will be **used against**
(i.e. repos that the token needs to read from or dispatch to). Sender repos do
NOT need the app installed — they only need the App ID and private key to
request a token.

1. Go to **Organization Settings → GitHub Apps → (your app) → Install App**,
   or go to **Organization Settings → Installations** and click **Configure**
   next to your app.

2. Select **Only select repositories** and choose the repos the app needs
   access to. For this example:
   - `gitops-manifest-example` (dispatch target)

   If any workflow also needs to clone a private repo (e.g. for smoke tests),
   add that repo here too.

3. Click **Install** (or **Save** if updating an existing installation).


## 4. Use in Workflows

### Sender workflow (dispatches to manifest repo)

```yaml
steps:
  - name: Generate GitHub App token
    id: app-token
    uses: actions/create-github-app-token@v2
    with:
      app-id: ${{ secrets.DEPLOY_APP_ID }}
      private-key: ${{ secrets.DEPLOY_APP_PRIVATE_KEY }}
      owner: your-org
      repositories: gitops-manifest-example

  - name: Dispatch to manifest repo
    uses: peter-evans/repository-dispatch@v3
    with:
      token: ${{ steps.app-token.outputs.token }}
      repository: your-org/gitops-manifest-example
      event-type: new-tag
      client-payload: |-
        {
          "tag": "${{ env.IMAGE_TAG }}",
          "type": "my-service",
          "project": "my-repo"
        }
```

### Receiver workflow (manifest repo)

No special token setup needed — the manifest repo just listens for the event:

```yaml
on:
  repository_dispatch:
    types: [new-tag]

jobs:
  handle-dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Use payload
        run: |
          echo "Tag: ${{ github.event.client_payload.tag }}"
          echo "Type: ${{ github.event.client_payload.type }}"
```

### Cloning a private repo (e.g. for smoke tests)

If a workflow needs to check out a different private repo, mint a token scoped
to that repo:

```yaml
steps:
  - name: Generate GitHub App token
    id: app-token
    uses: actions/create-github-app-token@v2
    with:
      app-id: ${{ secrets.DEPLOY_APP_ID }}
      private-key: ${{ secrets.DEPLOY_APP_PRIVATE_KEY }}
      owner: your-org
      repositories: the-private-repo

  - name: Checkout private repo
    uses: actions/checkout@v4
    with:
      repository: your-org/the-private-repo
      token: ${{ steps.app-token.outputs.token }}
```

## 6. Clean Up Old PATs

After verifying the GitHub App flow works end-to-end:

1. Delete the old PAT-based secrets (e.g. `MANIFEST_REPO_TOKEN`,
   `FRONTEND_REPO_TOKEN`) from each repo that had them.
2. Revoke the underlying PATs from **GitHub → Settings → Developer settings →
   Personal access tokens**.

## Troubleshooting

**"Resource not accessible by integration"**
- The app is not installed on the target repo. Go to the app's installation
  settings and add the repo.

**"Could not create installation access token"**
- The `DEPLOY_APP_ID` or `DEPLOY_APP_PRIVATE_KEY` secret is wrong or not
  accessible to the repo running the workflow. Check that the org secret grants
  access to that repo.

**Dispatch succeeds but no workflow runs on the manifest repo**
- The manifest repo must have a workflow on the default branch that listens for
  `repository_dispatch` with the matching event type.
- GitHub does not trigger workflows for `repository_dispatch` if the workflow
  file was just added — it must be merged to the default branch first.
