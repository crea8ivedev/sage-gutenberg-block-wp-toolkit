---
name: frontend-code-rules
description: 'Mandatory HTML/CSS/JS rules for Sage (Blade) + Tailwind v4 WordPress themes. Use whenever writing, editing, or reviewing any HTML markup, CSS (Tailwind @apply), or JavaScript in a project built on this stack — covers markup structure, design-token usage, responsive breakpoints, accessibility attributes, animation, and JS module conventions. Apply automatically, do not wait to be asked.'
---

# Front-end Code Rules (mandatory)

These rules apply to every HTML, CSS, and JS change in a Sage/Blade + Tailwind v4 theme. Follow them exactly — no exceptions.

========== HTML RULES ==========

1. No classes on body, html, header, footer. Style via CSS selectors in base.css or layout.css using @apply.
2. No inline styles (no style="..."). All styling in CSS files via @apply.
3. Every <a>: href, role="link", target, aria-label — each on its own line.
4. Every <button>: type="button"|"submit", aria-label — each on its own line.
5. Every <section>: descriptive class + section top/bottom padding (Figma Y-axis) + bg color, ALL as
    utility classes on the section tag. No .general-padding class. No section bg-color class in CSS.
    <section class="hero-section bg-black pt-100 pb-100"></section>
    Top/bottom padding: use pt-*/pb-* utilities with responsive variants for Figma's Y-axis spacing
    (e.g. pt-100 pb-100 max-1199:pt-50 max-1199:pb-50 max-768:pt-30 max-768:pb-30). Values from Figma exact.
    Background color: use the bg-{color} utility (color must exist in @theme{}, CSS rule 23) — never a
    per-section bg class in utilities.css.
    Inner horizontal padding: always reuse an existing .container-fluid* class (see CSS rule 10) —
    never invent a new custom px-* wrapper per section. Use the container whose px EQUALS Figma's
    X-axis spacing exactly. NEVER round to the "closest" existing value — that is the #1 cause of
    UI drift. If no container matches Figma's exact px, add that exact px as a new step to the SAME
    shared container classes (container-fluid / -md / -lg), not as a one-off section-specific class.
    This is a hard rule — do not re-derive per-section container padding from Figma each time; check
    the existing containers first, but match the exact value, never approximate.
    Centered content column: an inner wrapper may use a single max-w-* token (max-w allowed per rule
    12) to cap content width to Figma's exact content-frame width — the container classes never carry
    max-width, so centered layouts need this wrapper.
    <div class="container-fluid-lg"><div class="hero-section__inner">...</div></div>
6. Every <img>: width, height, alt, loading="lazy", decoding="async" — each on its own line. WebP/AVIF only. LCP image: loading="eager" + <link rel="preload"> in <head>.
   Standard: <img src="/assets/images/photo.webp" width="800" height="600" alt="..." loading="lazy" decoding="async" />
   LCP: <img src="/assets/images/hero.webp" width="1440" height="900" alt="..." loading="eager" decoding="async" />
7. Every h1–h6 wrapped in .title + color modifier (from Figma). No classes on heading tags directly.
    <div class="title title-black"><h1>text</h1></div>
8. Every <p> wrapped in .content + color modifier (from Figma). No classes on <p> directly.
    <div class="content content-black"><p>text</p></div>
9. alt never blank. Always descriptive.
10. Phone: tel:{number}. Email: mailto:{address}.
11. All <section> inside <main>.
12. Only these Tailwind classes in HTML: p-_, m-_, pt-_, pb-_ (+ responsive max-*:pt-_/pb-_), bg-_ (section bg only), max-w-_, flex, flex-col, flex-row, gap, grid, grid-cols-_. Everything else → CSS files via @apply.
13. SEO: semantic elements, accessibility attributes, single h1, clear heading hierarchy.
14. Every input has linked <label> (for="x" + id="x").
15. Clean indentation throughout.
16. All Figma-to-HTML output pixel-perfect by default — always match every spacing, font size, color, border-radius, shadow, and layout value exactly. Zero approximations.
    EVERY token value comes from Figma — never invent numbers to "fill a scale". If Figma only
    supplies 48px and 16px, define only those; do NOT guess --text-heading-1:56px etc. to complete
    the h1–h6 ladder. Missing heading steps = only add when a real Figma value needs them. Snapping
    a Figma value to the nearest existing token is a violation — add the exact value as a new token.
