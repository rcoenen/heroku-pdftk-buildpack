# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Heroku buildpack that installs **pdftk-java** on Heroku-24 for PDF manipulation. Unlike the legacy version, this buildpack uses the modern pdftk-java implementation from Ubuntu 24.04 apt repositories instead of compiling from source or bundling binaries.

**Supported Stack**: Heroku-24 (Ubuntu 24.04) only

## Architecture

This is a lightweight buildpack that works in conjunction with the Heroku APT buildpack.

### Buildpack Components

- **bin/detect**: Always returns success, making this buildpack compatible with any Heroku app
- **bin/compile**: Configuration script that:
  - Creates `.profile.d/pdftk.sh` to add `/app/.apt/usr/bin` to PATH
  - Verifies pdftk installation if available at build time
  - Outputs installation status messages
- **bin/release**: Returns empty config (no addons or config vars needed)
- **Aptfile**: Lists packages to install via heroku-buildpack-apt:
  - `pdftk-java` - The PDF toolkit (modern Java-based implementation)
  - `libbcprov-java` - Bouncy Castle cryptography provider
  - `libcommons-lang3-java` - Apache Commons utilities

### How It Works

1. **heroku-buildpack-apt** (must be added before this buildpack) installs packages from Aptfile to `/app/.apt/usr/bin/`
2. This buildpack configures PATH via `.profile.d/pdftk.sh` so pdftk is available at runtime
3. Applications can use `pdftk` command normally - it's CLI-compatible with original pdftk

### Key Design Decisions

- **No binary bundling**: pdftk-java is installed via apt, not pre-compiled binaries
- **Minimal buildpack**: Only handles PATH configuration; apt buildpack does the heavy lifting
- **Heroku-24 only**: Older stacks (heroku-20, heroku-22) are not supported in current version
- **pdftk-java vs original pdftk**: Uses actively-maintained Java implementation instead of obsolete GCJ-based C++ version

## Usage Pattern

Users must add TWO buildpacks in this order:

```bash
heroku buildpacks:add --index 1 https://github.com/heroku/heroku-buildpack-apt
heroku buildpacks:add https://github.com/USERNAME/heroku-pdftk-buildpack
```

The apt buildpack must come first to install packages before this buildpack configures paths.

## Compatibility Notes

- **CLI Compatibility**: pdftk-java has identical command-line syntax to original pdftk
- **Laravel/PHP**: No code changes needed when migrating from old buildpack
- **Stack Migration**: This version only supports Heroku-24; legacy binaries were removed

## No Build/Compilation Process

Unlike the legacy buildpack, there is NO compilation step. pdftk-java is installed directly from Ubuntu repositories. The old `scripts/build.sh` and binary directories have been removed.
