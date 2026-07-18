---
description: Scaffold a dynamic Gutenberg block (PHP class + Blade view + JSX editor + CSS) from raw HTML and notes
---

# Make Acorn Block — AI-Powered Dynamic Block Generator

Generate a fully dynamic Gutenberg block from real HTML + optional notes.

## Usage

```
/make-acorn-block
```

No arguments needed. Claude collects inputs interactively.

---

## Instructions

### Step 0 — Collect inputs

The user invoked this command with arguments: `$ARGUMENTS`

**Parse arguments for:**
- Block name (kebab-case slug, e.g. `hero-section`)
- HTML markup (anything between `<` and `>` tags)
- Notes (anything after "Notes:" or "Note:")

**If block name is missing** → ask: "Block slug? (kebab-case, e.g. `hero-section`)"
**If HTML is missing** → ask: "Paste the full HTML section for this block."
**If notes absent** → that's fine, proceed without.

Wait for user to provide missing inputs before proceeding. Collect all three (name, HTML, notes) before writing any file.

---

### Step 1 — Derive name variants

From the kebab-case block name:
- `kebabName` = `hero-section`
- `studlyName` = `HeroSection`
- `camelName` = `heroSection`
- `humanName` = `Hero Section`

---

### Step 2 — Analyze HTML → define attributes

Scan every element in the HTML. For each piece of **user-editable content**, define one attribute.

#### Element → attribute mapping

| HTML element / pattern | Attribute type | WP control | Notes |
|---|---|---|---|
| `<h1>` … `<h6>` text | `string` | `TextControl` | short text |
| `<p>` text | `string` | `TextareaControl` | long text |
| Inline `<span>`, `<strong>`, `<em>` text | `string` | `TextControl` | short text |
| `<a href="...">` | Two attrs: `xUrl` (string) + use existing text attr | `TextControl` for URL | if link text is standalone, add `xText` attr too |
| `<img src="..." alt="...">` | Three attrs: `xImageUrl` (string), `xImageId` (number, default 0), `xImageAlt` (string) | `MediaUpload` for url/id, `TextControl` for alt | |
| Background image in `style="background-image: url(...)"` | Same as img: url + id + alt | `MediaUpload` | |
| `<button>` or `.btn` text | `string` | `TextControl` | short text |
| Any small label / pre-title / tag text | `string` | `TextControl` | short text |
| Decorative elements, divs with no text, SVG icons | **skip** — no attribute | — | not user-editable |
| CSS class names | **skip** | — | hardcoded in templates |
| `data-*` attributes | **skip** unless they hold content | — | |

#### Attribute naming rules
- camelCase, descriptive, contextual: `heroTitle`, `ctaButtonText`, `ctaButtonUrl`, `teamImageUrl`
- If same element type appears multiple times: number them → `content1`, `content2` or prefix by section → `leftTitle`, `rightContent`
- Image triplet always named: `<purpose>ImageUrl`, `<purpose>ImageId`, `<purpose>ImageAlt`

#### Defaults
- Extract **exact text / URL / src** from the HTML as the default value
- Image defaults: `url: ''`, `id: 0`, `alt: ''`

#### Apply user notes
After auto-detection, read the notes parameter. Notes may:
- Add extra attributes not in the HTML (e.g. "add a toggle to show/hide the button")
- Override control type (e.g. "use color picker for the background")
- Add extra fields (e.g. "add a second paragraph field")
- Change grouping or panel labels

Apply all note-driven changes to the attribute list.

#### Control selection summary
- Short text (< ~80 chars default) → `TextControl`
- Long text (paragraph) → `TextareaControl`
- Image → `MediaUpload` + `MediaUploadCheck` + `TextControl` for alt
- URL → `TextControl` (label "URL")
- Toggle show/hide → `ToggleControl`
- Color → `ColorPicker` or `ColorPalette`
- Number / size → `RangeControl`

#### Grouping for InspectorControls
Group attributes into `PanelBody` sections based on **HTML structure**:
- If HTML has clear left/right columns → "Left Column" / "Right Column"
- If HTML has header + body + CTA → "Header" / "Content" / "Call to Action"
- If flat → one panel "Block Settings" or two: "Content" / "Media"
- Image fields always in own panel "Media" or grouped with their section

---

### Step 3 — Write PHP Block class

**Path:** `app/Blocks/{StudlyName}.php`

