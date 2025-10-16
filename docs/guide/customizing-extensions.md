# Customizing Extensions

This guide shows you how to customize existing isle-editor extensions to fit your specific needs. You can modify options, add new functionality, or override default behavior.

## Extending an Existing Extension

The `.extend()` method allows you to modify an existing extension without creating a completely new one.

### Basic Extension

Here's how to customize the Bold extension:

```javascript
import Bold from "@isle-editor/core/extensions/bold";

const CustomBold = Bold.extend({
  addOptions() {
    return {
      ...this.parent?.(),
      // Override the keyboard shortcut
      shortcutkeys: "Mod-Shift-B",
      // Add a custom CSS class
      HTMLAttributes: {
        class: "my-bold-text",
      },
    };
  },
});
```

### Changing Default Options

You can change default options when configuring the editor:

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
    // ... other extensions
  ],
});
```

## Customizing Node Extensions

### Modifying Attributes

Add custom attributes to existing nodes:

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

### Customizing Rendering

Override how nodes are rendered to HTML:

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

### Adding New Commands

Extend an existing node with additional commands:

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

## Customizing Mark Extensions

### Changing Mark Behavior

Customize how marks are applied:

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

## Customizing Keyboard Shortcuts

### Override Default Shortcuts

Change keyboard shortcuts for any extension:

```javascript
import CodeBlock from "@isle-editor/core/extensions/code-block";

const CustomCodeBlock = CodeBlock.extend({
  addKeyboardShortcuts() {
    return {
      ...this.parent?.(),
      // Change the shortcut for code block
      "Mod-Alt-C": () => this.editor.commands.toggleCodeBlock(),
      // Remove a shortcut by returning false
      "Mod-Shift-C": () => false,
    };
  },
});
```

### Add New Shortcuts

Add additional keyboard shortcuts:

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

## Customizing Input Rules

### Modify or Add Input Rules

Input rules trigger commands based on what users type:

```javascript
import { wrappingInputRule } from "@tiptap/core";
import Blockquote from "@isle-editor/core/extensions/blockquote";

const CustomBlockquote = Blockquote.extend({
  addInputRules() {
    return [
      // Add a new input rule
      wrappingInputRule({
        find: /^:::\s$/,
        type: this.type,
      }),
      // Keep existing rules
      ...this.parent?.(),
    ];
  },
});
```

## Customizing Extension Storage

Extensions can store state:

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

// Access storage
console.log(editor.storage.customExtension.count);
```

## Customizing Event Handlers

### Listen to Editor Events

```javascript
import { Extension } from "@tiptap/core";

const EventLogger = Extension.create({
  name: "eventLogger",

  onCreate() {
    console.log("Editor created");
  },

  onUpdate() {
    console.log("Editor updated");
  },

  onSelectionUpdate() {
    console.log("Selection changed");
  },

  onTransaction({ transaction }) {
    console.log("Transaction:", transaction);
  },

  onFocus() {
    console.log("Editor focused");
  },

  onBlur() {
    console.log("Editor blurred");
  },

  onDestroy() {
    console.log("Editor destroyed");
  },
});
```

## Customizing for Command Palette

isle-editor includes a command palette that can be triggered with `/`. Customize how your extensions appear:

```javascript
import { Node } from "@tiptap/core";

const CustomNode = Node.create({
  name: "customNode",

  addOptions() {
    return {
      // Show in slash command menu
      slash: true,
      // Name shown in command palette
      name: "Custom Block",
      // Description shown in command palette
      desc: "Insert a custom block element",
      // Command executed when selected from palette
      command: ({ editor, range }) => {
        range
          ? editor.chain().focus().deleteRange(range).setCustomNode().run()
          : editor.chain().focus().setCustomNode().run();
      },
      // Check if command is active
      isActive: ({ editor }) => editor.isActive("customNode"),
      // Check if command is disabled
      isDisabled: ({ editor }) => !editor.can().setCustomNode(),
      // Keyboard shortcut hint
      shortcutkeys: "Mod-Shift-C",
    };
  },
});
```

## Practical Examples

### Example 1: Custom Heading with Auto-Generated IDs

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

### Example 2: Read-Only Code Blocks

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
          return true; // Prevent deletion
        }
        return false;
      },
    };
  },
});
```

### Example 3: Enhanced Link with Validation

```javascript
import Link from "@tiptap/extension-link";

const ValidatedLink = Link.extend({
  addCommands() {
    return {
      ...this.parent?.(),
      setLink:
        (attributes) =>
        ({ chain }) => {
          // Validate URL
          if (!attributes.href) {
            return false;
          }

          try {
            new URL(attributes.href);
          } catch {
            console.error("Invalid URL");
            return false;
          }

          // Auto-add protocol if missing
          if (!/^https?:\/\//i.test(attributes.href)) {
            attributes.href = "https://" + attributes.href;
          }

          return chain().setMark(this.name, attributes).run();
        },
    };
  },
});
```

## Configuration Patterns

### Global Configuration

Configure extensions globally when initializing the editor:

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

### Conditional Configuration

Configure extensions based on environment or user preferences:

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

## Tips and Best Practices

1. **Always call parent methods** - Use `...this.parent?.()` to preserve original functionality
2. **Test thoroughly** - Test your customizations with different content and scenarios
3. **Document changes** - Keep notes on what you've customized and why
4. **Check compatibility** - Ensure your customizations work with other extensions
5. **Consider performance** - Avoid expensive operations in frequently-called methods like `onUpdate`
6. **Use TypeScript** - Type definitions help catch errors early

## Debugging Tips

### Log Extension Activity

```javascript
const DebugExtension = Extension.extend({
  onCreate() {
    console.log("Options:", this.options);
  },

  onUpdate() {
    console.log("Content:", this.editor.getJSON());
    console.log("HTML:", this.editor.getHTML());
  },
});
```

### Inspect Editor State

```javascript
// Get current selection
console.log(editor.state.selection);

// Get current node
console.log(editor.state.selection.$anchor.parent);

// Check if mark is active
console.log(editor.isActive("bold"));

// Get all active marks
console.log(editor.state.selection.$from.marks());
```

## Next Steps

- Learn how to [create custom extensions from scratch](./extension-development.md)
- Explore existing extensions in `packages/core/src/extensions/`
- Check out [REDAXO integration](./redaxo-integration.md) examples
- Read the [Tiptap documentation](https://tiptap.dev/) for advanced techniques
