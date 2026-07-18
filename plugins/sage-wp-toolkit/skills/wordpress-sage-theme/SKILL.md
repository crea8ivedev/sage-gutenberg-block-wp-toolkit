---
name: wordpress-sage-theme
description: 'Provides WordPress theme development patterns using Sage (roots/sage) framework. Use when creating, modifying, or debugging WordPress themes with Sage, including (1): creating new Sage themes from scratch, (2): setting up Blade templates and components, (3): configuring build tools (Vite, Bud), (4): working with WordPress theme templates and hierarchy, (5): implementing ACF fields integration, (6): theme customization and asset management.'
allowed-tools: Read, Write, Bash, Glob, Grep
---

# WordPress Sage Theme Development

## Overview

Sage is a WordPress theme framework by Roots that provides modern development practices including Blade templates, dependency management with Composer, and build tools with Vite (Sage 11) or Bud (legacy Sage 10).

## When to Use

- Creating new Sage themes from scratch or from composer templates
- Setting up Blade templates, layouts, and reusable components
- Configuring the Vite build (or Bud on legacy Sage 10) for asset compilation
- Working with WordPress template hierarchy in Blade format
- Integrating Advanced Custom Fields (ACF) with Blade templates
- Debugging theme rendering, asset loading, or build issues

## Instructions

1. **Set up the environment**: Install PHP 8.2+, Node.js 20+, Composer, and create a new Sage theme with `composer create-project roots/sage`
2. **Configure build tools**: Run `npm install && composer install`, then configure `vite.config.js` for asset entries and Tailwind v4
3. **Create Blade templates**: Place templates in `resources/views/`, using layouts in `layouts/`, components in `components/`
4. **Wire up WordPress templates**: Map WordPress template hierarchy to Blade files (e.g., `page.blade.php` for page templates)
5. **Integrate ACF fields**: Use `get_field()` for basic fields, `have_rows()` loops for repeaters and flexible content
6. **Build and verify**: Run `npm run build`, verify `public/manifest.json` exists, check browser console for asset errors
7. **Deploy**: Ensure the production build step (`npm run build`) runs during deployment; raw source files cannot be served directly

## Examples

**Create a new Sage theme:**

```bash
composer create-project roots/sage vabeau
cd vabeau
npm install && composer install
npm run dev
```

**Blade page template:**

```blade
@extends('layouts.app')

@section('content')
  <main class="content">
    <h1>{{ the_title() }}</h1>
    <div class="entry-content">
      {{ the_content() }}
    </div>
  </main>
@endsection
```

**ACF flexible content in Blade:**

```blade
@if (have_rows('flexible_content'))
  @while (have_rows('flexible_content'))
    @php the_row() @endphp
    @switch(get_row_layout())
      @case('hero_section')
        @include('components.hero')
        @break
    @endswitch
  @endwhile
@endif
```

## Quick Start

### Creating a New Sage Theme

**Prerequisites**: PHP 8.2+, Node.js 20+, Composer

```bash
# Create new Sage theme
wp scaffold theme-theme vabeau --theme_name="My Theme" --author="Your Name" --activate

# Or install Sage directly via Composer
composer create-project roots/sage vabeau
cd vabeau

# Install dependencies
npm install
composer install

# Build for development
npm run dev

# Build for production
npm run build
```

### Directory Structure

```
resources/
├── views/           # Blade templates
│   ├── layouts/     # Base layouts (app.blade.php)
│   ├── components/  # Reusable components
│   └── partials/    # Template partials
├── css/             # Stylesheets (Sage 11)
│   ├── app.css      # Main stylesheet entry
│   └── editor.css   # Gutenberg editor styles
└── js/              # JavaScript (Sage 11)
    ├── app.js       # Main JS entry
    └── editor.js    # Gutenberg editor entry
```
(Legacy Sage 10 used `resources/styles/` + `resources/scripts/` instead.)

## Blade Templates

### Layouts

**Base Layout** (`resources/views/layouts/app.blade.php`):

```blade
<!DOCTYPE html>
<html {{ site_html_language_attributes() }}>
  <head>
    {{ wp_head() }}
  </head>
  <body {{ body_class() }}>
    @yield('content')
    {{ wp_footer() }}
  </body>
</html>
```

### Template Hierarchy Mapping

| WordPress Template | Sage Blade File              |
| ------------------ | ---------------------------- |
| front-page.php     | `views/front-page.blade.php` |
| single.php         | `views/single.blade.php`     |
| page.php           | `views/page.blade.php`       |
| archive.php        | `views/archive.blade.php`    |
| index.php          | `views/index.blade.php`      |

