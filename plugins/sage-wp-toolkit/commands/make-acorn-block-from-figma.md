# Make Acorn Block — Figma MCP Dynamic Block Generator

Generate a fully dynamic Gutenberg block from a **Figma URL** (frame, component, or section) or raw HTML + optional notes.

## Usage

```
/make-acorn-block-from-figma
```

No arguments needed. Claude collects inputs interactively.

---

## Instructions

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
2. **`get_screenshot`** (or `mcp__claude_ai_Figma__get_screenshot`) — fetch a visual screenshot of the frame for reference
3. If variables or tokens are needed: **`get_variable_defs`**

Extract from the Figma response:
- **Colors** — hex values, opacities, gradients
- **Typography** — font families, sizes, weights, line-heights, letter-spacing
- **Spacing** — padding, margin, gap values (convert to Tailwind or arbitrary values)
- **Layout** — flex/grid direction, alignment, column structure
- **Content** — all visible text strings, image placeholders, icons
- **Components** — identify reusable sub-components

Generate production-ready, semantic HTML with Tailwind CSS v4 utility classes from the extracted design data.

#### HTML generation rules

**Semantics first:**
- Use correct semantic elements: `<header>`, `<section>`, `<article>`, `<nav>`, `<aside>`, `<footer>`, `<h1>`–`<h6>`, `<p>`, `<ul>`, `<ol>`, `<figure>`, `<figcaption>`, `<button>`, `<a>`
- Never use `<div>` when a semantic element fits
- Root element class = `{kebab-name} general-padding` — always these two classes on the root `<section>`
- All `<section>` elements must live inside `<main>`

**Tailwind CSS v4 — layout classes only in HTML:**
- Only these Tailwind classes go directly in HTML: `flex`, `flex-col`, `flex-row`, `grid`, `grid-cols-*`, `gap-*`, `p-*`, `m-*`, `max-w-*`
- All other styles (colors, typography, sizing, borders, shadows) → named CSS classes defined via `@apply` in Step 4
- No arbitrary values in HTML: no `text-[20px]`, `max-w-[452px]`, `bg-[#F68B32]` — define named utility classes in CSS
- Responsive: use `max-*` breakpoint prefixes, e.g. `max-768:flex-col`, `max-1199:grid-cols-1`
- Common responsive patterns:
  - Single column mobile → two columns desktop: `grid grid-cols-2 max-768:grid-cols-1`
  - Stack mobile → row desktop: `flex flex-row max-768:flex-col`

**Headings (h1–h6):**
- Never put classes directly on heading tags
- Always wrap: `<div class="title title-{color}"><h2>Text</h2></div>`
- Color class derived from Figma MCP (e.g. `title-orange`, `title-dark`, `title-white`)
- Define `.title-{color} h1, h2, ...` rule in Step 4 CSS via `@apply text-{color}`

**Paragraphs:**
- Never put classes directly on `<p>`
- Always wrap: `<div class="content content-{color}"><p>Text</p></div>`
- Color class derived from Figma MCP (e.g. `content-dark`, `content-white`)
- Define `.content-{color} p` rule in Step 4 CSS via `@apply text-{color}`

**Links:**
- Every `<a>` needs `href`, `role="link"`, `target`, `aria-label` — each attribute on its own line
- Phone: `href="tel:{n}"` · Email: `href="mailto:{e}"`

**Buttons:**
- Every `<button>` needs `type="button"|"submit"` and `aria-label` — each on its own line

**Custom fonts:**
- For custom fonts from Figma (e.g. `Amatic SC`, `Museo`) — assign a CSS class like `font-amatic` or `font-museo`
- Do NOT write `style="font-family: ..."` — the `@font-face` and class will be defined in Step 4 (block CSS)
- For any visual property that needs custom values — assign a descriptive class and define it in Step 4 via `@apply`

**Images:**
- Use `<img>` with `width`, `height`, `alt` — each attribute on its own line, matching Figma frame dimensions
- `alt` must never be empty — use Figma layer name or visible content
- Add `loading="lazy"` on below-fold images, `loading="eager"` on hero/first-visible images
- Add `decoding="async"` on all images

**Background images (dynamic):**
- Do NOT hardcode a `style="background-image: url(...)"` attribute
- Instead add the class `has-bg-image` to the element and define `.{kebab-name}.has-bg-image` in Step 4 CSS
- The Blade template will set only the CSS custom property: `style="--{kebab-name}-bg: url('...')"`
- That is the only permitted use of a style attribute — passing a dynamic URL into CSS

**Performance:**
- Zero inline styles — none at all (except the bg CSS custom property above)
- No unnecessary wrapper divs
- Keep nesting shallow (max 4–5 levels)
- Use CSS Grid or Flexbox — no floats, no tables for layout

