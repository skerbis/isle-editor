# REDAXO CMS 集成

本指南说明如何将 isle-editor 集成到 REDAXO CMS 中。REDAXO 是一个灵活的内容管理系统，在德语国家特别受欢迎，为构建自定义 Web 应用程序提供了强大的框架。

## 前提条件

- 已安装 REDAXO 5.x 或更高版本
- 基本的 REDAXO 插件开发知识
- Node.js 和 pnpm（用于构建 isle-editor）

## 集成方法

有几种方法可以将 isle-editor 集成到 REDAXO 中：

1. **作为 REDAXO 插件** - 与 REDAXO 后端完全集成
2. **通过 CDN** - 简单用例的快速集成
3. **自定义构建** - 将 isle-editor 与您的 REDAXO 主题一起构建和打包

## 方法 1：REDAXO 插件集成

### 步骤 1：创建 REDAXO 插件

在 `/redaxo/src/addons/isle_editor/` 中创建新的插件结构：

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

### 步骤 2：配置 package.yml

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

### 步骤 3：创建 boot.php

```php
<?php
/**
 * REDAXO 的 Isle Editor 插件
 */

if (rex::isBackend()) {
    // 将资源添加到后端
    rex_view::addCssFile($this->getAssetsUrl('css/isle-editor.css'));
    rex_view::addJsFile($this->getAssetsUrl('js/isle-editor.js'));
}

// 将编辑器注册为 REX_VAR
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

### 步骤 4：创建编辑器库

创建 `/lib/isle_editor.php`：

```php
<?php

