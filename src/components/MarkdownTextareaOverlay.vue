<template>
  <div class="tq-textarea-wrapper" ref="wrapper">
    <!-- Overlay for highlighting - click-through except for interactive elements -->
    <pre
      ref="overlay"
      class="tq-markdown tq-overlay"
      aria-hidden="true"
      :style="{ height: textareaHeight + 'px' }"
      v-html="decoratedContent"
    />

    <!-- Main textarea for user input (only in edit mode) -->
    <textarea
      v-if="!readonly"
      ref="textarea"
      v-model="text"
      class="tq-textarea"
      :style="{ height: textareaHeight + 'px' }"
      placeholder="Try typing **bold** text and @ to reference objects..."
      @input="onInput"
      @scroll="syncScroll"
      @keyup="updateMentionPopupPosition"
      @click="updateMentionPopupPosition"
      @focus="updateMentionPopupPosition"
    />

    <!-- Mention selector popup - positioned at caret and fully interactive (only in edit mode) -->
    <div
      v-if="!readonly && showMentionPopup && popupPosition"
      class="mention-popup"
      :style="{ 
        top: popupPosition.top + 'px', 
        left: popupPosition.left + 'px' 
      }"
      @mousedown.stop
    >
      <div class="mention-popup-header">
        <span class="mention-icon">@</span>
        <span>Reference Object</span>
        <button class="close-btn" @click="hideMentionPopup">×</button>
      </div>
      <div class="mention-popup-content">
        <div class="mention-search">
          <input 
            ref="mentionSearch"
            v-model="mentionQuery"
            placeholder="Search objects..."
            @keydown="handleMentionKeydown"
            @input="filterMentionResults"
          />
        </div>
        <div class="mention-results" v-if="filteredMentionResults.length > 0">
          <div class="mention-category" v-for="category in groupedMentionResults" :key="category.type">
            <div class="category-header">{{ category.label }}</div>
            <div 
              v-for="(item, index) in category.items" 
              :key="item.id"
              :class="['mention-item', { active: selectedMentionIndex === item.globalIndex }]"
              @click="selectMention(item)"
              @mouseenter="selectedMentionIndex = item.globalIndex"
            >
              <span class="mention-item-icon">{{ item.icon }}</span>
              <div class="mention-item-content">
                <div class="mention-item-title">{{ item.title }}</div>
                <div class="mention-item-subtitle">{{ item.subtitle }}</div>
              </div>
            </div>
          </div>
        </div>
        <div v-else class="no-results">
          No objects found matching "{{ mentionQuery }}"
        </div>
      </div>
    </div>

    <!-- Debug info (optional) -->
    <div v-if="showDebugInfo" class="debug-info">
      <p><strong>Caret Position:</strong> {{ caretPosition }}</p>
      <p><strong>Text Length:</strong> {{ text.length }}</p>
      <p><strong>Popup Position:</strong> {{ popupPosition }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, nextTick, watch } from 'vue'
import { marked } from 'marked'
import DOMPurify from 'dompurify'
import getCaretCoordinates from 'textarea-caret'
import 'highlight.js/styles/monokai.css'

// Props
interface Props {
  initialText?: string
  showDebugInfo?: boolean
  autoShowMentions?: boolean
  readonly?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  initialText: `# Welcome to the AI-Enhanced Markdown Editor!

Try typing some **markdown** text and see the live preview overlay.

## Features:
- Real-time markdown highlighting
- Object mentions triggered by typing @
- Click-through overlay that doesn't interfere with typing
- Synchronized scrolling and positioning

### Try these markdown features:
- **Bold text**
- *Italic text*
- \`inline code\`
- [Links](https://example.com)

### Try object mentions:
Type @ to reference:
- Custom fields
- Observables  
- Notes
- Attachments
- Events
- Cases

Type @ anywhere to open the mention selector!`,
  showDebugInfo: false,
  autoShowMentions: false,
  readonly: false
})

