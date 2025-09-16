# 🤖 AI-Enhanced Markdown Editor Demo

A comprehensive Vue 3 demo showcasing an advanced textarea overlay technique with real-time markdown highlighting and AI-powered suggestions positioned precisely at the caret.

## ✨ Features

- **Real-time Markdown Highlighting**: Live parsing and visual decoration of markdown syntax
- **Click-through Overlay**: Visual overlay that doesn't interfere with text editing
- **Precise AI Popup Positioning**: AI suggestions anchored exactly to the caret position
- **Synchronized Scrolling**: Overlay stays perfectly aligned with textarea content
- **Full Markdown Support**: Headers, bold, italic, links, code, lists, and custom highlights
- **Custom Highlight Syntax**: Use `==text==` for highlighted content
- **Keyboard Shortcuts**: Ctrl+Space for AI suggestions, Escape to dismiss

## 🚀 Demo Features

### Markdown Syntax Support

- **Bold** (`**text**`) and _italic_ (`*text*`)
- Custom highlights (`==text==`)
- `Inline code` and code blocks
- [Links](https://example.com) that open in new tabs
- Headers, lists, and blockquotes

### AI Integration

- Move your caret to see AI suggestions follow
- Press **Ctrl+Space** to manually trigger suggestions
- Click "Accept" to insert suggestions at caret position
- Press **Escape** to dismiss popups

### Technical Implementation

- Uses `pointer-events: none` for click-through overlay
- AI popup uses `pointer-events: auto` for full interactivity
- `textarea-caret` library provides pixel-perfect positioning
- DOMPurify ensures safe HTML rendering
- Synchronized scroll states between textarea and overlay

## 🛠️ Technical Stack

- **Vue 3** with Composition API and TypeScript
- **Marked.js** for markdown parsing
- **DOMPurify** for safe HTML sanitization
- **textarea-caret** for precise caret positioning
- **Highlight.js** for syntax highlighting
- **Vite** for development and building

## 📦 Installation & Setup

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build

# Type checking
npm run type-check
```

## 🚀 Deployment

This project includes automated CI/CD with GitHub Actions:

- **Production**: Deployed to GitHub Pages on push to `main`/`master`
- **Testing**: Automated builds on Node.js 20 & 22

### Quick Setup

1. Enable GitHub Pages in repository settings
2. Push to `main` branch to trigger deployment

See [`.github/DEPLOYMENT.md`](.github/DEPLOYMENT.md) for detailed setup instructions.

## 🎮 Usage

1. **Basic Editing**: Type in the textarea and see real-time markdown highlighting
2. **AI Suggestions**:
   - Move your caret around to see AI popup follow
   - Press `Ctrl+Space` to trigger AI suggestions manually
   - Use the popup buttons to accept/reject suggestions
3. **Markdown Features**:
   - Type `==highlight==` for custom highlights
   - Use standard markdown syntax for formatting
   - All links automatically open in new tabs

## 🏗️ Component Architecture

### `MarkdownTextareaOverlay.vue`

The main component that implements:

- Textarea with transparent background
- Overlay with rendered markdown (click-through)
- AI popup positioned at caret
- Synchronized scrolling and positioning

### Key Implementation Details

```vue
<!-- Click-through overlay -->
<pre class="tq-overlay" style="pointer-events: none">
  <!-- Rendered markdown content -->
</pre>

<!-- Interactive textarea -->
<textarea class="tq-textarea" style="background: transparent">
  <!-- User input -->
</textarea>

<!-- Interactive AI popup -->
<div class="ai-popup" style="pointer-events: auto">
  <!-- AI suggestions -->
</div>
```

## 🎨 Styling Approach

The component uses a layered approach:

1. **Overlay Layer** (z-index: 1): Visual markdown rendering, click-through
2. **Textarea Layer** (z-index: 2): User interaction, transparent background
3. **Popup Layer** (z-index: 10): AI suggestions, fully interactive

## 🔧 Customization

### Props

- `initialText`: Starting content for the editor
- `showDebugInfo`: Display debug information
- `autoShowAI`: Automatically show AI suggestions

### Exposed Methods

- `focusTextarea()`: Programmatically focus the textarea
- `insertText(text)`: Insert text at current caret position

## 🎯 Use Cases

This demo showcases techniques useful for:

- **Code Editors** with syntax highlighting
- **Markdown Editors** with live preview
- **AI-Powered Writing Tools** with contextual suggestions
- **Rich Text Editors** with overlay decorations
- **Documentation Tools** with interactive elements

## 🔍 Browser Compatibility

- Modern browsers supporting CSS `pointer-events`
- Vue 3 compatible environments
- ES2020+ JavaScript features

## 📝 License

This demo is provided as-is for educational and demonstration purposes.

---

**Built with ❤️ using Vue 3, TypeScript, and modern web technologies.**
