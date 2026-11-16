---
name: tmdr-theme-project
description: Use when working in this theme at all, including starting a new project (first detecting whether the current folder is a theme folder, a WordPress root, or wp-content/themes, then cloning the team's theme starter into the right tmdrx<project_name> folder) — project requirements, repository layout, functions.php load order, plugin prerequisites (ACF Pro, Polylang) and when to ask whether they are installed, whether the site is multilingual (Polylang) or not, and known intentional constraints. Consult this first for any task in the theme, even small edits.
---

# Project

Classic WordPress starter theme for Timedoor projects.

Requirements: WordPress, PHP 7.4+, ACF Pro. Polylang is **optional**, only for multilingual sites (see "Plugin prerequisites" and "Multilingual sites").

## Asking the user

Applies to every question in this skill. Ask only what cannot be inferred from the conversation, the request, or shell checks.

- **One question per message, always.** Never batch questions, including the plugin questions, even when the tool can ask several at once. Ask the next question only after the user answered the previous one.
- **Keep every message short.** A message that asks something contains only: one line of context, the question, and its choices. Do not list the steps you will do next, do not preview later questions, and do not explain skipped steps, other skills, or things you did not do (such as "no commit made"). Show a plan only if the user asks for it.
- **Offer 2 to 4 concrete choices built from what you found** (folder names, paths, detected values), not generic ones. Put the recommended choice first and mark it "(recommended)". The user can always type a different answer, so do not add an "other" choice yourself unless the tool needs one.
- **Use a tappable-choices tool when the environment has one** (for example `AskUserQuestion` in Claude Code, or `ask_user_input_v0` in Claude.ai). If there is none, write the choices as a short numbered list the user can answer with a number.
- **Yes / no questions** still get named choices that say what each answer does, for example "Yes, use this folder" and "No, create a new folder".
- **Never offer a choice you cannot carry out**, and never include a choice that deletes or overwrites existing content.

## Starting a new project

Every new project starts from the team's theme starter. Applies only when the task is a **new project** (e.g. "buat template theme WP untuk design/index.html") and the theme is not already in the workspace. Never clone over an existing theme.

Starter repo: `https://bitbucket.org/timedoor/fe-starter-wp.git`, branch `vanilla` (always this branch).

**Order matters.** Settle *where the theme goes* first, with shell checks and at most one question. Do not ask for a theme name, ACF Pro, or multilingual status up front, and never batch those questions together. Plugin questions are not asked up front: they come at the start of the ACF step (see "Build steps and suggesting the next step").

### Step 1: Detect where Claude is running (no questions yet)

The user may open Claude in the theme folder, the WordPress root, or somewhere else. Run these checks and read the results before asking anything:

```bash
pwd
ls -A
# WordPress root marker: wp-config.php (or wp-load.php) AND a wp-content/ folder
# Theme markers: functions.php that loads inc/, and acf-json/index.php
```

Also check the path: is the current folder inside `.../wp-content/themes/<folder>`? Design/reference files the user mentions (for example a `design/` folder or `index.html`) do not count as theme content. Ignore them when deciding whether the folder is "empty", and never move or delete them.

Classify the location, then follow the matching case:

| Case | How to recognise it | What to do |
| --- | --- | --- |
| **A. Existing theme** | Theme markers exist in the current folder | Skip this section entirely. Work in the existing repo. |
| **B. Candidate theme folder** | Current folder is directly inside `wp-content/themes/`, has no theme markers, and is empty (ignoring design/reference files) | Ask the confirmation question below. |
| **C. WordPress root** | `wp-config.php` or `wp-load.php` plus `wp-content/` in the current folder | Target is `wp-content/themes/tmdrx<project_name>`. Go to "Create a new folder". |
| **D. `wp-content/themes/` itself** | Current folder is named `themes` inside `wp-content` | Target is `./tmdrx<project_name>`. Go to "Create a new folder". |
| **E. Empty folder, not inside WordPress** | Nothing above matches | Do not guess. Ask where the theme should go. Offer the WordPress installs or `wp-content/themes/` folders you can find nearby (parent folders, sibling folders, common local paths) as choices, plus an open answer for a path you did not find. |
| **F. Non-empty, no markers, not WordPress** | Unknown content | Stop and ask. Choices: the folder you would use instead (if any), a path the user gives, or stop. Never delete or overwrite its contents. |

If the current folder is inside a WordPress install but deeper or elsewhere (for example `wp-content/plugins/...`), treat it as case E and ask for the target.

### Case B: confirm the current folder

Ask exactly one short question, and nothing else in the same message:

"This folder (`<folder name>`) is empty and sits inside `wp-content/themes/`. Is this where the theme should be placed?"

Choices: **"Yes, use this folder"** (recommended) and **"No, create a new `tmdrx<project_name>` folder in `wp-content/themes/`"**.

- **Yes:** use the current folder as the theme folder. Take the project name from the folder name, so do not ask for a theme name. Clone into it:
  - Truly empty: `git clone --branch vanilla --single-branch https://bitbucket.org/timedoor/fe-starter-wp.git .`
  - Contains only design/reference files: `git clone` refuses a non-empty folder, so clone to a temporary folder next to it, copy the starter's files into the theme folder **without overwriting** (`cp -rn` or `rsync -a --ignore-existing`, including dotfiles), then remove the temporary folder. If any starter file would collide with an existing file, stop and ask.
- **No:** go to "Create a new folder", using this folder's `wp-content/themes/` as the parent.

### Create a new folder (cases C, D, and Case B "No")

1. Folder name: `tmdrx` + the project name, lowercase, no spaces or special characters, for example `tmdrxbalivilla`. If the project name is not obvious from the request or the design folder name, ask for it. That is the only question. Suggest up to 3 names derived from what you found (the `<title>` or brand in the design's `index.html`, the WordPress site folder name, the reference folder name), each already shown as the final folder name, for example `tmdrxbalivilla`. If nothing can be derived, ask as an open question.
2. If `wp-content/themes/tmdrx<project_name>` already exists: if it has the starter's marker files, treat it as an existing theme (case A). If it is empty, clone into it. If it has other content, stop and ask. Never delete or overwrite it.
3. Otherwise clone into a new folder:

   ```bash
   cd wp-content/themes   # skip when already inside it (case D)
   git clone --branch vanilla --single-branch https://bitbucket.org/timedoor/fe-starter-wp.git tmdrx<project_name>
   ```

   If the workspace turns out not to have `wp-content/themes/`, fall back to case E.

### After cloning

- Confirm the marker files exist (`functions.php`, `inc/`, `acf-json/index.php`).
- Remove the starter's git history so the project starts with a new repository. Before deleting, confirm you are inside the folder you just cloned into and that `git remote get-url origin` points at `fe-starter-wp`; if either check fails, stop and ask. Then:

  ```bash
  rm -rf .git
  git init
  ```

  Delete only the `.git` folder of the theme folder, never anything else. In case B, if the *parent* of the theme folder or the folder itself was already a git repository before the clone, do not delete or re-init anything: skip this step and tell the user.
- **Keep the reference folder's images out of git.** The reference folder can have any name (`design`, `reference`, `mockup`, `html`, `figma-export`, ...), so never assume `design/`. Work out the folder from what the user gave:
  - The user pointed at a file (`@reference\index.html`): the reference folder is the folder that file sits in.
  - The user pointed at a folder: that folder is the reference folder.
  - There are several (for example a separate `images/` or `assets/` folder next to `index.html`): treat each one that holds reference images as a reference folder.
  - The files sit directly in the theme root with no folder, or you cannot tell which folders are reference material: ask once, "Which folder(s) hold the design reference images?", offering the candidate folders you found in the theme folder as choices (multiple choice allowed) plus "none, there are no reference images". Do not guess, because a wrong pattern could hide real theme images.

  If a reference folder sits inside the theme folder, add its images to the theme's `.gitignore` right after `git init`, before anything is staged. Convert Windows paths to forward slashes, make the path relative to the theme root, and never ignore the theme's own `assets/` folder. Append to the file if the starter already ships one, and do not duplicate lines. Set `REF_DIR` to the real relative path (no leading or trailing slash), once per reference folder:

  ```bash
  REF_DIR="reference"   # example: whatever the folder is really called, e.g. "mockup/v2"
  cat >> .gitignore <<GITIGNORE

  # Design reference images (reference only, not part of the theme)
  /${REF_DIR}/**/*.png
  /${REF_DIR}/**/*.jpg
  /${REF_DIR}/**/*.jpeg
  /${REF_DIR}/**/*.gif
  /${REF_DIR}/**/*.webp
  /${REF_DIR}/**/*.svg
  /${REF_DIR}/**/*.avif
  /${REF_DIR}/**/*.ico
  GITIGNORE
  git check-ignore -v "${REF_DIR}/path/to/some-image.png"   # must print a matching rule
  ```

  The rule covers only the reference folder. Images copied into `assets/images/` are theme assets and stay tracked. If the reference folder is outside the theme folder, no rule is needed. If the folder holds other image extensions (or uppercase ones like `.PNG`), add them too and re-check with `git status --short`: no reference image may show up. Tell the user which folder(s) were ignored.
- Do not add a remote or make the first commit unless the user asks. Tell them the repository is fresh with no remote, so they must add the project's own remote before pushing.
- Do not ask about the theme name or location again. Next, ask the starter-files question from "Starter code is a template, not a finished project", then start the first build step. Plugin questions are not asked yet: the first step uses static content and no plugin calls.

**Always try the clone yourself first.** Never assume it will fail or ask the user to run it without trying.

**If the clone fails** (private repo, authentication prompt, or the network blocks the host): do not look for workarounds, and never ask for or store passwords or tokens in the chat. Send one short message, and nothing else, with this content only:

1. One line: the clone needs the user's own Bitbucket credentials, so they should run it themselves.
2. The single `git clone` command, as a code block, with the real target path. Give only the clone, into the theme folder or into a temporary sibling folder (case B with design files). Do **not** include the copy, cleanup, `git init`, or `.gitignore` steps: Claude does those afterwards. Tell them to run it in their own terminal (in Claude Code they can type it with a leading `!`).
3. One question with choices: **"Done, it's cloned"** / **"It failed, here is the error"**.

Do not list what you will do after the clone, and do not ask the plugin questions in this message. When the user confirms, verify the marker files and continue from "After cloning".

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

**How to ask:** one short question, before writing any code that needs the plugin, and only after the theme folder is settled (see "Starting a new project"). Never ask about plugins before the location and theme folder are resolved. The first build step (static integration) needs no plugin, so ask at the start of the ACF step, not before. Ask one at a time, each with choices, in this order, and only the ones still unanswered. First: ACF Pro installed and active? **Yes / No / Not sure**. After the user answers, and only if the multilingual status is still unknown: site languages? **Single language / Multilingual (Polylang) / Not sure**. Never put both in the same message.

**If the answer is "no" or "not sure":**

- Say which plugin is missing and what will break (front-end fatal for ACF Pro, or for Polylang on multilingual sites).
- Ask the user to install and activate it themselves (WP Admin → Plugins, or the plugin notice the theme shows for plugins it registers). For "not sure", ask them to check WP Admin → Plugins first. ACF Pro is licensed and needs the user's own license key, so installation stays with them.
- Do not install the plugin yourself: no `wp plugin install`, no Composer, no downloading plugin files into the repo, and no editing `functions-plugin.php` or `wp-config.php` to force it.
- Wait for the user to confirm the plugin is installed and active before generating code that depends on it. In the meantime, offer work that does not depend on it (plain CSS, plain JS, static markup).
- Polylang only: the user may instead choose to proceed in single-language mode (see "Multilingual sites").
- Never add `function_exists()` fallback wrappers to work around a missing plugin.

## Starter code is a template, not a finished project

A fresh clone of the starter is **template code**: it contains demo files and placeholder content that exist to be adapted for the real project. The structure and rules of the theme are fixed. The demo content is not.

**Starter demo files** (expected to be adapted or replaced for a new project):

- `page.php` and `assets/dummy/intro.php` (starter documentation page)
- `page-templates/home.php` (starter home template). `blank.php` and `thank-you.php` stay unless the project needs them changed.
- `template-parts/components/TheNavbar.php` and `TheFooter.php` (hardcoded demo menu, contacts, placeholder logo and icons) and `ThemeButton.php` (echoes unescaped values)
- Demo and placeholder content anywhere else (placehold.co, lorem ipsum, icons8)

**What may change to match the reference:** the markup, text, and styling of the files above. Rewrite them to match the design, keeping the existing file name, location, and `Template Name:` header where a template already exists.

**What must not change** (the structure and rules): the file and folder layout, `functions.php` and the `inc/` load order, naming (`The*`, PascalCase components, `tmdr_` prefix), `components()` usage, the enqueue wiring in `inc/functions-style-script.php`, ACF JSON in `acf-json/` and option pages in PHP, escaping, plain CSS and JS, and `acf-json/index.php`. `layout.css` and `layout.js` are not edited: global additions go to `assets/css/global.css` and `assets/js/global.js` (see "Global or page-specific?"). The `header.php` and `footer.php` structure gets small targeted edits only. Starter demo files do not get deleted unless the new version replaces them (for example `assets/dummy/intro.php` only when `page.php` is replaced and nothing still requires it).

**Ask first, once.** Before modifying or replacing any starter demo file, ask this question, one per message, after the location is settled and the theme is cloned:

"The cloned theme still has starter demo files (`home.php`, `TheNavbar`, `TheFooter`, `page.php`). How should I handle them?"

Choices: **"Modify the starter files to match the design"** (recommended) / **"Leave the starter files, only add new files"** / **"Ask me per file"**.

Apply the answer for the rest of the task and do not ask again file by file, unless the user chose "Ask me per file". This section applies to a **new project** (the theme was just cloned in this task, or the files still hold the demo content). In an existing project with real client work, the normal working rules apply and no question is needed.

## Build steps and suggesting the next step

For a new page from a design reference, work in small steps. Finish one, then **suggest the next one** and wait. Never start the next step on your own.

| # | Step | What it covers |
| --- | --- | --- |
| 1 | **Integrate the page into the theme** | Template file in `page-templates/` with its `Template Name:` header, the reference markup as **static content**, page CSS/JS, copied assets, and the enqueue wiring. Fonts, `:root` variables, and base typography the page needs go to the global assets in this step. No ACF and no plugin calls. |
| 2 | **ACF for the page** | Plugin questions first (see "Plugin prerequisites"), then the field group JSON (English text, repeater or fixed per `tmdr-acf-field-group`), then replace the static content with `get_field()` reads. Empty fields render nothing. |
| 3 | **Header and footer** | `TheNavbar` and `TheFooter` markup from the reference, still static. Their CSS and JS go to `global.css` / `global.js`. |
| 4 | **Header and footer ACF** | Header and Footer option pages and fields, wired with their `post_id`. |
| 5 | **Next page, or finish** | Repeat steps 1 and 2 for another reference page, then run the verification commands from "Working rules". |

The order is a default, not a rule. If the user asks for a different order or for ACF from the start, follow the request. Skip steps that do not apply (for example the reference has no header or footer).

**How to end every step:**

1. One or two lines on what was done, plus any decision worth knowing (for example "Repeater: features. Fixed: hero.").
2. One question, "What next?", with 2 to 3 choices: the recommended next step first, an alternative if there is a sensible one, and **"Stop here"**.

Do not paste the whole roadmap or describe later steps in advance. This follows "Asking the user": one question, short message, choices.

## Repository layout

```text
inc/                    Theme setup, helpers, plugins, ACF, Polylang, auth
page-templates/         Optional page templates
template-parts/         Reusable PHP components only (template-parts/components/). Page content stays inline in page-templates/*.php, never in template-parts/
woocommerce/            WooCommerce template overrides (shop sites only; created per project)
assets/css/             Plain CSS (layout.css, global.css (added per project), pages/*.css, admin/*.css)
assets/js/              Plain JS (layout.js, global.js (added per project), pages/*.js)
assets/images/          Theme images
assets/dummy/intro.php  Demo content for the starter docs page (page.php)
acf-json/               ACF local JSON (only the index.php security stub in the starter; client groups are added per project)
```

`page.php` is the starter documentation page (it requires `assets/dummy/intro.php`). Replace it when real development begins. `show_debug_helper()` plus `template-parts/debug-console.php` is a dev-only debug panel; never leave a call to it in committed templates.

## Assets are plain CSS and JS

This project writes plain CSS and plain JS directly in `assets/css/` and `assets/js/`. There is no SCSS, no webpack, and no `source/` folder, so do not create or reference one, and do not add a build step.

- Add new styles as plain CSS files (one per page under `assets/css/pages/`) and new scripts as plain JS files (under `assets/js/pages/`).
- Scripts load with `defer` in the footer as classic scripts, so use no `import` / `export` and no bundler syntax.
- The starter's `assets/css/layout.css` and `assets/js/layout.js` were originally compiled output (the JS is a webpack bundle). Do not edit them. Global additions go to `global.css` / `global.js`, and page-specific code goes in its own file (see "Global or page-specific?").

### Global or page-specific?

Decide for every rule and script **before** placing it. A page-specific file loads on one template only. A global file loads on every page. Putting a global thing in a page file breaks every other page.

Run this test in order and stop at the first yes:

1. Does it style or load something rendered on **every page**: `<html>`, `<body>`, `header.php`, `footer.php`, `TheNavbar`, `TheFooter`? **Global.**
2. Is the selector not scoped to this page's own BEM block: `:root`, `html`, `body`, bare element selectors (`h1`, `a`, `img`), `.container` overrides, or an at-rule such as `@font-face`, `@import`, `@keyframes` or `@property`? **Global.**
3. Is it used by a component or class that appears on two or more templates (`ThemeButton`, shared utility classes)? **Global.**
4. Otherwise (a class or script used only by sections of this one template): **page-specific.**

If a thing affects every page, choose global when in doubt. If it only affects one section, choose page-specific.

| Item | Where |
| --- | --- |
| Fonts: `@font-face`, font files, Google Fonts or other font stylesheets | **Global** |
| `:root` custom properties (colors, spacing, font stacks, container widths), resets, base typography, `body`/heading/link defaults | **Global** |
| Navbar, header, and footer styles and scripts (menu toggle, sticky header) | **Global** |
| Shared component styles and utility classes used on 2+ templates | **Global** |
| Favicon, meta tags, analytics, and libraries used site-wide | **Global** (`header.php` or the global enqueue) |
| Section styles and scripts unique to one template | Page-specific |
| A library only one page uses (for example a slider on the home page) | Page-specific, in that page's enqueue block |

**Where global assets go:**

- Global CSS goes in `assets/css/global.css` and global JS in `assets/js/global.js`. Create each file the first time something global needs it, not empty in advance.
- Before creating them, open `inc/functions-style-script.php` and find where `layout.css` and `layout.js` are enqueued for every page. Register `global.css` / `global.js` in that same unconditional place, **right after** the layout files so they can override them, with unique handles. If you cannot find that place, ask the user.
- A font stylesheet from an external host is enqueued with `wp_enqueue_style()` in the global part of `tmdr_script_enqueue()`, **outside every `is_page_template()` block**, or linked in `header.php` (add `preconnect` there). Prefer self-hosted `@font-face` in `global.css` with the font files in `assets/fonts/`.
- Never put global assets inside an `is_page_template()`, `is_singular()`, or `is_archive()` block.

**Check before finishing a step:** scan each page CSS and JS file for `@font-face`, `:root`, `html`/`body` selectors, and header/footer selectors. Move anything global into `global.css` / `global.js`, and name what you moved in one line of the step summary.

**Mechanical check.** Run these from the theme root at the end of every step that added CSS or JS (use the editor's search tool with the same patterns if `grep` is not available). Skip a command if its folder does not exist yet.

```bash
# 1. Global leftovers in page CSS (must print nothing)
grep -nE '@font-face|@import|:root|^\s*(html|body)\b|^\s*(h[1-6]|a|img|p|ul|ol)\s*[,{]|\.(navbar|header|footer)([^a-zA-Z]|$)' assets/css/pages/*.css

# 2. Header/footer logic in page JS (review every hit)
grep -nE 'navbar|header|footer|offcanvas' assets/js/pages/*.js

# 3. Global files must be registered outside every condition (review the context)
grep -nE 'global\.(css|js)|layout\.(css|js)|is_page_template|is_singular|is_archive' inc/functions-style-script.php
```

- **Command 1:** any hit is a global rule in a page file. Move it to `global.css` and run it again until it prints nothing.
- **Command 2:** a hit is not automatically wrong (a page script may query a header element on purpose). Move it to `global.js` only if the logic belongs to the header or footer on every page.
- **Command 3:** the `global.css` / `global.js` lines must sit next to the `layout` lines, not inside an `is_page_template()`, `is_singular()` or `is_archive()` block. Read the surrounding lines to confirm, and fix the wiring if not.
- Put the result in the step summary in one line, for example "Global check: clean" or "Moved to global.css: @font-face, :root".

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

## Editing files

**Use the editor/write tools** (Write, Edit, `str_replace`, `create_file`) to create and change theme files. Do not rewrite files with python, `sed`, `awk`, or heredocs while the editor tools work: they make noisy diffs, can change line endings (CRLF/LF) and encoding on Windows paths such as `C:\laragon\...`, and can touch lines that were not meant to change. Keep each file's existing line endings and encoding.

Bash is for commands that are not file edits: `git`, `php -l`, `jq`, `node --check`, `openssl rand` (keys), `date -u +%s` (`modified`), and copying files (design assets, WooCommerce templates). Copy with bash, then change the copy with the editor tools.

**If an editor tool fails:**

1. **Read the error and fix the cause.** Typical causes: the file already exists (use an edit instead of create), the text to replace does not match or is not unique (re-read the file, copy the exact text, include enough context to make it unique), the path is wrong or points at a read-only location. Retry once. Do not loop.
2. **Still failing: use a minimal shell fallback.**
   - New file: write it with a quoted heredoc (`<<'EOF'`) so `$` and backticks in PHP are not expanded.
   - Existing file: replace only the exact string, never a regex, and check it matched exactly once before writing. Check the file's line endings first (`file <path>`) and write them back unchanged.
3. **Verify the result.** Re-read the changed region, run `git diff` for tracked files to confirm only the intended lines changed, then run the usual checks (`php -l`, `jq empty`, `node --check`).
4. **Tell the user in one line** that the editor tool failed and a shell fallback was used, and why.
5. **If the shell fails too** (permissions, read-only folder, locked file): stop and report the error. Do not change permissions, write the file somewhere else, or work around the block. Ask the user, with choices such as "I'll fix permissions and you tell me" or "Stop here".

## Working rules

1. Confirm required plugins per "Plugin prerequisites" before writing code that depends on them.
2. Read the relevant skill (`tmdr-page-template`, `tmdr-acf-field-group`, `tmdr-woocommerce`) before changing that area.
3. Search existing helpers, components, and CSS/JS before writing new code. The SCSS mixins and functions described in `assets/dummy/intro.php` (`color()`, `vwUnit()`, `typo()` and others) are not available in a plain-CSS project; write the equivalent CSS directly.
   Exception: never add thin wrapper helpers around native functions. Call WordPress and ACF directly in the template (`get_field()`, `wp_get_attachment_image()`, `wp_get_nav_menu_items()`, `get_posts()`, `sanitize_title()`, `wpautop()`, `wp_kses_post()`), and Polylang directly on multilingual sites (`pll__()`, `pll_the_languages()`). A helper in `inc/functions-custom.php` is only justified for site-specific logic shared by several templates (e.g. `tmdr_whatsapp_url()`, `tmdr_active_offer_ids()`, `tmdr_faq_groups()`).
4. Trace the complete caller flow for behavior changes.
5. Make the smallest change in the correct source file, using the editor tools (see "Editing files"). Exception: on a new project, starter demo files may be rewritten to match the reference once the user agreed (see "Starter code is a template, not a finished project").
6. Follow the "Frontend conventions" section above, and match neighboring files.
7. Avoid new dependencies unless an installed dependency cannot solve the requirement.
8. Do not modify unrelated existing work. Starter demo files in a new project are not "existing work" once the user agreed to modify them.
9. Escape all output (see `tmdr-page-template`, "Escaping output").
10. Verify before considering a change complete:
    - PHP: `php -l path/to/file.php`
    - JSON: `jq empty path/to/file.json`
    - JS: `node --check path/to/file.js` (for plain scripts)
    - CSS: load the page and check the browser console, or run the project's linter if one exists.