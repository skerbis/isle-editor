# REDAXO CMS Integration

This guide explains how to integrate isle-editor into REDAXO CMS, a flexible content management system. REDAXO is particularly popular in German-speaking countries and provides a powerful framework for building custom web applications.

## Prerequisites

- REDAXO 5.x or later installed
- Basic knowledge of REDAXO addon development
- Node.js and pnpm (for building isle-editor)

## Integration Methods

There are several ways to integrate isle-editor into REDAXO:

1. **As a REDAXO Addon** - Full integration with REDAXO's backend
2. **Via CDN** - Quick integration for simple use cases
3. **Custom Build** - Build and bundle isle-editor with your REDAXO theme

## Method 1: REDAXO Addon Integration

### Step 1: Create a REDAXO Addon

Create a new addon structure in `/redaxo/src/addons/isle_editor/`:

```
isle_editor/
├── assets/
│   ├── css/
│   │   └── isle-editor.css
│   └── js/
│       └── isle-editor.js
├── lib/
│   └── isle_editor.php
├── pages/
│   └── index.php
├── boot.php
└── package.yml
```

### Step 2: Configure package.yml

```yaml
package: isle_editor
version: "1.0.0"
author: Your Name
supportpage: https://github.com/yourusername/isle-editor-redaxo

page:
  title: "Isle Editor"
  perm: admin[]
  icon: rex-icon fa-edit

requires:
  redaxo: "^5.10"
  php:
    version: "^8.0"
```

### Step 3: Create boot.php

```php
<?php
/**
 * Isle Editor Addon for REDAXO
 */

if (rex::isBackend()) {
    // Add assets to backend
    rex_view::addCssFile($this->getAssetsUrl('css/isle-editor.css'));
    rex_view::addJsFile($this->getAssetsUrl('js/isle-editor.js'));
}

// Register the editor as a REX_VAR
if (!function_exists('rex_var_isle_editor')) {
    class rex_var_isle_editor extends rex_var {
        protected function getOutput() {
            $id = $this->getParsedArg('id', 1);
            $name = $this->getParsedArg('name', 'REX_ISLE_EDITOR_' . $id);
            
            return sprintf(
                'isle_editor::getEditorHTML(%s, %s, $this->getValue(%s))',
                rex_escape($name),
                rex_escape($id),
                $id
            );
        }
    }
}
```

### Step 4: Create the Editor Library

Create `/lib/isle_editor.php`:

```php
<?php

class isle_editor {
    /**
     * Generate HTML for the editor
     */
    public static function getEditorHTML($name, $id = 1, $content = '') {
        $content = htmlspecialchars($content, ENT_QUOTES, 'UTF-8');
        $editorId = 'isle-editor-' . $id;
        
        $html = '
        <div class="isle-editor-wrapper">
            <div id="' . $editorId . '" class="isle-editor-container"></div>
            <input type="hidden" name="' . $name . '" id="' . $editorId . '-input" value="' . $content . '">
        </div>
        
        <script>
        (function() {
            // Wait for DOM and isle-editor to be ready
            document.addEventListener("DOMContentLoaded", function() {
                if (typeof IsleEditor === "undefined") {
                    console.error("Isle Editor not loaded");
                    return;
                }
                
                const editorElement = document.getElementById("' . $editorId . '");
                const inputElement = document.getElementById("' . $editorId . '-input");
                
                const editor = new IsleEditor.Editor({
                    element: editorElement,
                    extensions: [
                        IsleEditor.StarterKit,
                        // Add more extensions as needed
                    ],
                    content: inputElement.value || "",
                    onUpdate: ({ editor }) => {
                        // Update hidden input with editor content
                        inputElement.value = editor.getHTML();
                    }
                });
            });
        })();
        </script>
        ';
        
        return $html;
    }
    
    /**
     * Get editor configuration
     */
    public static function getConfig() {
        return [
            'toolbar' => true,
            'menubar' => true,
            'placeholder' => rex_i18n::msg('isle_editor_placeholder'),
        ];
    }
}
```

### Step 5: Build and Copy Assets

Build isle-editor and copy the distribution files:

```bash
# In isle-editor repository
cd /path/to/isle-editor
pnpm install
pnpm build:packages

# Copy built files to REDAXO addon
cp packages/core/dist/index.umd.js /path/to/redaxo/src/addons/isle_editor/assets/js/isle-editor.js
cp packages/core/dist/style.css /path/to/redaxo/src/addons/isle_editor/assets/css/isle-editor.css
```

