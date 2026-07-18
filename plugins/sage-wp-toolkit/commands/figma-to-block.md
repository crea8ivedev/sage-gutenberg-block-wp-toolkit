---
description: Scaffold a dynamic Gutenberg block from a Figma URL via Figma MCP, with pixel-perfect screenshot verification
---

# Make Acorn Block — Figma MCP Dynamic Block Generator

Generate a fully dynamic Gutenberg block from a **Figma URL** (frame, component, or section) or raw HTML + optional notes.

---

## Usage

```
/figma-to-block
```

No arguments needed. Claude collects inputs interactively.

---

## Instructions

### Step -1 — Verify Figma MCP connection (hard gate)

**Before anything else**, when the input is a Figma URL, verify the Figma MCP server is connected and authenticated: call the Figma MCP `whoami` tool (or any cheap Figma MCP tool). 

If no Figma MCP tools are available, or the call fails with an auth/connection error:

1. Print exactly this and **STOP — do not continue to any other step, do not ask for inputs, do not write any files**:

   > ❌ **Figma is not connected.** This command needs the Figma MCP server.
   > Connect it first: run `/mcp` and authenticate the `figma` server (bundled with this plugin), or add the Figma connector, then re-run `/figma-to-block`.

2. Do NOT fall back to guessing the design from the URL, a cached screenshot, or memory. No Figma connection = no output.

(If the user provided raw HTML instead of a Figma URL, this gate does not apply — skip to Step 0.)

### Step 0 — Collect inputs

The user invoked this command with arguments: `$ARGUMENTS`

**Parse arguments for:**
- Block name (kebab-case slug, e.g. `hero-section`)
- Input type — one of:
  - **Figma URL** — a `figma.com/design/...` URL pointing to a frame, section, or component
  - **HTML** — raw markup pasted inline
- Notes (anything after "Notes:" or "Note:")

**If block name is missing** → ask: "Block slug? (kebab-case, e.g. `hero-section`)"
**If neither Figma URL nor HTML provided** → ask: "Share a Figma URL (frame/section/component) or paste the HTML for this block."
**If notes absent** → fine, proceed without.

Wait for all required inputs before proceeding. Do NOT write any files until block name + source (Figma URL or HTML) are both known.

---

### Step 1 — Derive name variants

From the kebab-case block name:
- `kebabName` = `hero-section`
- `studlyName` = `HeroSection`
- `camelName` = `heroSection`
- `humanName` = `Hero Section`

---

### Step 2 — Analyze Figma design and generate HTML

**Skip this step if the user provided raw HTML — go directly to Step 3.**

Use the Figma MCP tools to extract the design. Call tools in this order:

1. **`get_design_context`** (or `mcp__claude_ai_Figma__get_design_context`) — pass the Figma URL to get layout, components, styles, colors, typography, spacing, and content
2. **`get_screenshot`** (or `mcp__claude_ai_Figma__get_screenshot`) — fetch a visual screenshot of the frame. Keep this screenshot — it is the ground-truth target for the Step 9.5 verify loop, not just a reference.
3. If variables or tokens are needed: **`get_variable_defs`**

Extract from the Figma response:
- **Colors** — hex values, opacities, gradients
- **Typography** — font families, sizes, weights, line-heights, letter-spacing
- **Spacing** — padding, margin, gap values → map to Tailwind token utilities (with `--spacing:1px`, `p-24` = 24px). NEVER arbitrary `[...]` values.
- **Layout** — flex/grid direction, alignment, column structure, content-frame max-width
- **Content** — all visible text strings, image placeholders, icons
- **Components** — identify reusable sub-components

**Responsive — read real Figma frames.** If the design includes tablet/mobile frames, extract their EXACT values (font size, padding, gap, columns) at 1440/1024/768 and use those. Only if NO responsive frame exists may you fall back to scaling the heading ladder down — and when you do, note it inline as an assumption. Never auto-invent breakpoint values Figma already specifies (per CSS rule 21).