// Reactive state
const text = ref(props.initialText)
const textarea = ref<HTMLTextAreaElement | null>(null)
const overlay = ref<HTMLPreElement | null>(null)
const wrapper = ref<HTMLDivElement | null>(null)
const mentionSearch = ref<HTMLInputElement | null>(null)
const textareaHeight = ref(300)
const showMentionPopup = ref(false)
const popupPosition = ref<{ top: number; left: number } | null>(null)
const caretPosition = ref(0)
const mentionQuery = ref('')
const selectedMentionIndex = ref(0)
const mentionStartPosition = ref(0)

// Mock data for mentions (in real app, this would come from API)
const mockMentionData = [
  // Custom Fields
  { id: 'cf1', type: 'custom_field', icon: '🏷️', title: 'Priority Level', subtitle: 'High', category: 'Custom Fields' },
  { id: 'cf2', type: 'custom_field', icon: '🏷️', title: 'Analyst', subtitle: 'John Doe', category: 'Custom Fields' },
  { id: 'cf3', type: 'custom_field', icon: '🏷️', title: 'Source System', subtitle: 'SIEM-01', category: 'Custom Fields' },
  
  // Observables
  { id: 'obs1', type: 'observable', icon: '🔍', title: '192.168.1.100', subtitle: 'IP Address • Malicious', category: 'Observables' },
  { id: 'obs2', type: 'observable', icon: '🔍', title: 'malware.exe', subtitle: 'File Hash • Suspicious', category: 'Observables' },
  { id: 'obs3', type: 'observable', icon: '🔍', title: 'evil.com', subtitle: 'Domain • Blocked', category: 'Observables' },
  
  // Notes
  { id: 'note1', type: 'note', icon: '📝', title: 'Initial Analysis', subtitle: 'Created 2 hours ago', category: 'Notes' },
  { id: 'note2', type: 'note', icon: '📝', title: 'Follow-up Actions', subtitle: 'Updated 1 hour ago', category: 'Notes' },
  
  // Attachments
  { id: 'att1', type: 'attachment', icon: '📎', title: 'screenshot.png', subtitle: '2.3 MB • Image', category: 'Attachments' },
  { id: 'att2', type: 'attachment', icon: '📎', title: 'evidence.log', subtitle: '156 KB • Log file', category: 'Attachments' },
  
  // Events
  { id: 'evt1', type: 'event', icon: '⚡', title: 'Login Attempt', subtitle: 'Event ID: 4625 • Failed', category: 'Events' },
  { id: 'evt2', type: 'event', icon: '⚡', title: 'File Creation', subtitle: 'Event ID: 11 • Sysmon', category: 'Events' },
  
  // Cases
  { id: 'case1', type: 'case', icon: '📋', title: 'CASE-2024-001', subtitle: 'Phishing Investigation • High', category: 'Cases' },
  { id: 'case2', type: 'case', icon: '📋', title: 'CASE-2024-002', subtitle: 'Malware Analysis • Medium', category: 'Cases' },
]

const filteredMentionResults = ref(mockMentionData)

// Computed properties
const groupedMentionResults = computed(() => {
  const groups = new Map()
  let globalIndex = 0
  
  filteredMentionResults.value.forEach(item => {
    if (!groups.has(item.category)) {
      groups.set(item.category, {
        type: item.type,
        label: item.category,
        items: []
      })
    }
    groups.get(item.category).items.push({
      ...item,
      globalIndex: globalIndex++
    })
  })
  
  return Array.from(groups.values())
})

// Markdown renderer setup (based on your existing Markdown.vue)
const renderer = new marked.Renderer()

// Custom link renderer to open in new tab
renderer.link = (href, title, text) => {
  const titleAttr = title ? ` title="${title}"` : ''
  return `<a href="${href}"${titleAttr} target="_blank" rel="nofollow noreferrer">${text}</a>`
}

