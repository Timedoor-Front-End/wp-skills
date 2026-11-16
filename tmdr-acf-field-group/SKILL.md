---
name: tmdr-acf-field-group
description: Use when creating or editing ACF field groups, fields, tabs, repeaters, flexible content, or option pages for this theme. Also use whenever the user asks to "add a field," "add a repeater," "set up custom fields," "build an options page," or wire up any admin-editable content — even if they don't say "ACF" explicitly. Covers acf-json/ local JSON conventions, key generation, location rules, a JSON skeleton, English-only editor text, deciding repeater vs fixed fields, empty-value rendering, and template reading patterns.
---

# ACF JSON integration

**ACF Pro is required.** Repeaters, flexible content, galleries, option pages, and CPT/taxonomy JSON are Pro features, and templates fatal without the plugin. Before writing any group, confirm ACF Pro is installed and active, following `tmdr-theme-project` → "Plugin prerequisites" (infer from the conversation, WP-CLI, or the repo; otherwise ask once). If it is missing, stop, ask the user to install and activate it themselves (do not install it for them), and wait for their confirmation before generating field groups.

Client ACF field groups are stored as JSON files in `acf-json/`. This keeps field configuration under version control and reviewable in diffs, instead of living only in the database.

**Where things live:**

| What | Where |
| --- | --- |
| Client page/template field groups | JSON in `acf-json/` |
| Option pages and their field groups | PHP in `inc/functions-acf.php` |
| CPTs / taxonomies | `post_type_*.json` / `taxonomy_*.json` (ACF Pro) |

## Rules

- Create or edit client field groups as JSON only, so `acf-json/` stays the single source of truth and changes are reviewable like code.
- Do not register client field groups in `inc/functions-acf.php`. Only option pages (and their groups) go there. Mixing the two for client content makes it unclear which one wins.
- ACF loads `acf-json/` directly, and field groups are not edited in WP Admin. Do not write code that imports or syncs field groups into the database, and do not tell the user to sync them in WP Admin.
- Do not modify `acf-json/index.php`. It is a security stub that blocks directory listing, not project configuration.
- Use one field-group file per page template when their field structures differ. This keeps each template's fields scoped and easy to find.
- Name the file after the group key: `group_<13-hex>.json`. ACF itself writes files by key, so a mismatched filename leads to a duplicate file the next time the group is saved from wp-admin.
- Use unique export-style keys: `group_<13-hex>`, `field_<13-hex>`, `post_type_<13-hex>`, `taxonomy_<13-hex>`. Never use a section, template, CPT, taxonomy, or client name as a key. Keys are stable identifiers so ACF can track a field across renames.
- Keep field names semantic `snake_case`. Names (unlike keys) are what templates reference via `get_field('field_name')`.
- Write every editor-facing text in **English**: group titles, tab labels, field labels, `instructions`, `placeholder`, `button_label`, choice labels, and option page titles. This holds even when the user writes in Indonesian or the design's copy is in Indonesian. Do not put the design's copy into labels. Field names and keys are English `snake_case`.
- Give fields short English `instructions` when they help the editor: where the value appears, recommended image size or ratio, or a length limit. One plain sentence, no jargon. Skip instructions that only repeat the label. Never add boilerplate such as "Leave empty to hide it" to instructions: that is how every field behaves, so it is not worth repeating.
- Never set a `default_value`: leave it empty on every field. An empty field must render nothing (see "Empty values render nothing"). Keep `required` at `0` unless the client explicitly asks for a required field.
- Use one `type: "tab"` field per UI section (e.g. Hero, Intro, Gallery), with `placement: "left"`. A tab mirrors a section, and a section normally contains many fields: put all of that section's fields after its tab, until the next tab. The number of fields per tab is not limited, and a tab is never tied to a single field. Do not split one section across multiple tabs.
- Use a specific `page_template` location rule (e.g. `page-templates/home.php`), not a blanket "Post Type = Page" rule, which would show unrelated fields on every page.
- Page field groups use `hide_on_screen: ["the_content"]`, since these pages are built from structured fields. Leaving the editor visible invites content the template never renders.
- Preserve complete ACF export configuration, including `sub_fields`, `layouts`, `min`, `max`, `button_label`, return formats, `mime_types`, conditional logic, and wrappers. Dropping these during an edit silently changes validation and admin UI behavior.
- Update the group's top-level `"modified"` Unix timestamp whenever you edit a field-group JSON. Use `date -u +%s`, not a made-up number.

