# Add Branding Section

Add a new settings section to the branding admin page following this project's exact architectural pattern.

## What the user wants

The user will describe a section they want to add, for example:
- "Add a Hero section with tagline and background image fields"
- "Add a Contact Info section with phone, email, address"
- "Add a Social Proof section with a badge image and trust text"

Read their description, derive sensible field names and types, then implement ALL 5 layers below. Do not ask for clarification — infer and implement.

---

## Layer 1 — BrandingServiceProvider.php

File: `app/Providers/BrandingServiceProvider.php`

### 1a. Add option name constant (at top of class constants block)

```php
public const MY_SECTION_OPTION_NAME = 'radicle_my_section_settings';
```

Naming rule: `radicle_{snake_section_name}_settings`

### 1b. Add defaults constant

```php
public const DEFAULT_MY_SECTION_SETTINGS = [
    'field_one' => '',
    'field_two' => '',
    // image fields always need both id + url pair
    'image_id'  => 0,
    'image_url' => '',
];
```

Field types by use case:
- text input → `''`
- boolean/checkbox → `false`
- image → two fields: `'image_id' => 0` and `'image_url' => ''`
- select/radio → `'style-1'`
- repeating list → `[]`

### 1c. Add getter method

```php
public function getMySectionSettings(): array
{
    $saved = get_option(self::MY_SECTION_OPTION_NAME, []);
    $settings = array_merge(self::DEFAULT_MY_SECTION_SETTINGS, is_array($saved) ? $saved : []);

    // If section has an image field, resolve URL from ID if missing
    if ($settings['image_id'] && empty($settings['image_url'])) {
        $settings['image_url'] = wp_get_attachment_image_url($settings['image_id'], 'large') ?: '';
    }

    return $settings;
}
```

### 1d. Add AJAX handler method

```php
public function handleMySectionSave(): void
{
    check_ajax_referer('my_section_settings_nonce', 'nonce');

    if (! current_user_can('manage_options')) {
        wp_send_json_error(['message' => 'Unauthorized'], 403);
    }

    $settings = [
        'field_one' => sanitize_text_field($_POST['field_one'] ?? ''),
        'field_two' => sanitize_textarea_field($_POST['field_two'] ?? ''),
        'image_id'  => isset($_POST['image_id']) ? absint($_POST['image_id']) : 0,
    ];

    // Resolve image URL server-side after saving ID
    if ($settings['image_id']) {
        $settings['image_url'] = wp_get_attachment_image_url($settings['image_id'], 'large') ?: '';
    } else {
        $settings['image_url'] = '';
    }

    update_option(self::MY_SECTION_OPTION_NAME, $settings);

    wp_send_json_success([
        'message'  => 'My section settings saved successfully',
        'settings' => $settings,
    ]);
}
```

Sanitization rules — apply correct one per field:
| Field type | Sanitizer |
|---|---|
| Short text | `sanitize_text_field()` |
| Multi-line text | `sanitize_textarea_field()` |
| URL | `esc_url_raw()` |
| Integer / image ID | `absint()` |
| Boolean checkbox | `(bool) ($_POST['field'] ?? false)` |
| Hex color | `sanitize_hex_color()` after `isValidHexColor()` check |
| Repeating JSON list | `json_decode(stripslashes($_POST['items'] ?? '[]'), true)` then loop and sanitize each item |

### 1e. Register AJAX action in `register()` method

Add inside the existing `register()` method alongside the other `add_action` calls:

```php
add_action('wp_ajax_save_my_section_settings', [$this, 'handleMySectionSave']);
```

Action name rule: `save_{snake_section_name}_settings`

### 1f. Load and pass settings in `renderAdminPage()`

Add the getter call:
```php
$mySectionSettings = $this->getMySectionSettings();
```

Add the variable to the `compact()` call:
```php
echo view('admin.branding', compact(
    // ... existing variables ...
    'mySectionSettings',
))->render();
```

---

## Layer 2 — branding.blade.php: Alpine state

File: `resources/views/admin/branding.blade.php`

Find the `Alpine.data('brandingPage', () => ({` block near the bottom of the file. Add the new section's state inside that object alongside the existing sections:

```js
// My Section settings
mySectionSettings: @json($mySectionSettings),
savingMySection: false,
mySectionMessage: '',
mySectionMessageType: 'success',
// If section has an image field, also add:
mySectionMediaFrame: null,
```

State naming rule: `{camelSectionName}Settings`, `saving{PascalSection}`, `{camelSection}Message`, `{camelSection}MessageType`

---

## Layer 3 — branding.blade.php: save method

Inside the same `Alpine.data` object, add the save method following this exact structure (every save method in this project is identical except variable names and FormData fields):