**Accessibility:**
- All interactive elements keyboard-accessible
- `aria-label` on icon-only buttons
- Sufficient color contrast (check against WCAG AA)
- `<img>` always has non-empty `alt`
- Every `<input>` linked to a `<label>` via matching `for`/`id`

After generating HTML, proceed to Step 3 using this generated HTML as the source.

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

### Step 4 — Write block CSS

**Paths:** `resources/css/app.css` AND `resources/css/editor.css`

Read both files first. Then:

1. **Check for `@theme` breakpoints** — look for `--breakpoint-768` in each file. If missing, prepend the `@theme` block immediately after the `@import "tailwindcss"` line (before any `@source` or other rules):
```css
@theme {
  --breakpoint-1920: 1921px; --breakpoint-1600: 1601px; --breakpoint-1512: 1513px;
  --breakpoint-1440: 1441px; --breakpoint-1366: 1367px; --breakpoint-1199: 1200px;
  --breakpoint-1024: 1025px; --breakpoint-992:   993px; --breakpoint-768:   769px;
  --breakpoint-640:   641px; --breakpoint-576:   577px; --breakpoint-425:   426px;
  --breakpoint-375:   376px;
}
```
This is required for `max-768:`, `max-1199:` etc. responsive prefixes to work. Do this once per project — skip if already present.

2. **Append a block-scoped CSS section** to each file. The CSS written to both files must be **identical** — this is what makes the editor preview look the same as the frontend.

#### What goes in block CSS

**Custom fonts** (only if fonts are used in this block):
- Check if the font utility already exists (e.g. `@utility font-amatic`) — skip if present
- For externally loaded fonts (Google Fonts, Typekit), declare only the `@utility`:
```css
@utility font-amatic { font-family: 'Amatic SC', cursive; }
@utility font-museo  { font-family: 'museo', serif; }
```
- For locally hosted fonts (woff2 in `resources/fonts/`), add `@font-face` + `@utility`:
```css
@font-face {
  font-family: 'FontName';
  src: url('../fonts/FontName.woff2') format('woff2');
  font-weight: 400;
  font-display: swap;
}
@utility font-fontname { font-family: 'FontName', serif; }
```
- Font utilities MUST be `@utility` not plain `.class {}` — plain classes cannot be used in `@apply`

**Dynamic background image support:**
```css
/* {kebab-name} — background */
.{kebab-name} {
  background-image: var(--{kebab-name}-bg, none);
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}
```

**Gradient overlays on background images:**
```css
.{kebab-name}.has-bg-image {
  background-image:
    linear-gradient(0deg, rgba(0,0,0,0.4), rgba(0,0,0,0.4)),
    radial-gradient(48% 50% at 50% 50%, rgba(0,0,0,0.6) 0%, transparent 100%),
    var(--{kebab-name}-bg, none);
}
```

**Typography and color classes (via `@apply`):**
```css
/* {kebab-name} — typography */
.{kebab-name}__card-title {
  @apply text-2xl font-museo font-semibold;
}

.{kebab-name}__label {
  @apply text-sm uppercase tracking-widest font-semibold;
}
```

**Heading color helpers (scoped to this block):**
```css
/* {kebab-name} — title colors */
.{kebab-name} .title-orange h1,
.{kebab-name} .title-orange h2,
.{kebab-name} .title-orange h3 { @apply text-[#F68B32] }

.{kebab-name} .title-dark h1,
.{kebab-name} .title-dark h2,
.{kebab-name} .title-dark h3 { @apply text-[#1a1a1a] }
```

**Content color helpers (scoped to this block):**
```css
/* {kebab-name} — content colors */
.{kebab-name} .content-white p { @apply text-white }
.{kebab-name} .content-dark p  { @apply text-[#333333] }
```