## Generating keys

Keys must be exactly 13 lowercase hex characters after the prefix. Generate them instead of typing them:

```bash
echo "group_$(openssl rand -hex 7 | cut -c1-13)"
echo "field_$(openssl rand -hex 7 | cut -c1-13)"
```

Generate a fresh key for every group and every field. Never copy keys from the skeleton below or from another group.

## JSON skeleton

A minimal page group with one tab (the "Hero" section) holding two fields. A tab is a section divider, not a container for a single field: every field after a tab belongs to it until the next tab. A real group has several tabs and many fields. The keys and timestamp are placeholders — generate real ones as shown above.

```json
{
    "key": "group_0123456789abc",
    "title": "Home",
    "fields": [
        {
            "key": "field_0123456789abd",
            "label": "Hero",
            "name": "",
            "aria-label": "",
            "type": "tab",
            "instructions": "",
            "required": 0,
            "conditional_logic": 0,
            "wrapper": { "width": "", "class": "", "id": "" },
            "placement": "left",
            "endpoint": 0
        },
        {
            "key": "field_0123456789abe",
            "label": "Hero Title",
            "name": "hero_title",
            "aria-label": "",
            "type": "text",
            "instructions": "Main heading of the page.",
            "required": 0,
            "conditional_logic": 0,
            "wrapper": { "width": "", "class": "", "id": "" },
            "default_value": "",
            "maxlength": "",
            "placeholder": "",
            "prepend": "",
            "append": ""
        },
        {
            "key": "field_0123456789abf",
            "label": "Hero Text",
            "name": "hero_text",
            "aria-label": "",
            "type": "textarea",
            "instructions": "Short intro shown under the title.",
            "required": 0,
            "conditional_logic": 0,
            "wrapper": { "width": "", "class": "", "id": "" },
            "default_value": "",
            "maxlength": "",
            "rows": 4,
            "placeholder": "",
            "new_lines": ""
        }
    ],
    "location": [
        [
            { "param": "page_template", "operator": "==", "value": "page-templates/home.php" }
        ]
    ],
    "menu_order": 0,
    "position": "normal",
    "style": "default",
    "label_placement": "top",
    "instruction_placement": "label",
    "hide_on_screen": ["the_content"],
    "active": true,
    "description": "",
    "show_in_rest": 0,
    "modified": 1700000000
}
```

To add another section, add a new `type: "tab"` field after the last field of the previous section, then that section's fields.

When adding other field types (repeater, flexible content, image, link, gallery), copy the full shape from an existing group in `acf-json/` or from an ACF export so no required properties are dropped.

## Reading fields in templates

Read ACF data with ACF's native functions directly (`get_field()`, `the_field()`, `have_rows()` / `the_row()` / `get_sub_field()`). Do not create wrapper helpers for reading fields. The old `tmdr_field()` / `tmdr_option()` wrappers were removed so every call site shows exactly which ACF API and which `post_id` it reads from. Do not add default or fallback values at the call site: an empty field renders nothing (see "Empty values render nothing").

Repeaters and flexible content use `have_rows()` / `the_row()` / `get_sub_field()`; single fields (text, image, gallery, link) use `get_field()`. Guard array returns (`is_array()`) and empty values before rendering. An empty repeater or unset field returns `false` / `array()` rather than throwing, so unguarded output produces silent gaps or PHP notices.

Escape on output. ACF values are editor-supplied, not trusted:

```php
<?php $title = get_field('hero_title'); ?>
<?php if ($title) : ?>
    <h1><?php echo esc_html($title); ?></h1>
<?php endif; ?>
```

Use `esc_html()` for plain text, `esc_attr()` for attributes, `esc_url()` for URLs, and `wp_kses_post()` for WYSIWYG content.