```js
async saveMySectionSettings() {
    this.savingMySection = true;
    this.mySectionMessage = '';

    try {
        const formData = new FormData();
        formData.append('action', 'save_my_section_settings');
        formData.append('nonce', '{{ wp_create_nonce('my_section_settings_nonce') }}');
        formData.append('field_one', this.mySectionSettings.field_one);
        formData.append('field_two', this.mySectionSettings.field_two);
        formData.append('image_id', this.mySectionSettings.image_id ?? 0);

        const response = await fetch(ajaxurl, {
            method: 'POST',
            body: formData,
        });

        const data = await response.json();

        if (data.success) {
            this.mySectionMessage = data.data.message;
            this.mySectionMessageType = 'success';
        } else {
            this.mySectionMessage = data.data?.message || 'An error occurred while saving.';
            this.mySectionMessageType = 'error';
        }
    } catch (error) {
        this.mySectionMessage = 'An error occurred while saving.';
        this.mySectionMessageType = 'error';
        console.error('Save error:', error);
    } finally {
        this.savingMySection = false;
        setTimeout(() => { this.mySectionMessage = ''; }, 3000);
    }
},
```

**Nonce string** in `wp_create_nonce()` here must exactly match the string in `check_ajax_referer()` in the PHP handler. They are the same string — copy exactly.

For repeating list fields, send as JSON string:
```js
formData.append('items', JSON.stringify(this.mySectionSettings.items));
```

---

## Layer 4 — branding.blade.php: HTML section

Add the HTML section block in the blade file **before** the closing `</div>` of the main `x-data` wrapper and before the `<script>` block. Place it logically near related sections.

### Basic section template

```html
{{-- My Section --}}
<div class="branding-section branding-my-section-section">
    <h2>{{ __('My Section', 'radicle') }}</h2>
    <p class="section-description">{{ __('Manage my section settings.', 'radicle') }}</p>

    <table class="form-table">
        {{-- Text field --}}
        <tr>
            <th scope="row">{{ __('Field One', 'radicle') }}</th>
            <td>
                <input type="text"
                       x-model="mySectionSettings.field_one"
                       class="regular-text"
                       placeholder="{{ __('Enter value...', 'radicle') }}">
            </td>
        </tr>

        {{-- Textarea field --}}
        <tr>
            <th scope="row">{{ __('Field Two', 'radicle') }}</th>
            <td>
                <textarea x-model="mySectionSettings.field_two"
                          class="regular-text" rows="4"></textarea>
            </td>
        </tr>

        {{-- Boolean / checkbox field --}}
        <tr>
            <th scope="row">{{ __('Enable Feature', 'radicle') }}</th>
            <td>
                <label class="toggle-option">
                    <input type="checkbox" x-model="mySectionSettings.enable_feature">
                    <span class="toggle-label">
                        <strong>{{ __('Enable Feature', 'radicle') }}</strong>
                        <span class="toggle-description">{{ __('Description here.', 'radicle') }}</span>
                    </span>
                </label>
            </td>
        </tr>

        {{-- Radio / select field --}}
        <tr>
            <th scope="row">{{ __('Style', 'radicle') }}</th>
            <td>
                <label>
                    <input type="radio" name="my-section-style" value="style-1" x-model="mySectionSettings.style">
                    {{ __('Style 1', 'radicle') }}
                </label>
                <label>
                    <input type="radio" name="my-section-style" value="style-2" x-model="mySectionSettings.style">
                    {{ __('Style 2', 'radicle') }}
                </label>
            </td>
        </tr>

        {{-- Image upload field --}}
        <tr>
            <th scope="row">{{ __('Image', 'radicle') }}</th>
            <td>
                <div class="logo-preview" :class="{ 'has-logo': mySectionSettings.image_id }">
                    <template x-if="mySectionSettings.image_url">
                        <img :src="mySectionSettings.image_url" alt="" style="max-width:200px;">
                    </template>
                    <template x-if="!mySectionSettings.image_url">
                        <div class="logo-placeholder">
                            <span class="dashicons dashicons-format-image"></span>
                            <span>{{ __('No image set', 'radicle') }}</span>
                        </div>
                    </template>
                </div>
                <div style="margin-top:8px; display:flex; gap:8px;">
                    <button type="button" class="button button-secondary" @click="openMySectionMedia()">
                        <span x-show="!mySectionSettings.image_id">{{ __('Upload Image', 'radicle') }}</span>
                        <span x-show="mySectionSettings.image_id">{{ __('Change Image', 'radicle') }}</span>
                    </button>
                    <button type="button" class="button" x-show="mySectionSettings.image_id"
                            @click="mySectionSettings.image_id = 0; mySectionSettings.image_url = ''">
                        {{ __('Remove', 'radicle') }}
                    </button>
                </div>
            </td>
        </tr>
    </table>

    <div class="branding-actions">
        <button type="button" class="button button-primary"
                @click="saveMySectionSettings()" :disabled="savingMySection">
            <span x-show="!savingMySection">{{ __('Save My Section Settings', 'radicle') }}</span>
            <span x-show="savingMySection">{{ __('Saving...', 'radicle') }}</span>
        </button>
        <span x-show="mySectionMessage" x-text="mySectionMessage"
              :class="mySectionMessageType === 'success' ? 'notice-success' : 'notice-error'"
              class="inline-notice"></span>
    </div>
</div>
```

