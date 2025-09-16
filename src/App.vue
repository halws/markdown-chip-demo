<script setup lang="ts">
import { ref } from 'vue'
import MarkdownTextareaOverlay from './components/MarkdownTextareaOverlay.vue'

const showDebugInfo = ref(false)
const autoShowMentions = ref(false)
const readonlyMode = ref(false)

const demoText = `# AI-Enhanced Markdown Editor Demo

This is a **comprehensive demo** of a Vue 3 component that combines:

## 🎯 Key Features
- Real-time markdown parsing and highlighting
- Click-through overlay that doesn't interfere with typing
- AI suggestions triggered by typing @
- Synchronized scrolling and positioning
- Full markdown support

## 🚀 Try These Features:

### Markdown Syntax:
- **Bold text** and *italic text*
- \`inline code\` and code blocks
- [Links](https://example.com) that open in new tabs
- Lists and other markdown elements

### Object Mentions:
- Type **@** anywhere to open the mention selector
- Press **Ctrl+Space** to manually open mention selector
- Search and select objects to reference
- Press **Escape** to dismiss the selector

### Technical Details:
The overlay uses \`pointer-events: none\` for click-through behavior, while the AI popup uses \`pointer-events: auto\` for full interactivity. The \`textarea-caret\` library provides pixel-perfect caret positioning.

**Try editing this text and watch the magic happen!** ✨`
</script>

<template>
  <div class="app">
    <header class="app-header">
      <h1>🤖 AI-Enhanced Markdown Editor</h1>
      <p class="subtitle">
        A Vue 3 demo combining markdown highlighting, overlay techniques, and AI suggestions
      </p>
      
      <div class="controls">
        <label class="control-item">
          <input type="checkbox" v-model="showDebugInfo" />
          Show Debug Info
        </label>
        <label class="control-item">
          <input type="checkbox" v-model="autoShowMentions" />
          Auto-show Mention Selector
        </label>
        <label class="control-item">
          <input type="checkbox" v-model="readonlyMode" />
          Readonly Mode (Markdown Rendering)
        </label>
      </div>
    </header>

    <main class="app-main">
      <MarkdownTextareaOverlay 
        :initial-text="demoText"
        :show-debug-info="showDebugInfo"
        :auto-show-mentions="autoShowMentions"
        :readonly="readonlyMode"
      />
      
      <div class="instructions">
        <h3>🎮 How to Use:</h3>
        <ul>
          <li><strong>Edit Mode:</strong> Type <kbd>@</kbd> to open mention selector</li>
          <li><kbd>Ctrl</kbd> + <kbd>Space</kbd> - Manual mention selector</li>
          <li><kbd>↑</kbd>/<kbd>↓</kbd> - Navigate mentions</li>
          <li><kbd>Enter</kbd> - Select mention</li>
          <li><kbd>Escape</kbd> - Dismiss selector</li>
          <li><strong>Readonly Mode:</strong> Toggle to see full markdown rendering</li>
          <li>Use standard markdown syntax in both modes</li>
        </ul>
      </div>
    </main>

    <footer class="app-footer">
      <p>
        Built with Vue 3, TypeScript, and the power of overlay techniques. 
        <a href="https://github.com" target="_blank" rel="noopener">View Source</a>
      </p>
    </footer>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.app-header {
  text-align: center;
  padding: 2rem 1rem;
  background: rgba(255, 255, 255, 0.9);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(0, 0, 0, 0.1);
  
  h1 {
    margin: 0 0 0.5rem 0;
    font-size: 2.5rem;
    font-weight: 700;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }
  
  .subtitle {
    margin: 0 0 1.5rem 0;
    font-size: 1.1rem;
    color: #6a737d;
    max-width: 600px;
    margin-left: auto;
    margin-right: auto;
  }
}

.controls {
  display: flex;
  justify-content: center;
  gap: 2rem;
  flex-wrap: wrap;
  
  .control-item {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.9rem;
    color: #586069;
    cursor: pointer;
    
    input[type="checkbox"] {
      width: 16px;
      height: 16px;
      accent-color: #667eea;
    }
    
    &:hover {
      color: #24292e;
    }
  }
}

.app-main {
  padding: 2rem 1rem;
  max-width: 1200px;
  margin: 0 auto;
}

.instructions {
  margin-top: 2rem;
  padding: 1.5rem;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.1);
  
  h3 {
    margin: 0 0 1rem 0;
    color: #24292e;
    font-size: 1.2rem;
  }
  
  ul {
    margin: 0;
    padding-left: 1.5rem;
    
    li {
      margin: 0.5rem 0;
      color: #586069;
      
      kbd {
        background: #f6f8fa;
        border: 1px solid #d1d5da;
        border-radius: 3px;
        padding: 0.1em 0.3em;
        font-size: 0.85em;
        font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
      }
      
      code {
        background: #f6f8fa;
        padding: 0.1em 0.3em;
        border-radius: 3px;
        font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
        font-size: 0.85em;
      }
    }
  }
}

.app-footer {
  text-align: center;
  padding: 2rem 1rem;
  background: rgba(255, 255, 255, 0.5);
  border-top: 1px solid rgba(0, 0, 0, 0.1);
  
  p {
    margin: 0;
    color: #6a737d;
    font-size: 0.9rem;
    
    a {
      color: #0366d6;
      text-decoration: none;
      
      &:hover {
        text-decoration: underline;
      }
    }
  }
}

/* Responsive design */
@media (max-width: 768px) {
  .app-header h1 {
    font-size: 2rem;
  }
  
  .controls {
    gap: 1rem;
  }
  
  .app-main {
    padding: 1rem;
  }
}
</style>
