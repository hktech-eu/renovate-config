# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) presets for HK Tech projects.

## Presets

| Preset | Extends as | What it adds |
|---|---|---|
| `default.json` | `github>hktech-eu/renovate-config` | Auto-merge minor/patch when CI is green; group `docker/*` GitHub Actions |
| `symfony.json` | `github>hktech-eu/renovate-config//symfony` | Group all `symfony/*` composer packages into one PR |
| `php-version-sync.json` | `github>hktech-eu/renovate-config//php-version-sync` | Bump the FrankenPHP image tag (FrankenPHP + PHP patch) as one PR; disable stand-alone composer `php` bumps |

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

The FrankenPHP image tag is the PHP source of truth. The preset makes Renovate bump the
whole `<frankenphp>-php<php>` tag as one update and stops stand-alone composer `php`
bumps. The project must sync `composer.json` and `composer.lock` from the new tag with
a `postUpgradeTasks` rule, because the paths differ per project.

See [`docs/php-version-sync.md`](docs/php-version-sync.md) for the rule to copy and the
command allowlist.