#### Rules
- **All styles via `@apply` — zero raw CSS properties** (only exception: CSS custom properties for dynamic bg images and `@font-face` declarations)
- Scope all selectors to `.{kebab-name}` — no global side effects
- Group with a `/* {kebab-name} — section */` comment header for each group
- Append — never overwrite existing content in app.css or editor.css
- Write the same CSS to both files

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
        if (($block['blockName'] ?? '') !== 'radicle/{kebab-name}') {
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
- `blockName` check uses `radicle/{kebab-name}` — match namespace from existing project blocks
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

export const name = 'radicle/{kebab-name}';
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

Rules:
- `save: () => null` — always, dynamic block, PHP renders frontend
- **Zero inline styles** — no `style={{...}}` objects on any element except the root element's `bgStyle` which sets only CSS custom properties for dynamic image URLs
- Editor preview must use the **exact same CSS classes** as the Blade template — `editor.css` (loaded in WP admin) carries the same block CSS as `app.css`, so the preview looks identical to the frontend
- Only import components you actually use — remove unused imports
- `initialOpen={false}` on all PanelBody except the first
- `useBlockProps` className = kebab block name (matches Blade root element class)

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
- **No inline styles** — the only permitted `style` attribute is setting a CSS custom property for a dynamic background image URL
- Always append `{{ $className ?? '' }}` to the root element's `class` — forwards "Additional CSS class(es)" from the block's Advanced panel to the frontend

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
  <h2 class="font-amatic">Our Location</h2>
</div>

{{-- blade --}}
@if(!empty($title))
<div class="title title-orange">
  <h2 class="font-amatic">{{ $title }}</h2>
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
         * Render `radicle/{kebab-name}` block with Blade template
         */
        add_filter('render_block', [new {StudlyName}(), 'render'], 10, 2);
```

---

### Step 10 — Summary report

After all files written, output:

```
Block created: {kebab-name}
Source: figma-to-HTML

Files written:
  app/Blocks/{StudlyName}.php
  resources/js/editor/{kebab-name}.block.jsx
  resources/views/blocks/{kebab-name}.blade.php

Files updated:
  resources/css/app.css        (block CSS + fonts)
  resources/css/editor.css     (identical — ensures editor = frontend)
  resources/js/editor.js
  app/Providers/ThemeServiceProvider.php

Attributes ({count} total):
  attrName        string    TextControl       "Default from Figma"
  heroImageUrl    string    MediaUpload       ""
  heroImageId     number    MediaUpload       0
  heroImageAlt    string    TextControl       ""

Next steps:
  npm run dev   (or npm run build)
  WP Admin → find "{Human Name}" in block inserter
```

---

## Hard rules

- Namespace always `radicle/` — check existing `.block.jsx` files if unsure
- `save: () => null` always — dynamic block, PHP renders frontend
- PHP class never extends anything — plain class, matches project pattern
- Defaults pulled from actual Figma content — never invent placeholder text
- Keep all Tailwind classes and CSS classes in Blade — do not strip or modify
- Remove unused JS imports — only import what's used
- Each image = three attrs (url + id + alt) — always the triplet
- Notes override auto-detection — user knows their intent better than parser
- **Zero inline styles** — no `style="..."` attributes for visual design; no `style={{...}}` objects in JSX for visual design
- **Only permitted `style` attribute**: setting CSS custom properties for dynamic values (e.g. `style="--block-bg: url('...')"`) — this is the only exception
- **CSS in app.css AND editor.css** — every block-specific style goes in both files; never one without the other
- **All CSS via `@apply`** — zero raw CSS property declarations; only exceptions are `@font-face` rules and CSS custom property fallbacks (`var(--x, none)`)
- **Layout Tailwind classes only in HTML** — `flex`, `flex-col`, `flex-row`, `grid`, `grid-cols-*`, `gap-*`, `p-*`, `m-*`, `max-w-*` allowed; all typography, color, sizing, border, shadow classes → CSS via `@apply`
- **No arbitrary values in HTML** — no `text-[20px]`, `bg-[#hex]`, `max-w-[452px]`; define named classes in CSS and use `@apply` with arbitrary value there if needed
- **Headings wrapped** — never classes on `h1`–`h6` directly; always `<div class="title title-{color}"><hN>text</hN></div>`
- **Paragraphs wrapped** — never classes on `<p>` directly; always `<div class="content content-{color}"><p>text</p></div>`
- **Section root** = `{kebab-name} general-padding` — exactly these two classes on the root `<section>`; all sections inside `<main>`
- **Links** — every `<a>` has `href`, `role="link"`, `target`, `aria-label` on separate lines
- **Buttons** — every `<button>` has `type` and `aria-label` on separate lines
- **Images** — `width`, `height`, `alt` each on its own line; `alt` never empty
- Custom fonts: use `@utility font-name { font-family: '...'; }` — NOT a plain `.class {}` (plain classes can't be used in `@apply`); for local fonts add `@font-face` first, then `@utility`
- **`@theme` breakpoints required** — `max-768:`, `max-1199:` etc. only work if `@theme { --breakpoint-768: 769px; ... }` is declared in app.css AND editor.css; add the full 13-breakpoint block if not already present (Step 4 checks this before appending block CSS)
- Responsive: use `max-*` breakpoint prefixes (e.g. `max-768:flex-col`) — never `md:`, `lg:` Tailwind breakpoints
- LCP images get `loading="eager"`, all others `loading="lazy"` + `decoding="async"`
- Semantic elements preferred over `<div>` — use the right tag
- Editor preview = frontend — same CSS classes, same fonts, same layout; `editor.css` is what makes this true
- **Use Figma MCP** — always call `get_design_context` first, then `get_screenshot` for visual reference; extract exact colors, typography, spacing, and content from Figma data before writing any HTML
