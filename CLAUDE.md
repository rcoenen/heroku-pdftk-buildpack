# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository is **comprehensive documentation** for running pdftk-java on Heroku-24 using **standard buildpacks only**. It is NO LONGER a custom buildpack - it evolved from a custom buildpack into a community guide after discovering that standard buildpacks are sufficient.

**Repository Purpose**: Documentation and troubleshooting guide

**Custom Buildpack Status**: OBSOLETE - the `bin/` directory is preserved for historical reference only

## What This Repository Contains

### Active Documentation (README.md)

The README.md is the main deliverable - a complete guide covering:

1. **The Journey** - How we migrated from GCJ pdftk to pdftk-java on Heroku-24
2. **The Working Solution** - Required files (Aptfile, .profile) for Laravel apps
3. **Problems & Fixes** - Documented issues encountered (Java security symlinks, JAR paths)
4. **Complete Setup** - Step-by-step instructions anyone can follow
5. **Why pdftk-java** - Comparison with alternatives (qpdf, pdfcpu)

### Obsolete Code (bin/, Aptfile in repo)

- **bin/compile, bin/detect, bin/release**: Old buildpack scripts (UNUSED)
- **Aptfile in root**: Example only; users must create Aptfile in their app
- These files are preserved for git history but are not part of the solution

## The Actual Solution (What Users Need)

Users need to create two files in their **Laravel app root** (not this repo):

### 1. Aptfile (in user's app)
```
openjdk-17-jre-headless
pdftk-java
libbcprov-java
libcommons-lang3-java
```

### 2. .profile (in user's app)
- Fixes Java security configuration symlinks (broken by apt buildpack's /app/.apt/ install location)
- Creates pdftk wrapper script with correct JAR paths
- Runs on every dyno boot

### 3. Buildpack Configuration
- heroku-community/apt (index 1 - must be first)
- heroku/php or heroku/nodejs (as needed)
- NO custom pdftk buildpack needed

## Key Technical Discoveries

### Discovery 1: heroku-buildpack-apt handles PATH automatically
- Creates `.profile.d/000_apt.sh` that exports `PATH="$HOME/.apt/usr/bin:$PATH"`
- Custom buildpack for PATH configuration is redundant

### Discovery 2: Ubuntu's pdftk-java package has broken symlinks on Heroku
- Java config files are symlinks pointing to `/etc/java-17-openjdk/`
- But apt buildpack installs to `/app/.apt/etc/java-17-openjdk/`
- Result: `java.lang.InternalError: Error loading java.security file`
- Solution: `.profile` script recreates symlinks on boot

### Discovery 3: pdftk wrapper has hardcoded paths
- Ubuntu's `/usr/bin/pdftk.pdftk-java` contains: `/usr/share/java/bcprov.jar`
- On Heroku, JARs are at: `/app/.apt/usr/share/java/bcprov.jar`
- Solution: Custom wrapper in `.profile` with correct paths

## Why pdftk-java (Not qpdf)

Research showed:
- **pdftk-java is actively maintained** (2025) and 7-8x faster than qpdf
- **qpdf lacks form filling** - critical feature for Laravel apps
- **Laravel ecosystem uses mikehaertl/php-pdftk** - works with pdftk-java
- **CLI compatibility** - pdftk-java is drop-in replacement for original pdftk

## Repository Maintenance

When updating this repository:

1. **Focus on README.md** - The guide is the deliverable
2. **Update based on user feedback** - Document new issues/solutions
3. **Preserve old buildpack code** - Don't delete bin/ (historical reference)
4. **Keep CLAUDE.md current** - Document repository evolution

## For Future Work

If issues arise with the documented solution:
- Test the fix locally with Docker (`docker run ubuntu:24.04`)
- Document the problem and solution in README.md
- Update .profile example with fixes
- Consider whether a minimal buildpack wrapper is actually needed
