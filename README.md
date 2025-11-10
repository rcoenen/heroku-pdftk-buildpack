# PDFtk-Java on Heroku-24: The Complete Guide

> **Note**: This repository has evolved from a custom buildpack into a comprehensive guide for using pdftk-java on Heroku-24 with **standard buildpacks only**. No custom buildpack is needed!

## ⚠️ Important: Heroku Stack Dependency

**This solution is specifically designed for Heroku-24** (Ubuntu 24.04). It relies on:
- Heroku-24 stack architecture
- Ubuntu 24.04 package layouts
- Specific apt buildpack behavior

**Do NOT upgrade to future Heroku stacks (Heroku-26/28) without re-testing PDFtk!**

### Expected Lifetime
- ✅ **Safe until ~2029** - Heroku-24 LTS support
- ⚠️ **Re-test required** when Heroku deprecates Heroku-24
- 💡 **Migration path** - See [Alternative Approaches](#alternative-approaches) below

### Why This Matters

The solution works by fixing broken symlinks in the Java configuration. This is a **workaround for apt buildpack's non-standard installation paths**, not an official solution. Future Heroku stacks may:
- Change `/app/.apt/` structure
- Use different OpenJDK packaging
- Modify how apt buildpack works

**Bottom line:** This is a "we know what we're doing" hack. Document it, test it, plan for eventual migration.

## Table of Contents

1. [Overview](#overview)
2. [The Journey](#the-journey)
3. [The Working Solution](#the-working-solution)
4. [Problems Encountered & Fixes](#problems-encountered--fixes)
5. [Complete Setup Instructions](#complete-setup-instructions)
6. [Why pdftk-java?](#why-pdftk-java)

---

## Overview

This guide documents how to successfully run **pdftk-java** on Heroku-24 (Ubuntu 24.04) using standard Heroku buildpacks. The original pdftk (C++/GCJ version) no longer works on modern Heroku stacks, but pdftk-java is a drop-in replacement that's actively maintained and 100% CLI-compatible.

**TL;DR**: You need an `Aptfile` + `.profile` script in your app. No custom buildpack required.

---

## The Journey

### The Goal

Replace the obsolete GCJ-based pdftk with modern pdftk-java on Heroku-24 while maintaining backward compatibility with existing Laravel applications that use pdftk for PDF form filling.

### Initial Approach: Custom Buildpack

We started by creating a custom buildpack that would:
- Bundle pre-compiled binaries (like the old approach)
- Handle Java classpath configuration
- Work alongside heroku-buildpack-apt

### The Discovery

Through testing and research, we discovered:

1. **heroku-buildpack-apt already handles PATH** - It automatically creates `.profile.d/000_apt.sh` that adds `/app/.apt/usr/bin` to PATH
2. **Ubuntu 24.04's pdftk-java package is well-designed** - It creates proper symlinks and wrapper scripts
3. **A custom buildpack was overkill** - The standard apt buildpack + simple configuration was sufficient

### The Implementation

We switched to using:
- **Aptfile** - Declares packages to install via heroku-buildpack-apt
- **.profile** - Boot script to fix Java configuration issues
- **Standard buildpacks** - heroku/nodejs, heroku/php, heroku-community/apt

---

## The Working Solution

### Required Files in Your App Root

**1. Aptfile** (declares system packages)
```
openjdk-17-jre-headless
pdftk-java
libbcprov-java
libcommons-lang3-java
```

**2. .profile** (fixes Java config + creates pdftk wrapper)
```bash
#!/bin/bash

# Fix Java security configuration symlinks
# The apt buildpack installs to /app/.apt/ but Java config symlinks point to /etc/
JAVA_SECURITY_DIR="/app/.apt/usr/lib/jvm/java-17-openjdk-amd64/conf/security"

if [ -d "$JAVA_SECURITY_DIR" ]; then
    echo "Fixing Java security configuration symlinks..."
    rm -f "$JAVA_SECURITY_DIR/java.policy" "$JAVA_SECURITY_DIR/java.security" "$JAVA_SECURITY_DIR/nss.cfg"
    ln -sf /app/.apt/etc/java-17-openjdk/security/java.policy "$JAVA_SECURITY_DIR/java.policy"
    ln -sf /app/.apt/etc/java-17-openjdk/security/java.security "$JAVA_SECURITY_DIR/java.security"
    ln -sf /app/.apt/etc/java-17-openjdk/security/nss.cfg "$JAVA_SECURITY_DIR/nss.cfg"
    echo "Java security configuration fixed."
fi

# Create pdftk wrapper with correct paths
mkdir -p /app/bin
cat > /app/bin/pdftk << 'EOF'
#!/bin/sh
java -cp /app/.apt/usr/share/java/bcprov.jar:/app/.apt/usr/share/java/commons-lang3.jar:/app/.apt/usr/share/pdftk/pdftk.jar com.gitlab.pdftk_java.pdftk "$@"
EOF
chmod +x /app/bin/pdftk

echo "pdftk-java configured and ready!"
```

**3. Buildpack Configuration**

```bash
heroku buildpacks:clear
heroku buildpacks:add heroku/nodejs
heroku buildpacks:add heroku/php
heroku buildpacks:add --index 1 heroku-community/apt
```

Order matters! The apt buildpack must be **first** so packages are available to other buildpacks.

---

## Problems Encountered & Fixes

### Problem 1: Java Security File Error

**What happened:**
```
java.lang.InternalError: Error loading java.security file
```

**Symptoms:**
- ✅ Read operations worked (`pdftk dump_data_fields`)
- ❌ Write operations failed (`pdftk fill_form`)

**Root Cause:**

When apt buildpack installs OpenJDK to `/app/.apt/`, the Java configuration files are symlinks pointing to system paths:

```
/app/.apt/usr/lib/jvm/java-17-openjdk-amd64/conf/security/java.security
  → /etc/java-17-openjdk/security/java.security  (doesn't exist!)
```

But the actual files are at:
```
/app/.apt/etc/java-17-openjdk/security/java.security
```

**The Fix:**

The `.profile` script recreates these symlinks on every dyno boot to point to the correct `/app/.apt/etc/` location:

```bash
JAVA_SECURITY_DIR="/app/.apt/usr/lib/jvm/java-17-openjdk-amd64/conf/security"
rm -f "$JAVA_SECURITY_DIR/java.security" "$JAVA_SECURITY_DIR/java.policy" "$JAVA_SECURITY_DIR/nss.cfg"
ln -sf /app/.apt/etc/java-17-openjdk/security/java.policy "$JAVA_SECURITY_DIR/java.policy"
ln -sf /app/.apt/etc/java-17-openjdk/security/java.security "$JAVA_SECURITY_DIR/java.security"
ln -sf /app/.apt/etc/java-17-openjdk/security/nss.cfg "$JAVA_SECURITY_DIR/nss.cfg"
```

### Problem 2: Hardcoded JAR Paths

**What happened:**

Ubuntu's pdftk wrapper script (`/usr/bin/pdftk.pdftk-java`) contains:
```sh
UBUNTUCP=/usr/share/java/bcprov.jar:/usr/share/java/commons-lang3.jar
```

But on Heroku, JARs are at `/app/.apt/usr/share/java/`

**The Fix:**

Create a custom wrapper in `.profile` with correct paths:
```bash
cat > /app/bin/pdftk << 'EOF'
#!/bin/sh
java -cp /app/.apt/usr/share/java/bcprov.jar:/app/.apt/usr/share/java/commons-lang3.jar:/app/.apt/usr/share/pdftk/pdftk.jar com.gitlab.pdftk_java.pdftk "$@"
EOF
```

---

## Complete Setup Instructions

### For a Laravel/PHP App

**1. Create Required Files**

In your Laravel app root, create:

`Aptfile`:
```
openjdk-17-jre-headless
pdftk-java
libbcprov-java
libcommons-lang3-java
```

`.profile`:
```bash
#!/bin/bash

# Fix Java security configuration symlinks
JAVA_SECURITY_DIR="/app/.apt/usr/lib/jvm/java-17-openjdk-amd64/conf/security"

if [ -d "$JAVA_SECURITY_DIR" ]; then
    echo "Fixing Java security configuration symlinks..."
    rm -f "$JAVA_SECURITY_DIR/java.policy" "$JAVA_SECURITY_DIR/java.security" "$JAVA_SECURITY_DIR/nss.cfg"
    ln -sf /app/.apt/etc/java-17-openjdk/security/java.policy "$JAVA_SECURITY_DIR/java.policy"
    ln -sf /app/.apt/etc/java-17-openjdk/security/java.security "$JAVA_SECURITY_DIR/java.security"
    ln -sf /app/.apt/etc/java-17-openjdk/security/nss.cfg "$JAVA_SECURITY_DIR/nss.cfg"
    echo "Java security configuration fixed."
fi

# Create pdftk wrapper with correct paths
mkdir -p /app/bin
cat > /app/bin/pdftk << 'EOF'
#!/bin/sh
java -cp /app/.apt/usr/share/java/bcprov.jar:/app/.apt/usr/share/java/commons-lang3.jar:/app/.apt/usr/share/pdftk/pdftk.jar com.gitlab.pdftk_java.pdftk "$@"
EOF
chmod +x /app/bin/pdftk

echo "pdftk-java configured and ready!"
```

**2. Configure Buildpacks**

```bash
heroku buildpacks:clear
heroku buildpacks:add heroku/nodejs  # if you use Node
heroku buildpacks:add heroku/php
heroku buildpacks:add --index 1 heroku-community/apt
```

**3. Commit and Deploy**

```bash
git add Aptfile .profile
git commit -m "Add pdftk-java support for Heroku-24"
git push heroku main
```

**4. Verify Installation**

```bash
heroku run bash
# Inside the dyno:
pdftk --version
# Should output: pdftk port to java 3.3.3 a Handy Tool for Manipulating PDF Documents
```

**5. Test PDF Operations**

```bash
# Test form filling
heroku run "pdftk template.pdf fill_form data.fdf output filled.pdf"
```

### Using in Laravel Code

Your existing Laravel code should work without changes:

```php
use mikehaertl\pdftk\Pdf;

$pdf = new Pdf('storage/template.pdf');
$pdf->fillForm([
    'name' => 'John Doe',
    'email' => 'john@example.com',
    'date' => date('Y-m-d')
])
->flatten()
->saveAs('storage/filled.pdf');
```

---

## Why pdftk-java?

### Actively Maintained (2025)
- Latest version: 3.3.3 (2024)
- Active development through October 2025
- Available in Ubuntu 24.04 repositories

### Performance
- **7-8x faster** than qpdf for batch operations
- Benchmark: 27s vs 201s (pdftk vs qpdf)

### Laravel Ecosystem
- Works with `mikehaertl/php-pdftk` package
- Drop-in replacement for original pdftk
- No code changes needed

### Built-in Form Filling
- ✅ pdftk-java: Native `fill_form` command
- ❌ qpdf: No form filling capability

### Comparison: pdftk-java vs Alternatives

| Feature | pdftk-java | qpdf | pdfcpu |
|---------|------------|------|--------|
| **Maintenance** | Active (2025) | Very Active | Active |
| **Performance** | Fast (27s) | Slow (201s) | Fast |
| **Form Filling** | ✅ Built-in | ❌ None | ❌ None |
| **Laravel Support** | ✅ php-pdftk | ❌ Custom | ❌ Custom |
| **CLI Compatibility** | ✅ Original pdftk | ❌ Different | ❌ Different |
| **Heroku-24** | ✅ Native package | ✅ Available | ⚠️ Manual install |

---

## Troubleshooting

### pdftk command not found

Check your buildpack order:
```bash
heroku buildpacks
# Should show apt buildpack FIRST
```

Verify Aptfile exists:
```bash
git ls-files Aptfile
```

### Java security file error

Make sure your `.profile` includes the Java security symlink fix (see above).

### Form filling fails

Test with a simple command:
```bash
heroku run "pdftk --version"
```

If version works but form filling doesn't, check:
1. Java security symlinks are fixed
2. Wrapper script uses correct JAR paths
3. Input PDF has fillable form fields

---

## Success Stories

This setup successfully powers production Laravel applications on Heroku-24, generating multi-page PDF packages with form filling, including:
- Immigration application forms
- Multi-page document packages (28+ pages)
- Automated PDF generation at scale

---

## Alternative Approaches

If this symlink-fixing approach seems too fragile for your use case, consider these alternatives:

### 1. Cloud PDF Services (Recommended for Long-term)
**Pros:**
- No infrastructure management
- Auto-scaling
- Professional support
- No Heroku stack dependency

**Options:**
- **AWS Lambda** with PDFtk layer
- **DocRaptor API** - PDF generation as a service
- **PDF.co API** - PDF manipulation API

**Cons:** Monthly costs, vendor lock-in

### 2. Revert to This Custom Buildpack
**Pros:**
- Simpler than symlink fixing
- Isolated from system changes
- Clear, single-purpose solution

**Cons:**
- Unmaintained (archived)
- May break on future stacks
- Extra buildpack to manage

**When to use:** If the symlink approach breaks and you need a quick fix

### 3. Docker/Container Deployment
**Pros:**
- Full control over environment
- No symlink hacks needed
- Install PDFtk normally

**Cons:**
- Requires Heroku plan upgrade
- More complex deployment
- Different workflow

**When to use:** If you're already containerizing or have complex dependencies

### Decision Matrix

| Scenario | Recommended Approach |
|----------|---------------------|
| **Small app, low volume** | Current symlink solution |
| **Enterprise, mission-critical** | Cloud PDF service |
| **Symlink solution breaks** | Revert to custom buildpack |
| **Already using Docker** | Container deployment |
| **Budget constraints** | Current symlink solution |

---

## Repository Status

This repository has evolved from a custom buildpack into comprehensive documentation. The custom buildpack code (in `bin/`) is now **obsolete and unused**. Use the standard buildpacks + configuration approach documented above.

For historical reference, the old buildpack approach is preserved in git history.

---

## Contributing

If you encounter issues or have improvements to the setup process, please open an issue or pull request. This guide benefits the entire Laravel/Heroku community!

---

## License

This documentation is provided as-is for the benefit of the community. The code examples are freely usable.
