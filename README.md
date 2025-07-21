# Setup RapidFort Catalog Action

[![Test](https://github.com/rapidfort/setup-rfcatalog/actions/workflows/test.yml/badge.svg)](https://github.com/rapidfort/setup-rfcatalog/actions/workflows/test.yml)

A GitHub Action to install and configure the `rfcatalog` CLI for managing RapidFort curated container images.

## Features

- 🚀 **One-line setup** - Install rfcatalog with a single action
- ⚡ **Smart caching** - Cached installations for faster builds  
- 🔄 **Auto-updates** - Always installs the latest version by default
- 🔐 **Built-in authentication** - Optionally authenticate with RapidFort
- 🎯 **Cross-platform** - Works on Linux, macOS runners
- ✅ **Zero dependencies** - No sudo or additional tools required

## Quick Start

```yaml
steps:
  - uses: actions/checkout@v4
  
  - name: Setup RapidFort Catalog
    uses: rapidfort/setup-rfcatalog@v1
    with:
      access-id: ${{ secrets.RF_ACCESS_ID }}
      secret: ${{ secrets.RF_SECRET_ACCESS_KEY }}
      
  - name: Generate Renovate datasource
    run: rfcatalog -r alpine -f renovate > alpine.json



