# Heroku PDFtk Buildpack

Custom buildpack that installs **pdftk-java** on Heroku for PDF manipulation. Supports **Heroku-24** (Ubuntu 24.04).

This buildpack uses the modern [pdftk-java](https://gitlab.com/pdftk-java/pdftk) implementation, which is a drop-in replacement for the original pdftk. It has identical command-line syntax, so your existing code will work without modifications.

## Why pdftk-java?

The original pdftk (C++/GCJ version) is no longer maintained and doesn't work on modern Heroku stacks. pdftk-java is:
- Actively maintained
- Compatible with modern Ubuntu/Heroku stacks
- 100% CLI-compatible with original pdftk
- Based on OpenJDK instead of obsolete GCJ

## Requirements

This buildpack requires the **Heroku APT buildpack** to install system packages.

## How to Use

### 1. Create Aptfile in Your Application Root

**IMPORTANT**: You must create an `Aptfile` in your Laravel/PHP application root (not in the buildpack repo). This tells the apt buildpack which packages to install.

Create a file named `Aptfile` in your application root with these contents:

```
pdftk-java
libbcprov-java
libcommons-lang3-java
```

Commit this file to your repository:

```bash
git add Aptfile
git commit -m "Add Aptfile for pdftk-java dependencies"
```

### 2. Add Buildpacks to Your App

You need to add **two buildpacks** in this order:

```bash
# First, add the apt buildpack (to install pdftk-java via apt)
heroku buildpacks:add --index 1 https://github.com/heroku/heroku-buildpack-apt

# Then, add this pdftk buildpack (to configure paths)
heroku buildpacks:add https://github.com/rcoenen/heroku-pdftk-buildpack
```

**Important**: The apt buildpack must come before this buildpack.

### 3. Deploy Your App

```bash
git push heroku main
```

### 4. Use pdftk in Your Application

The `pdftk` command will be available in your application's runtime environment:

```bash
pdftk input.pdf output output.pdf
```

For PHP/Laravel applications:

```php
exec('pdftk input.pdf fill_form data.fdf output output.pdf');
```

## What This Buildpack Does

1. Works with the apt buildpack to install `pdftk-java` and its dependencies (specified in your app's `Aptfile`)
2. Configures the PATH to include `/app/.apt/usr/bin` where pdftk is installed
3. Verifies pdftk installation during build

## Dependencies

The following packages are installed via the `Aptfile` **in your application root** (not the buildpack repo):

- `pdftk-java` - The PDF toolkit
- `libbcprov-java` - Bouncy Castle cryptography provider
- `libcommons-lang3-java` - Apache Commons utilities

**Note**: The `Aptfile` in this buildpack repository is just a reference example. You must create your own `Aptfile` in your Laravel/PHP application root.

## Compatibility

- **Supported Stacks**: Heroku-24 (Ubuntu 24.04)
- **Legacy Support**: For older stacks (Heroku-20, Heroku-22), use an earlier version of this buildpack from the git history

## Migrating from Old pdftk Buildpack

If you're upgrading from the old GCJ-based buildpack:

1. No code changes needed - pdftk-java has identical CLI syntax
2. Update your buildpack configuration as shown above
3. Deploy to Heroku-24 stack
4. Test your PDF operations

## Troubleshooting

### pdftk command not found

Make sure:
1. You created an `Aptfile` in your **application root** (see step 1 above)
2. The apt buildpack is added and comes before this buildpack
3. You've deployed your app after adding the buildpacks and the Aptfile

### Checking pdftk version

```bash
heroku run pdftk --version
```

You should see output indicating pdftk-java version 3.3.3 or newer.

## Support

This buildpack is designed for Heroku-24. For issues or questions, please open an issue on GitHub.
