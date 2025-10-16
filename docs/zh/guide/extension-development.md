# 扩展开发

本指南将帮助您为 isle-editor 创建自定义扩展。扩展允许您向编辑器添加新功能、节点类型、标记和特性。

## 理解扩展

isle-editor 基于 [Tiptap](https://tiptap.dev/) 构建，而 Tiptap 又基于 [ProseMirror](https://prosemirror.net/)。扩展是为编辑器添加特定功能的模块化组件。

主要有三种类型的扩展：

1. **节点（Nodes）** - 块级内容（段落、标题、图片等）
2. **标记（Marks）** - 内联格式（粗体、斜体、链接等）
3. **扩展（Extensions）** - 通用功能（键盘快捷键、命令等）

## 创建简单的标记扩展

让我们创建一个自定义的高亮标记，允许用户用背景颜色高亮文本。

### 步骤 1：创建扩展文件

在 `packages/core/src/extensions/highlight.js` 中创建新文件：

```javascript
import { Mark, mergeAttributes } from "@tiptap/core";

const source = {
  name: "highlight",
  desc: "高亮文本",
  command: ({ editor }) => editor.chain().focus().toggleHighlight().run(),
  isActive: ({ editor }) => editor.isActive("highlight"),
  isDisabled: ({ editor }) => !editor.can().toggleHighlight(),
  shortcutkeys: "Mod-Shift-H",
};

export default Mark.create({
  name: "highlight",

  addOptions() {
    return {
      HTMLAttributes: {},
      ...source,
    };
  },

  parseHTML() {
    return [
      {
        tag: "mark",
      },
      {
        style: "background-color",
        getAttrs: (value) => !!value && null,
      },
    ];
  },

  renderHTML({ HTMLAttributes }) {
    return [
      "mark",
      mergeAttributes(this.options.HTMLAttributes, HTMLAttributes),
      0,
    ];
  },

  addCommands() {
    return {
      setHighlight:
        () =>
        ({ commands }) => {
          return commands.setMark(this.name);
        },
      toggleHighlight:
        () =>
        ({ commands }) => {
          return commands.toggleMark(this.name);
        },
      unsetHighlight:
        () =>
        ({ commands }) => {
          return commands.unsetMark(this.name);
        },
    };
  },

  addKeyboardShortcuts() {
    return {
      "Mod-Shift-h": () => this.editor.commands.toggleHighlight(),
    };
  },
});
```

### 步骤 2：导出扩展

将扩展添加到 `packages/core/src/index.js`：

```javascript
// ... 其他导入
export { default as Highlight } from "./extensions/highlight.js";
```

### 步骤 3：使用扩展

现在您可以在编辑器中使用自定义扩展：

```javascript
import { Editor } from "@isle-editor/core";
import { Highlight } from "@isle-editor/core";

const editor = new Editor({
  extensions: [
    // ... 其他扩展
    Highlight,
  ],
});
```

## 创建节点扩展

让我们创建一个自定义的标注/警告节点：

```javascript
import { Node, mergeAttributes } from "@tiptap/core";

const source = {
  slash: true,
  name: "callout",
  desc: "添加标注块",
  command: ({ editor, range }) => {
    range
      ? editor.chain().focus().deleteRange(range).setCallout().run()
      : editor.chain().focus().setCallout().run();
  },
  isActive: ({ editor }) => editor.isActive("callout"),
  isDisabled: ({ editor }) => !editor.can().setCallout(),
};

export default Node.create({
  name: "callout",

  addOptions() {
    return {
      HTMLAttributes: {
        class: "callout",
      },
      types: ["info", "warning", "error", "success"],
      ...source,
    };
  },

  content: "block+",

  group: "block",

  defining: true,

  addAttributes() {
    return {
      type: {
        default: "info",
        parseHTML: (element) => element.getAttribute("data-type"),
        renderHTML: (attributes) => {
          return {
            "data-type": attributes.type,
            class: `callout callout-${attributes.type}`,
          };
        },
      },
    };
  },

  parseHTML() {
    return [
      {
        tag: "div[data-type]",
      },
    ];
  },

  renderHTML({ HTMLAttributes }) {
    return [
      "div",
      mergeAttributes(this.options.HTMLAttributes, HTMLAttributes),
      0,
    ];
  },

  addCommands() {
    return {
      setCallout:
        (attributes) =>
        ({ commands }) => {
          return commands.wrapIn(this.name, attributes);
        },
      toggleCallout:
        (attributes) =>
        ({ commands }) => {
          return commands.toggleWrap(this.name, attributes);
        },
      unsetCallout:
        () =>
        ({ commands }) => {
          return commands.lift(this.name);
        },
    };
  },
});
```

## 创建功能扩展

功能扩展添加通用功能而不渲染内容。这是一个字数统计扩展的示例：

```javascript
import { Extension } from "@tiptap/core";

export default Extension.create({
  name: "wordCount",

  addOptions() {
    return {
      limit: null,
    };
  },

  addStorage() {
    return {
      words: 0,
      characters: 0,
    };
  },

  onUpdate() {
    const text = this.editor.state.doc.textContent;
    this.storage.characters = text.length;
    this.storage.words = text.split(/\s+/).filter((word) => word.length > 0)
      .length;
  },
});
```

## 扩展选项

扩展可以定义选项，允许用户自定义行为：

```javascript
addOptions() {
  return {
    // 默认选项
    HTMLAttributes: {},
    placeholder: "输入一些内容...",
    maxLength: null,
    
    // 命令面板的源对象选项
    name: "myExtension",
    desc: "命令面板的描述",
    command: ({ editor }) => editor.commands.doSomething(),
    isActive: ({ editor }) => editor.isActive("myExtension"),
    isDisabled: ({ editor }) => !editor.can().doSomething(),
    shortcutkeys: "Mod-Shift-X",
    slash: true, // 在斜杠命令菜单中显示
  };
}
```

## 扩展命令

命令是修改编辑器状态的函数：

```javascript
addCommands() {
  return {
    setMyNode:
      (attributes) =>
      ({ commands }) => {
        return commands.insertContent({
          type: this.name,
          attrs: attributes,
        });
      },
    toggleMyNode:
      () =>
      ({ commands }) => {
        return commands.toggleNode(this.name, "paragraph");
      },
  };
}
```

## 键盘快捷键

为扩展添加键盘快捷键：

```javascript
addKeyboardShortcuts() {
  return {
    // Mod 在 Mac 上是 Cmd，在 Windows/Linux 上是 Ctrl
    "Mod-Shift-x": () => this.editor.commands.setMyNode(),
    "Mod-Alt-x": () => this.editor.commands.toggleMyNode(),
    Enter: () => {
      // 自定义 Enter 键行为
      return this.editor.commands.createParagraphNear();
    },
  };
}
```

## 输入规则

输入规则允许您根据用户输入触发命令：

```javascript
import { wrappingInputRule, textblockTypeInputRule } from "@tiptap/core";

addInputRules() {
  return [
    // 输入 ">" 时用引用块包装选择
    wrappingInputRule({
      find: /^\s*>\s$/,
      type: this.type,
    }),
    
    // 输入 "# " 时转换为标题
    textblockTypeInputRule({
      find: /^(#{1,6})\s$/,
      type: this.type,
      getAttributes: (match) => ({ level: match[1].length }),
    }),
  ];
}
```

## 最佳实践

1. **保持扩展专注** - 每个扩展应该做好一件事
2. **遵循命名约定** - 扩展名称使用驼峰命名法
3. **提供良好的默认值** - 使扩展开箱即用
4. **记录选项** - 帮助用户了解如何配置扩展
5. **全面测试** - 在不同场景下测试扩展
6. **考虑可访问性** - 确保扩展可通过键盘访问
7. **处理边缘情况** - 考虑空内容、选择等情况

## 测试扩展

创建测试文件以验证扩展是否正常工作：

```javascript
import { Editor } from "@tiptap/core";
import Document from "@tiptap/extension-document";
import Text from "@tiptap/extension-text";
import Paragraph from "@tiptap/extension-paragraph";
import { MyExtension } from "./my-extension";

const editor = new Editor({
  extensions: [Document, Text, Paragraph, MyExtension],
  content: "<p>你好世界！</p>",
});

// 测试命令
editor.commands.setMyNode();
console.log(editor.getHTML());

// 测试是否激活
console.log(editor.isActive("myExtension"));
```

## 下一步

- 浏览 `packages/core/src/extensions/` 中的现有扩展以获取灵感
- 阅读 [Tiptap 文档](https://tiptap.dev/guide/custom-extensions)了解更多高级功能
- 查看[自定义扩展](./customizing-extensions.md)指南以修改现有扩展
- 了解如何[与 REDAXO CMS 集成](./redaxo-integration.md)
