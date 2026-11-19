# ThemeKraft release tooling

Shared release pipeline for ThemeKraft plugins and themes. Replaces `tk_script` (Robo).

A release is a git tag on the product repo:

- `x.y.z-beta.N`: builds the zip and deploys it to Freemius as a beta. GitHub prerelease.
- `x.y.z`: deploys to Freemius as released, then downloads the free build that Freemius generates and commits it to wordpress.org SVN (only when `wporg: true`). GitHub release with notes from Conventional Commits.

The wordpress.org build is always the free zip downloaded back from Freemius, never the repo contents, because Freemius strips the `__premium_only` code.

## What is in here

- `.github/workflows/release.yml`: reusable release workflow (`workflow_call`).
- `.github/workflows/ci.yml`: reusable CI: `php -l` on the oldest and newest PHP, `check-free-js` on the built zip, and WordPress Plugin Check on the built zip.
- `bin/build-zip`: builds `SLUG.zip` with `SLUG/` as root from a checkout, in `.release-build/` by default (so a plugin's own `build/` ships). Honors `.distignore`, runs `composer install --no-dev` in the copy (even when `.distignore` keeps `composer.json` out of the zip), never touches the source tree. Does not minify or rewrite code.
- `bin/freemius`: small client for the Freemius developer API (`tags`, `deploy`, `set-mode`, `download`). Python 3 standard library only.
- `bin/check-free-js`: simulates what Freemius strips from JavaScript for the free build (every `/* <fs_premium_only> */ ... /* </fs_premium_only> */` span, markers included, plus `__premium_only` files and `@fs_premium_only` header paths) and runs `node --check` on every `.js`/`.mjs`/`.cjs` file that is left (`check-free-js .release-build/SLUG`). It fails when a free version does not parse, and says whether the stripping caused it. Keep the markers around whole statements, never inside minified expressions: a minifier that joins statements (`sequences`, `join_vars`) can leave the free build with a syntax error even though the premium file parses (TKT-123).
- `bin/strip-requires-plugins`: removes slugs from the `Requires Plugins` header of a built plugin (`strip-requires-plugins .release-build/SLUG sfwd-lms`). The CI uses it, through the `plugin-check-skip-dependencies` input, on the copy Plugin Check runs on, because `wordpress/plugin-check-action` installs every required plugin from wordpress.org and stops at the first one that is not there (LearnDash `sfwd-lms`, WooCommerce Subscriptions). Never run it on a zip that gets released.
- `bin/gh-pc-results`: pulls the Plugin Check findings of a product's latest CI run into a JSON file (`gh-pc-results Themekraft/wc4bp ci/release-pipeline pc.json`).
- `bin/pcp-autofix`: applies the Plugin Check fixes that cannot change behavior, from that JSON, run in the product root: ABSPATH guards, `translators:` comments (named after the sprintf arguments, inner comments moved in front of the call) and the WordPress/PHP/Tested-up-to headers. It lists every other error as manual. Review the diff, especially any translator comment that says "value".
- `cliff.toml`: git-cliff config for the release notes.
- `templates/`: `release.yml`, `ci.yml` and `.distignore` to copy into a product repo.

## Adding a product

1. Copy `templates/release.yml` and `templates/ci.yml` to the product's `.github/workflows/`, and set `slug` (install slug, which is not always the repo name), `freemius-id` and `main-file`. Set `wporg: false` for products that are not on wordpress.org.
2. Copy `templates/.distignore` and adjust it to the product.
3. Remove the `.tk` submodule, `tk.sh`, `RoboFile.php` and `.semver`.
4. Open the PR, then run `gh-pc-results` and `pcp-autofix` on the CI findings. tk_script injected ABSPATH guards at build time, so the first run of a migrated product always reports them.
5. If the plugin compiles assets (for example `@wordpress/scripts` into `build/`), set `build-command` in both `release.yml` and `ci.yml` (`pnpm install --frozen-lockfile && pnpm run build`) and declare pnpm in `package.json` (`"packageManager": "pnpm@x.y.z"`); Node and pnpm are only installed when `build-command` is set. Add `.release-build/` to the product's `.gitignore`.
6. If errors that need real code changes remain (escaping, SQL, HTTP API, settings sanitizers), set `plugin-check-blocking: false` in the product's `ci.yml` and track them in Jira; remove the flag once they are fixed.

## Release cadence

Stable releases are batched. Every stable release shows an update notice on every site that runs the product and costs a full round of testing, approval and wp.org checks, so we prefer fewer releases that carry more changes.

- Urgent changes go out right away as a patch release: security fixes, fatal errors or regressions introduced by the previous release, data loss, broken checkout, payments, login or registration, and breakage caused by a new WordPress, WooCommerce or BuddyPress version.
- Everything else is merged into `develop`, added to the product's next GitHub milestone (named after the version, for example `3.6.5`) and released in the next release window.
- Release window: once a month per product. If the milestone is empty, that product skips the month.
- Betas are not limited: tag a beta whenever something lands in `develop` and test it on the debug site, so the stable at the end of the cycle is already tested. Freemius only offers betas to sites that opted into them.
- Security fixes never wait for the window. Most product repos are public, so a fix merged into `develop` discloses the vulnerability to anyone who reads the diff. Either release it as a patch the same day it is merged, or prepare it in private (for example in the private fork of a GitHub security advisory) and merge it on release day.

## Releasing

1. On `develop`, bump the `Version:` header (and the version constant, if the product has one) and set `Stable tag:` in `readme.txt` to the same version, betas included: Plugin Check fails with `stable_tag_mismatch` otherwise (wp.org only reads `Stable tag` from the stable release, so a beta value in `develop` never reaches it). For a stable release also add the changelog entry.
2. Merge `develop` into `main` through a PR, tag the release commit on `main` with the bare version (`2.3.0` or `2.3.0-beta.1`) and push the tag.
3. Stable releases wait in the `production` environment if it has required reviewers.

A stable release fails if `readme.txt` declares a `Tested up to` older than the current WordPress major (read from the wordpress.org API at release time). Test the release on the current WordPress and update the header first; betas are not checked.

If a later step fails (for example SVN), re-run the failed job: the Freemius deploy is idempotent and reuses an existing tag with the same version.

Large plugins can hit `svn: E175012: Connection timed out` right after "Transmitting file data". The commit often landed anyway (BuddyForms 2.10.0 did): before re-running, check `https://plugins.svn.wordpress.org/SLUG/tags/VERSION/readme.txt` and `trunk/readme.txt`. SVN commits are atomic, so if the tag is there it is complete.

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
