---
name: tmdr-page-template
description: Use when creating or editing WordPress page templates or reusable PHP components in this theme — even when the user just says "buat halaman baru," "tambah komponen," or describes a new page/section without naming a file. Covers components() usage, The*/Base* naming, template structure, image rendering, output escaping, optional Polylang strings, and CSS/JS enqueue wiring for new pages.
---

# Page templates and PHP components

Templates call ACF (and Polylang on multilingual sites) directly, so confirm the required plugins first, following `tmdr-theme-project` → "Plugin prerequisites". Do not write templates that call a plugin's functions until that plugin is confirmed available. If it is missing, ask the user to install and activate it themselves; never install it for them.

## Standard templates

At the theme root: `index.php`, `single.php`, `archive.php`, `search.php`, `page.php`, `404.php`, `comments.php`, `header.php`, `footer.php`.

Starter page templates in `page-templates/`:

- `blank.php` — blank starter template.
- `home.php` — home starter template.
- `thank-you.php` — thank-you page.

## PHP components

Reusable components live in `template-parts/components/` with PascalCase filenames. Call them through `components()`, never `get_template_part()` directly. `components()` standardizes how `$args` reach a template part; calling `get_template_part()` directly leaves each caller to invent its own way of passing data, which drifts over time.

```php
components('ThemeButton', array(
    'text' => 'Contact',
    'link' => home_url('/contact/'),
));
```

`components()` is defined in `inc/functions-custom.php` and is a thin wrapper: `get_template_part('/template-parts/components/' . $name, null, $args)`. So:

- The first argument is the filename without `.php`, resolved inside `template-parts/components/`.
- Inside the component, the data arrives as `$args`. Read each value with a default: `$text = $args['text'] ?? 'Click me!';`.
- The starter's `ThemeButton.php` echoes its values unescaped. Do not copy that: new components must escape (see "Escaping output").

Naming:

- `The*`: single-instance layout component, e.g. `TheNavbar`, `TheFooter`. The prefix signals there is exactly one on the page, so nobody reuses it as a repeatable block.
- `Base*`: reusable base component. None currently exists.
- Other reusable components: descriptive PascalCase, e.g. `ThemeButton`.

Existing components (this is the full list — do not call components that are not here unless you create them):

```text
template-parts/components/TheNavbar.php
template-parts/components/TheFooter.php
template-parts/components/ThemeButton.php
```

`TheNavbar.php` currently contains demo hardcoded menu content. Remove or replace it when implementing a real project, since shipping the demo menu to a client site would expose placeholder links and copy.

Registered menu locations: `main_menu` and `footer`. `TheNavbar` renders `main_menu` through `wp_nav_menu()` inside a Bootstrap offcanvas, then hardcoded demo items (Contact, a mega menu) that must be removed or replaced.

## Header and footer

`TheNavbar` and `TheFooter` hold demo content. When implementing or changing a real header or footer:

- Keep the markup in those two components (call them only from `header.php` / `footer.php`, as the starter does).
- Make every editable piece of content (logo, contact info, address, social links, copyright text, extra links, CTA button) come from ACF fields on **their own option pages**: one **Header** option page and one **Footer** option page, not General Settings and not a page template field group. Create them as described in `tmdr-acf-field-group` under "Header and footer option pages".
- Read the values with native ACF calls and the page's `post_id`, e.g. `get_field('cta_text', 'header')` and `get_field('copyright_text', 'footer')`, with inline defaults and escaping.
- Menus stay in WordPress menus (`main_menu` and `footer`, rendered with `wp_nav_menu()`), not in ACF repeaters.
- Remove the placeholder content (placehold.co logo, lorem ipsum mega menu, fake contacts, icons8 icons) rather than leaving it next to the real content.

## Images

Prefer `wp_get_attachment_image()` for WordPress attachment images. Use a raw `<img>` only for external, generated, or otherwise non-attachment sources. This keeps responsive `srcset` / `sizes` output instead of a flat `<img>`.

Pass `loading` explicitly: `'lazy'` by default, `'eager'` only for above-the-fold heroes. Components are rendered outside the normal content flow, and WordPress's automatic loading heuristics depend on version and context, so being explicit keeps behavior predictable. Check the result in the page source on the project's WordPress version.

