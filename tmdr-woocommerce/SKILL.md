---
name: tmdr-woocommerce
description: Use when customizing WooCommerce in this theme — changing the shop, product, cart, checkout, My Account, or email layout, overriding WooCommerce templates, or adding WooCommerce styles/scripts. Also use when the user says "ubah halaman produk," "ubah tampilan cart/checkout," "custom shop page," "override template woocommerce," or mentions single-product, archive-product, or the WooCommerce my-account area, even without naming a file. Also use for any custom WooCommerce logic, such as extra checkout fields, validation, order meta, pricing or cart rules, custom emails, or AJAX. Covers the plugin prerequisite, hook-vs-override choice, copying single template files into the theme's woocommerce/ folder, keeping all custom PHP in one file (inc/functions-woocommerce.php), block vs shortcode cart/checkout, styling, and verification.
---

# WooCommerce customization

WooCommerce lets a theme replace a plugin template by placing a file with the **same relative path** inside the theme's `woocommerce/` folder. WooCommerce loads the theme copy first and falls back to the plugin file. Editing files inside `wp-content/plugins/woocommerce/` is never correct: the next plugin update overwrites it.

## Prerequisite: WooCommerce must be installed

Confirm WooCommerce is installed and active before writing any code that depends on it, following `tmdr-theme-project` → "Plugin prerequisites" (conversation, then WP-CLI `wp plugin list --status=active --field=name`, then repo evidence, then ask once).

If it is missing or the answer is "not sure", ask the user to install and activate it themselves (WP Admin → Plugins). Do not install it for them: no `wp plugin install`, no Composer, no downloading plugin files into the repo. Wait for their confirmation before generating WooCommerce code. Never add `function_exists()` fallbacks around WooCommerce calls.

Also check theme support once. The theme must declare it, in `inc/functions-woocommerce.php` (see "Custom logic goes in one file"):

```php
add_action('after_setup_theme', 'tmdr_wc_theme_support');
function tmdr_wc_theme_support() {
    add_theme_support('woocommerce');
    add_theme_support('wc-product-gallery-zoom');
    add_theme_support('wc-product-gallery-lightbox');
    add_theme_support('wc-product-gallery-slider');
}
```

Add only the gallery features the design actually uses. Without `add_theme_support('woocommerce')`, WooCommerce shows a compatibility notice and styles the shop with fallback behavior.

## Choose the lightest change that works

Work down this list and stop at the first option that does the job:

1. **CSS only.** Restyle WooCommerce's existing markup in a plain CSS file. Safest: no template to maintain.
2. **Hooks.** Move, remove, or add content with `remove_action()` / `add_action()` / filters (for example reordering the single-product summary, changing the "Add to cart" text, or removing breadcrumbs). Put them in a `tmdr_wc_`-prefixed function in `inc/functions-woocommerce.php` (see "Custom logic goes in one file").
3. **Template override.** Only when the HTML structure itself must change and hooks cannot do it.

Every override is a file that must be re-checked after each WooCommerce update, so prefer hooks. Say which option you chose and why when it is not obvious.

## Overriding a template

When the user wants to change a WooCommerce template, copy **only the specific file** that needs changing into the theme, keeping its path under `templates/`. Do not copy the whole `templates/` folder: every copied file becomes the theme's responsibility, and untouched copies silently go stale when WooCommerce updates.

Procedure:

