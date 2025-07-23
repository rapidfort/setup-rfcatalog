# Setup RapidFort Catalog Action

[![Test](https://github.com/rapidfort/setup-rfcatalog/actions/workflows/test.yml/badge.svg)](https://github.com/rapidfort/setup-rfcatalog/actions/workflows/test.yml)

A GitHub Action to install and configure the `rfcatalog` CLI for managing RapidFort curated container images.

## Features

- 🚀 **One-line setup** - Install rfcatalog with a single action
- ⚡ **Smart caching** - Cached installations for faster builds  
- 🔄 **Auto-updates** - Always installs the latest version by default
- 🔐 **Built-in authentication** - Optionally authenticate with RapidFort
- 🎯 **Cross-platform** - Works on Linux and macOS runners

## Quick Start

```yaml
- uses: rapidfort/setup-rfcatalog@v1
  with:
    access-id: ${{ secrets.RF_ACCESS_ID }}
    secret: ${{ secrets.RF_SECRET_ACCESS_KEY }}
```

## Usage

### Basic Installation (No Authentication)

```yaml
- uses: rapidfort/setup-rfcatalog@v1
  with:
    authenticate: false
```

### With Authentication

```yaml
- uses: rapidfort/setup-rfcatalog@v1
  with:
    access-id: ${{ secrets.RF_ACCESS_ID }}
    secret: ${{ secrets.RF_SECRET_ACCESS_KEY }}
```

### Specific Version

```yaml
- uses: rapidfort/setup-rfcatalog@v1
  with:
    version: '1.4.3100'
    access-id: ${{ secrets.RF_ACCESS_ID }}
    secret: ${{ secrets.RF_SECRET_ACCESS_KEY }}
```

## Complete Example: Update Docker Images

Create `.github/workflows/update-images.yml`:

```yaml
name: Update RapidFort Images
on:
  schedule:
    - cron: '0 0 * * *'  # Daily
  workflow_dispatch:

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: rapidfort/setup-rfcatalog@v1
        with:
          access-id: ${{ secrets.RF_ACCESS_ID }}
          secret: ${{ secrets.RF_SECRET_ACCESS_KEY }}
      
      - name: Check for updates
        env:
          RF_CONTAINER_ENGINE: docker
        run: |
          # Get current digest from Dockerfile
          CURRENT=$(grep "FROM.*alpine.*@sha256:" Dockerfile | grep -o "sha256:[a-f0-9]*" | cut -d: -f2)
          
          # Get latest digest from RapidFort (for amd64)
          LATEST=$(rfcatalog -r alpine -f renovate | jq -r '.releases[] | select(.version == "3.21-rfcurated" and .architecture == "amd64") | .digest' | head -1)
          
          if [ "$CURRENT" != "$LATEST" ]; then
            echo "New digest available!"
            echo "current_digest=$CURRENT" >> $GITHUB_ENV
            echo "new_digest=$LATEST" >> $GITHUB_ENV
            echo "update_needed=true" >> $GITHUB_ENV
          else
            echo "Already up to date"
            echo "update_needed=false" >> $GITHUB_ENV
          fi
      
      - name: Update Dockerfile
        if: env.update_needed == 'true'
        run: |
          sed -i "s|FROM --platform=linux/amd64 quay.io/rfcurated/alpine:3.21-rfcurated.*|FROM --platform=linux/amd64 quay.io/rfcurated/alpine:3.21-rfcurated@sha256:${{ env.new_digest }}|" Dockerfile
          
      - name: Create Pull Request
        if: env.update_needed == 'true'
        uses: peter-evans/create-pull-request@v5
        with:
          title: "Update Alpine to latest digest"
          body: |
            Updates Alpine image to latest digest
            
            **Old digest**: `${{ env.current_digest }}`
            **New digest**: `${{ env.new_digest }}`
          branch: update-alpine-digest
          commit-message: "chore: update alpine to latest digest"
```

## Action Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `version` | Version of rfcatalog to install | No | `latest` |
| `access-id` | RapidFort Access ID for authentication | No | - |
| `secret` | RapidFort Secret for authentication | No | - |
| `authenticate` | Whether to authenticate with RapidFort | No | `auto` |

## Action Outputs

| Output | Description |
|--------|-------------|
| `version` | Version of rfcatalog that was installed |
| `cache-hit` | Whether the installation was restored from cache |

## Environment Variables

If you're using container-related commands, set:

```yaml
env:
  RF_CONTAINER_ENGINE: docker  # or podman
```

## Requirements

### 1. Get RapidFort Credentials

1. Sign up at [rapidfort.com](https://rapidfort.com)
2. Create a service account:
   - Go to Settings → Service Accounts
   - Click "Create Service Account"
   - Save the Access ID and Secret Access Key

### 2. Add GitHub Secrets

In your repository:
1. Go to Settings → Secrets and variables → Actions
2. Add these secrets:
   - `RF_ACCESS_ID` - Your RapidFort Access ID
   - `RF_SECRET_ACCESS_KEY` - Your RapidFort Secret Access Key

### 3. What the action does

- Installs the latest rfcatalog CLI to `~/.local/bin/rfcatalog`
- Adds it to PATH for subsequent steps
- Configures authentication if credentials provided
- Sets up RF_CONTAINER_ENGINE=docker automatically

## Manual Commands

After setup, you can run:

```bash
# See all available Alpine versions
rfcatalog -r alpine -f renovate | jq '.releases[] | {version, architecture, digest}'

# Get latest digest for specific version/arch
rfcatalog -r alpine -f renovate | jq -r '.releases[] | select(.version == "3.21-rfcurated" and .architecture == "amd64") | .digest'
```

## Support

- 📧 Email: support@rapidfort.com
- 📚 Docs: [docs.rapidfort.com](https://docs.rapidfort.com)

## License

Apache 2.0