**Example Page Template** (`resources/views/page.blade.php`):

```blade
@extends('layouts.app')

@section('content')
  <main class="content">
    <h1>{{ the_title() }}</h1>
    <div class="entry-content">
      {{ the_content() }}
    </div>
  </main>
@endsection
```

### Components

**Reusable Button Component** (`resources/views/components/button.blade.php`):

```blade
@props(['url' => '#', 'text' => 'Click', 'variant' => 'primary'])

<a href="{{ $url }}" class="btn btn-{{ $variant }}">
  {{ $text }}
</a>
```

**Usage**:

```blade
<x-button url="/contact" text="Contact Us" variant="secondary" />
```

## ACF Integration

### Displaying ACF Fields

**Basic Field**:

```blade
@while(the_post())
  <h1>{{ get_field('hero_title') ?? the_title() }}</h1>
  <p>{{ get_field('hero_description') }}</p>
@endwhile
```

**Flexible Content**:

```blade
@if (have_rows('flexible_content'))
  @while (have_rows('flexible_content'))
    @php the_row() @endphp

    @switch(get_row_layout())
      @case('hero_section')
        @include('components.hero')
        @break

      @case('features_grid')
        @include('components.features-grid')
        @break
    @endswitch
  @endwhile
@endif
```

**Repeater Field**:

```blade
@if (have_rows('testimonials'))
  <div class="testimonials">
    @while (have_rows('testimonials'))
      @php the_row() @endphp
      <blockquote>
        <p>{{ get_sub_field('testimonial_text') }}</p>
        <cite>{{ get_sub_field('author_name') }}</cite>
      </blockquote>
    @endwhile
  </div>
@endif
```

## Build Configuration (Vite — Sage 11 default)

Sage 11 uses Vite (`vite.config.js`), not Bud. Bud applies only to older Sage 10 projects — see [bud.md](references/bud.md) if you're maintaining one.

**Install Tailwind v4**:

```bash
npm install -D tailwindcss @tailwindcss/vite
```

**Configure** (`vite.config.js`):

```js
import { defineConfig } from 'vite'
import laravel from 'laravel-vite-plugin'
import { wordpressPlugin, wordpressThemeJson } from '@roots/vite-plugin'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  base: '/app/themes/sage/public/build/',
  plugins: [
    tailwindcss(),
    laravel({
      input: [
        'resources/css/app.css',
        'resources/js/app.js',
        'resources/css/editor.css',
        'resources/js/editor.js',
      ],
      refresh: true,
    }),
    wordpressPlugin(),
    wordpressThemeJson({ disableTailwindColors: false }),
  ],
  resolve: {
    alias: {
      '@scripts': '/resources/js',
      '@styles': '/resources/css',
      '@fonts': '/resources/fonts',
      '@images': '/resources/images',
    },
  },
})
```

**Tailwind CSS v4** (`resources/css/app.css`) — no `tailwind.config.js`; theme tokens live in CSS:

```css
@import "tailwindcss";

@theme {
  --color-brand: #0f172a;
  --font-inter: "Inter", sans-serif;
}
```

Keep `@wordpress/*` packages out of the bundle — WordPress provides them at runtime (enqueue via `app/setup.php`).

## Advanced Patterns

### Conditional Logic

```blade
@if (is_front_page())
  @include('components.hero')
@elseif (is_singular('post'))
  @include('components.post-meta')
@endif
```

### Custom Queries

```blade
@php
  $args = [
    'post_type' => 'service',
    'posts_per_page' => 6,
  ];
  $query = new WP_Query($args);
@endphp

@if ($query->have_posts())
  @while ($query->have_posts())
    @php $query->the_post() @endphp
    <article>
      <h2>{{ the_title() }}</h2>
    </article>
  @endwhile
  @php wp_reset_postdata() @endphp
@endif
```

### Theme Customization

**functions.php additions**:

```php
// Custom image sizes
add_image_size('hero-lg', 1920, 1080, true);

// Custom post types
add_action('init', function() {
  register_post_type('service', [
    'label' => 'Services',
    'public' => true,
    'has_archive' => true,
    'supports' => ['title', 'editor', 'thumbnail'],
  ]);
});
```

## References

- **Sage Documentation**: See [sage.md](references/sage.md) for complete framework reference
- **Blade Templates**: See [blade.md](references/blade.md) for advanced Blade patterns
- **Bud Configuration**: See [bud.md](references/bud.md) for build tool configuration
- **ACF Integration**: See [acf.md](references/acf.md) for ACF field examples