1. **Identify the exact file** using the map below (or by looking in the plugin's `templates/` folder).
2. **Find the plugin source.** Look for `wp-content/plugins/woocommerce/templates/` relative to the WordPress root (the theme lives in `wp-content/themes/<theme>/`). If the plugin folder is not available in this environment, ask the user to upload or paste the original file. Do not write the template from memory: the markup and `@version` differ between WooCommerce releases, and an invented copy can drop required hooks.
3. **Copy that one file**, creating the subfolders as needed. Example for the single product title:

   ```bash
   # from the theme root
   mkdir -p woocommerce/single-product
   cp ../../plugins/woocommerce/templates/single-product/title.php woocommerce/single-product/title.php
   ```

   The relative path must match exactly: `templates/single-product/title.php` becomes `woocommerce/single-product/title.php` in the theme.
4. **Keep the file header intact**, including the docblock with `@version` and the `defined('ABSPATH') || exit;` guard. The `@version` is how WooCommerce detects an outdated override. Do not bump it by hand.
5. **Make the smallest edit** that achieves the change. Keep every `do_action()` / `apply_filters()` call WooCommerce placed there, because plugins and other code hook into them. Mark your changes with a short comment (`// TMDR: reason`) so a future diff against a newer plugin version is easy.
6. **Follow the theme rules inside the override:** escape all output (`tmdr-page-template` → "Escaping output"), render images with `wp_get_attachment_image()` (`tmdr-page-template` → "Images"), use native ACF calls with the product ID for custom fields (`get_field('badge_text', $product->get_id())`), and follow the project's single-language or Polylang mode (`tmdr-theme-project` → "Multilingual sites"). Keep WooCommerce's own text domain (`'woocommerce'`) for strings that already use it.
7. **Wire styles if needed** (see "Styling and assets").

If a template override replaces a file that already exists in the theme's `woocommerce/` folder, edit that file in place instead of copying again.

## Common templates

| To change | Copy this file from `templates/` |
| --- | --- |
| Shop / category loop wrapper | `archive-product.php` |
| One product card in the loop | `content-product.php` |
| Single product page layout | `single-product.php`, `content-single-product.php` |
| Single product title, price, gallery, tabs | `single-product/title.php`, `single-product/price.php`, `single-product/product-image.php`, `single-product/tabs/tabs.php` |
| Shortcode cart page | `cart/cart.php`, `cart/cart-totals.php` |
| Shortcode checkout page | `checkout/form-checkout.php`, `checkout/form-billing.php`, `checkout/review-order.php` |
| My Account pages | `myaccount/my-account.php`, `myaccount/dashboard.php`, `myaccount/form-login.php` |
| Order emails | `emails/` (only when the design requires it; email clients are fragile) |

Use this as a starting point, not a complete list. Confirm the file exists in the installed plugin version before copying.

## Custom logic goes in one file

Any custom WooCommerce PHP that is not template markup lives in **one file: `inc/functions-woocommerce.php`**. This is deliberately open-ended: checkout fields, validation, saving order data, pricing or cart rules, order status actions, emails, admin columns, AJAX endpoints, or anything else a client asks for. The scope is not limited. What is fixed is the location, so the logic never ends up scattered across templates, `functions.php`, and other `inc/` files.

Rules:

- **Templates only render.** Files in `woocommerce/` and `page-templates/` must not register hooks, define functions, validate input, or save data. If a template needs a computed value, write a `tmdr_` function in `inc/functions-woocommerce.php` and call it from the template.
- **Do not put WooCommerce logic anywhere else:** not in `functions.php`, `functions-custom.php`, `functions-theme-setup.php`, or a template. Two exceptions follow the project's existing conventions: CSS/JS enqueue stays in `inc/functions-style-script.php`, and ACF field groups stay in `acf-json/` (or `functions-acf.php` for option pages).
- **Create the file if it does not exist**, then add its `require` to `functions.php` after the existing `inc/` files and to the load-order list in `tmdr-theme-project` → "Runtime flow". It only registers hooks, so its position does not matter. Start it with `if (!defined('ABSPATH')) { exit; }`.
- **Keep it one file.** Do not split it into several files unless the user asks.
- **Group by section comments** so it stays readable as it grows, in this order, adding a section only when it is needed:

  ```php
  /* ---------- Theme support ---------- */
  /* ---------- Layout hooks (shop, product, cart) ---------- */
  /* ---------- Checkout (fields, validation, saving, display) ---------- */
  /* ---------- Cart and pricing ---------- */
  /* ---------- Orders and emails ---------- */
  /* ---------- Admin ---------- */
  /* ---------- AJAX ---------- */
  ```

- **Prefix everything** with `tmdr_wc_` (for example `tmdr_wc_save_checkout_fields`), so each function is easy to find and cannot clash with WooCommerce or other plugins.
- **One feature stays together.** Keep all the functions for one client request adjacent under one comment, such as "Delivery note field": the field, its validation, saving, and display.
- **Use WooCommerce's own APIs.** Read and write orders and products through `wc_get_order()`, `wc_get_product()`, `$order->update_meta_data()` and `$order->get_meta()`, not raw SQL or `get_post_meta()` on orders, so the code works with High-Performance Order Storage (HPOS).
- **Security:** sanitize every submitted value (`sanitize_text_field()`, `sanitize_textarea_field()`, `sanitize_email()`), escape on output, and for custom AJAX endpoints verify a nonce and check capabilities.
- **Strings:** use the project's single-language or Polylang mode for new strings, and the `'woocommerce'` text domain only for strings WooCommerce itself already defines.

Example of a typical request, a custom field at checkout, kept in that one file:

```php
/* ---------- Checkout: delivery note ---------- */

// Show the field after the order notes.
add_action('woocommerce_after_order_notes', 'tmdr_wc_delivery_note_field');
function tmdr_wc_delivery_note_field($checkout) {
    woocommerce_form_field('tmdr_delivery_note', array(
        'type'     => 'text',
        'label'    => 'Delivery note',
        'required' => false,
    ), $checkout->get_value('tmdr_delivery_note'));
}

// Save it on the order (HPOS-safe).
add_action('woocommerce_checkout_create_order', 'tmdr_wc_save_delivery_note', 10, 2);
function tmdr_wc_save_delivery_note($order, $data) {
    if (!empty($_POST['tmdr_delivery_note'])) {
        $order->update_meta_data('_tmdr_delivery_note', sanitize_text_field(wp_unslash($_POST['tmdr_delivery_note'])));
    }
}

// Show it in the admin order screen.
add_action('woocommerce_admin_order_data_after_billing_address', 'tmdr_wc_show_delivery_note');
function tmdr_wc_show_delivery_note($order) {
    $note = $order->get_meta('_tmdr_delivery_note');
    if ($note) {
        echo '<p><strong>' . esc_html__('Delivery note', 'theme-text-domain') . ':</strong> ' . esc_html($note) . '</p>';
    }
}
```

Add validation (`woocommerce_checkout_process` or `woocommerce_after_checkout_validation`) and email output (`woocommerce_email_order_meta_fields`) in the same block when the request needs them. The label strings above are examples: replace them with the project's own translation approach.

These classic hooks apply to the shortcode checkout. The block checkout has its own API for additional checkout fields, which depends on the installed WooCommerce version, so check which checkout the site uses first (see below) and look up the current documentation for that version before writing it.

## Block cart and checkout

Recent WooCommerce versions create the Cart and Checkout pages with **blocks** by default. The PHP files `cart/cart.php` and `checkout/form-checkout.php` only apply to the **classic shortcode** versions (`[woocommerce_cart]`, `[woocommerce_checkout]`). Overriding them does nothing on a block-based page.

Before changing cart or checkout layout, find out which version the site uses (open the Cart/Checkout page in the editor, or ask the user), then:

- **Shortcode pages:** override the PHP templates as above.
- **Block pages:** restyle with CSS, or use the supported block filters. Do not override the PHP templates and expect them to render.

## Styling and assets

Plain CSS and JS, no build step (`tmdr-theme-project` → "Assets are plain CSS and JS"). Put WooCommerce styles in their own file under `assets/css/pages/` (for example `woocommerce.css`) and load it only on WooCommerce screens, extending `tmdr_script_enqueue()` in `inc/functions-style-script.php` the same way as page templates:

```php
if (is_woocommerce() || is_cart() || is_checkout() || is_account_page()) {
    tmdr_print_css('woocommerceCss', 'pages/woocommerce.css');   // assets/css/pages/woocommerce.css
}
```

Narrow the condition when only some screens need the file (`is_shop()`, `is_product()`, `is_cart()`, `is_checkout()`, `is_account_page()`). Handles must be unique across the theme. Scripts follow the same rules as other page scripts: classic deferred script, no `import` / `export`, initialization inside `DOMContentLoaded`.

Use BEM classes for markup you add. Do not rename or remove WooCommerce's own classes and IDs in overridden templates (`woocommerce-`, `product`, `.single_add_to_cart_button`, form field IDs): WooCommerce's JavaScript and extensions select them.

**Image sizes:** this theme disables WordPress intermediate image sizes, so WooCommerce thumbnails may fall back to full-size uploads. Tell the user this is expected, and recommend compressing product images before upload. Do not re-enable image sizes without being asked.

## Where code goes

| What | Where |
| --- | --- |
| All custom WooCommerce PHP: theme support, hooks, filters, checkout fields, validation, saving, pricing, orders, emails, AJAX | `inc/functions-woocommerce.php` only (create it if missing), `tmdr_wc_` prefixed functions |
| Template overrides | `woocommerce/` at the theme root, mirroring the plugin's `templates/` paths |
| Styles and scripts | `assets/css/pages/`, `assets/js/pages/`, wired in `inc/functions-style-script.php` |
| Custom product fields | ACF JSON in `acf-json/` with a `post_type == product` location rule (`tmdr-acf-field-group`) |

## Verify

- PHP: `php -l woocommerce/path/to/file.php` for every file you added or changed, and for `inc/functions-woocommerce.php`.
- Confirm each override's relative path mirrors the plugin's `templates/` path exactly (a wrong folder name means the override is silently ignored).
- Ask the user to open **WooCommerce → Status → System status → Templates**. Overridden files are listed there, and any marked outdated need to be compared against the plugin's current version of that file.
- Load the affected page (shop, product, cart, checkout, My Account) and check the browser console and the PHP error log.
- After a WooCommerce update, compare each override's `@version` with the plugin's current file and merge new markup in.
