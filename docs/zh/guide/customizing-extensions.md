# 自定义扩展

本指南向您展示如何自定义现有的 isle-editor 扩展以满足您的特定需求。您可以修改选项、添加新功能或覆盖默认行为。

## 扩展现有扩展

`.extend()` 方法允许您修改现有扩展，而无需创建全新的扩展。

### 基本扩展

以下是如何自定义 Bold 扩展：

```javascript
import Bold from "@isle-editor/core/extensions/bold";

const CustomBold = Bold.extend({
  addOptions() {
    return {
      ...this.parent?.(),
      // 覆盖键盘快捷键
      shortcutkeys: "Mod-Shift-B",
      // 添加自定义 CSS 类
      HTMLAttributes: {
        class: "my-bold-text",
      },
    };
  },
});
```

### 更改默认选项

您可以在配置编辑器时更改默认选项：

```javascript
import { Editor } from "@isle-editor/core";
import Bold from "@isle-editor/core/extensions/bold";

const editor = new Editor({
  extensions: [
    Bold.configure({
      HTMLAttributes: {
        class: "font-bold",
      },
    }),
    // ... 其他扩展
  ],
});
```

## 自定义节点扩展

### 修改属性

向现有节点添加自定义属性：

```javascript
import Heading from "@tiptap/extension-heading";

const CustomHeading = Heading.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      id: {
        default: null,
        parseHTML: (element) => element.getAttribute("id"),
        renderHTML: (attributes) => {
          if (!attributes.id) {
            return {};
          }
          return { id: attributes.id };
        },
      },
    };
  },
});
```

### 自定义渲染

覆盖节点渲染为 HTML 的方式：

```javascript
import Blockquote from "@isle-editor/core/extensions/blockquote";

const CustomBlockquote = Blockquote.extend({
  renderHTML({ HTMLAttributes }) {
    return [
      "div",
      { class: "custom-blockquote" },
      ["blockquote", HTMLAttributes, 0],
    ];
  },
});
```

### 添加新命令

使用额外的命令扩展现有节点：

```javascript
import Image from "@tiptap/extension-image";

const EnhancedImage = Image.extend({
  addCommands() {
    return {
      ...this.parent?.(),
      setImageWithCaption:
        (options) =>
        ({ commands }) => {
          return commands.insertContent({
            type: "figure",
            content: [
              {
                type: this.name,
                attrs: options,
              },
              {
                type: "figcaption",
                content: [{ type: "text", text: options.caption || "" }],
              },
            ],
          });
        },
    };
  },
});
```

## 自定义标记扩展

### 更改标记行为

自定义标记的应用方式：

```javascript
import Link from "@tiptap/extension-link";

const CustomLink = Link.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      rel: {
        default: "noopener noreferrer nofollow",
      },
      target: {
        default: "_blank",
      },
    };
  },

  addCommands() {
    return {
      ...this.parent?.(),
      setLink:
        (attributes) =>
        ({ chain }) => {
          return chain()
            .setMark(this.name, attributes)
            .setMeta("preventAutolink", true)
            .run();
        },
    };
  },
});
```

## 自定义键盘快捷键

### 覆盖默认快捷键

更改任何扩展的键盘快捷键：

```javascript
import CodeBlock from "@isle-editor/core/extensions/code-block";

const CustomCodeBlock = CodeBlock.extend({
  addKeyboardShortcuts() {
    return {
      ...this.parent?.(),
      // 更改代码块的快捷键
      "Mod-Alt-C": () => this.editor.commands.toggleCodeBlock(),
      // 通过返回 false 移除快捷键
      "Mod-Shift-C": () => false,
    };
  },
});
```

### 添加新快捷键

添加额外的键盘快捷键：

```javascript
import BulletList from "@tiptap/extension-bullet-list";

const EnhancedBulletList = BulletList.extend({
  addKeyboardShortcuts() {
    return {
      ...this.parent?.(),
      "Shift-Ctrl-8": () => this.editor.commands.toggleBulletList(),
    };
  },
});
```

## 自定义输入规则

### 修改或添加输入规则

输入规则根据用户输入触发命令：

```javascript
import { wrappingInputRule } from "@tiptap/core";
import Blockquote from "@isle-editor/core/extensions/blockquote";

const CustomBlockquote = Blockquote.extend({
  addInputRules() {
    return [
      // 添加新的输入规则
      wrappingInputRule({
        find: /^:::\s$/,
        type: this.type,
      }),
      // 保留现有规则
      ...this.parent?.(),
    ];
  },
});
```

## 自定义扩展存储

扩展可以存储状态：

```javascript
import { Extension } from "@tiptap/core";

const CustomExtension = Extension.create({
  name: "customExtension",

  addStorage() {
    return {
      count: 0,
      history: [],
    };
  },

  onUpdate() {
    this.storage.count += 1;
    this.storage.history.push({
      timestamp: Date.now(),
      content: this.editor.getJSON(),
    });
  },
});

// 访问存储
console.log(editor.storage.customExtension.count);
```

## 自定义事件处理器

### 监听编辑器事件

```javascript
import { Extension } from "@tiptap/core";

const EventLogger = Extension.create({
  name: "eventLogger",

  onCreate() {
    console.log("编辑器已创建");
  },

  onUpdate() {
    console.log("编辑器已更新");
  },

  onSelectionUpdate() {
    console.log("选择已更改");
  },

  onTransaction({ transaction }) {
    console.log("事务：", transaction);
  },

  onFocus() {
    console.log("编辑器已聚焦");
  },

  onBlur() {
    console.log("编辑器已失焦");
  },

  onDestroy() {
    console.log("编辑器已销毁");
  },
});
```

