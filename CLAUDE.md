# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Drupal 11 site ("gonzi") hosted on Pantheon, built from Pantheon's `drupal-composer-managed` upstream. It uses Pantheon's **Integrated Composer**: Drupal core, contrib modules/themes, and `vendor/` are *not* committed. Pantheon runs `composer install` on deploy (`build_step: true` in `pantheon.upstream.yml`). There's no custom code yet: no custom modules, themes, or tests.

## Commands

```bash
composer install                       # build core, contrib and vendor locally
composer require drupal/<module>       # add a contrib module (lands in web/modules/contrib)
composer update                        # also runs the ComposerScripts pre-update hook (see below)
vendor/bin/drush <cmd>                 # Drush 13 (e.g. cr, cim, cex, updb)
vendor/bin/phpunit -c web/core <path>  # run tests (drupal/core-dev is in require-dev)
vendor/bin/phpcs --standard=Drupal,DrupalPractice <path>
```

## Layout and how the pieces fit

- `web/` is the docroot (`web_docroot: true`). Only `web/sites/default/settings.php` is tracked. It includes `settings.pantheon.php` (scaffolded by `pantheon-systems/drupal-integrations`) and then an untracked `settings.local.php` if one exists. Use `settings.local.php` for local overrides.
- Custom code goes in `web/modules/custom`, `web/themes/custom`, and `web/profiles/custom`. Contrib paths are gitignored and installed by Composer (see `extra.installer-paths` in `composer.json`).
- `config/` is the config sync directory (`drush cex` / `drush cim`).
- `recipes/` is gitignored except for its `.gitignore`. Composer installs Drupal recipes there.
- `upstream-configuration/` is a Composer path package (`pantheon-upstreams/upstream-configuration`). It holds `scripts/ComposerScripts.php`, which runs on `composer update`. That script rewrites the root `composer.json`: it syncs `config.platform.php` to the `php_version` in `pantheon.yml`/`pantheon.upstream.yml`, and it re-adds the plugins and hooks it requires. Expect `composer.json` diffs after an update, and change the PHP version in the Pantheon YAML rather than only in `composer.json`.
- `pantheon.upstream.yml` belongs to the upstream. Put site-specific Pantheon overrides (PHP version, protected paths, etc.) in `pantheon.yml`.
- Patches use `cweagans/composer-patches` (`enable-patching: true`, core patch level `-p2`, `composer-exit-on-patch-failure: true`).

## Git remotes and deployment

GitHub is the source of truth. The remotes are:

- `origin`: GitHub (`mariohernandez/gonzi`). Branches, PRs, and pushes go here.
- `pantheon`: Pantheon's Git repo for the site. Don't push to it by hand. Its `master` branch feeds the Dev environment.

`.github/workflows/pantheon-deploy.yml` runs on every push to `main`, and can also be started by hand. It uses `pantheon-systems/push-to-pantheon` with `skip_build_tools: true`, so it does a plain `git push --force` of `main` to Pantheon's `master` branch, with no Build Tools plugin and no database or files cloning.

- Commits made only on Pantheon (dashboard or SFTP) get overwritten by the next deploy. Make code changes in GitHub.
- Pantheon Dev must be in Git mode, not SFTP mode, or the push fails.
- The workflow needs the repo secrets `PANTHEON_PRIVATE_SSH_KEY` and `PANTHEON_MACHINE_TOKEN`. Both must belong to the Pantheon account that owns the site (`mario@mariohernandez.io`), not another account. Use an RSA key (`ssh-keygen -t rsa -b 4096 -m PEM`); Pantheon doesn't accept ed25519.
- Don't set `fetch-depth: 0` on the checkout. The action runs `git fetch --unshallow` itself (Pantheon rejects shallow pushes), and that command fails on a full clone.
- `push-to-pantheon` reports success even when its push fails (bad token, SSH key not on the Pantheon account). The last step, "Verify Pantheon received the commit", fails the job unless Pantheon's `master` matches the deployed commit.
- The job needs `permissions: contents: read, deployments: write`. The action checks for these and fails without them.
- Actions are pinned to full commit SHAs with the version in a trailing comment (`@<sha> # v7.0.1`). Keep that format; Dependabot (`.github/dependabot.yml`) opens weekly PRs to bump them.
- Pushing changes under `.github/workflows/` requires a GitHub token with the `workflow` scope (`gh auth refresh -h github.com -s workflow`).
