# Deploy GitHub Action



This repository contains a custom GitHub action for deploying apps to Kubernetes clusters via helm. It is designed for Packhelp organisation specific needs. However, with some alterations, it should be possible to re-use in other environments.

## Basic usage

Create an `envs.yaml` file in your app's `helm-chart` directory. Use it to declare available environments, clusters they should be deployed to, and `values.yaml` files they should use:

```yaml
default:
  cluster: np
  values: values.dev.yaml
staging:
  cluster: np
  values: values.staging.yaml
production:
  cluster: pr
  values: values.production.yaml
```

*NOTE:* the `default` configuration will be used for unknown env names. This usually means development environments with temporary names.

Then, create a GitHub Actions workflow (e.g. `.github/workflows/deploy.yaml`):

```yaml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      env:
        description: Deployment target environment
        type: choice
        required: true
        default: "staging"
        options:
          - staging
          - production

jobs:
  build:
    ... # build Docker image and push to registry
  deploy:
    name: Deploy to Kubernetes
    runs-on: [self-hosted, deploy]
    needs: [build]
    steps:
      - uses: actions/checkout@v4
      - uses: packhelp/deploy@v1
        with:
          app: myapp
          env: ${{ inputs.env }}
          kubeconfig_np: "${{ secrets.<np_cluster_kubeconfig> }}"
          kubeconfig_pr: "${{ secrets.<pr_cluster_kubeconfig> }}"

```

The above workflow enables triggering a deployment manually from **Actions** page in your repository. Keep in mind, that the workflow will only be visible once you merge it to the default branch (e.g. `main`). Once there, you can iterate on another branch and switch using `Use workflow from:` select.

## Deploying on push to `main`

```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  build:
    ... # build Docker image and push to registry
  deploy:
    name: Deploy to Kubernetes
    runs-on: [self-hosted, deploy]
    needs: [build]
    steps:
      - uses: actions/checkout@v4
      - uses: packhelp/deploy@v1
        with:
          app: myapp
          env: staging
          kubeconfig_np: "${{ secrets.<np_cluster_kubeconfig> }}"
          kubeconfig_pr: "${{ secrets.<pr_cluster_kubeconfig> }}"
```

This workflow will deploy to `staging` env on each push to `main` branch.

## Deploying on a new release published

```yaml
name: Deploy

on:
  release:
    types:
      - published

jobs:
  build:
    ... # build Docker image and push to registry
  deploy:
    name: Deploy to Kubernetes
    runs-on: [self-hosted, deploy]
    needs: [build]
    steps:
      - uses: actions/checkout@v4
      - uses: packhelp/deploy@v1
        with:
          app: myapp
          env: production
          release_version: ${{ github.event.release.tag_name }}
          kubeconfig_np: "${{ secrets.<np_cluster_kubeconfig> }}"
          kubeconfig_pr: "${{ secrets.<pr_cluster_kubeconfig> }}"
```

This workflow deploys to `production` on each release published event. It also sets `release_version` to the GitHub release tag. This can be used to monitor which app is running at a time (e.g. Datadog's Universal Service Monitoring).

## Multiple triggers

Tying it all together, we use a single workflow to deploy after various triggers. Below is a workflow capable of all the above: manual workflow trigger, on-push deployment, on-release deployment.

```yaml
name: Deploy

on:
  release:
    types:
      - published
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      env:
        description: Deployment target environment
        type: choice
        required: true
        default: "staging"
        options:
          - staging
          - production

jobs:
  build:
    ... # build Docker image and push to registry
  deploy:
    name: Deploy to Kubernetes
    runs-on: [self-hosted, deploy]
    needs: [build]
    steps:
      - uses: actions/checkout@v4
      - name: Set deployment env for push to 'main'
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: echo "ENV=staging" >> $GITHUB_ENV
      - name: Set deployment env for new releases
        if: github.event_name == 'release' && github.event.action == 'published'
        run: echo "ENV=production" >> $GITHUB_ENV
      - uses: packhelp/deploy@v1
        with:
          app: myapp
          env: ${{ env.ENV || inputs.env }}
          release_version: ${{ github.event.release.tag_name || github.sha }}
          kubeconfig_np: "${{ secrets.<np_cluster_kubeconfig> }}"
          kubeconfig_pr: "${{ secrets.<pr_cluster_kubeconfig> }}"


```

Aside from defining multiple workflow triggers, we need to handle two more details.

First, we set up the environment for automatic workflow triggers (push, release). That's because in these cases, `inputs` does not contain any values and `env` would be empty.

Secondly, we set `release_version` to `github.event.release.tag_name`. For release published type events, this will hold the release tag. For other deployments (push, manual), we just use the git sha value. This could be set to other values as well, according to your process. For example, in some cases using branch names (`github.ref_name`) could be a viable option.