Pattern (match existing project blocks exactly — no `extends Block`):

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
            // one line per attribute, default = value from HTML
            'attrName'   => $attrs['attrName']   ?? 'default value',
            // image triplet:
            'heroImageUrl' => $attrs['heroImageUrl'] ?? '',
            'heroImageId'  => $attrs['heroImageId']  ?? 0,
            'heroImageAlt' => $attrs['heroImageAlt']  ?? '',
            // always include — powers "Additional CSS class(es)" from WP sidebar
            'extraClass' => $attrs['className'] ?? '',
        ])->render();
    }
}
```

Rules:
- `blockName` check uses `{namespace}/{kebab-name}` — match namespace from existing project blocks
- Pass **every** attribute with its default as fallback
- Always pass `'extraClass' => $attrs['className'] ?? ''` — WordPress stores "Additional CSS class(es)" here
- Use `->render()` at the end of the view() call

---

### Step 4 — Write JSX editor file

**Path:** `resources/js/editor/{kebab-name}.block.jsx`

Pattern (match existing project JSX files):

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
    attrName: { type: 'string', default: 'Default text from HTML' },
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

    return (
        <>
            <InspectorControls>
                <PanelBody title={__('Content', 'sage')}>
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

                <PanelBody title={__('Media', 'sage')} initialOpen={false}>
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
                                    {heroImageUrl && <img src={heroImageUrl} alt={heroImageAlt} style={{ maxWidth: '100%', marginBottom: '8px' }} />}
                                    <Button onClick={open} variant="secondary">
                                        {heroImageUrl ? __('Replace Image', 'sage') : __('Select Image', 'sage')}
                                    </Button>
                                    {heroImageUrl && (
                                        <Button
                                            onClick={() => setAttributes({ heroImageUrl: '', heroImageId: 0, heroImageAlt: '' })}
                                            variant="link"
                                            isDestructive
                                            style={{ marginLeft: '8px' }}
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
            </InspectorControls>

            {/* Editor preview — mirror the HTML structure, show live attribute values */}
            <section {...blockProps}>
                {/* replicate the exact HTML layout here, using {attrName} for dynamic values */}
                {/* for images: heroImageUrl && <img src={heroImageUrl} alt={heroImageAlt} /> */}
            </section>
        </>
    );
};

export const save = () => null;
```

Rules:
- `save: () => null` — always, because PHP renders frontend
- Editor preview in `edit()` must mirror the real HTML structure so WYSIWYG works in Gutenberg
- Only import components you actually use
- Use `initialOpen={false}` on all PanelBody except the first
- `useBlockProps` className = kebab block name (matches Blade root element class)

---

### Step 5 — Write Blade template

**Path:** `resources/views/blocks/{kebab-name}.blade.php`

Take the **original HTML** and transform it:
- Text content → `{{ $attrName }}`
- `href="..."` → `href="{{ $attrUrl }}"`
- `src="..."` → `src="{{ $imageUrl }}"`
- `alt="..."` → `alt="{{ $imageAlt }}"`
- Optional elements (could be empty) → wrap in `@if(!empty($attrName)) ... @endif`
- Keep **all** CSS classes, data attributes, aria attributes exactly as in original HTML
- Do NOT add extra wrapper divs
- **Always** append `{{ !empty($extraClass) ? ' ' . $extraClass : '' }}` inside the `class="..."` of the outermost element — this applies the WP "Additional CSS class(es)" field

Example transformation:
```html
<!-- original -->
<h4>WHO WE ARE</h4>

<!-- blade -->
 @if(!empty($preTitle))
<h4>{{ $preTitle }}</h4>
@endif
```

```html
<!-- original -->
<img src="/img/team.jpg" alt="Our team">

<!-- blade -->
@if(!empty($teamImageUrl))
    <img src="{{ $teamImageUrl }}" alt="{{ $teamImageAlt }}">
@endif
```

```html
<!-- original -->
<a href="/about" class="btn-yellow">Our Model</a>

<!-- blade -->
@if(!empty($buttonText))
    <a href="{{ $buttonUrl }}" class="btn-yellow">{{ $buttonText }}</a>
@endif
```

```html
<!-- original outer wrapper -->
<section class="hero-banner">

<!-- blade — extraClass always appended to outermost element -->
<section class="hero-banner{{ !empty($extraClass) ? ' ' . $extraClass : '' }}">
```

---

### Step 6 — Update editor.js

**Path:** `resources/js/editor.js`

Read the file. Find the last `import * as` line. Add new import after it:

```js
import * as {camelName}Block from "./editor/{kebab-name}.block";
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

### Step 7 — Update ThemeServiceProvider.php

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

### Step 8 — Summary report

After all files written, output:

```
Block created: {kebab-name}

Files written:
  app/Blocks/{StudlyName}.php
  resources/js/editor/{kebab-name}.block.jsx
  resources/views/blocks/{kebab-name}.blade.php

Files updated:
  resources/js/editor.js
  app/Providers/ThemeServiceProvider.php

Attributes ({count} total):
  attrName        string    TextControl       "Default from HTML"
  heroImageUrl    string    MediaUpload       ""
  heroImageId     number    MediaUpload       0
  heroImageAlt    string    TextControl       ""

Next steps:
  npm run dev   (or npm run build)
  WP Admin → find "{Human Name}" in block inserter
```

---

## Hard rules

- `{namespace}` = the project's block namespace — read it from the project's CLAUDE.md or existing `.block.jsx` files; never invent a new one
- `save: () => null` always — dynamic block, PHP renders frontend
- PHP class never extends anything — plain class, matches project pattern
- Defaults pulled from actual HTML content — never invent placeholder text
- Keep all original CSS classes in Blade — do not strip or modify
- Remove unused JS imports — only import what's used
- Each image = three attrs (url + id + alt) — always the triplet
- Notes override auto-detection — user knows their intent better than HTML parser
- Always pass `'extraClass' => $attrs['className'] ?? ''` in every PHP block's view() call
- Always append `{{ !empty($extraClass) ? ' ' . $extraClass : '' }}` to the outermost element's class in every Blade template — this wires up WP's "Additional CSS class(es)" sidebar field
