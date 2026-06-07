# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) presets for HK Tech projects.

## Presets

| Preset | Extends as | What it adds |
|---|---|---|
| `default.json` | `github>hktech-eu/renovate-config` | Auto-merge minor/patch when CI is green; group `docker/*` GitHub Actions |
| `symfony.json` | `github>hktech-eu/renovate-config//symfony` | Group all `symfony/*` composer packages into one PR |

## Usage

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    "github>hktech-eu/renovate-config",
    "github>hktech-eu/renovate-config//symfony"
  ]
}
```

Add only the presets that apply to the project.

## PHP version sync

When Renovate bumps the `php` version in `composer.json`, three files must change in one PR:

| File | What changes |
|---|---|
| `composer.json` | PHP version bumped by Renovate |
| `composer.lock` | `content-hash` recomputed, `platform.php` updated |
| `Dockerfile` | Base image PHP suffix updated (e.g. `-php8.4.1` → `-php8.4.2`) |

This isn't a shareable preset because the file paths differ per project. Add this rule directly to the project's `renovate.json` and adjust the paths:

```json
{
  "description": "Sync composer.lock and Dockerfile when the php version is bumped",
  "matchManagers": ["composer"],
  "matchDepNames": ["php"],
  "postUpgradeTasks": {
    "commands": [
      "composer -d <php-dir> update --no-install --ignore-platform-reqs --no-scripts",
      "sed -i \"s/\\\"php\\\": \\\"[0-9][^\\\"]*\\\"/\\\"php\\\": \\\"{{{newValue}}}\\\"/\" <php-dir>/composer.lock",
      "sed -i \"s/-php[0-9][0-9.]*/-php{{{newValue}}}/g\" <dockerfile>"
    ],
    "fileFilters": ["<php-dir>/composer.lock", "<dockerfile>"]
  }
}
```

Also allowlist all three commands in `.github/workflows/renovate.yml`:

```yaml
RENOVATE_ALLOWED_COMMANDS: '[
  "^composer -d <php-dir> update --no-install --ignore-platform-reqs --no-scripts$",
  "^sed -i \"s/.+\" <php-dir>/composer\\.lock$",
  "^sed -i \"s/.+\" <dockerfile>$"
]'
```

Replace `<php-dir>` and `<dockerfile>` with the actual paths:

| Project structure | `<php-dir>` | `<dockerfile>` |
|---|---|---|
| `composer.json` at repo root | `.` (and drop `-d .`) | `Dockerfile` |
| PHP app under `app/` | `app` | `app/Dockerfile` |
| PHP app under `api/` | `api` | `api/Dockerfile` |

See [`docs/php-version-sync.md`](docs/php-version-sync.md) for the full explanation.
