# Publishing a Scala Library to Maven Central with sbt-ci-release

Publishing a Scala library to Maven Central can seem daunting, but with `sbt-ci-release` and GitHub Actions, the process
becomes remarkably straightforward. This guide walks you through the complete setup, from configuring your build to
watching your library appear on Maven Central.

## Overview

We'll cover:

1. Setting up `sbt-ci-release` in your project
2. Configuring `build.sbt` for publishing
3. Creating a Sonatype Central Portal account
4. Generating and publishing GPG keys
5. Setting up GitHub Actions for automated releases

## Step 1: Add sbt-ci-release Plugin

Add the plugin to your `project/plugins.sbt`:

```scala
addSbtPlugin("com.github.sbt" % "sbt-ci-release" % "1.11.2")
```

> **Note:** Version 1.11.0+ uses the new Sonatype Central Portal instead of the legacy OSSRH. The Central Portal does *
*not** support SNAPSHOT publishing.

## Step 2: Configure build.sbt

Add the required metadata to your `build.sbt`:

```scala
inThisBuild(
  List(
    organization := "io.github.yourusername",
    homepage := Some(url("https://github.com/yourusername/your-project")),
    licenses := List("Apache-2.0" -> url("http://www.apache.org/licenses/LICENSE-2.0")),
    developers := List(
      Developer(
        "yourusername",
        "Your Name",
        "your@email.com",
        url("https://github.com/yourusername")
      )
    ),
    scmInfo := Some(
      ScmInfo(
        url("https://github.com/yourusername/your-project"),
        "scm:git:git@github.com:yourusername/your-project.git"
      )
    )
  )
)
```

**Important notes:**

- For GitHub-based namespaces, use `io.github.yourusername` as your organization
- The `sonatypeProfileName` setting is no longer needed with sbt-ci-release 1.11+
- All metadata fields are required for Maven Central validation

## Step 3: Set Up Sonatype Central Portal

1. Go to [https://central.sonatype.com](https://central.sonatype.com)
2. Sign in with your GitHub account
3. Navigate to **Namespaces** and verify your namespace
    - For `io.github.yourusername`, verification is automatic via GitHub
4. Go to **Account** → **Generate User Token**
5. Save the generated username and password (these are NOT your login credentials)

## Step 4: Generate and Publish GPG Keys

### Generate a new GPG key

```bash
gpg --full-generate-key
```

Choose:

- RSA and RSA
- 4096 bits
- No expiration (or your preferred expiration)
- Your name and email

### Find your key ID

```bash
gpg --list-secret-keys --keyid-format long
```

Output looks like:

```
sec   rsa4096/ABCD1234EFGH5678 2024-01-01 [SC]
      1234567890ABCDEF1234567890ABCDEF12345678
uid           [ultimate] Your Name <your@email.com>
```

Your key ID is `ABCD1234EFGH5678`.

### Publish your public key to key servers

**This step is crucial!** Sonatype Central Portal validates signatures against public key servers:

```bash
gpg --keyserver keyserver.ubuntu.com --send-keys YOUR_KEY_ID
gpg --keyserver keys.openpgp.org --send-keys YOUR_KEY_ID
```

Wait a few minutes for propagation before attempting to publish.

### Export your private key for CI

```bash
gpg --armor --export-secret-keys YOUR_KEY_ID | base64 | pbcopy
```

This copies the base64-encoded private key to your clipboard (macOS). On Linux, use `xclip` instead of `pbcopy`.

## Step 5: Configure GitHub Secrets

In your GitHub repository, go to **Settings** → **Secrets and variables** → **Actions** and add:

| Secret Name         | Value                                               |
|---------------------|-----------------------------------------------------|
| `SONATYPE_USERNAME` | Token username from Central Portal (not your email) |
| `SONATYPE_PASSWORD` | Token password from Central Portal                  |
| `PGP_SECRET`        | Base64-encoded GPG private key                      |
| `PGP_PASSPHRASE`    | Your GPG key passphrase                             |

## Step 6: Create GitHub Actions Workflow

Create `.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    tags: [ "v*" ]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: sbt/setup-sbt@v1
      - uses: olafurpg/setup-gpg@v3
      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/coursier/v1
          key: ${{ runner.os }}-coursier-${{ hashFiles('build.sbt') }}
          restore-keys: ${{ runner.os }}-coursier-
      - name: Publish
        run: sbt +ci-release
        env:
          PGP_PASSPHRASE: ${{ secrets.PGP_PASSPHRASE }}
          PGP_SECRET: ${{ secrets.PGP_SECRET }}
          SONATYPE_PASSWORD: ${{ secrets.SONATYPE_PASSWORD }}
          SONATYPE_USERNAME: ${{ secrets.SONATYPE_USERNAME }}
```

**Key points:**

- Triggers only on version tags (`v*`)
- Uses Java 17 (recommended for Scala 3)
- The `+ci-release` command cross-compiles and publishes for all Scala versions

## Step 7: Publish a Release

Commit your changes and create a version tag:

```bash
git add .
git commit -m "Prepare for release"
git push origin main

git tag v1.0.0
git push origin v1.0.0
```

The GitHub Action will automatically:

1. Compile for all Scala versions
2. Generate sources and javadoc JARs
3. Sign all artifacts with GPG
4. Upload to Sonatype Central Portal
5. Publish to Maven Central

## Troubleshooting

### 403 Error on SNAPSHOT publish

Sonatype Central Portal doesn't support SNAPSHOTs. Ensure your workflow only triggers on tags:

```yaml
on:
  push:
    tags: [ "v*" ]
```

### "Could not find public key" error

Your GPG public key isn't on the key servers. Upload it:

```bash
gpg --keyserver keyserver.ubuntu.com --send-keys YOUR_KEY_ID
```

Wait a few minutes and retry.

### "Bad passphrase" error

1. Verify your `PGP_PASSPHRASE` secret has no extra whitespace
2. Ensure `PGP_SECRET` contains the correct key:
   ```bash
   # Verify passphrase works locally
   echo "test" | gpg --clearsign --local-user YOUR_KEY_ID
   ```

### Version shows as 0.0.0+1-abc123-SNAPSHOT

This happens when there's no git tag. The version is derived from git tags by `sbt-dynver`. Create a proper version tag:

```bash
git tag v1.0.0
git push origin v1.0.0
```

## Conclusion

With `sbt-ci-release` and GitHub Actions, publishing to Maven Central becomes a simple `git tag` and `git push`. The
plugin handles all the complexity of signing, bundling, and uploading artifacts.

Your users can now add your library with:

```scala
"io.github.yourusername" %% "your-library" % "1.0.0"
```

Happy publishing!