## Troubleshooting

**Build workflow validation** (always run in order):

1. `npm run build` completes without errors
2. Verify `public/manifest.json` exists and contains asset entries
3. Test locally in browser before production push
4. Check browser console for asset 404 errors

**Build issues**:

```bash
# Clear cache
npm run clean

# Rebuild
rm -rf node_modules public
npm install
npm run build

# Validate build output
ls -la public/
cat public/manifest.json | head -20
```

**Blade not compiling**: Check `public/manifest.json` exists after build

**AC fields not showing**: Verify field names match exactly (case-sensitive)

## Best Practices

### Blade Template Organization

- **Use components for reusable UI elements**: Create blade components in `resources/views/components/` for buttons, cards, and other repeated elements
- **Keep layouts minimal**: Base layouts should only contain structural HTML; delegate styling to components
- **Leverage Blade directives**: Use `@include`, `@each`, and `@component` for better code organization
- **Avoid inline PHP**: Use `@php` blocks sparingly; move complex logic to Composers or service classes

### ACF Field Management

- **Group related fields**: Use ACF field groups to organize fields logically by content type
- **Use field keys for references**: When referencing fields programmatically, use field keys (e.g., `field_12345678`) for reliability
- **Implement fallback values**: Always provide default values for optional fields to prevent empty output
- **Cache expensive queries**: Use WordPress transients for complex ACF queries on high-traffic pages

### Asset Management

- **Use Vite for asset compilation**: Let Vite handle versioning and optimization instead of manual asset management
- **Minimize HTTP requests**: Combine CSS/JS files where appropriate using Vite's entry points
- **Optimize images**: Use WordPress image sizes and lazy loading; consider WebP conversion
- **Cache busting**: Vite automatically handles cache busting via the build manifest

### Code Organization

- **Use Composers for data**: Move data retrieval logic from templates to Sage Composers (`app/View/Composers/`)
- **Separate concerns**: Keep business logic in service classes, presentation in Blade templates
- **Follow WordPress coding standards**: Maintain consistency with WordPress PHP coding standards
- **Use type declarations**: PHP 8.0+ allows typed properties and arguments for better code quality

### Security Considerations

- **Escape all output**: Use `{{ }}` (Blade auto-escapes) or WordPress functions like `esc_html()`, `esc_url()`
- **Validate ACF input**: Sanitize and validate custom field inputs on the backend
- **Nonce verification**: Always verify nonces for form submissions
- **Capability checks**: Use `current_user_can()` before privileged operations

## Constraints and Warnings

### Version Requirements

- **PHP 8.2+ required**: Sage 11 requires PHP 8.2 or higher (legacy Sage 10: PHP 8.0+)
- **Node.js 20+ required**: Build tools require modern Node.js; older versions may cause compilation errors
- **WordPress 6.0+ recommended**: While Sage works with WordPress 5.x, version 6.0+ is recommended for full feature support
- **Composer required**: Dependency management requires Composer; manual installation is not supported

### Build Tool Limitations

- **Hot reload limitations**: Vite HMR may not work correctly with some WordPress multisite configurations
- **Production builds required for testing**: Some features work differently in development vs production; always test with `npm run build` before deployment
- **Manifest.json dependency**: Theme relies on `public/manifest.json`; missing this file breaks asset loading

### Common Pitfalls

- **Template hierarchy confusion**: Sage uses Blade files but follows WordPress template hierarchy; ensure file names match WordPress expectations
- **Direct file access**: Do not access Blade files directly via URL; they must be rendered through WordPress
- **Plugin conflicts**: Some caching and security plugins may interfere with Vite's dev server or asset serving
- **Theme updates**: Updating Sage via Composer may overwrite customizations; use child themes for extensive modifications

### Performance Considerations

- **ACF Repeater performance**: Large repeater fields can impact page load; consider pagination or alternative storage for large datasets
- **Blade compilation overhead**: First load after cache clear compiles all templates; use OPcache in production
- **Asset bundle size**: Monitor compiled asset sizes; large bundles affect page load performance
- **Database queries**: Use Query Monitor plugin to identify and optimize expensive queries in theme templates

### Deployment Constraints

- **Build step required**: Production deployments must run `npm run build`; raw source files cannot be served
- **Environment-specific configuration**: the Vite `base` path may need adjustment for different deployment environments (Bedrock vs plain WP)
- **File permissions**: Ensure `public/` directory is writable during builds; incorrect permissions cause build failures