17. Scripts: defer or type="module" (preferred). Never bare <script> in <head>.
    Correct: <script defer src="..."> / <script type="module" src="...">
    Avoid: <script src="...">
18. Preload LCP image + above-fold fonts in <head>. No non-critical preloads.
    <link rel="preload" as="image" href="/assets/images/hero.webp" />
    <link rel="preload" as="font" type="font/woff2" href="/assets/fonts/inter.woff2" crossorigin />
19. Never use Figma-hosted URLs. Workflow: Export via MCP → save to /assets/images/ → convert to .webp → use local path.
    Correct: <img src="/assets/images/hero.webp" ... />
    Avoid: <img src="https://figma-alpha-api.s3.us-west-2.amazonaws.com/..." ... />
20. Icons via <img> tags only — never inline SVG. Dimensions must match Figma exactly.
    <img src="/assets/icons/arrow.webp" width="24" height="24" alt="Arrow icon" loading="lazy" decoding="async" />
21. One <nav> for all screen sizes. No separate mobile nav. Reuse desktop nav with responsive CSS.
    Correct: <nav class="main-nav">...</nav>
    Avoid: <nav class="desktop-nav">...</nav> + <nav class="mobile-nav">...</nav>
22. Never extract overlay or border-radius images from Figma. Overlays = CSS gradient/bg color.
    Rounded corners = CSS border-radius on the element. Export only the real content image.
23. List bullets: never use an <img> dot inside <ul><li>. Draw the dot in CSS via ::before/::after
    pseudo-element. Wrap every list in .list + a color modifier class; dot color comes from the
    second class (e.g. .list.blue → blue dot). Structure:
    <div class="list blue">
      <ul>
        <li>1</li>
      </ul>
    </div>
    CSS: .list ul li{ @apply relative pl-* } .list ul li::before{ ...dot via bg-color... }
    .list.blue ul li::before{ @apply bg-blue } — dot color driven by the modifier class.

========== CSS RULES ==========

1. CSS files: style.css, base.css, component.css, layout.css, utilities.css.
2. All class names kebab-case. Correct: .hero-section. Avoid: .hero_section.
3. All styles via @apply with Tailwind v4 utilities. No raw CSS (except @font-face).
4. Breakpoints from style.css only. max-_/min-_ prefixes. No @media queries.
5. All responsive classes in same @apply line.
   h1,.h1{ @apply text-heading-1 max-768:text-20 }
6. No [] arbitrary values anywhere. Avoid: @apply text-[20px] max-w-[452px]
7. No fixed h-* or min-h-*. Layouts content-driven. h-full OK.
8. No fixed w-* or min-w-*. Layouts content-driven. w-full OK.
9. Tailwind classes in HTML: see HTML rule 12.

10. style.css — only @theme{} containing:

    IMPORT ORDER:
    @import "base.css"; @import "component.css"; @import "layout.css"; @import "utilities.css";

    BREAKPOINTS:
    --breakpoint-1920:1921px; --breakpoint-1600:1601px; --breakpoint-1512:1513px;
    --breakpoint-1440:1441px; --breakpoint-1366:1367px; --breakpoint-1199:1200px;
    --breakpoint-1024:1025px; --breakpoint-992:993px; --breakpoint-768:769px;
    --breakpoint-640:641px; --breakpoint-576:577px; --breakpoint-425:426px;
    --breakpoint-375:376px;

    SPACING: --spacing: 1px;
    FONT FAMILY: From Figma MCP, dynamic. Example: --font-inter: "Inter", sans-serif;
    COLORS: From Figma MCP, dynamic. Example: --color-white: #FFFFFF;
    FONT SIZES: From Figma MCP. Largest→h1 descending. --text-heading-1 through --text-heading-6 mandatory. Rest: --text-{size}.
    Example: --text-heading-1:48px; --text-heading-2:40px; ... --text-20:20px; --text-16:16px;