### Repeating list field (like announcements)

Add methods to the Alpine object:
```js
addMySectionItem() {
    this.mySectionSettings.items.push({ title: '', image_id: 0, image_url: '' });
},
removeMySectionItem(index) {
    this.mySectionSettings.items.splice(index, 1);
},
```

HTML for the repeating list:
```html
<template x-for="(item, index) in mySectionSettings.items" :key="index">
    <div style="display:flex; gap:10px; margin-bottom:10px; align-items:center;">
        <input type="text" x-model="mySectionSettings.items[index].title"
               class="regular-text" placeholder="{{ __('Title...', 'radicle') }}">
        <button type="button" class="button button-secondary"
                @click="removeMySectionItem(index)">
            <span class="dashicons dashicons-trash" style="margin-top:4px;"></span>
        </button>
    </div>
</template>
<button type="button" class="button" @click="addMySectionItem()">
    {{ __('Add Item', 'radicle') }}
</button>
```

### Image upload media library method (add to Alpine object if section has image):

```js
openMySectionMedia() {
    if (this.mySectionMediaFrame) {
        this.mySectionMediaFrame.open();
        return;
    }
    this.mySectionMediaFrame = wp.media({
        title: 'Select Image',
        button: { text: 'Use this image' },
        multiple: false,
        library: { type: 'image' },
    });
    this.mySectionMediaFrame.on('select', () => {
        const attachment = this.mySectionMediaFrame.state().get('selection').first().toJSON();
        this.mySectionSettings.image_id = attachment.id;
        this.mySectionSettings.image_url = attachment.url;
    });
    this.mySectionMediaFrame.open();
},
```

---

## Layer 5 — storage/framework/providers.php (only if creating a brand new ServiceProvider)

If the branding section lives in an **existing** `BrandingServiceProvider.php`, no change needed here — it is already registered.

If adding a **new standalone ServiceProvider** for this section, add the class to `storage/framework/providers.php`:

```php
App\Providers\MySectionServiceProvider::class,
```

**Do not modify `config/app.php` providers array** — this project registers providers via `storage/framework/providers.php` (Laravel Acorn/WordPress pattern). That file is the authoritative list.

---

## Checklist before finishing

After implementing, verify:

- [ ] Constant name: `radicle_{snake}_settings`
- [ ] AJAX action: `save_{snake}_settings` — matches in `add_action`, `formData.append('action', ...)`, and PHP handler
- [ ] Nonce string: exact same value in `check_ajax_referer()`, `wp_create_nonce()` in blade
- [ ] Getter passed to `compact()` in `renderAdminPage()`
- [ ] Alpine state uses `@json($variable)` for initial value
- [ ] Save method `finally` block resets `saving{X}` to `false` and clears message after 3000ms
- [ ] All `$_POST` values sanitized with correct sanitizer for field type
- [ ] Image fields: save ID, resolve URL server-side in handler, also resolve in getter if URL missing

---

## Example usage

User: "Add a Contact Bar section with a phone number field, an email field, and a toggle to show/hide it."

You would:
1. Constant: `CONTACT_BAR_OPTION_NAME = 'radicle_contact_bar_settings'`
2. Defaults: `['phone' => '', 'email' => '', 'enabled' => true]`
3. Getter: `getContactBarSettings()` using `get_option` + `array_merge`
4. Handler: `handleContactBarSave()` — sanitize phone as `sanitize_text_field`, email as `sanitize_email`, enabled as `(bool)`
5. Register: `add_action('wp_ajax_save_contact_bar_settings', [$this, 'handleContactBarSave'])`
6. Pass: `$contactBarSettings = $this->getContactBarSettings()` + add to `compact()`
7. Alpine state: `contactBarSettings: @json($contactBarSettings), savingContactBar: false, ...`
8. Save method: `saveContactBarSettings()` with `action: 'save_contact_bar_settings'`, nonce `'contact_bar_settings_nonce'`
9. HTML: table with text inputs for phone/email, checkbox for enabled, save button + message span
