---
name: tmdr-page-template
description: Use when creating or editing WordPress page templates or reusable PHP components in this theme — even when the user just says "buat halaman baru," "tambah komponen," or describes a new page/section without naming a file. Covers components() usage, The*/Base* naming, template structure, and CSS/JS enqueue wiring for new pages.
---

# Page templates and PHP components — global

## Standard templates

At the theme root: `index.php`, `single.php`, `archive.php`, `search.php`, `page.php`, `404.php`, `comments.php`, `header.php`, `footer.php`.

Starter page templates in `page-templates/`:

- `blank.php` — blank starter template.
- `home.php` — home starter template.
- `thank-you.php` — thank-you page.

## PHP components

Reusable components live in `template-parts/components/` with PascalCase filenames. Call them through `components()` — never `get_template_part()` directly. `components()` standardizes how `$args` get passed into a template part; calling `get_template_part()` directly bypasses that and leaves each caller to reinvent how it hands data to the part, which drifts inconsistent across the codebase over time.

Prefer `wp_get_attachment_image()` for WordPress attachment images; use a raw `<img>` only for external, generated, or otherwise non-attachment sources. This keeps images benefiting from WordPress's responsive `srcset`/`sizes` output instead of a flat `<img>` that ignores registered image sizes.

```php
components('ThemeButton', array(
    'text' => 'Contact',
    'link' => home_url('/contact/'),
));
```

`components()` is defined in `inc/functions-custom.php` and passes `$args` to the component.

Naming:

- `The*`: single-instance layout component, e.g. `TheNavbar`, `TheFooter`. The prefix signals at a glance that there's exactly one of these on the page, so no one accidentally reuses it as a repeatable block.
- `Base*`: reusable base component. None currently exists.
- Other reusable components: descriptive PascalCase, e.g. `ThemeButton`.

Existing components:

```text
template-parts/components/TheNavbar.php
template-parts/components/TheFooter.php
template-parts/components/ThemeButton.php
```

`TheNavbar.php` currently contains demo hardcoded menu content. Remove or replace it when implementing a real project, since shipping the demo menu to a client site would expose placeholder links and copy.

Registered menu locations: `main_menu` and `footer`.

## New page template wiring

A new page template that needs its own stylesheet or script requires two edits, because the enqueue logic doesn't auto-discover new templates — each new asset has to be wired in explicitly or it simply won't load:

1. Add the template to the enqueue condition in `inc/functions-style-script.php` (`is_page_template()` list, `tmdr_print_css()` / JS equivalent).
2. Give the template a `Template Name:` header comment, which is how WordPress makes the template selectable in the page editor:

```php
<?php
/*
    Template Name: Theme - Company History
*/
```

Standard section order: `get_header()`, `PageBanner` component, page sections, `CompanyRelated` (company pages), `get_footer()`.