11. base.css contains:

    HEADINGS — base font sizes always defined. Responsive at 1440 + 1024 always included.
    h1,.h1{ @apply text-heading-1 max-1440:text-heading-2 max-1024:text-heading-3 max-768:text-22 }
    h2,.h2{ @apply text-heading-2 max-1440:text-heading-3 max-1024:text-heading-4 max-768:text-20 }
    h3,.h3{ @apply text-heading-3 max-1440:text-heading-4 max-1024:text-heading-5 max-768:text-18 }
    h4,.h4{ @apply text-heading-4 max-1440:text-heading-5 max-1024:text-heading-6 max-768:text-16 }
    h5,.h5{ @apply text-heading-5 max-1440:text-heading-6 max-1024:text-20 max-768:text-16 }
    h6,.h6{ @apply text-heading-6 max-1440:text-20 max-1024:text-18 max-768:text-14 }
    NOTE: Actual values from Figma MCP, not above example.

    PARAGRAPHS:
    .content p{ @apply text-16 }
    .content p + p{ @apply mt-10 }

    TITLE COLORS (from Figma, dynamic):
    .title-white h1,.title-white h2,.title-white h3,
    .title-white h4,.title-white h5,.title-white h6{ @apply text-white }

    CONTENT COLORS (from Figma, dynamic):
    .content-white p{ @apply text-white }
    .content-black p{ @apply text-black }

    CONTAINERS (Figma X-axis spacing, no max-width). These are the ONLY horizontal-padding
    wrappers allowed anywhere in the theme — every section's container-level px must reuse one
    of these three. Never add a new custom-px container class for a specific section (see HTML
    rule 5). If a new px value shows up in Figma, extend one of these three shared classes with a
    new breakpoint step instead of creating a fourth class.
    .container-fluid{ @apply px-50 }
    .container-fluid-md{ @apply px-80 }
    .container-fluid-lg{ @apply px-100 }

    NOTE: Section top/bottom padding (Y-axis) is NOT a .general-padding class — apply pt-*/pb-*
    utilities directly on the <section> tag (HTML rules 5, 12). Section bg color is also inline via
    the bg-* utility, never a per-section class here.

12. component.css — buttons, inputs, textarea, select (all from Figma via @apply):
    .btn{ @apply px-16 py-8 inline-flex text-center rounded-10 border-1 border-solid cursor-pointer }
    .btn-black{ @apply text-white bg-black border-black hover:bg-white hover:text-white }
    All button variants have hover + transition. Input/textarea/select match Figma exactly.

13. layout.css — header and footer styles only.
14. utilities.css — section-related CSS and extra helper classes.
15. No CSS variables in :root or any CSS file. Use style.css @theme{} vars only.
16. No @layer blocks in any CSS file.
17. Always Tailwind v4 classes.
18. Animate only transform and opacity. Never width, height, top, left, margin, padding.
    Correct: .card{ @apply transition-transform duration-300 ease-out hover:scale-105 }
    Avoid: .card{ @apply transition-all hover:h-80 }
19. Prefer CSS animations over JS. Scroll effects: see JS rule 6.
20. Always font-display: swap on @font-face. Never block or auto.
21. Responsive always generated — even without explicit request. PREFER real Figma responsive frames:
    if the design has tablet/mobile frames, read their EXACT values (font size, padding, gap, cols) at
    1440/1024/768 and use those. Only when Figma has NO responsive frame may you fall back to scaling
    down the heading ladder — and when you do, note it inline as an assumption. Never auto-invent
    breakpoint values when Figma already specifies them. All sections/containers/spacing get responsive
    adjustments from Figma's actual frames.
22. Never page-specific or content-specific class names. Always global layout-pattern names.
    Correct: .image-with-content, .grid-section, .slider-section, .zigzag-section, .cards-section
    Avoid: .about-zigzag, .services-grid, .home-slider, .team-cards
