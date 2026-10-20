---
name: tmdr-theme-project
description: Use when working in this theme at all — project requirements, repository layout, functions.php load order, and known intentional constraints.
---

# Project — global

Classic WordPress starter theme for Timedoor projects.

Requirements: WordPress, PHP 7.4+, ACF Pro.

## Repository layout

```text
inc/                    Theme setup, helpers, plugins, ACF, Polylang, auth
page-templates/         Optional page templates
template-parts/         Reusable PHP components
assets/css/             CSS
assets/js/              JavaScript
assets/images/          Theme images
acf-json/               ACF local JSON
```

## Runtime flow

`functions.php` loads `inc/` in this order:

1. `class-tgm-plugin-activation.php`
2. `functions-custom.php`
3. `functions-plugin.php`
4. `functions-style-script.php`
5. `functions-acf.php`
6. `functions-polylang.php`
7. `functions-theme-setup.php`
8. `functions-basic-auth.php`

Main responsibilities:

- `functions-custom.php`: helpers, admin/login hardening.
- `functions-plugin.php`: TGMPA plugin registration.
- `functions-style-script.php`: CSS/JS enqueue helpers.
- `functions-acf.php`: ACF option pages and field groups.
- `functions-polylang.php`: Polylang hooks.
- `functions-theme-setup.php`: theme setup, menus, logo, login branding, admin schemes.
- `functions-basic-auth.php`: staging Basic Auth and IP whitelist.

Do not change load order without checking hook dependencies.

## Basic Auth

Runs only while WordPress Search Engine Visibility is disabled (`blog_public = 0`). Credentials belong in `wp-config.php` or environment variables:

```php
define('TMDR_BASIC_AUTH_USER', 'your-user');
define('TMDR_BASIC_AUTH_PASS', 'your-password');
```

The implementation intentionally uses `REMOTE_ADDR`; do not trust forwarded headers without a trusted proxy configuration.

## Known intentional constraints

- WordPress intermediate image sizes are disabled by the theme; uploads may remain full-size.
- `lazysizes` and `gsap` are installed but not currently imported.
- `acf-json/` contains synced page field groups; global option groups remain registered in PHP.
- Templates call ACF and Polylang functions directly (`get_field()`, `pll__()`), so both plugins must stay active; deactivating either fatals the front end (no fallback layer by design).

## Working rules

1. Read the relevant skill (`tmdr-page-template`, `tmdr-acf-field-group`) before changing that area.
2. Search existing helpers, components, mixins, and imports before writing new code.
   Exception: never add thin wrapper helpers around native functions. Call WordPress, ACF and Polylang directly in the template (`get_field()`, `wp_get_attachment_image()`, `pll__()`, `pll_the_languages()`, `wp_get_nav_menu_items()`, `get_posts()`, `sanitize_title()`, `wpautop()`, `wp_kses_post()`). A helper in `inc/functions-custom.php` is only justified for site-specific logic shared by several templates (e.g. `tmdr_whatsapp_url()`, `tmdr_active_offer_ids()`, `tmdr_faq_groups()`).
3. Trace the complete caller flow for behavior changes.
4. Make the smallest change in the correct source file.
5. Preserve naming, BEM structure, and JS initialization format.
6. Avoid new dependencies unless an installed dependency cannot solve the requirement.
7. Do not modify unrelated existing work.
8. Verify affected PHP/CSS/JS syntax before considering a change complete.