Generate production-ready, semantic HTML with Tailwind CSS v4 token utility classes from the extracted design data.

**Follow `html-css-js-rules` for all HTML markup, semantics, class usage, accessibility, image/link/button attributes, and responsive conventions.**

After generating HTML, proceed to Step 2.6 (fonts) then Step 3.

---

### Step 2.6 — Download + verify fonts (CRITICAL — #1 accuracy fix)

**A `@font-face` pointing at a missing `.woff2` = browser falls back to a system font = every text metric wrong = whole UI looks off. This is the single biggest cause of inaccurate output. Do NOT skip.**

For every font family + weight the design uses:

1. **Identify** the fonts from `get_design_context` / `get_variable_defs` (family, weights, styles).
2. **Obtain the `.woff2`:**
   - Try Figma assets first: `download_assets` / `upload_assets` MCP tools.
   - If the font is a known web font (Google Fonts etc.), fetch the `.woff2` from its source.
   - Prefer subset `.woff2` (used chars only) per Performance rule 5.
3. **Save** to `resources/fonts/` with kebab names matching the `@font-face` `src` (e.g. `poppins-regular.woff2`, `big-caslon-cc-bold.woff2`).
4. **Verify files exist** — list `resources/fonts/` and confirm every `@font-face` `src` resolves to a real file. Never leave a `@font-face` whose file is absent.
5. If a font `.woff2` genuinely cannot be obtained, STOP and tell the user which font is missing — do not silently ship a broken fallback.

`@font-face` blocks (with `font-display:swap`) live in `base.css` — the only raw-CSS exception.

---

### Step 3 — Analyze HTML → define attributes

Scan every element in the HTML. For each piece of **user-editable content**, define one attribute.

#### Element → attribute mapping

| HTML element / pattern | Attribute type | WP control | Notes |
|---|---|---|---|
| `<h1>` … `<h6>` text | `string` | `TextControl` | short text |
| `<p>` text | `string` | `TextareaControl` | long text |
| Inline `<span>`, `<strong>`, `<em>` text | `string` | `TextControl` | short text |
| `<a href="...">` | Two attrs: `xUrl` (string) + use existing text attr | `TextControl` for URL | if link text is standalone, add `xText` attr too |
| `<img src="..." alt="...">` | Three attrs: `xImageUrl` (string), `xImageId` (number, default 0), `xImageAlt` (string) | `MediaUpload` for url/id, `TextControl` for alt | |
| Dynamic background image | Three attrs: `bgImageUrl` (string), `bgImageId` (number, default 0), `bgImageAlt` (string) | `MediaUpload` | |
| `<button>` or `.btn` text | `string` | `TextControl` | short text |
| Any small label / pre-title / tag text | `string` | `TextControl` | short text |
| Decorative elements, divs with no text, SVG icons | **skip** — no attribute | — | not user-editable |
| Tailwind class names | **skip** | — | hardcoded in templates |
| `data-*` attributes | **skip** unless they hold content | — | |

#### Attribute naming rules
- camelCase, descriptive, contextual: `heroTitle`, `ctaButtonText`, `ctaButtonUrl`, `teamImageUrl`
- If same element type appears multiple times: number them → `content1`, `content2` or prefix by section → `leftTitle`, `rightContent`
- Image triplet always named: `<purpose>ImageUrl`, `<purpose>ImageId`, `<purpose>ImageAlt`

#### Defaults
- Extract **exact text / URL / src** from the HTML as the default value
- Image defaults: `url: ''`, `id: 0`, `alt: ''`

#### Apply user notes
After auto-detection, apply any note-driven changes:
- Add extra attributes not visible in design/HTML
- Override control type
- Add extra fields
- Change grouping or panel labels

#### Control selection summary
- Short text (< ~80 chars default) → `TextControl`
- Long text (paragraph) → `TextareaControl`
- Image → `MediaUpload` + `MediaUploadCheck` + `TextControl` for alt
- URL → `TextControl` (label "URL")
- Toggle show/hide → `ToggleControl`
- Color → `ColorPicker` or `ColorPalette`
- Number / size → `RangeControl`

