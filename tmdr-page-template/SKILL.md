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

**Page content never goes in `template-parts/`.** The sections of a page (hero, intro, gallery, and so on) are written inline in the page template file itself, in `page-templates/<page>.php`. Do not create `template-parts/home-content.php`, `template-parts/<page>-*.php`, per-page or per-section partials, or call `get_template_part()` / `components()` to split one page into pieces. A long template file is expected and fine.

Create a component only when **both** are true: the same markup is used in two or more places (on different pages, or repeated across templates), or it is a single-instance layout part (`TheNavbar`, `TheFooter`); and the user's design or request actually needs it. A section that appears once on one page is not a component, even if it is large. If unsure, keep it inline.

Reusable components live in `template-parts/components/` with PascalCase filenames, and that is the only place in `template-parts/` where new files are created. Call them through `components()`, never `get_template_part()` directly. `components()` standardizes how `$args` reach a template part; calling `get_template_part()` directly leaves each caller to invent its own way of passing data, which drifts over time.

```php
components('ThemeButton', array(
    'text' => 'Contact',
    'link' => home_url('/contact/'),
));
```

`components()` is defined in `inc/functions-custom.php` and is a thin wrapper: `get_template_part('/template-parts/components/' . $name, null, $args)`. So:

- The first argument is the filename without `.php`, resolved inside `template-parts/components/`.
- Inside the component, the data arrives as `$args`. Read each value without a fallback text: `$text = $args['text'] ?? '';`, and render the element only when the value is not empty (see `tmdr-acf-field-group` → "Empty values render nothing").
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
- Read the values with native ACF calls and the page's `post_id`, e.g. `get_field('cta_text', 'header')` and `get_field('copyright_text', 'footer')`, without default values, rendering each piece only when it is not empty, and with escaping.
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
<?php endif; ?>
```

When the image field is empty, render nothing: no placeholder image and no fallback markup. Do not use external placeholder services.

## Escaping output

Escape everything at the point of output:

- `esc_html()` for plain text, `esc_attr()` for attribute values, `esc_url()` for URLs and `href` / `src`.
- `wp_kses_post()` for trusted-markup content such as WYSIWYG fields.
- Never echo `get_field()` or request data raw.

```php
<?php if (!empty($args['link']) && !empty($args['text'])) : ?>
    <a class="block__link" href="<?php echo esc_url($args['link']); ?>">
        <?php echo esc_html($args['text']); ?>
    </a>
<?php endif; ?>
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

Standard section order: `get_header()`, the page's sections as inline markup in the template file (use a component only where it meets the rules in "PHP components"), `get_footer()`. Each section is a `<section class="block">` wrapping a Bootstrap `container`, with BEM class names (`block__element--modifier`). Only add banner or related-content components if they exist in `template-parts/components/` or you create them in the same change. When a section reads ACF fields, define its variables in a `<?php ... ?>` block directly above that section, never at the top of the template file (see `tmdr-acf-field-group` → "Empty values render nothing").

The template's ACF group is **not** created in this step. It is created in the ACF step as JSON under `acf-json/`, scoped to this template with a `page_template` location rule (see `tmdr-acf-field-group`). Until then the template holds the design's static content, and calls no plugin functions.

Only **page-specific** assets go inside the `is_page_template()` block. Fonts, `:root` variables, header/footer styles, and anything else global go in `global.css` / `global.js`, registered outside every condition (see `tmdr-theme-project` → "Global or page-specific?").

## Building from a design reference

Applies when the user points to a design or reference folder (for example `design/index.html`). That folder is **read-only input**: never edit, move, or delete anything in it, never link to it from the theme (no `<link>` or `<script>` pointing at the design folder), and never commit its images (`tmdr-theme-project` → "Starting a new project" adds the `.gitignore` rule). Everything the theme needs is **copied** into the theme's own folders.

0. **Starter files:** `home.php`, `TheNavbar`, `TheFooter`, and `ThemeButton` are demo code. Whether to modify them is decided by the question in `tmdr-theme-project` → "Starter code is a template, not a finished project". Do not rewrite them before that question is answered.
1. **Read and classify first.** Read the design's HTML, CSS, and JS before writing markup. Run every rule and script through the test in `tmdr-theme-project` → "Global or page-specific?" before copying it. Fonts, `:root` variables, resets, and base typography are global, even when they appear at the top of the design's page CSS.
2. **Page-specific CSS: start from the design's CSS, do not rewrite from scratch.** Copy the page's own rules into `assets/css/pages/<page>.css` and adapt them to the theme:
   - Plain CSS only: flatten SCSS nesting, turn `$variables` into CSS custom properties, drop mixins.
   - Rename classes to BEM (`block__element--modifier`) and match the class names in the template markup.
   - Drop unused rules, and rules that duplicate Bootstrap 5 or `layout.css`. Search `layout.css` before copying a rule.
   - Rewrite `url()` paths for the new location: from `assets/css/pages/`, images are at `../../images/<file>`.
3. **Global CSS and JS:** put them in `assets/css/global.css` and `assets/js/global.js`, wired as described in `tmdr-theme-project` → "Global or page-specific?". Do not edit `layout.css` or `layout.js`, and do not leave `@font-face`, `:root`, or `html`/`body` rules in the page file.
   - **Fonts:** copy the font files the design uses to `assets/fonts/` and declare `@font-face` in `global.css` with paths like `../fonts/<file>`. If the design loads a Google Fonts stylesheet instead, enqueue it globally, never from the page's `is_page_template()` block and never as a `<link>` copied into the page template.
   - Header and footer styles and scripts belong here too (in the header and footer step).
4. **Assets:** copy only the images, fonts, and icons the design actually uses, not the whole folder. Images go to `assets/images/`, fonts to `assets/fonts/` (create it if missing). Copied files are theme assets and **are** tracked in git; only the originals in the reference folder are ignored. Relative paths: from `global.css` use `../images/` and `../fonts/`; from `pages/<page>.css` use `../../images/`.
   - In the integration step, photos and banners stay static `<img>` markup pointing at the copied files. In the ACF step they become image fields and the static markup is replaced, with no placeholder fallback (see "Images").
   - Static or decorative images (logo, icons, background patterns) are referenced directly from CSS or with `get_theme_file_uri()`.
5. **Markup:** convert the design's HTML into the template structure below, keeping the design's own text and images as static content. Do not wire ACF yet: that is the next step (`tmdr-theme-project` → "Build steps and suggesting the next step"), where static content becomes fields and every output is escaped.
6. **JS:** page-only logic goes to `assets/js/pages/<page>.js`, global logic to `assets/js/global.js`, both as plain classic scripts (no `import` / `export`, init in `DOMContentLoaded`). Do not copy design CDN links or the design's own copy of Bootstrap.
7. **Before finishing the step,** re-scan the page CSS and JS for global leftovers and move them (see the check in `tmdr-theme-project`).