**Alt text:** do not pass `'alt' => ''` for content images. When `alt` is omitted, `wp_get_attachment_image()` uses the attachment's own alt text. Pass `'alt' => ''` only for purely decorative images.

**ACF image fields** (`return_format: "array"`) may be empty, so branch inline:

```php
<?php
$image = get_field('message_image');
$image_id = is_array($image) ? ($image['ID'] ?? 0) : (int) $image;
?>

<?php if ($image_id) : ?>
    <?php echo wp_get_attachment_image($image_id, 'full', false, array(
        'class' => 'block__image',
        'loading' => 'lazy',
    )); ?>
<?php else : ?>
    <img class="block__image" src="<?php echo esc_url(get_theme_file_uri('assets/images/placeholder.jpg')); ?>" alt="" loading="lazy" decoding="async">
<?php endif; ?>
```

The placeholder must be a real file in `assets/images/` (use the design's placeholder when one exists). Do not use external placeholder services. `esc_url()` encodes the spaces some asset file names contain. A fallback placeholder is decorative, so empty `alt` is correct there.

## Escaping output

Escape everything at the point of output:

- `esc_html()` for plain text, `esc_attr()` for attribute values, `esc_url()` for URLs and `href` / `src`.
- `wp_kses_post()` for trusted-markup content such as WYSIWYG fields.
- Never echo `get_field()` or request data raw.

```php
<a class="block__link" href="<?php echo esc_url($args['link'] ?? ''); ?>">
    <?php echo esc_html($args['text'] ?? ''); ?>
</a>
```

## Text strings and multilingual sites

Polylang is optional. Before writing any user-facing string, determine whether the site is multilingual by following `tmdr-theme-project` → "Plugin prerequisites" and "Multilingual sites" (a TGMPA entry alone does not prove Polylang is installed or needed). Ask once if it cannot be inferred.

- **Multilingual site:** use Polylang directly: `pll__('Book now')` for translatable strings (register them in `inc/functions-polylang.php` with `pll_register_string()`), `pll_the_languages()` for the switcher in the navbar, `pll_home_url()` instead of `home_url('/')` where the home link must stay in the current language. Use the existing `getCurrentLang()` helper when you need the language code.
- **Single-language site:** write plain text or `esc_html__('Book now', 'theme-text-domain')`. Do not call any `pll_*` function.

Never mix the two in one project.

## New page template wiring

A new page template that needs its own stylesheet or script requires two edits, because the enqueue logic does not auto-discover new templates. Each new asset has to be wired in explicitly or it will not load.

1. Add the template to the enqueue condition in `inc/functions-style-script.php`. Open that file first, find the existing `is_page_template()` list and the `tmdr_print_css()` / JS calls, and extend them the same way existing templates do. The shape is:

   The helpers take a unique handle and a path relative to `assets/css/` or `assets/js/`. Versioning follows `style.css`, and scripts load deferred in the footer. This is the existing pattern for `home` and `thank-you`:

   ```php
   if (is_page_template('page-templates/about.php')) {
       tmdr_print_css('aboutCss', 'pages/about.css');   // assets/css/pages/about.css
       tmdr_print_js('aboutJs', 'pages/about.js');      // only if the page needs JS
   }
   ```

   Add the block inside `tmdr_script_enqueue()`. Handles must be unique across the theme. Use `is_singular('post_type')` or `is_archive()` for non-page-template screens (see the commented examples in the file). Create the plain CSS file (and plain JS file, if needed) at exactly that path under `assets/`. There is no build step. A page script runs as a classic deferred script, so use no `import` / `export`, and wrap initialization in `DOMContentLoaded`.

2. Give the template a `Template Name:` header comment, which is how WordPress makes it selectable in the page editor:

   ```php
   <?php
   /*
       Template Name: Theme - About Template
   */
   ```

   The starter names templates `Theme - <Name> Template`; follow that.

Standard section order: `get_header()`, the page's sections (components or inline markup), `get_footer()`. Each section is a `<section class="block">` wrapping a Bootstrap `container`, with BEM class names (`block__element--modifier`). Only add banner or related-content components if they exist in `template-parts/components/` or you create them in the same change.

Also create the template's ACF group as JSON under `acf-json/`, scoped to this template with a `page_template` location rule (see `tmdr-acf-field-group`).
