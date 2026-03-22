---
name: make-site-multilingual
description: >-
  Convert a non-multilingual site to use the Rosey/RCC/CloudCannon stack for
  translation management. Use when the user wants to add multilingual support,
  add translations, internationalize a site, or set up Rosey with CloudCannon.
---

# Make a Site Multilingual with Rosey/RCC/CloudCannon

Step-by-step workflow for converting a single-language site into a multilingual site managed through CloudCannon's Visual Editor using Rosey and the Rosey CloudCannon Connector (RCC).

## Prerequisites

- The site must build to static HTML (SSG). Rosey operates on built output, not source files.
- CloudCannon must be the CMS (RCC depends on CloudCannon's JS API), and at least some of the pages on the site should be editable in CloudCannon's visual editor. A significant part of what the RCC does is connecting the locales files Rosey uses, to the parts of the page that they're used in - in CloudCannon's visual editor.
- Confirm which locales the user wants to support (e.g., `fr,de,es`).

## Phase 1: Audit the Site

Before touching code, understand what needs to be translated.

1. **Find all translatable text.** Search templates, components, and layouts for user-visible text:
   - Headings, paragraphs, button labels, link text, alt text
   - Text in markdown frontmatter that renders into HTML (titles, descriptions)
   - Text in global data files (navigation labels, footer text, company info)
   - Hardcoded strings in template files

2. **Identify the build output directory.** Common values: `dist/`, `_site/`, `build/`, `out/`. Check the framework config (e.g., `astro.config.mjs`, `eleventy.js`).

3. **Map out the page/content structure.** Understand how pages are generated -- dynamic routes, content collections, data-driven pages. This determines how to set `data-rosey-root` values.

## Phase 2: Install Dependencies

**Fastest path (recommended for agents):** Run the setup wizard in non-interactive mode. It handles installation, postbuild creation, `rosey.yml`, and CloudCannon config in one command with no prompts:

```bash
npx rosey-cloudcannon-connector init --yes --locales fr,de
```

Override any default as needed:

```bash
npx rosey-cloudcannon-connector init --yes \
  --locales fr,de,es \
  --default-language en \
  --build-dir dist \
  --rosey-dir rosey \
  --content-at-root \
  --collection
```

The manual steps below (Phases 3-6) are still needed for tagging templates and importing the RCC, but the wizard covers everything in Phase 5 (Configure CloudCannon) automatically.

**Interactive mode** (if a human is running it):

```bash
npx rosey-cloudcannon-connector init
```

The wizard prompts for locales, default language, build dir, and more.

**Manual install** (if you prefer or need to skip the wizard entirely):

```bash
npm install rosey
npm install rosey-cloudcannon-connector@github:tomrcc/rcc-v2
```

> RCC v2 is currently installed from the GitHub repo while in development. This will change to a regular npm package once published.

## Phase 3: Tag Templates with `data-rosey`

Add Rosey attributes to the HTML output. Work from the outermost layout inward.

### 3a. Set up `data-rosey-root` on page containers

Each page needs a root namespace so keys don't collide across pages. Add `data-rosey-root` to a top-level element (typically `<main>`) using the page's slug or path:

```html
<main data-rosey-root="about">
```

For dynamic pages, use the slug variable:

```astro
<main data-rosey-root={slug}>
```

### 3b. Add `data-rosey-ns` for component namespacing

Wrap reusable sections with `data-rosey-ns` to namespace their keys:

```html
<section data-rosey-ns="hero">
  <h1 data-rosey="title">Welcome</h1>
  <p data-rosey="description">Our product helps you...</p>
</section>
```

This produces keys like `index:hero:title` and `index:hero:description`.

### 3c. Add `data-rosey` to translatable elements

Tag every element containing user-visible text:

```html
<h1 data-rosey="title">Welcome to Our Site</h1>
<p data-rosey="description">We build great products.</p>
<a data-rosey="cta_text" href="/signup">Get Started</a>
```

**Important considerations:**
- `data-rosey` only captures the **text content** of the element
- For elements that already have CloudCannon `data-editable` / `data-prop` attributes, add `data-rosey` alongside them -- they serve different purposes
- Use `data-rcc-ignore` on elements that have `data-rosey` but should not appear in the locale switcher
- **Skip proper nouns**: Do not tag names, author names, designations, or other identity values that should remain the same across locales
- **Rich-text body content**: Place `data-rosey` on the `<editable-text>` element, not a parent wrapper, so Rosey captures only the inner HTML and not the custom element tags

### 3d. Handle shared/global content

For content shared across pages (navigation, footer), decide on a namespace strategy:
- Option A: Use `data-rosey-root=""` to reset the namespace, giving global elements flat keys
- Option B: Use a dedicated root like `data-rosey-root="global"` on the wrapper

## Phase 4: Import RCC in the Root Layout

Add the RCC import to the site's root layout. It must be lazy-loaded and only run inside the CloudCannon editor:

```html
<script>
  if (window?.inEditorMode) {
    import("rosey-cloudcannon-connector");
  }
</script>
```

Place this in the `<body>` of the root layout, after the main content.

## Phase 5: Configure CloudCannon

> **If you ran `npx rosey-cloudcannon-connector init` in Phase 2**, the wizard has already handled 5a, 5b, and optionally 5c below. Skip to Phase 6.

### 5a. Add `data_config` for locale files

In `cloudcannon.config.yml`, add an entry for each locale. The key **must** follow the format `locales_{code}`:

```yaml
data_config:
  locales_fr:
    path: rosey/locales/fr.json
  locales_de:
    path: rosey/locales/de.json
```

### 5b. Update the postbuild script

Replace or update `.cloudcannon/postbuild` with the Rosey pipeline. Adjust `--source dist` to match the build output directory. On first run, add `--locales fr,de` to create the initial locale files; subsequent runs auto-detect:

```bash
#!/usr/bin/env bash

npx rosey generate --source dist
npx rosey-cloudcannon-connector write-locales --source rosey --dest dist
mv ./dist ./_untranslated_site
npx rosey build --source _untranslated_site --dest dist --default-language-at-root
cp -r _untranslated_site/_rcc dist/_rcc
```

This script:
1. Generates `rosey/base.json` from the built HTML
2. Creates/updates locale JSON files (preserving existing translations, removing stale keys) and writes the locale manifest to `dist/_rcc/locales.json`
3. Moves the original build aside
4. Rebuilds the site with Rosey translations injected
5. Copies `_rcc/locales.json` back into `dist/` (Rosey excludes `_`-prefixed directories from its output, but the RCC client needs this file at runtime)

### 5c. (Optional) Expose locales as a browsable collection

If editors need to edit translations that don't appear visually on a page (HTML attributes, `<head>` values, etc.), expose the locale files as a CloudCannon collection:

```yaml
collections_config:
  locales:
    path: rosey/locales
    name: Locales
    icon: translate
    disable_add: true
    disable_add_folder: true
    disable_file_actions: true
    _inputs:
      value:
        type: html
        label: Translation
        cascade: true
      original:
        hidden: true
        cascade: true
      _base_original:
        disabled: true
        cascade: true
```

This makes locale files browsable in the CloudCannon sidebar. The `_inputs` config shows `value` as an HTML editor, hides the internal `original` field, and displays `_base_original` as read-only context.

## Phase 6: Generate and Verify

1. **Build the site locally:**
   ```bash
   npm run build
   ```

2. **Generate the Rosey base file:**
   ```bash
   npx rosey generate --source dist
   ```

3. **Create locale files** (first time, specify locales explicitly; subsequent runs auto-detect):
   ```bash
   npx rosey-cloudcannon-connector write-locales --source rosey --dest dist --locales fr,de
   ```

4. **Verify `rosey/base.json`** -- confirm it contains all expected keys with correct namespacing.

5. **Verify locale files** (e.g., `rosey/locales/fr.json`) -- confirm keys match `base.json` and `original`/`value` fields are populated.

6. **Test the full pipeline:**
   ```bash
   mv ./dist ./_untranslated_site
   npx rosey build --source _untranslated_site --dest dist --default-language-at-root
   cp -r _untranslated_site/_rcc dist/_rcc
   ```
   Verify the translated site output in `dist/` and confirm `dist/_rcc/locales.json` exists.

## Phase 7: Split-by-Directory for Body Content (Optional)

For pages with large body content (blog posts, articles, documentation pages), Rosey's single-key approach is impractical -- the entire body becomes one massive translation key. A better approach is **split-by-directory**: create a separate content collection per locale and let the SSG build those pages natively to the correct locale URLs.

### When to use split-by-directory

- The page has long-form body content (blog posts, documentation, case studies)
- The body content uses MDX components or rich formatting that doesn't map well to a single Rosey key
- Editors need a familiar content editing experience (CloudCannon's Content Editor) rather than the Visual Editor's inline translation

### How it works

1. **Create per-locale content directories** mirroring the English collection (e.g., `src/content/blog_fr/`, `src/content/blog_de/`). Seed with copies of the English files as starting points.

2. **Define Astro content collections** for each locale in `content.config.ts`, using the same schema as the English collection.

3. **Create locale routes** using a dynamic `[locale]` parameter:
   - `src/pages/[locale]/blog/[...slug].astro` -- individual posts
   - `src/pages/[locale]/blog/[...page].astro` -- paginated index
   - `src/pages/[locale]/blog/tag/[tag]/[...page].astro` -- tag filter pages
   - `getStaticPaths` iterates locale codes and fetches from the matching collection.

4. **Extract shared rendering components** to avoid duplicating template logic across English and locale route files. Pass `locale` as a prop for locale-aware links, date formatting, and collection selection.

5. **Align Rosey roots** so shared UI strings translate correctly. Locale pages must set `data-rosey-root` to the **English-equivalent** path (e.g., `blog/my-post`, not `fr/blog/my-post`). Add a `roseyRoot` prop to the page layout and compute it by stripping the locale prefix.

6. **Remove `data-rosey` from body content and frontmatter-driven fields** (title, description, tag) since those are natively translated in the locale files. Keep `data-rosey` on shared UI strings (breadcrumbs, sidebar headings, share buttons) so Rosey still translates them.

7. **Add CloudCannon collections** for each locale blog (`blog_fr`, `blog_de`) in `cloudcannon.config.yml`, with `url: /{locale}/blog/[full_slug]/`.

8. **Create a locale config utility** (`src/lib/locales.ts`) mapping locale codes to collection names, date locale strings, and labels. Single source of truth for adding new locales.

### Rosey coexistence

The postbuild script is unchanged. When Rosey encounters an existing page at a locale URL (e.g., `/fr/blog/my-post/`), it **respects the existing content** and only translates elements with `data-rosey` attributes. This means:
- Body content stays as-is (natively French/German from the locale collection)
- Shared UI strings (breadcrumbs, "Share this article:", "Latest News") get translated from the Rosey locale files
- Non-blog pages continue using Rosey for full translation as before

## Phase 8: Visitor-Facing Locale Picker (Optional)

**Ask the user first:** "Would you like a visitor-facing locale picker (language switcher) added to the site, or do you already have one / prefer to bring your own?"

If the user declines or has their own, remind them that any links pointing to locale URLs must have `data-rosey-ignore` to prevent Rosey from rewriting them (see gotcha below).

If the user wants one:

1. **Create a locale picker component** in the site's navigation area. The component should:
   - Parse the current page URL to detect the active locale (check if the first path segment is a known locale code)
   - Strip the locale prefix to get the base path
   - Render links for each locale: `/{locale}{basePath}` for non-default, `{basePath}` for the default language
   - Add **`data-rosey-ignore`** on every `<a>` element (critical -- prevents Rosey from double-prefixing locale URLs)
   - Add `hreflang` attributes for SEO
   - Include a small client-side script to fix the active-state highlight on Rosey-generated pages (since build-time HTML always reflects the English page's active state)

2. **Place the component** in both the desktop nav and mobile nav.

See the "Visitor-Facing Locale Picker" section in `rosey-multilingual-context.mdc` for a full example.

## Checklist

- [ ] All user-visible text elements have `data-rosey` attributes
- [ ] Each page/route has a `data-rosey-root` set to a unique slug
- [ ] Reusable sections use `data-rosey-ns` for namespacing
- [ ] RCC is imported conditionally in the root layout (`window?.inEditorMode`)
- [ ] Root `<html>` tag has `lang="{defaultLanguage}"` set (e.g. `<html lang="en">`)
- [ ] `cloudcannon.config.yml` has `data_config` entries for each locale (`locales_{code}`)
- [ ] `.cloudcannon/postbuild` runs the full Rosey pipeline
- [ ] `write-locales --dest` generates the locale manifest at `{build_dir}/_rcc/locales.json`
- [ ] `rosey/base.json` generates with correct keys
- [ ] Locale files are created with correct structure

## Learnings and Gotchas

> This section is a living document. When you discover new patterns, issues, or improvements while making sites multilingual, **ask the user** before appending them here. See the living-docs-protocol rule.

- **Stale translation detection.** When `original` and `_base_original` differ in a locale file, the RCC shows an amber dashed border and warning badge in the Visual Editor. Editors can either update the translation or click "Mark as reviewed" to acknowledge the change.
- **Empty `data-rosey-root`.** Setting `data-rosey-root=""` on an element resets the namespace -- child keys won't inherit anything above it. Useful for global components like navigation and footer.
- **Key collisions.** If two pages have the same `data-rosey-root` value and the same element keys, their translations will collide. Always use unique root values (typically the page slug).
- **Rosey operates on built HTML.** It does not see source files, markdown, or frontmatter directly. If text from frontmatter is rendered into the HTML, Rosey will pick it up from the rendered output.
- **`write-locales` preserves existing translations but removes stale keys.** Running it again adds new keys and removes keys that no longer exist in `base.json`. It never overwrites existing `value` fields on keys that still exist.
- **Snapshot boundary.** The RCC clones the boundary container (`<main>` by default, or `[data-rcc]`) when switching locales. Content outside the boundary (navigation, footer) is not affected by locale switching. If you need a custom boundary, add `data-rcc` to the desired container.
- **`data-rosey` must go on the innermost text element.** Rosey captures `innerHTML` of the tagged element. If `data-rosey` is placed on an outer element (e.g., `<h2>`) that wraps `<editable-text>` or other custom elements, the captured original will include those wrapper tags. Always place `data-rosey` on the `<editable-text>` element itself, or on the innermost element containing just the text.
- **Shared components need explicit `data-rosey` passthrough.** Components like Heading, Button, or Text that wrap content in `<editable-text>` cannot simply spread `data-rosey` via rest props -- it would land on the outer tag, not the inner `<editable-text>`. Destructure `data-rosey` alongside `data-editable`/`data-prop` and forward it to the inner wrapper element.
- **Content block namespacing with `_name` + index.** For CMS page builders using `content_blocks`, add `data-rosey-ns` with `{block._name}-{index}` (e.g., `global-hero-0`, `global-feature-1`) on each block wrapper. This prevents key collisions when a page has multiple blocks of the same type while keeping keys human-readable.
- **Nav/footer use `data-rosey-ns`, not `data-rosey-root`.** Navigation and footer sit outside `<main>` and have no `data-rosey-root` ancestor. Use `data-rosey-ns="nav"` / `data-rosey-ns="footer"` for organization. Rosey deduplicates identical keys across pages automatically, so no root is needed.
- **Slug derivation via `Astro.url.pathname`.** Rather than threading a slug prop through the layout chain, use `Astro.url.pathname.replace(/^\/|\/$/g, '') || 'index'` directly in the component that renders `<main>`. Works for any page type and keeps changes self-contained.
- **Array items within blocks need nested `data-rosey-ns`.** For repeating items (testimonials, team members, FAQ, pricing features, etc.), add `data-rosey-ns={String(i)}` on each array item wrapper. This produces keys like `index:global-testimonial-2:0:author` and prevents collisions within the same block.
- **Rosey excludes `_rcc/` from its build output.** The `write-locales --dest` command writes `_rcc/locales.json` into the build dir, but `rosey build` does not carry files prefixed with `_` into its output. The postbuild script must copy the `_rcc/` directory back after the Rosey build: `cp -r _untranslated_site/_rcc dist/_rcc`.
- **Don't translate names.** Props that represent proper nouns — author names, person names, designations/titles — should **not** get `data-rosey` attributes. These are not translatable text; they're identity values that stay the same across locales.
- **Blog body content: put `data-rosey` on `<editable-text>`, not the wrapper.** For rich-text body content (e.g., `<editable-text data-prop="@content">`), place `data-rosey` directly on the `<editable-text>` element. Putting it on a parent `<div>` causes Rosey to capture the `<editable-text>` wrapper tags as part of the original, which corrupts the translation.
- **Auto-derive `data-rosey` from `data-prop` in reusable building blocks.** For component-heavy sites, modify core building blocks (Heading, Text, SimpleText, Button, ListItem, Testimonial) to automatically derive `data-rosey` from the existing `data-prop` value. Pattern: `const roseyProp = Astro.props["data-rosey"]; const effectiveDataRosey = roseyProp === false ? null : (roseyProp ?? effectiveDataProp ?? null);` then spread `roseyAttributes` on the inner text element. This avoids manually tagging every component instance across all pages.
- **Destructure `data-rosey` in component props to prevent DOM leaking.** When a component uses `...htmlAttributes` (rest spread) on an outer wrapper, any undeclared prop ends up in `htmlAttributes`. If `data-rosey` isn't explicitly destructured (`"data-rosey": roseyProp`), it leaks onto the outer element instead of reaching the inner text element where it's needed. Always destructure it alongside `data-prop`.
- **`data-rosey={false}` for per-instance opt-out.** When auto-deriving from `data-prop`, pass `data-rosey={false}` on instances that should not be translated (e.g., `<Heading data-prop="name" data-rosey={false}>` for team member names). The auto-derive logic checks for `false` and suppresses the attribute.
- **Non-editable components need explicit `data-rosey`.** When `editable={false}` on a building block, the auto-derive mechanism won't produce a `data-rosey` attribute (since there's no `data-prop`). Hardcoded strings in non-editable components (e.g., "All" filter buttons, "No posts found" fallback text) need an explicit `data-rosey="key"` prop.
- **Multi-level nav link keys need depth encoding.** For navigation with dropdowns and nested submenus, encode the nesting depth in the `data-rosey` key: `link-{topIndex}` for top-level, `link-{topIndex}-{childIndex}` for children, `link-{topIndex}-{childIndex}-{grandchildIndex}` for grandchildren. This prevents collisions between nav levels while keeping keys deterministic.
- **Split-by-directory for body-heavy content.** Blog posts, articles, and documentation pages are better translated as separate per-locale content collections than as single Rosey keys. Create `blog_fr/`, `blog_de/` directories, define Astro collections, and build locale pages natively. Rosey still handles shared UI strings on those pages.
- **Rosey root alignment for locale pages.** When Astro builds a page at `/fr/blog/my-post/`, `Page.astro` derives `data-rosey-root` from the URL as `fr/blog/my-post`. This creates keys like `fr/blog/my-post:breadcrumb-blog` which don't match the locale file entries (keyed as `blog/my-post:breadcrumb-blog`). Fix by adding a `roseyRoot` prop to the page layout that lets locale pages pass the English-equivalent root.
- **Rosey merges with pre-existing locale pages.** When Rosey encounters an already-built page at a locale URL during `rosey build`, it respects the existing content and only translates `data-rosey` elements. It does not create a duplicate or overwrite the page. This enables the hybrid approach where body content comes from locale collections and UI strings come from Rosey.
- **Use snake_case for collection names.** Astro and CloudCannon both handle kebab-case, but snake_case (`blog_fr`, `blog_de`) is more consistent with CloudCannon conventions like `data_config` keys (`locales_fr`, `locales_de`).
- **`data-rosey={false}` on frontmatter-driven fields in shared templates.** When a shared blog template is used for both English and locale pages, frontmatter-driven fields (title, description, tag) must suppress `data-rosey` with `data-rosey={false}`. Otherwise Rosey would overwrite the already-translated content from the locale collection with whatever is in the Rosey locale file.
- **Rosey rewrites internal links on generated pages but not on pre-existing pages.** When Rosey copies a page to a locale URL (e.g., `/about/` to `/fr/about/`), it rewrites all `<a href>` values that match known site URLs to prepend the locale prefix. However, for split-by-directory pages that already exist at the locale URL, Rosey only touches `data-rosey` elements — links are left as-is. This means nav links on split-by-directory pages still point to English paths (e.g., `/blog/` instead of `/fr/blog/`).
- **Locale picker links need `data-rosey-ignore`.** Any visitor-facing locale switcher must add `data-rosey-ignore` to its `<a>` elements. Without it, Rosey rewrites the English link (e.g., `/about/`) to the current locale (e.g., `/fr/about/`) on generated pages, breaking the "switch to English" action. `data-rosey-ignore` tells Rosey to leave the link unchanged.
- **Locale picker active state needs client-side JS.** Build-time HTML always reflects the English page's perspective. On Rosey-generated locale pages, the "English" link would incorrectly appear active. A small client-side script that compares link hrefs against `window.location.pathname` fixes this at runtime.
- **Bookshop button `data-rosey` captures SVG icon markup.** When `data-rosey` is placed on an `<a>` or `<button>` that contains both text and a Bookshop icon component (e.g., arrow icons), Rosey captures the full `innerHTML` including the rendered SVG and Bookshop live-edit comments. The translation `value` must preserve the icon markup; only the text portion should change. For cleaner translations, wrap the button text in a `<span data-rosey="button_text">` and leave the icon outside, but this requires restructuring the button component. For Bookshop sites, the trade-off is acceptable since editors use the RCC Visual Editor rather than editing raw JSON.
- **Eleventy slug derivation for `data-rosey-root`.** In Eleventy, `page.url` gives the URL path (e.g., `/booking/`). Derive a Rosey root slug with `{% assign rosey_slug = page.url | replace: '/', '' %}` and fall back for the index page: `{% if rosey_slug == '' %}{% assign rosey_slug = 'index' %}{% endif %}`. This works for flat URL structures; nested paths (e.g., `/blog/my-post/`) would need a split/join approach instead of a blanket replace.
- **Bookshop `page.eleventy.liquid` is the ideal block namespacing point.** For Bookshop sites, the shared `page.eleventy.liquid` template (which loops `content_blocks`) is the single best place to add `data-rosey-ns` wrappers. Use `{% assign block_ns = block._bookshop_name | split: "/" | last | append: "-" | append: forloop.index0 %}` to derive names like `left-right-simple-0`, `price-list-1`. One change covers all content blocks across all pages.
- **Duplicate desktop/mobile nav elements share the same Rosey key.** When a nav has both desktop and mobile versions of the same links (common in responsive designs), both can use the same `data-rosey` key (e.g., `link-0`). Rosey records them as multiple occurrences of the same key on the same page. Both instances get the same translated value, which is the desired behavior.
- **Eleventy locale picker uses `page.url | split: "/"`** to parse the path and detect the current locale. The first meaningful segment is at index 1 (`path_segments[1]`) since index 0 is empty from the leading `/`. This differs from the Astro pattern (`Astro.url.pathname`) documented elsewhere in this file.
