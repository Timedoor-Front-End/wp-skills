---
name: tmdr-theme-project
description: Use when working in this theme at all — project requirements, repository layout, functions.php load order, plugin prerequisites (ACF Pro, Polylang) and when to ask whether they are installed, whether the site is multilingual (Polylang) or not, and known intentional constraints. Consult this first for any task in the theme, even small edits.
---

# Project

Classic WordPress starter theme for Timedoor projects.

Requirements: WordPress, PHP 7.4+, ACF Pro. Polylang is **optional**, only for multilingual sites (see "Plugin prerequisites" and "Multilingual sites").

## Plugin prerequisites

Before generating code that depends on a plugin, confirm it is available. Do this once per task, not once per file, and only for plugins the task actually needs. Templates call plugin functions directly with no fallback layer, so a missing plugin fatals the front end.

| Plugin | Needed for | Required? |
| --- | --- | --- |
| ACF Pro | Any `get_field()` call, `acf-json/` groups, option pages, repeaters, flexible content, gallery, CPT/taxonomy JSON | Always |
| Polylang (+ Polylang Slug) | `pll_*` calls, language switcher | Multilingual sites only |
| WooCommerce | Shop, product, cart, checkout, My Account, `woocommerce/` template overrides (see `tmdr-woocommerce`) | Shop sites only |

**Detection order (stop at the first answer):**

1. The conversation or an earlier answer already states it.
2. Shell access with WP-CLI: `wp plugin list --status=active --field=name`.
3. Repository evidence: `acf-json/` contains real groups, or existing templates already call `get_field()` / `pll__()`. A TGMPA entry in `inc/functions-plugin.php` only means the theme *requests* the plugin, not that it is installed or active, so it is not proof.
4. Otherwise ask.

**How to ask:** one short question, before writing any code, combined with any other open questions. Offer yes / no / not sure. Example: "Is ACF Pro installed and active on this site? And is it multilingual (Polylang) or single-language?"

**If the answer is "no" or "not sure":**

- Say which plugin is missing and what will break (front-end fatal for ACF Pro, or for Polylang on multilingual sites).
- Ask the user to install and activate it themselves (WP Admin → Plugins, or the plugin notice the theme shows for plugins it registers). For "not sure", ask them to check WP Admin → Plugins first. ACF Pro is licensed and needs the user's own license key, so installation stays with them.
- Do not install the plugin yourself: no `wp plugin install`, no Composer, no downloading plugin files into the repo, and no editing `functions-plugin.php` or `wp-config.php` to force it.
- Wait for the user to confirm the plugin is installed and active before generating code that depends on it. In the meantime, offer work that does not depend on it (plain CSS, plain JS, static markup).
- Polylang only: the user may instead choose to proceed in single-language mode (see "Multilingual sites").
- Never add `function_exists()` fallback wrappers to work around a missing plugin.

## Repository layout

```text
inc/                    Theme setup, helpers, plugins, ACF, Polylang, auth
page-templates/         Optional page templates
template-parts/         Reusable PHP components
woocommerce/            WooCommerce template overrides (shop sites only; created per project)
assets/css/             Plain CSS (layout.css, pages/*.css, admin/*.css)
assets/js/              Plain JS (layout.js, pages/*.js)
assets/images/          Theme images
assets/dummy/intro.php  Demo content for the starter docs page (page.php)
acf-json/               ACF local JSON (only the index.php security stub in the starter; client groups are added per project)
```

`page.php` is the starter documentation page (it requires `assets/dummy/intro.php`). Replace it when real development begins. `show_debug_helper()` plus `template-parts/debug-console.php` is a dev-only debug panel; never leave a call to it in committed templates.

## Assets are plain CSS and JS

This project writes plain CSS and plain JS directly in `assets/css/` and `assets/js/`. There is no SCSS, no webpack, and no `source/` folder, so do not create or reference one, and do not add a build step.

- Add new styles as plain CSS files (one per page under `assets/css/pages/`) and new scripts as plain JS files (under `assets/js/pages/`).
- Scripts load with `defer` in the footer as classic scripts, so use no `import` / `export` and no bundler syntax.
- The starter's `assets/css/layout.css` and `assets/js/layout.js` were originally compiled output (the JS is a webpack bundle). Avoid large edits inside them. Put new page-specific code in its own file, and only make small, targeted changes to the layout files when the change must be global.

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
9. `functions-woocommerce.php` (shop sites only; created per project, see `tmdr-woocommerce`)

Main responsibilities:

- `functions-custom.php`: helpers (`components()`, `getCurrentLang()`, `getPageData()`, `show_debug_helper()`), admin/login hardening.
- `functions-plugin.php`: TGMPA plugin registration (what the theme requests, not what is installed).
- `functions-style-script.php`: CSS/JS enqueue helpers.
- `functions-acf.php`: ACF **option pages and their field groups** (PHP). Client page field groups are JSON in `acf-json/`, never here.
- `functions-polylang.php`: Polylang string registration, already wrapped in `if (function_exists('pll_register_string'))`, so it is harmless without the plugin.
- `functions-theme-setup.php`: theme setup, menus, logo, login branding, admin schemes.
- `functions-basic-auth.php`: staging Basic Auth and IP whitelist.
- `functions-woocommerce.php`: all custom WooCommerce PHP (theme support, hooks, checkout fields, pricing, emails, AJAX) in one file, on shop sites only.

Do not change load order without checking hook dependencies.

## Multilingual sites (Polylang is optional)

Decide this once per project, before writing templates, using the detection order in "Plugin prerequisites". Do not infer it from `functions-plugin.php` alone.

- **Site needs multiple languages:** Polylang must stay active. Call its functions directly in templates (`pll__()`, `pll_the_languages()`, `pll_home_url()`), with no wrapper layer. Deactivating it fatals the front end by design.
- **Site is single-language:** do not use any `pll_*` function in templates. Use plain text or `esc_html__()` / `__()` with the theme text domain. In `inc/functions-plugin.php`, remove the `Polylang` and `Polylang Slug` entries (the starter lists both with `'required' => false`). Leave `functions-polylang.php` as is, since it is already guarded.
- Never add `pll_*` calls "just in case". A single-language site with a stray `pll__()` call fatals when the plugin is absent.
- The starter already ships one language helper that works in both modes: `getCurrentLang()` (uses `pll_current_language()` when available, else `get_locale()`), used by `header.php` for `<html lang>`. Use it for the language code. Do not add other `function_exists()` fallback wrappers to make both modes work in one template; pick the mode per project.

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
- `acf-json/` holds client page field groups once a project adds them (the starter ships only the `index.php` stub); global option groups remain registered in PHP.
- Templates call ACF directly (`get_field()`), so ACF Pro must stay active; deactivating it fatals the front end (no fallback layer by design). Polylang follows the same rule on multilingual sites only.

## Frontend conventions

Adapted from the starter's own coding standards (`assets/dummy/intro.php`, header of `layout.js`) for a plain CSS/JS workflow:

- **HTML:** semantic elements (`section`, `header`, `footer`, `article`) over generic `div` wrappers, no inline styles, and consistent indentation matching the surrounding file.
- **Layout:** Bootstrap 5 grid (`container`, `row`, `col-*`) and components (offcanvas, modal, tab) as the starter already uses them.
- **CSS:** plain CSS (no SCSS syntax such as nesting with `&`, `@include`, or `$variables`; use CSS custom properties instead). BEM, `block__element--modifier` (e.g. `thankyou__subtitle`). One stylesheet per page template under `assets/css/pages/`, loaded only on that template.
- **JS:** plain classic scripts: small single-purpose functions, one focused file per feature, no `import` / `export`, and all initialization inside `DOMContentLoaded`.
- **PHP:** custom functions use the `tmdr_` prefix. The older helpers (`getCurrentLang()`, `getPageData()`) predate that rule; do not copy their naming.

## Working rules

1. Confirm required plugins per "Plugin prerequisites" before writing code that depends on them.
2. Read the relevant skill (`tmdr-page-template`, `tmdr-acf-field-group`, `tmdr-woocommerce`) before changing that area.
3. Search existing helpers, components, and CSS/JS before writing new code. The SCSS mixins and functions described in `assets/dummy/intro.php` (`color()`, `vwUnit()`, `typo()` and others) are not available in a plain-CSS project; write the equivalent CSS directly.
   Exception: never add thin wrapper helpers around native functions. Call WordPress and ACF directly in the template (`get_field()`, `wp_get_attachment_image()`, `wp_get_nav_menu_items()`, `get_posts()`, `sanitize_title()`, `wpautop()`, `wp_kses_post()`), and Polylang directly on multilingual sites (`pll__()`, `pll_the_languages()`). A helper in `inc/functions-custom.php` is only justified for site-specific logic shared by several templates (e.g. `tmdr_whatsapp_url()`, `tmdr_active_offer_ids()`, `tmdr_faq_groups()`).
4. Trace the complete caller flow for behavior changes.
5. Make the smallest change in the correct source file.
6. Follow the "Frontend conventions" section above, and match neighboring files.
7. Avoid new dependencies unless an installed dependency cannot solve the requirement.
8. Do not modify unrelated existing work.
9. Escape all output (see `tmdr-page-template`, "Escaping output").
10. Verify before considering a change complete:
    - PHP: `php -l path/to/file.php`
    - JSON: `jq empty path/to/file.json`
    - JS: `node --check path/to/file.js` (for plain scripts)
    - CSS: load the page and check the browser console, or run the project's linter if one exists.
