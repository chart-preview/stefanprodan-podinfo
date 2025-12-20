# Chart Preview Setup Guide

Welcome to Chart Preview! This guide will help you configure automated chart preview deployments for your repository.

## 🚀 Quick Start

Your Chart Preview system is almost ready! Follow these steps to complete the setup:

### 1. Create GitHub Workflow

⚠️ **Manual Step Required**: Due to GitHub App security restrictions, the workflow file could not be automatically placed in `.github/workflows/`.

**Please follow these steps:**

1. Merge this PR to get the setup guide into your repository
2. Go to your repository on GitHub
3. Click **Add file** → **Create new file**
4. Name the file: `.github/workflows/chart-preview.yml`
5. Paste the workflow content below
6. Commit directly to `main` branch

⚠️ **Important**: Create the workflow file directly on `main`, not via a pull request.

### 2. Add Your PREVIEW_TOKEN Secret

Your Chart Preview token:
```
rpt_OycQaLQI.OycQaLQIT4laI3h2porFNRa3XQcN5i2Hg0rxm2kdk5U
```

**To add the secret:**
1. Go to your repository **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `PREVIEW_TOKEN`
4. Value: The token above

### 3. Update the Chart Path

Edit the workflow to set the correct path to your Helm chart:

```json
"git": {
  "chartPath": "charts/myapp"  // ← Update this to your chart location
}
```

---

## Workflow Options

### Option A: Git Chart Flow (Recommended)

The simplest approach - deploy directly from your repository. No need to package or publish charts.

```yaml
name: Chart Preview
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Preview
        run: |
          curl -fsSL -X POST \
            -H "Authorization: Bearer ${{ secrets.PREVIEW_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "repo": "${{ github.repository }}",
              "pr": ${{ github.event.pull_request.number }},
              "git": {
                "provider": "github",
                "owner": "${{ github.repository_owner }}",
                "repo": "${{ github.event.repository.name }}",
                "ref": "${{ github.event.pull_request.head.sha }}",
                "chartPath": "charts/myapp"
              },
              "values": {
                "ingress": {
                  "enabled": true,
                  "ingressClassName": "nginx"
                }
              },
              "head": {
                "owner": "${{ github.repository_owner }}",
                "repo": "${{ github.event.repository.name }}",
                "sha": "${{ github.event.pull_request.head.sha }}"
              }
            }' \
            "https://api.previews.chart-preview.com/previews"
```

**Benefits:**
- No Docker build required
- No chart packaging/publishing
- Chart changes in your PR are automatically deployed
- Works with private repositories

### Option B: Docker Build + OCI Chart

For applications that need to build Docker images as part of the preview:

```yaml
name: Chart Preview
on:
  pull_request:
    types: [opened, synchronize, reopened]
    paths-ignore:
      - '.github/**'
      - '**.md'

permissions:
  contents: read
  packages: write
  statuses: write

jobs:
  # Job 1: Build and push Docker image
  build:
    runs-on: ubuntu-latest
    outputs:
      image: ghcr.io/${{ github.repository }}
      digest: ${{ steps.push.outputs.digest }}
      tags: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Generate Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=pr
            type=sha,prefix=pr-${{ github.event.pull_request.number }}-
            type=raw,value=pr-${{ github.event.pull_request.number }}

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          provenance: false
          sbom: false
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Update commit status (image-build)
        if: success()
        run: |
          curl -X POST \
               -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
               -H "Accept: application/vnd.github+json" \
               -H "X-GitHub-Api-Version: 2022-11-28" \
               "https://api.github.com/repos/${{ github.repository }}/statuses/${{ github.sha }}" \
               -d '{
                 "state": "success",
                 "context": "image-build",
                 "description": "Docker image built and pushed successfully"
               }'

      - name: Update commit status on failure (image-build)
        if: failure()
        run: |
          curl -X POST \
               -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
               -H "Accept: application/vnd.github+json" \
               -H "X-GitHub-Api-Version: 2022-11-28" \
               "https://api.github.com/repos/${{ github.repository }}/statuses/${{ github.sha }}" \
               -d '{
                 "state": "failure",
                 "context": "image-build",
                 "description": "Docker image build failed"
               }'

  # Job 2: Deploy preview environment
  deploy-preview:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy Preview Environment
        run: |
          cat > payload.json <<EOF
          {
            "repo": "${{ github.repository }}",
            "pr": ${{ github.event.pull_request.number }},
            "chart": {
              "source": "oci://ghcr.io/${{ github.repository_owner }}/helm-chart",
              "version": "1.0.0"
            },
            "values": {
              "image": {
                "repository": "${{ needs.build.outputs.image }}",
                "digest": "${{ needs.build.outputs.digest }}"
              }
            },
            "head": {
              "owner": "${{ github.repository_owner }}",
              "repo": "${{ github.event.repository.name }}",
              "sha": "${{ github.event.pull_request.head.sha }}"
            }
          }
          EOF

          # Note: For private images, add imagePullSecrets and secrets fields.
          # See PREVIEW_SETUP.md for details.

          echo "Deploying preview environment..."
          curl -fsSL -X POST \
            -H "Authorization: Bearer ${{ secrets.PREVIEW_TOKEN }}" \
            -H "Content-Type: application/json" \
            --data @payload.json \
            "https://api.previews.chart-preview.com/previews"
```

**Use this when:**
- You need to build and test application code changes
- Your chart references a Docker image that should be built from the PR
- You have a Dockerfile in your repository

---

## Customizing Values

Update the `values` section in the workflow to configure your chart:

```json
"values": {
  "ingress": {
    "enabled": true,
    "ingressClassName": "nginx"
  },
  "replicaCount": 1,
  "resources": {
    "limits": { "cpu": "200m", "memory": "256Mi" }
  }
}
```

## Monitoring & Debugging

- **Dashboard**: `https://dashboard.previews.chart-preview.com`
- **API**: `https://api.previews.chart-preview.com/previews?repo=stefanprodan-podinfo`

### Common Issues

| Issue | Solution |
|-------|----------|
| `chartPath` not found | Update the path to where your `Chart.yaml` is located |
| 401 Unauthorized | Check that `PREVIEW_TOKEN` secret is set correctly |
| Helm lint errors | Run `helm lint stefanprodan-podinfo/` locally to debug |

## Next Steps

1. [ ] Merge this PR
2. [ ] Add the workflow file (if not auto-created)
3. [ ] Add the `PREVIEW_TOKEN` secret
4. [ ] Update `chartPath` to your chart location
5. [ ] Create a test PR to verify

---

*Generated by Chart Preview*
