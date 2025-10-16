# Extension Development

This guide will help you create custom extensions for isle-editor. Extensions allow you to add new functionality, node types, marks, and features to the editor.

## Understanding Extensions

isle-editor is built on top of [Tiptap](https://tiptap.dev/), which in turn is built on [ProseMirror](https://prosemirror.net/). Extensions are modular pieces that add specific functionality to the editor.

There are three main types of extensions:

1. **Nodes** - Block-level content (paragraphs, headings, images, etc.)
2. **Marks** - Inline formatting (bold, italic, links, etc.)
3. **Extensions** - General functionality (keyboard shortcuts, commands, etc.)

## Creating a Simple Mark Extension

Let's create a custom highlight mark that allows users to highlight text with a background color.

### Step 1: Create the Extension File

Create a new file in `packages/core/src/extensions/highlight.js`:

```javascript
import { Mark, mergeAttributes } from "@tiptap/core";

const source = {
  name: "highlight",
  desc: "Highlight text",
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

### Step 2: Export the Extension

Add your extension to `packages/core/src/index.js`:

```javascript
// ... other imports
export { default as Highlight } from "./extensions/highlight.js";
```

### Step 3: Use the Extension

Now you can use your custom extension in your editor:

```javascript
import { Editor } from "@isle-editor/core";
import { Highlight } from "@isle-editor/core";

const editor = new Editor({
  extensions: [
    // ... other extensions
    Highlight,
  ],
});
```

## Creating a Node Extension

Let's create a custom callout/alert node:

```javascript
import { Node, mergeAttributes } from "@tiptap/core";

const source = {
  slash: true,
  name: "callout",
  desc: "Add a callout block",
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

## Creating a Feature Extension

Feature extensions add general functionality without rendering content. Here's an example of a word count extension:

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

## Extension Options

Extensions can define options that allow users to customize behavior:

```javascript
addOptions() {
  return {
    // Default options
    HTMLAttributes: {},
    placeholder: "Type something...",
    maxLength: null,
    
    // Options from source object for command palette
    name: "myExtension",
    desc: "Description for command palette",
    command: ({ editor }) => editor.commands.doSomething(),
    isActive: ({ editor }) => editor.isActive("myExtension"),
    isDisabled: ({ editor }) => !editor.can().doSomething(),
    shortcutkeys: "Mod-Shift-X",
    slash: true, // Show in slash command menu
  };
}
```

## Extension Commands

Commands are functions that modify the editor state:

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

## Keyboard Shortcuts

Add keyboard shortcuts to your extension:

```javascript
addKeyboardShortcuts() {
  return {
    // Mod is Cmd on Mac and Ctrl on Windows/Linux
    "Mod-Shift-x": () => this.editor.commands.setMyNode(),
    "Mod-Alt-x": () => this.editor.commands.toggleMyNode(),
    Enter: () => {
      // Custom Enter key behavior
      return this.editor.commands.createParagraphNear();
    },
  };
}
```

## Input Rules

Input rules allow you to trigger commands based on what the user types:

```javascript
import { wrappingInputRule, textblockTypeInputRule } from "@tiptap/core";

addInputRules() {
  return [
    // Wrap selection with blockquote when typing ">"
    wrappingInputRule({
      find: /^\s*>\s$/,
      type: this.type,
    }),
    
    // Convert to heading when typing "# "
    textblockTypeInputRule({
      find: /^(#{1,6})\s$/,
      type: this.type,
      getAttributes: (match) => ({ level: match[1].length }),
    }),
  ];
}
```

## Best Practices

1. **Keep extensions focused** - Each extension should do one thing well
2. **Follow naming conventions** - Use camelCase for extension names
3. **Provide good defaults** - Make extensions work well out of the box
4. **Document your options** - Help users understand how to configure your extension
5. **Test thoroughly** - Test your extension in different scenarios
6. **Consider accessibility** - Ensure your extension is keyboard accessible
7. **Handle edge cases** - Consider what happens with empty content, selections, etc.

## Testing Your Extension

Create a test file to verify your extension works:

```javascript
import { Editor } from "@tiptap/core";
import Document from "@tiptap/extension-document";
import Text from "@tiptap/extension-text";
import Paragraph from "@tiptap/extension-paragraph";
import { MyExtension } from "./my-extension";

const editor = new Editor({
  extensions: [Document, Text, Paragraph, MyExtension],
  content: "<p>Hello World!</p>",
});

// Test commands
editor.commands.setMyNode();
console.log(editor.getHTML());

// Test if active
console.log(editor.isActive("myExtension"));
```

## Next Steps

- Explore existing extensions in `packages/core/src/extensions/` for inspiration
- Read the [Tiptap documentation](https://tiptap.dev/guide/custom-extensions) for more advanced features
- Check out the [Customizing Extensions](./customizing-extensions.md) guide to modify existing extensions
- See how to [integrate with REDAXO CMS](./redaxo-integration.md)