#### Grouping for InspectorControls
Group attributes into `PanelBody` sections based on HTML structure:
- Clear left/right columns → "Left Column" / "Right Column"
- Header + body + CTA → "Header" / "Content" / "Call to Action"
- Flat → one panel "Block Settings" or two: "Content" / "Media"
- Image fields always in own panel "Media" or grouped with their section

---

### Step 3.5 — Bootstrap token system + CSS architecture (run ONCE per project)

**CRITICAL — do this BEFORE writing any block CSS.** `html-css-js-rules` mandates a token-driven, split-file CSS system. Most repos do NOT have it yet (only `app.css` + `editor.css` with a bare `@theme` of breakpoints). Without it, every block is forced into `[...]` arbitrary values and inline font utilities — which VIOLATES the rules. Bootstrap the system first, then all block CSS can comply.

**Detection:** check whether `resources/css/_tokens.css` exists.
- **Exists** → token system already bootstrapped. Skip to Step 4. Only ADD missing tokens (new colors/fonts/sizes from this Figma) into `_tokens.css`.
- **Missing** → run the full bootstrap below.

**Bootstrap steps:**

1. **Extract ALL design tokens from Figma** via `get_variable_defs` + `get_design_context` — every unique color, font family, font size, X-axis (container) spacing, Y-axis (section) spacing, border radius.

2. **Create `resources/css/_tokens.css`** — a single `@theme {}` block holding (per CSS rule 10):
   - Breakpoints (the standard list from `html-css-js-rules`)
   - `--spacing: 1px;` — so every `p-*/m-*/gap-*/px-*/py-*` number = that many px (removes need for brackets)
   - `--color-{name}` for each Figma color (semantic names, no hardcoded example values)
   - `--font-{name}` for each Figma font family
   - `--text-heading-1` … only the heading steps Figma actually uses (largest→h1 descending) + `--text-{size}` for every non-heading size.
     **EXACT VALUES ONLY — never invent numbers to "fill the h1–h6 scale."** If Figma supplies only 48px and 16px, define only those. Do NOT guess `--text-heading-1:56px` etc. to complete the ladder (per HTML rule 16). Add a heading step only when a real Figma value needs it.
   - `--radius-{n}` for each corner radius
   - Tailwind v4 auto-generates `bg-{color}`, `text-{color}`, `font-{name}`, `text-heading-1`, `text-16`, `rounded-10` etc. from these — so NO `[...]` needed anywhere.