### Step 6: Use in Templates

In your REDAXO modules, use the editor:

```php
<?php
// In your module input (Eingabe)
echo isle_editor::getEditorHTML('REX_INPUT_VALUE[1]', 1, 'REX_VALUE[1]');
?>
```

Or using REX_VAR syntax:

```php
REX_ISLE_EDITOR[id=1 name="article_content"]
```

For output (Ausgabe):

```php
<?php
// Simply output the HTML content
echo 'REX_VALUE[1]';
?>
```

## Method 2: CDN Integration

For a quick setup, you can use isle-editor via CDN:

### Step 1: Add to REDAXO Template

```php
<?php
// In your template head
?>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@isle-editor/core/dist/style.css">
<script src="https://cdn.jsdelivr.net/npm/@isle-editor/core/dist/index.umd.js"></script>

<?php
// In your module input
?>
<div id="isle-editor-1"></div>
<input type="hidden" name="REX_INPUT_VALUE[1]" id="editor-content-1" value="REX_VALUE[1]">

<script>
document.addEventListener('DOMContentLoaded', function() {
    const editor = new IsleEditor.Editor({
        element: document.getElementById('isle-editor-1'),
        extensions: [IsleEditor.StarterKit],
        content: document.getElementById('editor-content-1').value,
        onUpdate: ({ editor }) => {
            document.getElementById('editor-content-1').value = editor.getHTML();
        }
    });
});
</script>
```

## Method 3: Custom Build with Vue

If you want to use the Vue 3 components in REDAXO:

### Step 1: Create a Vue Application

Create `assets/src/editor-app.js`:

```javascript
import { createApp } from "vue";
import { IsleEditor } from "@isle-editor/vue3";
import "@isle-editor/core/dist/style.css";

// Initialize editors on all elements with class 'isle-editor-mount'
document.addEventListener("DOMContentLoaded", () => {
  const editorMounts = document.querySelectorAll(".isle-editor-mount");

  editorMounts.forEach((mount) => {
    const inputId = mount.dataset.inputId;
    const inputElement = document.getElementById(inputId);

    const app = createApp({
      components: { IsleEditor },
      data() {
        return {
          content: inputElement ? inputElement.value : "",
        };
      },
      watch: {
        content(newValue) {
          if (inputElement) {
            inputElement.value = newValue;
          }
        },
      },
      template: `
        <isle-editor
          v-model="content"
          :extensions="extensions"
          placeholder="Start typing..."
        />
      `,
      computed: {
        extensions() {
          return ["StarterKit"];
        },
      },
    });

    app.mount(mount);
  });
});
```

### Step 2: Build with Vite

Create `vite.config.js`:

```javascript
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
  plugins: [vue()],
  build: {
    outDir: "../../redaxo/src/addons/isle_editor/assets",
    rollupOptions: {
      input: "src/editor-app.js",
      output: {
        entryFileNames: "js/editor-app.js",
        assetFileNames: "css/editor-app.css",
      },
    },
  },
});
```

### Step 3: Use in REDAXO

```php
<?php
// Module input
?>
<div class="isle-editor-mount" data-input-id="editor-content-1"></div>
<input type="hidden" name="REX_INPUT_VALUE[1]" id="editor-content-1" value="REX_VALUE[1]">
```

## Advanced Integration

### Custom REDAXO Form Element

Create a custom form element class:

```php
<?php
class rex_form_isle_editor_element extends rex_form_element {
    private $editorConfig = [];
    
    public function setEditorConfig(array $config) {
        $this->editorConfig = $config;
        return $this;
    }
    
    public function formatElement() {
        $value = htmlspecialchars($this->getValue(), ENT_QUOTES, 'UTF-8');
        $name = $this->getAttribute('name');
        $id = $this->getAttribute('id');
        
        $config = json_encode($this->editorConfig);
        
        return '
        <div class="isle-editor-wrapper">
            <div id="' . $id . '-editor" class="isle-editor"></div>
            <input type="hidden" name="' . $name . '" id="' . $id . '" value="' . $value . '">
        </div>
        <script>
        (function() {
            const config = ' . $config . ';
            const editor = new IsleEditor.Editor({
                element: document.getElementById("' . $id . '-editor"),
                content: "' . $value . '",
                onUpdate: ({ editor }) => {
                    document.getElementById("' . $id . '").value = editor.getHTML();
                },
                ...config
            });
        })();
        </script>
        ';
    }
}
```

### Image Upload Integration

Integrate with REDAXO's media pool:

```javascript
// In your editor initialization
const editor = new IsleEditor.Editor({
  element: document.getElementById("editor"),
  extensions: [
    IsleEditor.StarterKit,
    IsleEditor.Image.configure({
      // Custom upload handler
      uploadFn: async (file) => {
        const formData = new FormData();
        formData.append("file", file);
        formData.append("rex-api-call", "isle_editor_upload");

        const response = await fetch(
          "/index.php?rex-api-call=isle_editor_upload",
          {
            method: "POST",
            body: formData,
          }
        );

        const data = await response.json();
        return data.url; // URL to uploaded image
      },
    }),
  ],
});
```

Create the API endpoint in `lib/api.php`:

```php
<?php
class rex_api_isle_editor_upload extends rex_api_function {
    protected $published = true;
    
    public function execute() {
        // Check permissions
        if (!rex::getUser() || !rex::getUser()->hasPerm('media[upload]')) {
            return new rex_api_result(false, 'Permission denied');
        }
        
        // Handle file upload
        if (isset($_FILES['file'])) {
            $file = $_FILES['file'];
            
            // Use REDAXO media manager
            $media = rex_media::get($file['name']);
            if (!$media) {
                $upload = rex_media_service::addMedia([
                    'name' => $file['name'],
                    'file' => $file['tmp_name'],
                ], rex_media_category::get(1), [], rex::getUser()->getLogin());
                
                if ($upload['ok']) {
                    $media = rex_media::get($upload['filename']);
                }
            }
            
            if ($media) {
                return new rex_api_result(true, null, [
                    'url' => rex_url::media($media->getFileName())
                ]);
            }
        }
        
        return new rex_api_result(false, 'Upload failed');
    }
}
```

## Localization

Add translations to your addon in `/lang/de_de.lang`:

```ini
isle_editor_title = Isle Editor
isle_editor_placeholder = Beginnen Sie mit dem Tippen...
isle_editor_toolbar_bold = Fett
isle_editor_toolbar_italic = Kursiv
isle_editor_upload_error = Upload fehlgeschlagen
```

And `/lang/en_gb.lang`:

```ini
isle_editor_title = Isle Editor
isle_editor_placeholder = Start typing...
isle_editor_toolbar_bold = Bold
isle_editor_toolbar_italic = Italic
isle_editor_upload_error = Upload failed
```

## Tips and Best Practices

1. **Version Control** - Keep track of which version of isle-editor you're using
2. **Asset Management** - Use REDAXO's asset management for cache busting
3. **Permissions** - Integrate with REDAXO's permission system
4. **Media Pool** - Use REDAXO's media pool for image management
5. **Multilingual** - Support REDAXO's multi-language capabilities
6. **Backup** - Always backup before updating the addon

## Troubleshooting

### Editor Not Initializing

Check browser console for errors:
- Ensure isle-editor scripts are loaded
- Check for JavaScript errors
- Verify DOM element exists

### Content Not Saving

- Check that the hidden input field is properly updated
- Verify form submission works
- Check REDAXO logs for errors

### Styling Issues

- Ensure CSS is loaded
- Check for CSS conflicts with REDAXO's backend styles
- Use specific CSS selectors to avoid conflicts

## Example: Complete Module

Here's a complete example of a REDAXO module with isle-editor:

**Module Input:**
```php
<?php
// Load required assets if not already loaded
if (!rex_view::hasJsFile('isle-editor')) {
    rex_view::addCssFile($this->getAssetsUrl('../../addons/isle_editor/assets/css/isle-editor.css'));
    rex_view::addJsFile($this->getAssetsUrl('../../addons/isle_editor/assets/js/isle-editor.js'));
    rex_view::setJsProperty('isle-editor', true);
}

$id = 'REX_MODULE_ID';
$content = 'REX_VALUE[1]';
?>
<div class="form-group">
    <label for="isle-editor-<?php echo $id; ?>">Content</label>
    <?php echo isle_editor::getEditorHTML('REX_INPUT_VALUE[1]', $id, $content); ?>
</div>
```

**Module Output:**
```php
<div class="article-content">
    REX_VALUE[1]
</div>
```

## Next Steps

- Read the [Extension Development](./extension-development.md) guide to create custom extensions
- Learn about [Customizing Extensions](./customizing-extensions.md) for REDAXO-specific needs
- Check out REDAXO's documentation at [redaxo.org](https://www.redaxo.org/)
- Join the REDAXO community on [Slack](https://redaxo.org/slack/)
