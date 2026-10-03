# ThemeKraft release tooling

Shared release pipeline for ThemeKraft plugins and themes. Replaces `tk_script` (Robo).

A release is a git tag on the product repo:

- `x.y.z-beta.N`: builds the zip and deploys it to Freemius as a beta. GitHub prerelease.
- `x.y.z`: deploys to Freemius as released, then downloads the free build that Freemius generates and commits it to wordpress.org SVN (only when `wporg: true`). GitHub release with notes from Conventional Commits.

The wordpress.org build is always the free zip downloaded back from Freemius, never the repo contents, because Freemius strips the `__premium_only` code.

## What is in here

- `.github/workflows/release.yml`: reusable release workflow (`workflow_call`).
- `.github/workflows/ci.yml`: reusable CI: `php -l` on the oldest and newest PHP, and WordPress Plugin Check on the built zip.
- `bin/build-zip`: builds `SLUG.zip` with `SLUG/` as root from a checkout. Honors `.distignore`, runs `composer install --no-dev` in the copy, never touches the source tree. Does not minify or rewrite code.
- `bin/freemius`: small client for the Freemius developer API (`tags`, `deploy`, `set-mode`, `download`). Python 3 standard library only.
- `cliff.toml`: git-cliff config for the release notes.
- `templates/`: `release.yml`, `ci.yml` and `.distignore` to copy into a product repo.

## Adding a product

1. Copy `templates/release.yml` and `templates/ci.yml` to the product's `.github/workflows/`, and set `slug` (install slug, which is not always the repo name), `freemius-id` and `main-file`. Set `wporg: false` for products that are not on wordpress.org.
2. Copy `templates/.distignore` and adjust it to the product.
3. Remove the `.tk` submodule, `tk.sh`, `RoboFile.php` and `.semver`.

## Releasing

1. On `develop`, bump the `Version:` header (and the version constant, if the product has one). For a stable release also set `Stable tag:` in `readme.txt` and add the changelog entry.
2. Merge `develop` into `main` through a PR, tag the release commit on `main` with the bare version (`2.3.0` or `2.3.0-beta.1`) and push the tag.
3. Stable releases wait in the `production` environment if it has required reviewers.

If a later step fails (for example SVN), re-run the failed job: the Freemius deploy is idempotent and reuses an existing tag with the same version.

## Secrets

Organization secrets in Themekraft, available to public repos: `FS_DEV_ID`, `FS_PUBLIC_KEY`, `FS_SECRET_KEY` (Freemius developer key), `SVN_USERNAME`, `SVN_PASSWORD` (wordpress.org). Private repos on the free GitHub plan cannot read organization secrets, so they need the same secrets at repo level. `CHECKOUT_TOKEN` is only needed while a submodule is private.

## Freemius CLI locally

```
export FS_DEV_ID=342
# op-sa = 1Password CLI as the service account for the Jaime vault (no approval prompt)
export FS_PUBLIC_KEY=$(op-sa read "op://Jaime/Freemius Developer API Key/username")
export FS_SECRET_KEY=$(op-sa read "op://Jaime/Freemius Developer API Key/credential")
bin/freemius tags 426
bin/freemius download 426 --version 2.2.14 --out free.zip
```