**Images:** ACF image fields use `return_format: "array"`. Rendering (`wp_get_attachment_image()`, lazy loading, placeholders, alt text) is defined once in `tmdr-page-template` under "Images". Follow it there instead of repeating it in field code.

Nested repeaters: the inner `have_rows('entries')` call runs inside the outer row loop; sub_fields of a nested repeater carry `parent_repeater` keys in JSON.

## Repeater or fixed fields?

Decide per section, before writing the group. Look at the reference markup: **repeated sibling elements with the same structure inside one container** (cards, accordion or FAQ items, slider slides, logo strip, stats, testimonials, timeline steps, tabs) are repeater candidates.

| Situation | Use |
| --- | --- |
| Items share one structure and are interchangeable (removing or reordering one does not break the layout), and the editor could reasonably add or remove items | **Repeater** |
| Count is fixed by the design and each item has its own role, position, or styling (hero left/right columns, one-off highlight blocks) | **Fixed fields** (one set of fields per item, e.g. `left_title`, `right_title`) |
| Only images, no per-image text | **Gallery** |
| Rows differ in structure, or the editor must mix and reorder different block types | **Flexible content** |
| Items are standalone content reused elsewhere with their own page or archive (posts, products, team members, locations) | **CPT** + post object or relationship field, not a repeater |
| A single value (title, image, link, text) | Plain field |
| A list of plain text values | Repeater with one sub field |

If the design shows 2 or 3 identical items, decide by whether the client would expect to add another one: likely yes means repeater. If it is genuinely unclear, ask once (one question, with choices): **"Repeater (editor can add and remove items)"** / **"Fixed fields (N items)"**. Otherwise decide without asking, and list the decisions in one line when the step ends (for example "Repeater: features, faq. Fixed: hero.").

Repeater settings:

- Name the field in plural `snake_case` (`feature_items`, `faq_items`), and name sub fields without repeating the parent (`title`, not `feature_title`).
- `min` `0` by default (an empty repeater renders nothing). Set `max` only when the layout has a real cap (for example 6 for a 3x2 grid); leave it empty when the layout wraps.
- `layout`: `block` for rows with an image, textarea, or many fields; `row` for 2 to 4 short fields; `table` for at most 3 small fields.
- `button_label` in English, naming the item (`Add Feature`, `Add Question`).
- Set `collapsed` to the title sub field key so long lists stay readable.
- Nest at most 2 levels. Anything deeper becomes flexible content or a CPT.
- Put the repeater under its section's tab like any other field.

## Empty values render nothing

Any field can be empty, and an empty field must **not appear on the page**: no empty tag, no placeholder text, no fallback image.

- No default values in JSON and no fallbacks in templates (`?: 'Some text'`, `?? 'Click me!'`). `placeholder` in JSON is only an admin hint and may stay.
- Read the value into a variable, then render the **whole element**, including its wrapper and classes, only when it has a value. Never output an empty `<h2>`, `<p>`, `<a>`, or `<div>`.
- Images render only when an attachment ID exists. Links render only when both `url` and `title` exist.
- Repeaters: wrap the whole list container in `have_rows()`, and skip a row that has no values at all.
- Sections: read the section's fields first, and when every one is empty, do not render the section wrapper at all.
- **Define the variables directly above their own section**: a small `<?php ... ?>` block placed right before that section's markup (and before the `if` that wraps it). Never collect the fields of all sections in one block at the top of the template file, and never use a variable that another section's block defined.
- Prefix variable names with the section name (`$hero_title`, `$intro_title`), and define every variable a section uses inside its own block. This stops sections from overwriting each other's values or reusing a stale one.