## 自定义命令面板

isle-editor 包含一个可以用 `/` 触发的命令面板。自定义扩展在其中的显示方式：

```javascript
import { Node } from "@tiptap/core";

const CustomNode = Node.create({
  name: "customNode",

  addOptions() {
    return {
      // 在斜杠命令菜单中显示
      slash: true,
      // 命令面板中显示的名称
      name: "自定义块",
      // 命令面板中显示的描述
      desc: "插入自定义块元素",
      // 从面板选择时执行的命令
      command: ({ editor, range }) => {
        range
          ? editor.chain().focus().deleteRange(range).setCustomNode().run()
          : editor.chain().focus().setCustomNode().run();
      },
      // 检查命令是否激活
      isActive: ({ editor }) => editor.isActive("customNode"),
      // 检查命令是否禁用
      isDisabled: ({ editor }) => !editor.can().setCustomNode(),
      // 键盘快捷键提示
      shortcutkeys: "Mod-Shift-C",
    };
  },
});
```

## 实际示例

### 示例 1：带自动生成 ID 的自定义标题

```javascript
import Heading from "@tiptap/extension-heading";

function generateId(text) {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/(^-|-$)/g, "");
}

const HeadingWithId = Heading.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      id: {
        default: null,
        rendered: false,
      },
    };
  },

  onUpdate() {
    const { state } = this.editor;
    const { doc } = state;

    doc.descendants((node, pos) => {
      if (node.type.name === this.name) {
        const id = generateId(node.textContent);
        if (node.attrs.id !== id) {
          this.editor
            .chain()
            .setTextSelection(pos)
            .updateAttributes(this.name, { id })
            .run();
        }
      }
    });
  },
});
```

### 示例 2：只读代码块

```javascript
import CodeBlock from "@isle-editor/core/extensions/code-block";

const ReadOnlyCodeBlock = CodeBlock.extend({
  addAttributes() {
    return {
      ...this.parent?.(),
      readOnly: {
        default: false,
        parseHTML: (element) => element.getAttribute("data-readonly") === "true",
        renderHTML: (attributes) => {
          if (attributes.readOnly) {
            return { "data-readonly": "true" };
          }
          return {};
        },
      },
    };
  },

  addKeyboardShortcuts() {
    return {
      ...this.parent?.(),
      Backspace: () => {
        const { selection } = this.editor.state;
        const node = selection.$anchor.parent;

        if (node.type.name === this.name && node.attrs.readOnly) {
          return true; // 阻止删除
        }
        return false;
      },
    };
  },
});
```

### 示例 3：带验证的增强链接

```javascript
import Link from "@tiptap/extension-link";

const ValidatedLink = Link.extend({
  addCommands() {
    return {
      ...this.parent?.(),
      setLink:
        (attributes) =>
        ({ chain }) => {
          // 验证 URL
          if (!attributes.href) {
            return false;
          }

          try {
            new URL(attributes.href);
          } catch {
            console.error("无效的 URL");
            return false;
          }

          // 如果缺少协议，自动添加
          if (!/^https?:\/\//i.test(attributes.href)) {
            attributes.href = "https://" + attributes.href;
          }

          return chain().setMark(this.name, attributes).run();
        },
    };
  },
});
```

## 配置模式

### 全局配置

初始化编辑器时全局配置扩展：

```javascript
import { Editor } from "@isle-editor/core";

const editor = new Editor({
  extensions: [
    Document,
    Paragraph.configure({
      HTMLAttributes: {
        class: "my-paragraph",
      },
    }),
    Heading.configure({
      levels: [1, 2, 3],
    }),
    Link.configure({
      openOnClick: false,
      linkOnPaste: true,
    }),
  ],
});
```

### 条件配置

根据环境或用户偏好配置扩展：

```javascript
const isDarkMode = window.matchMedia("(prefers-color-scheme: dark)").matches;

const editor = new Editor({
  extensions: [
    CodeBlock.configure({
      HTMLAttributes: {
        class: isDarkMode ? "code-dark" : "code-light",
      },
    }),
  ],
});
```

## 提示和最佳实践

1. **始终调用父方法** - 使用 `...this.parent?.()` 保留原始功能
2. **全面测试** - 使用不同的内容和场景测试您的自定义
3. **记录更改** - 记录您自定义的内容及原因
4. **检查兼容性** - 确保您的自定义与其他扩展兼容
5. **考虑性能** - 避免在频繁调用的方法（如 `onUpdate`）中进行昂贵的操作
6. **使用 TypeScript** - 类型定义有助于早期发现错误

## 调试技巧

### 记录扩展活动

```javascript
const DebugExtension = Extension.extend({
  onCreate() {
    console.log("选项：", this.options);
  },

  onUpdate() {
    console.log("内容：", this.editor.getJSON());
    console.log("HTML：", this.editor.getHTML());
  },
});
```

### 检查编辑器状态

```javascript
// 获取当前选择
console.log(editor.state.selection);

// 获取当前节点
console.log(editor.state.selection.$anchor.parent);

// 检查标记是否激活
console.log(editor.isActive("bold"));

// 获取所有激活的标记
console.log(editor.state.selection.$from.marks());
```

## 下一步

- 了解如何[从头开始创建自定义扩展](./extension-development.md)
- 浏览 `packages/core/src/extensions/` 中的现有扩展
- 查看 [REDAXO 集成](./redaxo-integration.md)示例
- 阅读 [Tiptap 文档](https://tiptap.dev/)了解高级技术