// Configure marked with mention extension
marked.use({ 
  renderer,
  extensions: [
    {
      name: 'mention',
      level: 'inline',
      start(src) { return src.match(/mention:\/\//)?.index },
      tokenizer(src, tokens) {
        const rule = /^mention:\/\/([^\/\s]+)\/([^\s]+)/
        const match = rule.exec(src)
        if (match) {
          return {
            type: 'mention',
            raw: match[0],
            objectType: match[1],
            objectId: match[2]
          }
        }
      },
      renderer(token) {
        return `<mention-component data-type="${token.objectType}" data-id="${token.objectId}"></mention-component>`
      }
    }
  ]
})

// DOMPurify hook for links and mentions
DOMPurify.addHook('afterSanitizeAttributes', (node) => {
  if ('target' in node) {
    node.setAttribute('target', '_blank')
    node.setAttribute('rel', 'nofollow noreferrer')
  }
  if (!node.hasAttribute('target') && (node.hasAttribute('xlink:href') || node.hasAttribute('href'))) {
    node.setAttribute('xlink:show', 'new')
  }
})

// Allow custom mention components in DOMPurify
DOMPurify.addHook('uponSanitizeElement', (node, data) => {
  if (data.tagName === 'mention-component') {
    data.allowedTags['mention-component'] = true
  }
})

// Computed properties
const decoratedContent = computed(() => {
  const textContent = text.value || ''
  
  if (props.readonly) {
    // Readonly mode: render markdown with mention components
    let rawContent = marked.parse(textContent)
    
    // Allow mention-component tags in DOMPurify
    const sanitized = DOMPurify.sanitize(rawContent as string, { 
      FORBID_TAGS: ['style'],
      ALLOW_TAGS: ['mention-component', 'mark', 'strong', 'em', 'code', 'pre', 'a', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'p', 'ul', 'ol', 'li', 'blockquote'],
      ALLOW_ATTR: ['data-type', 'data-id', 'data-text']
    })
    
    // Replace mention-component tags with actual Vue components
    const processedContent = sanitized.replace(
      /<mention-component data-type="([^"]+)" data-id="([^"]+)"><\/mention-component>/g,
      (match, type, id) => {
        const mentionData = mockMentionData.find(item => item.id === id && item.type === type)
        if (mentionData) {
          return renderMentionComponent(mentionData)
        }
        return `<span class="mention-error">Unknown ${type}: ${id}</span>`
      }
    )
    
    return processedContent + '\n'
  } else {
    // Edit mode: render plain text with mention URIs as visual components
    // First, escape HTML entities for safety
    let processedContent = textContent
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;')
    
    // Then replace mention URIs with visual components
    processedContent = processedContent.replace(
      /mention:\/\/([^\/\s]+)\/([^\s]+)/g,
      (match, type, id) => {
        const mentionData = mockMentionData.find(item => item.id === id && item.type === type)
        if (mentionData) {
          return renderMentionComponent(mentionData)
        }
        return `<span class="mention-error">Unknown ${type}: ${id}</span>`
      }
    )
    
    return processedContent + '\n'
  }
})

// Function to render mention components as HTML
function renderMentionComponent(mentionData: any) {
  const { type, icon, title, subtitle, id } = mentionData
  
  switch (type) {
    case 'custom_field':
      return `<span class="mention mention-custom-field" data-id="${id}"><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-value">${subtitle}</span></span></span>`
      
    case 'observable':
      return `<span class="mention mention-observable" data-id="${id}"><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'note':
      return `<span class="mention mention-note" data-id="${id}"><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'attachment':
      return `<span class="mention mention-attachment" data-id="${id}"><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'event':
      return `<span class="mention mention-event" data-id="${id}"><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'case':
      return `<span class="mention mention-case" data-id="${id}"><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    default:
      return `<span class="mention mention-unknown" data-id="${id}"><span class="mention-icon">❓</span><span class="mention-content"><span class="mention-title">Unknown: ${title}</span></span></span>`
  }
}

// Methods
function syncScroll() {
  if (overlay.value && textarea.value) {
    overlay.value.scrollTop = textarea.value.scrollTop
    overlay.value.scrollLeft = textarea.value.scrollLeft
  }
}

function onInput(event: Event) {
  syncScroll()
  
  // Check if @ was typed to trigger mention selector
  const target = event.target as HTMLTextAreaElement
  const inputType = (event as InputEvent).inputType
  const data = (event as InputEvent).data
  
  if (inputType === 'insertText' && data === '@') {
    mentionStartPosition.value = target.selectionStart - 1 // Position of @
    showMentionPopupManually()
  } else {
    updateMentionPopupPosition()
  }
}

function updateMentionPopupPosition() {
  nextTick(() => {
    const ta = textarea.value
    if (!ta || !wrapper.value) return

    caretPosition.value = ta.selectionStart
    
    try {
      const { top, left } = getCaretCoordinates(ta, ta.selectionStart)
      const wrapperRect = wrapper.value.getBoundingClientRect()
      const textareaRect = ta.getBoundingClientRect()
      
      popupPosition.value = {
        top: top + textareaRect.top - wrapperRect.top + 25 - ta.scrollTop,
        left: left + textareaRect.left - wrapperRect.left + 5 - ta.scrollLeft
      }
    } catch (error) {
      console.warn('Error calculating caret position:', error)
    }
  })
}

function selectMention(item: any) {
  const ta = textarea.value
  if (!ta) return

  // Create mention link in URI format only
  const mentionLink = `mention://${item.type}/${item.id}`
  
  // Replace @ with the mention link
  const beforeMention = text.value.substring(0, mentionStartPosition.value)
  const afterCaret = text.value.substring(ta.selectionStart)
  
  text.value = beforeMention + mentionLink + afterCaret
  
  // Move caret to end of inserted mention
  nextTick(() => {
    const newPosition = beforeMention.length + mentionLink.length
    ta.setSelectionRange(newPosition, newPosition)
    ta.focus()
  })
  
  hideMentionPopup()
}

function filterMentionResults() {
  if (!mentionQuery.value.trim()) {
    filteredMentionResults.value = mockMentionData
  } else {
    const query = mentionQuery.value.toLowerCase()
    filteredMentionResults.value = mockMentionData.filter(item =>
      item.title.toLowerCase().includes(query) ||
      item.subtitle.toLowerCase().includes(query) ||
      item.category.toLowerCase().includes(query)
    )
  }
  selectedMentionIndex.value = 0
}

function handleMentionKeydown(event: KeyboardEvent) {
  const totalItems = filteredMentionResults.value.length
  
  switch (event.key) {
    case 'ArrowDown':
      event.preventDefault()
      selectedMentionIndex.value = (selectedMentionIndex.value + 1) % totalItems
      break
    case 'ArrowUp':
      event.preventDefault()
      selectedMentionIndex.value = selectedMentionIndex.value === 0 ? totalItems - 1 : selectedMentionIndex.value - 1
      break
    case 'Enter':
      event.preventDefault()
      if (filteredMentionResults.value[selectedMentionIndex.value]) {
        selectMention(filteredMentionResults.value[selectedMentionIndex.value])
      }
      break
    case 'Escape':
      event.preventDefault()
      hideMentionPopup()
      break
  }
}

function hideMentionPopup() {
  showMentionPopup.value = false
  mentionQuery.value = ''
  selectedMentionIndex.value = 0
}

function showMentionPopupManually() {
  updateMentionPopupPosition()
  showMentionPopup.value = true
  mentionQuery.value = ''
  selectedMentionIndex.value = 0
  
  // Focus search input
  nextTick(() => {
    mentionSearch.value?.focus()
  })
}

// Keyboard shortcuts
function handleKeydown(event: KeyboardEvent) {
  // Ctrl+Space to trigger mention selector
  if (event.ctrlKey && event.code === 'Space') {
    event.preventDefault()
    mentionStartPosition.value = textarea.value?.selectionStart || 0
    showMentionPopupManually()
  }
  
  // Escape to hide mention popup
  if (event.code === 'Escape' && showMentionPopup.value) {
    event.preventDefault()
    hideMentionPopup()
  }
}

// Lifecycle
onMounted(() => {
  syncScroll()
  updateMentionPopupPosition()
  
  // Add keyboard event listeners
  document.addEventListener('keydown', handleKeydown)
})

// Watch for text changes to update overlay
watch(text, () => {
  nextTick(() => {
    syncScroll()
  })
})

// Expose methods for parent components if needed
defineExpose({
  focusTextarea: () => textarea.value?.focus(),
  insertText: (text: string) => {
    if (!textarea.value) return
    const start = textarea.value.selectionStart
    const end = textarea.value.selectionEnd
    const beforeCaret = text.value.substring(0, start)
    const afterCaret = text.value.substring(end)
    text.value = beforeCaret + text + afterCaret
    nextTick(() => {
      const newPosition = start + text.length
      textarea.value?.setSelectionRange(newPosition, newPosition)
    })
  }
})
</script>

<style scoped lang="scss">
/* Modern CSS Reset for consistent rendering */
*, *::before, *::after {
  box-sizing: border-box;
}

* {
  margin: 0;
}

.tq-textarea-wrapper {
  position: relative;
  width: 100%;
  max-width: 800px;
  margin: 0 auto;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  isolation: isolate;
}

.tq-overlay {
  pointer-events: none; /* Click-through overlay */
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  min-height: 300px;
  z-index: 10; /* Higher z-index to appear above textarea */
  
  /* Match textarea styling EXACTLY */
  font-family: inherit;
  font-size: 14px;
  line-height: 1.5;
  padding: 12px;
  margin: 0;
  border: 1px solid transparent; /* Match textarea border width */
  border-radius: 6px; /* Match textarea border radius */
  box-sizing: border-box;
  
  /* Text styling - must match textarea exactly */
  white-space: pre-wrap;
  word-wrap: break-word;
  overflow-wrap: break-word;
  color: #24292e; /* Match textarea text color */
  background: transparent;
  overflow: hidden; /* Prevent scrollbars on overlay */
  
  /* Reset all text styling to match textarea */
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  
  /* Markdown styling */
  h1, h2, h3, h4, h5, h6 {
    margin: 0.5em 0;
    font-weight: bold;
  }
  
  h1 { font-size: 1.8em; color: #2c5aa0; }
  h2 { font-size: 1.5em; color: #2c5aa0; }
  h3 { font-size: 1.3em; color: #2c5aa0; }
  
  p {
    margin: 0.5em 0;
  }
  
  strong {
    font-weight: bold;
    color: #d73a49;
  }
  
  em {
    font-style: italic;
    color: #6f42c1;
  }
  
  code {
    background: rgba(175, 184, 193, 0.2);
    padding: 0.1em 0.3em;
    border-radius: 3px;
    font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
    font-size: 0.9em;
  }
  
  pre {
    background: rgba(175, 184, 193, 0.1);
    padding: 8px;
    border-radius: 4px;
    overflow: auto;
    margin: 0.5em 0;
  }
  
  a {
    color: #0366d6;
    text-decoration: underline;
    
    &:hover {
      text-decoration: none;
    }
  }
  
  ul, ol {
    padding-left: 2em;
    margin: 0.5em 0;
  }
  
  blockquote {
    border-left: 4px solid #dfe2e5;
    padding-left: 1em;
    margin: 0.5em 0;
    color: #6a737d;
  }
}

.tq-textarea {
  position: relative;
  z-index: 1; /* Lower z-index so overlay appears above */
  width: 100%;
  min-height: 300px;
  
  /* Styling - base layer */
  font-family: inherit;
  font-size: 14px;
  line-height: 1.5;
  padding: 12px;
  border: 1px solid #d1d5da;
  border-radius: 6px;
  background: rgba(255, 255, 255, 0.9);
  color: #24292e;
  
  /* Behavior */
  resize: vertical;
  box-sizing: border-box;
  caret-color: #0366d6;
  
  /* Text rendering to match overlay */
  white-space: pre-wrap;
  word-wrap: break-word;
  overflow-wrap: break-word;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  
  &:focus {
    outline: none;
    border-color: #0366d6;
    box-shadow: 0 0 0 3px rgba(3, 102, 214, 0.1);
  }
  
  &::placeholder {
    color: #6a737d;
  }
}

/* Highlight styling */
:deep(mark) {
  background: linear-gradient(120deg, #a8e6cf 0%, #dcedc1 100%);
  border-radius: 3px;
  padding: 0 2px;
  box-decoration-break: clone;
}

/* Mention Components Styling */
:deep(.mention) {
  display: inline-flex;
  align-items: center;
  vertical-align: baseline;
  padding: 1px 4px;
  margin: 0;
  border-radius: 3px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  text-decoration: none;
  line-height: 1.5; /* Match textarea line height exactly */
  height: auto;
  max-height: 1.5em; /* Constrain to line height */
  
  /* Reset margins and padding to prevent line height issues */
  * {
    margin: 0;
    padding: 0;
  }
  
  .mention-icon {
    margin-right: 3px;
    font-size: 12px;
    line-height: 1;
    display: inline-block;
    vertical-align: middle;
  }
  
  .mention-content {
    display: inline-flex;
    align-items: center;
    line-height: 1;
    vertical-align: middle;
  }
  
  .mention-title {
    font-weight: 500;
    font-size: 12px;
    line-height: 1;
    white-space: nowrap;
  }
  
  .mention-subtitle,
  .mention-value {
    font-size: 10px;
    opacity: 0.8;
    margin-left: 3px;
    line-height: 1;
    white-space: nowrap;
  }
  
  &:hover {
    transform: translateY(-1px);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  }
}

/* Custom Field Mentions */
:deep(.mention-custom-field) {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
  color: white;
  border: 1px solid rgba(240, 147, 251, 0.3);
  
  &:hover {
    background: linear-gradient(135deg, #f5576c 0%, #f093fb 100%);
  }
}

/* Observable Mentions */
:deep(.mention-observable) {
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
  color: white;
  border: 1px solid rgba(79, 172, 254, 0.3);
  
  &:hover {
    background: linear-gradient(135deg, #00f2fe 0%, #4facfe 100%);
  }
}

/* Note Mentions */
:deep(.mention-note) {
  background: linear-gradient(135deg, #43e97b 0%, #38f9d7 100%);
  color: #1a202c;
  border: 1px solid rgba(67, 233, 123, 0.3);
  
  &:hover {
    background: linear-gradient(135deg, #38f9d7 0%, #43e97b 100%);
  }
}

/* Attachment Mentions */
:deep(.mention-attachment) {
  background: linear-gradient(135deg, #fa709a 0%, #fee140 100%);
  color: #1a202c;
  border: 1px solid rgba(250, 112, 154, 0.3);
  
  &:hover {
    background: linear-gradient(135deg, #fee140 0%, #fa709a 100%);
  }
}

/* Event Mentions */
:deep(.mention-event) {
  background: linear-gradient(135deg, #a8edea 0%, #fed6e3 100%);
  color: #1a202c;
  border: 1px solid rgba(168, 237, 234, 0.3);
  
  &:hover {
    background: linear-gradient(135deg, #fed6e3 0%, #a8edea 100%);
  }
}

/* Case Mentions */
:deep(.mention-case) {
  background: linear-gradient(135deg, #d299c2 0%, #fef9d7 100%);
  color: #1a202c;
  border: 1px solid rgba(210, 153, 194, 0.3);
  
  &:hover {
    background: linear-gradient(135deg, #fef9d7 0%, #d299c2 100%);
  }
}

/* Error/Unknown Mentions */
:deep(.mention-error),
:deep(.mention-unknown) {
  background: #f56565;
  color: white;
  border: 1px solid rgba(245, 101, 101, 0.3);
  
  &:hover {
    background: #e53e3e;
  }
}

/* Mention Popup */
.mention-popup {
  position: absolute;
  z-index: 10;
  pointer-events: auto; /* Make popup interactive */
  
  background: white;
  border: 1px solid #e1e4e8;
  border-radius: 8px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  
  min-width: 320px;
  max-width: 400px;
  max-height: 300px;
  
  color: #24292e;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  
  animation: popupSlideIn 0.2s ease-out;
}

@keyframes popupSlideIn {
  from {
    opacity: 0;
    transform: translateY(-10px) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

.mention-popup-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: 1px solid #e1e4e8;
  background: #f6f8fa;
  border-radius: 8px 8px 0 0;
  
  .mention-icon {
    font-size: 16px;
    margin-right: 8px;
    color: #0366d6;
    font-weight: bold;
  }
  
  .close-btn {
    background: none;
    border: none;
    color: #6a737d;
    font-size: 18px;
    cursor: pointer;
    padding: 0;
    width: 24px;
    height: 24px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    
    &:hover {
      background: #e1e4e8;
      color: #24292e;
    }
  }
}

.mention-popup-content {
  padding: 8px 0;
  max-height: 240px;
  overflow-y: auto;
}

.mention-search {
  padding: 8px 16px;
  border-bottom: 1px solid #e1e4e8;
  
  input {
    width: 100%;
    padding: 8px 12px;
    border: 1px solid #d1d5da;
    border-radius: 6px;
    font-size: 14px;
    outline: none;
    
    &:focus {
      border-color: #0366d6;
      box-shadow: 0 0 0 3px rgba(3, 102, 214, 0.1);
    }
  }
}

.mention-results {
  max-height: 180px;
  overflow-y: auto;
}

.mention-category {
  .category-header {
    padding: 8px 16px 4px 16px;
    font-size: 12px;
    font-weight: 600;
    color: #6a737d;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    background: #f6f8fa;
    border-bottom: 1px solid #e1e4e8;
  }
}

.mention-item {
  display: flex;
  align-items: center;
  padding: 8px 16px;
  cursor: pointer;
  transition: background-color 0.1s ease;
  
  &:hover,
  &.active {
    background: #f1f8ff;
  }
  
  .mention-item-icon {
    font-size: 16px;
    margin-right: 12px;
    width: 20px;
    text-align: center;
  }
  
  .mention-item-content {
    flex: 1;
    min-width: 0;
    
    .mention-item-title {
      font-size: 14px;
      font-weight: 500;
      color: #24292e;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }
    
    .mention-item-subtitle {
      font-size: 12px;
      color: #6a737d;
      margin-top: 2px;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }
  }
}

.no-results {
  padding: 16px;
  text-align: center;
  color: #6a737d;
  font-size: 14px;
}

/* Debug info */
.debug-info {
  position: absolute;
  top: 100%;
  left: 0;
  margin-top: 16px;
  padding: 12px;
  background: #f8f9fa;
  border: 1px solid #e1e4e8;
  border-radius: 6px;
  font-size: 12px;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  
  p {
    margin: 4px 0;
  }
}

/* Responsive design */
@media (max-width: 768px) {
  .tq-textarea-wrapper {
    margin: 0 16px;
  }
  
  .ai-popup {
    min-width: 250px;
    max-width: calc(100vw - 32px);
  }
}
</style>