```php
<?php get_header(); ?>

<?php
$hero_title    = get_field('hero_title');
$hero_text     = get_field('hero_text');
$hero_image    = get_field('hero_image');
$hero_image_id = is_array($hero_image) ? ($hero_image['ID'] ?? 0) : (int) $hero_image;
?>
<?php if ($hero_title || $hero_text || $hero_image_id) : ?>
    <section class="hero">
        <div class="container">
            <?php if ($hero_title) : ?>
                <h1 class="hero__title"><?php echo esc_html($hero_title); ?></h1>
            <?php endif; ?>
            <?php if ($hero_text) : ?>
                <div class="hero__text"><?php echo wp_kses_post($hero_text); ?></div>
            <?php endif; ?>
            <?php if ($hero_image_id) : ?>
                <?php echo wp_get_attachment_image($hero_image_id, 'full', false, array('class' => 'hero__image', 'loading' => 'eager')); ?>
            <?php endif; ?>
        </div>
    </section>
<?php endif; ?>

<?php
$intro_title = get_field('intro_title');
$intro_text  = get_field('intro_text');
?>
<?php if ($intro_title || $intro_text) : ?>
    <section class="intro">
        <div class="container">
            <?php if ($intro_title) : ?>
                <h2 class="intro__title"><?php echo esc_html($intro_title); ?></h2>
            <?php endif; ?>
            <?php if ($intro_text) : ?>
                <div class="intro__text"><?php echo wp_kses_post($intro_text); ?></div>
            <?php endif; ?>
        </div>
    </section>
<?php endif; ?>

<?php get_footer(); ?>
```

## Option pages

Option pages are global, site-wide settings rather than per-template content, so they stay in PHP in `inc/functions-acf.php`, not JSON.

Read an option field by passing the option page's `post_id` as the second argument: `get_field('booking_url', 'header')`.

The starter registers only **Theme Settings → Login** (`login`) and **Theme Settings → Page Links** (`page-links`). Their field groups are added in PHP with `acf_add_local_field_group()` on the `acf/include_fields` hook. Add new option pages and their groups the same way in `inc/functions-acf.php`, following the commented examples at the top of that file. Do not copy keys from those PHP groups: a few are not valid 13-hex keys (e.g. `field_64db4a60bfbfg`), and the validation below targets `acf-json/` files only.

`getPageData($page_title)` (in `inc/functions-custom.php`) returns the `page_object` (a `WP_Post`) from the Page Links repeater row whose `page_title` matches. It returns an **error string** when no row matches, and `get_field()` returns `false` for an empty repeater, so guard the result before using `->ID`:

```php
$page = getPageData('Contact');
if (is_object($page) && isset($page->ID)) {
    $url = get_permalink($page->ID);
}
```

Before relying on any other option page, confirm it is registered in `inc/functions-acf.php`.

## Header and footer option pages

The navbar and footer content each get a separate option sub-page under Theme Settings, with their own `post_id`. Do not put it in General Settings or in a template's field group, because the header and footer render on every page.

Register them in `tmdr_register_option_page()` in `inc/functions-acf.php`, at the "Place for other custom subpage" comment:

```php
acf_add_options_sub_page( array(
    'page_title'  => 'Header',
    'menu_slug'   => 'header',
    'post_id'     => 'header',
    'parent_slug' => 'theme-settings',
    'menu_title'  => 'Header',
    'redirect'    => false,
) );

acf_add_options_sub_page( array(
    'page_title'  => 'Footer',
    'menu_slug'   => 'footer',
    'post_id'     => 'footer',
    'parent_slug' => 'theme-settings',
    'menu_title'  => 'Footer',
    'redirect'    => false,
) );
```

Then add one field group per page in the same file with `acf_add_local_field_group()` on `acf/include_fields`, following the existing `login` and `page-links` groups. Use `options_page == header` and `options_page == footer` as the location rules. Use one tab per section (for example Logo, Contact, Social, Copyright), and many fields per tab. Generate fresh keys as described above.

Read the fields in `TheNavbar.php` and `TheFooter.php` with `get_field('field_name', 'header')` / `get_field('field_name', 'footer')`. Menus stay as WordPress menus (`main_menu`, `footer`), not ACF fields.

## Validation

After writing a group JSON, run:

```bash
f=acf-json/group_xxxxxxxxxxxxx.json
jq empty "$f"                                                  # parses
jq -r '.. | .key? // empty' "$f" | grep -Evc '^(group|field)_[0-9a-f]{13}$'   # must print 0
jq -r '.. | .key? // empty' "$f" | sort | uniq -d              # must print nothing
jq .modified "$f"                                              # must be a fresh timestamp
```

Also confirm the filename equals the top-level `key`.