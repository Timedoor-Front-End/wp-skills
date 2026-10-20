---
name: tmdr-acf-field-group
description: Use when creating or editing ACF field groups, fields, tabs, repeaters, flexible content, or option pages for this theme. Also use whenever the user asks to "add a field," "add a repeater," "set up custom fields," "build an options page," or wire up any admin-editable content — even if they don't say "ACF" explicitly. Covers acf-json/ local JSON conventions, key naming, location rules, and template reading patterns.
---

# ACF JSON integration — global

Client ACF field groups are stored as JSON files in `acf-json/`. This keeps field configuration under version control and reviewable in diffs, instead of living only in the database.

## Rules

- Create or edit client field groups as JSON only. This is what keeps `acf-json/` the single source of truth and lets changes be reviewed like code.
- Do not register client field groups in `inc/functions-acf.php`. Mixing JSON-defined and code-defined client groups makes it unclear which one wins and breaks the "JSON is source of truth" convention.
- Do not auto-sync field groups through code. Developers sync manually through `WP Admin → Custom Fields → Field Groups → Sync`. Manual sync gives a developer a chance to review what's about to change on a given environment before it takes effect, rather than having it happen silently on deploy.
- Do not modify `acf-json/index.php`. It's a security stub (blocks directory listing/direct access), not project configuration — editing it doesn't affect field groups and risks weakening that protection.
- Use one field-group file per page template when their field structures differ. This keeps each template's fields scoped and easy to find, instead of one sprawling group with conditional logic for every page.
- The filename must match the field group's key: `group_<13-hex>.json`. ACF resolves local JSON by scanning `acf-json/` for a file matching the group's key — a mismatched filename means the sync UI won't find or update the right file.
- Use unique export-style keys: `group_<13-hex>`, `field_<13-hex>`, `post_type_<13-hex>`, and `taxonomy_<13-hex>`. Never use a section, template, CPT, taxonomy, or client name as a key — keys must stay stable identifiers so ACF can track a field across renames; a human-readable key would silently break if the field or section is ever renamed. Keys are 13 hex chars.
- Keep field names semantic `snake_case` — field names (unlike keys) are what templates reference via `get_field('field_name')`, so they should read clearly at the call site.
- Use one `type: "tab"` field per UI section, with `placement: "left"`; do not split fields from the same section across multiple tabs. Place that section's fields after the tab until the next tab. This keeps the admin edit screen matching the mental model of "one tab = one section," so editors aren't hunting across tabs for related fields.
- Use a specific `page_template` location rule, for example `page-templates/home.php`, instead of applying page fields to every page. A blanket "Post Type = Page" rule would surface unrelated fields on every page in wp-admin, confusing editors.
- Page field groups use `hide_on_screen: ["the_content"]`, since these pages are built from structured fields, not the default WYSIWYG editor — leaving it visible invites editors to add content that the template never renders.
- ACF Pro entity files (`post_type_*.json`, `taxonomy_*.json`) contain entity configuration only, without field groups or tabs. With free ACF, register CPTs and taxonomies in code with `register_post_type()` or `register_taxonomy()`, since the JSON-registration feature is Pro-only.
- Preserve complete ACF export configuration, including `sub_fields`, `layouts`, `min`, `max`, `button_label`, return formats, `mime_types`, conditional logic, and wrappers when used. Dropping these during an edit silently changes validation/UI behavior in the admin, often in ways that aren't obvious until an editor hits the missing constraint.
- Update the group's top-level `"modified"` Unix timestamp whenever you edit a field-group JSON. ACF only offers a group in the Sync UI when the JSON's `"modified"` value is higher than the `post_modified` date stored in the database — an edited JSON with a stale `"modified"` parses fine but never appears as available for sync. Set it to the current time, e.g. `date -u +%s` (or an equivalent), not a made-up value.

## Reading fields in templates

Repeaters and flexible content use `have_rows()` / `the_row()` / `get_sub_field()`; single fields (text, image, gallery, link) use `get_field()`. Guard array returns (`is_array()`) and empty values before rendering, since an empty repeater or unset field returns `false`/`array()` rather than throwing — unguarded output produces silent gaps or PHP notices instead of a clear failure.

ACF image fields use `return_format: "array"`; render through `wp_get_attachment_image()` with the attachment ID and an explicit fallback when empty, so the image benefits from WordPress's responsive `srcset`/`sizes` handling and the page never ships a broken `<img>` tag when the field is empty:

```php
<?php
$image = get_field('message_image');
$image_id = is_array($image) ? ($image['ID'] ?? 0) : (int) $image;
?>

<?php if ($image_id) : ?>
    <?php echo wp_get_attachment_image($image_id, 'full', false, array(
        'class' => 'block__image',
        'alt' => '',
        'loading' => 'lazy',
    )); ?>
<?php else : ?>
    <img class="block__image" src="https://placehold.co/..." alt="" loading="lazy">
<?php endif; ?>
```

Nested repeaters: the inner `have_rows('entries')` call runs inside the outer row loop; sub_fields of a nested repeater carry `parent_repeater` keys in JSON.

## Option pages

ACF option pages are registered in `inc/functions-acf.php` (keep these in PHP, not JSON) — they're global/site-wide settings rather than per-template content, so they don't fit the "one field group per template" JSON convention above:

- `Theme Settings → Login`
- `Theme Settings → Page Links`

`getPageData($page_title)` reads the Page Links repeater. It returns an error string when no mapping exists; guard the result before accessing `->ID`.

## Validation

After writing a group JSON, validate it parses (`JSON.parse` / `jq`), that every `group_`/`field_` key suffix is exactly 13 hex characters and unique, and that the top-level `modified` value was updated to the current Unix timestamp. Sync stays manual in WP Admin.
