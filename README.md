# Setup RapidFort Catalog

This action installs `rfcatalog` and helps you update Dockerfiles with the latest RapidFort image digests.

## What it does

1. Installs `rfcatalog` CLI tool
2. Checks for new image digests from RapidFort
3. Updates your Dockerfile with the latest digest
4. Creates a PR when digests change

## Quick Start

### 1. Add to your workflow

```yaml
- uses: rapidfort/setup-rfcatalog@v1
  with:
    access-id: ${{ secrets.RF_ACCESS_ID }}
    secret: ${{ secrets.RF_SECRET_ACCESS_KEY }}
```

### 2. Example: Update Alpine Image

Start with this Dockerfile:
```dockerfile
FROM --platform=linux/amd64 quay.io/rfcurated/alpine:3.21-rfcurated
RUN apk add --no-cache curl
```

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
          # Update to digest pinning format
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

## For ARM64 Architecture

Change the platform and architecture selection:

```dockerfile
FROM --platform=linux/arm64 quay.io/rfcurated/alpine:3.21-rfcurated
```

And in the workflow, select arm64:
```yaml
LATEST=$(rfcatalog -r alpine -f renovate | jq -r '.releases[] | select(.version == "3.21-rfcurated" and .architecture == "arm64") | .digest' | head -1)
```

## Setup Requirements

1. Add GitHub Secrets:
   - `RF_ACCESS_ID` - Your RapidFort Access ID
   - `RF_SECRET_ACCESS_KEY` - Your RapidFort Secret

2. The workflow will:
   - Run daily
   - Check if a new digest is available
   - Update your Dockerfile
   - Create a PR with the changes

## Manual Commands

After setup, you can run these commands:

```bash
# See all available Alpine versions
rfcatalog -r alpine -f renovate | jq '.releases[] | {version, architecture, digest}'

# Get latest digest for specific version/arch
rfcatalog -r alpine -f renovate | jq -r '.releases[] | select(.version == "3.21-rfcurated" and .architecture == "amd64") | .digest'
```

## Result

When RapidFort publishes a new image, you'll get a PR that changes:

```diff
- FROM --platform=linux/amd64 quay.io/rfcurated/alpine:3.21-rfcurated
+ FROM --platform=linux/amd64 quay.io/rfcurated/alpine:3.21-rfcurated@sha256:d63e8b573b219ff955de4110facff828d73936ae62d6e95ef42f216ba293b4ef
```

This ensures you always use the exact same image (digest pinning) while getting automated updates.