class isle_editor {
    /**
     * 生成编辑器的 HTML
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
            // 等待 DOM 和 isle-editor 准备就绪
            document.addEventListener("DOMContentLoaded", function() {
                if (typeof IsleEditor === "undefined") {
                    console.error("Isle Editor 未加载");
                    return;
                }
                
                const editorElement = document.getElementById("' . $editorId . '");
                const inputElement = document.getElementById("' . $editorId . '-input");
                
                const editor = new IsleEditor.Editor({
                    element: editorElement,
                    extensions: [
                        IsleEditor.StarterKit,
                        // 根据需要添加更多扩展
                    ],
                    content: inputElement.value || "",
                    onUpdate: ({ editor }) => {
                        // 用编辑器内容更新隐藏输入
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
     * 获取编辑器配置
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

### 步骤 5：构建并复制资源

构建 isle-editor 并复制分发文件：

```bash
# 在 isle-editor 仓库中
cd /path/to/isle-editor
pnpm install
pnpm build:packages

# 将构建的文件复制到 REDAXO 插件
cp packages/core/dist/index.umd.js /path/to/redaxo/src/addons/isle_editor/assets/js/isle-editor.js
cp packages/core/dist/style.css /path/to/redaxo/src/addons/isle_editor/assets/css/isle-editor.css
```

### 步骤 6：在模板中使用

在您的 REDAXO 模块中使用编辑器：

```php
<?php
// 在您的模块输入（Eingabe）中
echo isle_editor::getEditorHTML('REX_INPUT_VALUE[1]', 1, 'REX_VALUE[1]');
?>
```

或使用 REX_VAR 语法：

```php
REX_ISLE_EDITOR[id=1 name="article_content"]
```

输出（Ausgabe）：

```php
<?php
// 简单地输出 HTML 内容
echo 'REX_VALUE[1]';
?>
```

## 方法 2：CDN 集成

对于快速设置，您可以通过 CDN 使用 isle-editor：

### 步骤 1：添加到 REDAXO 模板

```php
<?php
// 在您的模板头部
?>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@isle-editor/core/dist/style.css">
<script src="https://cdn.jsdelivr.net/npm/@isle-editor/core/dist/index.umd.js"></script>

<?php
// 在您的模块输入中
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

## 方法 3：使用 Vue 的自定义构建

如果您想在 REDAXO 中使用 Vue 3 组件：

### 步骤 1：创建 Vue 应用程序

创建 `assets/src/editor-app.js`：

```javascript
import { createApp } from "vue";
import { IsleEditor } from "@isle-editor/vue3";
import "@isle-editor/core/dist/style.css";

// 在所有具有 'isle-editor-mount' 类的元素上初始化编辑器
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
          placeholder="开始输入..."
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

### 步骤 2：使用 Vite 构建

创建 `vite.config.js`：

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

### 步骤 3：在 REDAXO 中使用

```php
<?php
// 模块输入
?>
<div class="isle-editor-mount" data-input-id="editor-content-1"></div>
<input type="hidden" name="REX_INPUT_VALUE[1]" id="editor-content-1" value="REX_VALUE[1]">
```

## 高级集成

### 自定义 REDAXO 表单元素

创建自定义表单元素类：

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

### 图片上传集成

与 REDAXO 的媒体池集成：

```javascript
// 在编辑器初始化中
const editor = new IsleEditor.Editor({
  element: document.getElementById("editor"),
  extensions: [
    IsleEditor.StarterKit,
    IsleEditor.Image.configure({
      // 自定义上传处理器
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
        return data.url; // 上传图片的 URL
      },
    }),
  ],
});
```

在 `lib/api.php` 中创建 API 端点：

```php
<?php
class rex_api_isle_editor_upload extends rex_api_function {
    protected $published = true;
    
    public function execute() {
        // 检查权限
        if (!rex::getUser() || !rex::getUser()->hasPerm('media[upload]')) {
            return new rex_api_result(false, '权限被拒绝');
        }
        
        // 处理文件上传
        if (isset($_FILES['file'])) {
            $file = $_FILES['file'];
            
            // 使用 REDAXO 媒体管理器
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
        
        return new rex_api_result(false, '上传失败');
    }
}
```

## 本地化

在 `/lang/de_de.lang` 中添加翻译：

```ini
isle_editor_title = Isle Editor
isle_editor_placeholder = Beginnen Sie mit dem Tippen...
isle_editor_toolbar_bold = Fett
isle_editor_toolbar_italic = Kursiv
isle_editor_upload_error = Upload fehlgeschlagen
```

和 `/lang/en_gb.lang`：

```ini
isle_editor_title = Isle Editor
isle_editor_placeholder = Start typing...
isle_editor_toolbar_bold = Bold
isle_editor_toolbar_italic = Italic
isle_editor_upload_error = Upload failed
```

## 提示和最佳实践

1. **版本控制** - 记录您使用的 isle-editor 版本
2. **资源管理** - 使用 REDAXO 的资源管理进行缓存清除
3. **权限** - 与 REDAXO 的权限系统集成
4. **媒体池** - 使用 REDAXO 的媒体池进行图片管理
5. **多语言** - 支持 REDAXO 的多语言功能
6. **备份** - 更新插件前始终备份

## 故障排除

### 编辑器未初始化

检查浏览器控制台错误：
- 确保 isle-editor 脚本已加载
- 检查 JavaScript 错误
- 验证 DOM 元素是否存在

### 内容未保存

- 检查隐藏输入字段是否正确更新
- 验证表单提交是否正常工作
- 检查 REDAXO 日志错误

### 样式问题

- 确保 CSS 已加载
- 检查与 REDAXO 后端样式的 CSS 冲突
- 使用特定的 CSS 选择器避免冲突

## 示例：完整的模块

以下是带有 isle-editor 的完整 REDAXO 模块示例：

**模块输入：**
```php
<?php
// 如果尚未加载，加载所需资源
if (!rex_view::hasJsFile('isle-editor')) {
    rex_view::addCssFile($this->getAssetsUrl('../../addons/isle_editor/assets/css/isle-editor.css'));
    rex_view::addJsFile($this->getAssetsUrl('../../addons/isle_editor/assets/js/isle-editor.js'));
    rex_view::setJsProperty('isle-editor', true);
}

$id = 'REX_MODULE_ID';
$content = 'REX_VALUE[1]';
?>
<div class="form-group">
    <label for="isle-editor-<?php echo $id; ?>">内容</label>
    <?php echo isle_editor::getEditorHTML('REX_INPUT_VALUE[1]', $id, $content); ?>
</div>
```

**模块输出：**
```php
<div class="article-content">
    REX_VALUE[1]
</div>
```

## 下一步

- 阅读[扩展开发](./extension-development.md)指南以创建自定义扩展
- 了解[自定义扩展](./customizing-extensions.md)以满足 REDAXO 特定需求
- 查看 REDAXO 文档：[redaxo.org](https://www.redaxo.org/)
- 在 [Slack](https://redaxo.org/slack/) 加入 REDAXO 社区