3. **Create the 4 split files** (per CSS rules 11–14):
   - `resources/css/base.css` — `@font-face` blocks, `h1..h6`/`.h1..h6` (using `text-heading-*`), `.content p` + `.content p + p`, `.title-{color}` + `.content-{color}` color helpers, `.container-fluid` (X padding), `.general-padding` (Y padding, responsive).
   - `resources/css/component.css` — `.btn` + `.btn-{variant}` (with hover transition), inputs/textarea/select.
   - `resources/css/layout.css` — header/footer (placeholder comment ok).
   - `resources/css/utilities.css` — block/section-specific classes (this is where each block's CSS goes — see Step 4).

4. **Rewrite `app.css` AND `editor.css`** to import in this exact order (keep `app.css`'s existing `theme(static)` variant; `editor.css` uses plain import):
   ```css
   @import "tailwindcss";           /* app.css: @import "tailwindcss" theme(static); */
   @import "./_tokens.css";
   @source "../../app/**/*.php";
   @source "../**/*.blade.php";
   @source "../**/*.js";
   @import "./base.css";
   @import "./component.css";
   @import "./layout.css";
   @import "./utilities.css";
   ```
   Both entries import the SAME split files → editor preview automatically matches frontend, and block CSS is written ONCE (not duplicated).

5. **Migrate any pre-existing blocks** (e.g. `hero-section`) off `[...]` values into the token system + move their section CSS into `utilities.css`. Strip font utilities out of their Blade/JSX markup (fonts belong in CSS via `@apply`).

6. **Verify:** run `npm run build` — must compile with no unknown-utility errors before proceeding.

---

### Step 4 — Write block CSS

**Path:** `resources/css/utilities.css` (block/section-specific styles live here — see Step 3.5).

Global concerns (headings, `.content`, `.title-{color}`/`.content-{color}`, `.btn`, container, general-padding) are already defined in `base.css`/`component.css` from Step 3.5 — **reuse them, do NOT redefine per block.** Only add NEW color/font/size tokens to `_tokens.css` if this design introduces them.

**Because both `app.css` and `editor.css` `@import "./utilities.css"`, you write the CSS ONCE here — it applies to editor + frontend identically. Do NOT hand-copy CSS into two files anymore.**

**Follow `html-css-js-rules` for all CSS authoring conventions — `@apply` only, token classes only.**

Hard constraints (enforce strictly — these are the rules the old workflow broke):
- **NEVER use `[...]` arbitrary-value syntax** (`text-[48px]`, `bg-[#fff]`, `rounded-[50px]`). Use the tokens from `_tokens.css` (`text-heading-1`, `bg-cream`, `rounded-full`). If a needed value has no token, ADD it to `_tokens.css` first.
- **Use EXACT Figma values — never snap to the nearest existing token.** If Figma says 55px padding and only a 50px container exists, add the 55px step; do NOT reuse 50px because it is "close." Rounding to nearest is the #1 cause of layout drift (per HTML rules 5 + 16).
- **NEVER put font/color/typography utilities in HTML markup** — only `p-* m-* max-w-* flex flex-col flex-row gap grid grid-cols-*` are allowed in markup (HTML rule 12). `font-*`, `text-*`, `bg-*` go in CSS via `@apply` on the element's class.
- **Centered content column** → cap width with a single `max-w-*` token on an inner wrapper (max-w is allowed in markup); the container classes never carry max-width.
- Scope all selectors to `.{kebab-name}` — no global side effects.
- Group with a `/* {kebab-name} */` comment header.
- Append to `utilities.css` — never overwrite existing block sections.
- No fixed `h-*`/`w-*`/`min-h-*`/`min-w-*` on sections (CSS rules 7–8) — `h-full`/`w-full`/`max-w-*` ok.
- Animations use only `transform`/`opacity` (CSS rule 18) — no `transition-all`.

---

### Step 5 — Write PHP Block class

**Path:** `app/Blocks/{StudlyName}.php`

```php
<?php

namespace App\Blocks;

class {StudlyName}
{
    public function render(string $blockContent, array $block): string
    {
        if (($block['blockName'] ?? '') !== '{namespace}/{kebab-name}') {
            return $blockContent;
        }

        $attrs = $block['attrs'] ?? [];

        return view('blocks.{kebab-name}', [
            // one line per attribute, default = value from Figma design
            'attrName' => $attrs['attrName'] ?? 'default value',
            // image triplet:
            'heroImageUrl' => $attrs['heroImageUrl'] ?? '',
            'heroImageId'  => $attrs['heroImageId']  ?? 0,
            'heroImageAlt' => $attrs['heroImageAlt']  ?? '',
            // always pass className — required for "Additional CSS class(es)" from block Advanced panel
            'className'    => $attrs['className']    ?? '',
        ])->render();
    }
}
```

Rules:
- `blockName` check uses `{namespace}/{kebab-name}` — match namespace from existing project blocks
- Pass **every** attribute with its default as fallback
- Use `->render()` at the end of the view() call

---

### Step 6 — Write JSX editor file

**Path:** `resources/js/editor/{kebab-name}.block.jsx`

```jsx
/** @jsx wp.element.createElement */
/** @jsxFrag wp.element.Fragment */
import { __ } from '@wordpress/i18n';
import {
    InspectorControls,
    useBlockProps,
    MediaUpload,
    MediaUploadCheck,
} from '@wordpress/block-editor';
import {
    PanelBody,
    TextControl,
    TextareaControl,
    ToggleControl,
    Button,
} from '@wordpress/components';

// only import what you actually use — remove unused imports

export const name = '{namespace}/{kebab-name}';
export const title = __('{Human Name}', 'sage');
export const category = 'design';
export const icon = '{appropriate-dashicon}'; // e.g. 'layout', 'format-image', 'text'

export const attributes = {
    // string attribute:
    attrName: { type: 'string', default: 'Default text from Figma' },
    // image triplet:
    heroImageUrl: { type: 'string', default: '' },
    heroImageId:  { type: 'number', default: 0 },
    heroImageAlt: { type: 'string', default: '' },
};

export const edit = ({ attributes, setAttributes }) => {
    const blockProps = useBlockProps({ className: '{kebab-name}' });
    const {
        // destructure all attributes here
    } = attributes;

    // For dynamic background image: set CSS custom property only — no other inline styles
    const bgStyle = heroImageUrl ? { '--{kebab-name}-bg': `url(${heroImageUrl})` } : undefined;

    return (
        <>
            <InspectorControls>
                <PanelBody title={__('Background', 'sage')}>
                    <MediaUploadCheck>
                        <MediaUpload
                            onSelect={(media) => setAttributes({
                                heroImageUrl: media.url,
                                heroImageId: media.id,
                                heroImageAlt: media.alt,
                            })}
                            allowedTypes={['image']}
                            value={heroImageId}
                            render={({ open }) => (
                                <div>
                                    {heroImageUrl && (
                                        <img src={heroImageUrl} alt={heroImageAlt} className="w-full mb-2" />
                                    )}
                                    <Button onClick={open} variant="secondary">
                                        {heroImageUrl ? __('Replace Image', 'sage') : __('Select Image', 'sage')}
                                    </Button>
                                    {heroImageUrl && (
                                        <Button
                                            onClick={() => setAttributes({ heroImageUrl: '', heroImageId: 0, heroImageAlt: '' })}
                                            variant="link"
                                            isDestructive
                                            className="ml-2"
                                        >
                                            {__('Remove', 'sage')}
                                        </Button>
                                    )}
                                </div>
                            )}
                        />
                    </MediaUploadCheck>
                    <TextControl
                        label={__('Image Alt Text', 'sage')}
                        value={heroImageAlt}
                        onChange={(value) => setAttributes({ heroImageAlt: value })}
                    />
                </PanelBody>

                <PanelBody title={__('Content', 'sage')} initialOpen={false}>
                    <TextControl
                        label={__('Title', 'sage')}
                        value={attrName}
                        onChange={(value) => setAttributes({ attrName: value })}
                    />
                    <TextareaControl
                        label={__('Paragraph', 'sage')}
                        value={paragraphAttr}
                        onChange={(value) => setAttributes({ paragraphAttr: value })}
                    />
                </PanelBody>
            </InspectorControls>

            {/* Editor preview — use exact same classes as Blade template */}
            <section {...blockProps} style={bgStyle}>
                {/* replicate exact HTML structure + Tailwind classes from Blade here */}
                {/* dynamic text: {attrName} */}
                {/* conditional image: heroImageUrl && <img src={heroImageUrl} alt={heroImageAlt} ... /> */}
            </section>
        </>
    );
};

export const save = () => null;
```

**Follow `html-css-js-rules` for JSX/JS conventions — inline style restrictions, import hygiene, and editor/frontend class parity.**

Rules specific to this workflow:
- `save: () => null` — always, dynamic block, PHP renders frontend
- `useBlockProps` className = kebab block name (matches Blade root element class)
- `initialOpen={false}` on all PanelBody except the first

---

### Step 7 — Write Blade template

**Path:** `resources/views/blocks/{kebab-name}.blade.php`

Take the HTML (generated or provided) and transform it:
- Text content → `{{ $attrName }}`
- `href="..."` → `href="{{ $attrUrl }}"`
- `src="..."` → `src="{{ $imageUrl }}"`
- `alt="..."` → `alt="{{ $imageAlt }}"`
- Optional elements (could be empty) → wrap in `@if(!empty($attrName)) ... @endif`
- Keep **all** Tailwind classes and CSS classes exactly as generated
- Do NOT add extra wrapper divs
- Always append `{{ $className ?? '' }}` to the root element's `class` — forwards "Additional CSS class(es)" from the block's Advanced panel to the frontend

**Follow `html-css-js-rules` for markup/style constraints (e.g. the CSS custom property exception for dynamic background images).**

#### Dynamic background image in Blade

```html
{{-- dynamic background image via CSS custom property --}}
<section
    class="{kebab-name} has-bg-image flex ... {{ $className ?? '' }}"
    @if(!empty($bgImageUrl)) style="--{kebab-name}-bg: url('{{ $bgImageUrl }}')" @endif
>
```

This sets only the URL variable — all visual styling (gradients, size, position) lives in the CSS from Step 4.

#### Other example transformations

```html
{{-- original heading (wrapped per skill rule) --}}
<div class="title title-orange">
  <h2>Our Location</h2>
</div>

{{-- blade --}}
@if(!empty($title))
<div class="title title-orange">
  <h2>{{ $title }}</h2>
</div>
@endif
```

```html
{{-- original paragraph (wrapped per skill rule) --}}
<div class="content content-dark">
  <p>Some description text here.</p>
</div>

{{-- blade --}}
@if(!empty($description))
<div class="content content-dark">
  <p>{{ $description }}</p>
</div>
@endif
```

```html
{{-- original image --}}
<img
  src="/img/team.jpg"
  alt="Our team"
  class="w-full"
  width="800"
  height="600"
  loading="lazy"
  decoding="async">

{{-- blade --}}
@if(!empty($teamImageUrl))
    <img
      src="{{ $teamImageUrl }}"
      alt="{{ $teamImageAlt }}"
      class="w-full"
      width="800"
      height="600"
      loading="lazy"
      decoding="async">
@endif
```

```html
{{-- original link --}}
<a
  href="/about"
  role="link"
  target="_self"
  aria-label="Learn more about us"
  class="btn btn-orange">Learn More</a>

{{-- blade --}}
@if(!empty($buttonText))
    <a
      href="{{ $buttonUrl }}"
      role="link"
      target="_self"
      aria-label="{{ $buttonText }}"
      class="btn btn-orange">{{ $buttonText }}</a>
@endif
```

---

### Step 8 — Update editor.js

**Path:** `resources/js/editor.js`

Read the file. Find the last `import * as` line. Add new import after it:

```js
import * as {camelName}Block from './editor/{kebab-name}.block';
```

Find the `registerBlockType` section. Add:

```js
  registerBlockType({camelName}Block.name, {
    apiVersion: 3,
    title: {camelName}Block.title,
    category: {camelName}Block.category,
    icon: {camelName}Block.icon,
    attributes: {camelName}Block.attributes,
    edit: {camelName}Block.edit,
    save: {camelName}Block.save,
  });
```

---

### Step 9 — Update ThemeServiceProvider.php

**Path:** `app/Providers/ThemeServiceProvider.php`

1. Add use statement after existing `use` lines:
```php
use App\Blocks\{StudlyName};
```

2. Add filter inside `register()` method before its closing `}`:
```php
        /**
         * Render `{namespace}/{kebab-name}` block with Blade template
         */
        add_filter('render_block', [new {StudlyName}(), 'render'], 10, 2);
```

---

### Step 9.5 — Visual verify loop (CRITICAL — closes the accuracy gap)

**The old workflow was fire-and-forget: it fetched a Figma screenshot as a loose "reference" and never checked the rendered result. That is why output drifted. Now verify.**

1. **Build:** run `npm run build` (or `npm run dev`) — must compile clean, no unknown-utility errors.
2. **Render:** view the block on a page (frontend) at the Figma frame's width. Take a screenshot of the rendered block.
3. **Compare** the rendered screenshot against the Step 2 Figma screenshot, side by side. Check:
   - Fonts actually loaded (NOT a system fallback) — if text looks wrong, re-check Step 2.6.
   - Spacing / padding / gap match exactly (no rounding).
   - Font sizes, colors, radius, shadows match.
   - Layout: alignment, column widths, content max-width, wrapping.
   - Responsive: repeat at 1440 / 1024 / 768 against Figma responsive frames.
4. **Fix** every mismatch — adjust the exact token in `_tokens.css` / the `@apply` in `utilities.css`. Never "close enough."
5. **Repeat** steps 1–4 until the render matches Figma. Only then report done.

If a mismatch cannot be resolved (e.g. font unavailable, Figma effect not reproducible in CSS), flag it explicitly in the Step 10 report.

---

### Step 10 — Summary report

After all files written AND Step 9.5 verify passes, output:

```
Block created: {kebab-name}
Source: figma-to-HTML

Files written:
  app/Blocks/{StudlyName}.php
  resources/js/editor/{kebab-name}.block.jsx
  resources/views/blocks/{kebab-name}.blade.php

Files updated:
  resources/css/utilities.css  (block section CSS — written once, imported by both entries)
  resources/css/_tokens.css    (only if new tokens added)
  resources/js/editor.js
  app/Providers/ThemeServiceProvider.php

Fonts:
  resources/fonts/{font}.woff2   (downloaded + verified present)

  (Step 3.5 bootstrap files listed separately if this was the first block)

Attributes ({count} total):
  attrName        string    TextControl       "Default from Figma"
  heroImageUrl    string    MediaUpload       ""
  heroImageId     number    MediaUpload       0
  heroImageAlt    string    TextControl       ""

Verify (Step 9.5):
  Build: pass
  Visual match vs Figma: pass  (or list unresolved mismatches)

Next steps:
  npm run dev   (or npm run build)
  WP Admin → find "{Human Name}" in block inserter
```

---

## Hard rules

- **Follow `html-css-js-rules` for all HTML, CSS, and JS/JSX writing conventions.** This file only covers block-generation workflow, file structure, and Gutenberg/Laravel wiring.
- **Download + verify fonts (Step 2.6) BEFORE build** — a `@font-face` with a missing file is the #1 cause of inaccurate UI. Never ship one.
- **Verify visually (Step 9.5)** — build, screenshot render, diff against Figma, fix, repeat. No fire-and-forget.
- **EXACT Figma values only** — never invent numbers to fill a scale, never snap to the nearest existing token/container. Missing value → add the exact value as a new token.
- **Read real Figma responsive frames** for 1440/1024/768 — only fall back to scaling when no responsive frame exists (and mark it).
- `{namespace}` = the project's block namespace — read it from the project's CLAUDE.md or existing `.block.jsx` files; never invent a new one
- `save: () => null` always — dynamic block, PHP renders frontend
- PHP class never extends anything — plain class, matches project pattern
- Defaults pulled from actual Figma content — never invent placeholder text
- Each image = three attrs (url + id + alt) — always the triplet
- Notes override auto-detection — user knows their intent better than parser
- **Bootstrap the token system FIRST (Step 3.5)** — if `_tokens.css` is missing, build the token + split-CSS architecture before any block CSS. This is what makes rule-compliance possible.
- **NEVER `[...]` arbitrary values, NEVER font/color/typography utilities in HTML** — the two biggest rule violations. Tokens in `_tokens.css`, styling in CSS via `@apply`, markup limited to the HTML-rule-12 utility whitelist.
- **Block CSS lives in `utilities.css`, written ONCE** — both `app.css` and `editor.css` `@import` it, so editor = frontend automatically. Do NOT hand-duplicate CSS across two files.
- **Use Figma MCP** — always call `get_design_context` first, then `get_screenshot` (keep it as verify target); extract exact colors, typography, spacing, and content from Figma data before writing any HTML
