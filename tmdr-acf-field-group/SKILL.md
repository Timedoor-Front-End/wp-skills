---
name: tmdr-acf-field-group
description: Use when creating or editing ACF field groups, fields, tabs, repeaters, flexible content, or option pages for this theme. Also use whenever the user asks to "add a field," "add a repeater," "set up custom fields," "build an options page," or wire up any admin-editable content — even if they don't say "ACF" explicitly. Covers acf-json/ local JSON conventions, key generation, location rules, a JSON skeleton, and template reading patterns.
---

# ACF JSON integration

Client ACF field groups are stored as JSON files in `acf-json/`. This keeps field configuration under version control and reviewable in diffs, instead of living only in the database.

**Where things live:**

| What | Where |
| --- | --- |
| Client page/template field groups | JSON in `acf-json/` |
| Option pages and their field groups | PHP in `inc/functions-acf.php` |
| CPTs / taxonomies | `post_type_*.json` / `taxonomy_*.json` (ACF Pro), or `register_post_type()` / `register_taxonomy()` in code (free ACF) |

## Rules

- Create or edit client field groups as JSON only, so `acf-json/` stays the single source of truth and changes are reviewable like code.
- Do not register client field groups in `inc/functions-acf.php`. Only option pages (and their groups) go there. Mixing the two for client content makes it unclear which one wins.
- Do not auto-sync field groups through code. Developers sync manually through `WP Admin → Custom Fields → Field Groups → Sync`. Manual sync lets a developer review what will change on a given environment instead of it happening silently on deploy.
- Do not modify `acf-json/index.php`. It is a security stub that blocks directory listing, not project configuration.
- Use one field-group file per page template when their field structures differ. This keeps each template's fields scoped and easy to find.
- Name the file after the group key: `group_<13-hex>.json`. ACF itself writes files by key, so a mismatched filename leads to a duplicate file the next time the group is saved from wp-admin.
- Use unique export-style keys: `group_<13-hex>`, `field_<13-hex>`, `post_type_<13-hex>`, `taxonomy_<13-hex>`. Never use a section, template, CPT, taxonomy, or client name as a key. Keys are stable identifiers so ACF can track a field across renames.
- Keep field names semantic `snake_case`. Names (unlike keys) are what templates reference via `get_field('field_name')`.
- Use one `type: "tab"` field per UI section (e.g. Hero, Intro, Gallery), with `placement: "left"`. A tab mirrors a section, and a section normally contains many fields: put all of that section's fields after its tab, until the next tab. The number of fields per tab is not limited, and a tab is never tied to a single field. Do not split one section across multiple tabs.
- Use a specific `page_template` location rule (e.g. `page-templates/home.php`), not a blanket "Post Type = Page" rule, which would show unrelated fields on every page.
- Page field groups use `hide_on_screen: ["the_content"]`, since these pages are built from structured fields. Leaving the editor visible invites content the template never renders.
- Preserve complete ACF export configuration, including `sub_fields`, `layouts`, `min`, `max`, `button_label`, return formats, `mime_types`, conditional logic, and wrappers. Dropping these during an edit silently changes validation and admin UI behavior.
- Update the group's top-level `"modified"` Unix timestamp whenever you edit a field-group JSON. ACF only offers a group in the Sync UI when the JSON's `modified` is higher than the database `post_modified`. A stale value parses fine but never appears for sync. Use `date -u +%s`, not a made-up number.

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
            "instructions": "",
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
            "instructions": "",
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

Read ACF data with ACF's native functions directly (`get_field()`, `the_field()`, `have_rows()` / `the_row()` / `get_sub_field()`). Do not create wrapper helpers for reading fields. The old `tmdr_field()` / `tmdr_option()` wrappers were removed so every call site shows exactly which ACF API and which `post_id` it reads from. Defaults go inline at the call site, e.g. `get_field('cta_title', 'footer') ?: ''`.

Repeaters and flexible content use `have_rows()` / `the_row()` / `get_sub_field()`; single fields (text, image, gallery, link) use `get_field()`. Guard array returns (`is_array()`) and empty values before rendering. An empty repeater or unset field returns `false` / `array()` rather than throwing, so unguarded output produces silent gaps or PHP notices.

Escape on output. ACF values are editor-supplied, not trusted:

```php
<h1><?php echo esc_html(get_field('hero_title') ?: ''); ?></h1>
<div><?php echo wp_kses_post(get_field('hero_text') ?: ''); ?></div>
```

Use `esc_html()` for plain text, `esc_attr()` for attributes, `esc_url()` for URLs, and `wp_kses_post()` for WYSIWYG content.

**Images:** ACF image fields use `return_format: "array"`. Rendering (`wp_get_attachment_image()`, lazy loading, placeholders, alt text) is defined once in `tmdr-page-template` under "Images". Follow it there instead of repeating it in field code.

Nested repeaters: the inner `have_rows('entries')` call runs inside the outer row loop; sub_fields of a nested repeater carry `parent_repeater` keys in JSON.

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

Also confirm the filename equals the top-level `key`. Sync stays manual in WP Admin.