23. When fetching Figma via MCP: if any color, font size, or button style is NOT in style.css @theme{}, add it there first before writing any CSS. Never hardcode values in component.css/utilities.css.
    Workflow: Fetch Figma → check @theme{} → missing? add to style.css → then use via @apply.
    Correct: style.css: --color-accent:#E94F2E; / component.css: .btn-accent{ @apply bg-color-accent }
    Avoid: component.css: .btn-accent{ @apply bg-[#E94F2E] }

========== PERFORMANCE RULES ==========

1. CLS — width + height on every <img>/<video>. Dynamic content: use aspect-ratio container.
   .media-wrapper{ @apply aspect-video w-full overflow-hidden }
2. IMAGES — max 500 KB. Use srcset + sizes for responsive.
   <img src="/assets/images/photo.webp" srcset="...400.webp 400w, ...800.webp 800w" sizes="(max-width:768px) 100vw, 50vw" width="800" height="600" alt="..." loading="lazy" decoding="async" />
3. JAVASCRIPT — defer/async, split files, lean bundles, CSS over JS animation. See JS rules 1–4, 6–7.
4. CSS — used classes only, no transition-all, flat selectors, no speculative utilities.
5. FONTS — WOFF2 in assets/fonts/ (no CDNs). @font-face + font-display:swap in base.css. Preload above-fold fonts only. Subset to used chars.
   @font-face { font-family:"Inter"; src:url("/assets/fonts/inter.woff2") format("woff2"); font-weight:400; font-style:normal; font-display:swap; }
   NOTE: @font-face is the only exception to the no-raw-CSS rule.
   MANDATORY — the actual .woff2 files MUST exist on disk before build. A @font-face rule pointing at a
   missing file = browser falls back to a system font = every text metric wrong = whole UI looks off.
   This is the #1 cause of inaccurate Figma output. For each font family/weight in @theme{}: download
   the .woff2 (Figma assets or the font source), save to the fonts dir, and VERIFY the file exists.
   Never ship a @font-face whose src file is absent.
6. A11Y + SEO — semantic elements, descriptive alt, one h1, no skipped levels, keyboard accessible, WCAG AA contrast (4.5:1 body / 3:1 large), meta description + title on every page.
7. CODE QUALITY — minimal DOM depth, no pattern repeated 3+ times (extract to class), comments only for non-obvious WHY, no dead/commented code.
8. PAGESPEED — Performance:90+ | Accessibility:95+ | Best Practices:95+ | SEO:95+ | LCP<2.5s | CLS<0.1 | TBT<200ms | INP<200ms. Document tradeoffs inline.
9. OUTPUT — production-ready only. Flag CWV-harming Figma patterns before implementing.

========== JS RULES ==========

1. FILE STRUCTURE — one file per feature in assets/js/. Each exports single default init fn.
   assets/js/ ├── app.js (entry only) ├── header.js ├── slider.js ├── accordion.js ├── animation.js └── tabView.js
2. ENTRY POINT — app.js: imports + one DOMContentLoaded calling each init fn. No feature logic.
   import initHeader from './header.js'
   document.addEventListener('DOMContentLoaded', function(){ initHeader() })
3. MODULE FORMAT — ES modules (export default) only. No CommonJS. <script type="module" src="/assets/js/app.js">
4. PERFORMANCE — type="module" on app.js. No document.write(). No sync XHR. Guard missing nodes:
   export default function initAccordion(){ const el=document.querySelector('.accordion'); if(!el) return; }
5. EVENT LISTENERS — inside init fn only. Never at module top level.
6. SCROLL & RESIZE — IntersectionObserver over scroll listeners. Unavoidable listeners: debounce min 100ms. Visual updates: requestAnimationFrame not setInterval.
   export default function initAnimation(){ const els=document.querySelectorAll('[data-animate]'); if(!els.length) return; const obs=new IntersectionObserver(e=>e.forEach(x=>x.target.classList.toggle('is-visible',x.isIntersecting))); els.forEach(el=>obs.observe(el)); }
7. LIBRARIES — no whole-library imports for vanilla-achievable tasks. If needed, import specific module in feature file only (not app.js).
   // slider.js: import Swiper from 'swiper' ✓
   // app.js: import _ from 'lodash' ✗
