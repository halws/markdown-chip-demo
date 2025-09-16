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

    <!-- Hidden measurement container for width calculations -->
    <div 
      ref="measurementContainer" 
      class="measurement-container"
      aria-hidden="true"
    ></div>

    <!-- Debug info (optional) -->
    <div v-if="showDebugInfo" class="debug-info">
      <p><strong>Caret Position:</strong> {{ caretPosition }}</p>
      <p><strong>Rendered Text Length:</strong> {{ renderedText.length }}</p>
      <p><strong>Original Text Length:</strong> {{ originalText.length }}</p>
      <p><strong>Popup Position:</strong> {{ popupPosition }}</p>
      <p><strong>Character Width:</strong> {{ CHAR_WIDTH_PX }}px</p>
      <p><strong>Fixed ID Width:</strong> {{ FIXED_ID_TOTAL_WIDTH }} chars</p>
      <p><strong>Next ID:</strong> [{{ (mentionIdCounter + 1).toString().padStart(ID_WIDTH, '0') }}]</p>
      <div class="debug-texts">
        <div><strong>Rendered Text:</strong> <code>{{ renderedText }}</code></div>
        <div><strong>Original Text:</strong> <code>{{ originalText }}</code></div>
        <div><strong>Mention Mapping:</strong> <code>{{ JSON.stringify(mentionMapping, null, 2) }}</code></div>
        <div><strong>Width Mapping:</strong> <code>{{ JSON.stringify(mentionWidthMapping, null, 2) }}</code></div>
      </div>
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
const originalText = ref(props.initialText) // Store the actual user input
const renderedText = ref(props.initialText) // Store the text with surrogate characters
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

// Mapping between surrogate IDs and original mention URIs
const mentionMapping = ref<Record<string, string>>({})
// Mapping between surrogate IDs and target widths for precise scaling
const mentionWidthMapping = ref<Record<string, number>>({})
let mentionIdCounter = 0

// Fixed-width ID configuration
const ID_WIDTH = 5 // Always use 5 digits: [00001], [00002], etc.
const ID_PREFIX = '[' 
const ID_SUFFIX = ']'
const FIXED_ID_TOTAL_WIDTH = ID_PREFIX.length + ID_WIDTH + ID_SUFFIX.length // "[00001]" = 7 chars

// Characters used for width calculation (different widths for precision)
const SURROGATE_CHAR = 'm' // Base character (full width)
let THIN_CHAR = 'i' // Thinner character for fractional adjustments (will be updated)
const SPACE_CHAR = ' ' // Space for fine-tuning
let CHAR_WIDTH_PX = 8.4 // Will be measured dynamically
let THIN_CHAR_WIDTH_PX = 4.2 // Will be measured dynamically
let SPACE_WIDTH_PX = 4.2 // Will be measured dynamically

// Hidden measurement container for precise width calculations
const measurementContainer = ref<HTMLDivElement | null>(null)

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
  const textContent = renderedText.value || ''
  
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
    
    // Replace surrogate patterns with visual components
    processedContent = processedContent.replace(
      /\[(\d{5})\]m*/g, // Match IDs with m characters (monospace font)
      (match, surrogateId) => {
        const originalMention = mentionMapping.value[surrogateId]
        const targetWidth = mentionWidthMapping.value[surrogateId]
        if (originalMention) {
          const uriMatch = originalMention.match(/mention:\/\/([^\/\s]+)\/([^\s]+)/)
          if (uriMatch) {
            const [, type, id] = uriMatch
            const mentionData = mockMentionData.find(item => item.id === id && item.type === type)
            if (mentionData) {
              return renderMentionComponent(mentionData, targetWidth)
            }
          }
        }
        return `<span class="mention-error">Unknown mention: ${surrogateId}</span>`
      }
    )
    
    return processedContent + '\n'
  }
})

// Computed property to sync rendered text with textarea and update original text
const text = computed({
  get: () => renderedText.value,
  set: (value: string) => {
    renderedText.value = value
    originalText.value = restoreOriginalText(value)
  }
})

// Computed property to expose the original text for external use
const originalTextComputed = computed(() => originalText.value)

// Function to generate fixed-width ID
function generateFixedWidthId(): string {
  mentionIdCounter++
  return mentionIdCounter.toString().padStart(ID_WIDTH, '0') // e.g., "00001", "00002"
}

// Function to measure actual component width by rendering it
function measureComponentWidth(mentionData: any): Promise<number> {
  return new Promise((resolve) => {
    if (!measurementContainer.value) {
      console.warn('⚠️ measurementContainer not available, using fallback width')
      resolve(100) // fallback width
      return
    }
    
    // Create temporary element with the exact same HTML as the mention component
    const tempElement = document.createElement('span')
    tempElement.innerHTML = renderMentionComponent(mentionData)
    tempElement.style.position = 'absolute'
    tempElement.style.visibility = 'hidden'
    tempElement.style.whiteSpace = 'nowrap'
    
    // Apply the same styles as the overlay
    tempElement.style.fontFamily = 'Monaco, Menlo, Ubuntu Mono, monospace'
    tempElement.style.fontSize = '14px'
    tempElement.style.lineHeight = '1.5'
    
    measurementContainer.value.appendChild(tempElement)
    
    // Measure the actual rendered width
    nextTick(() => {
      const width = tempElement.getBoundingClientRect().width
      console.log('📏 Component measurement:', {
        mentionType: mentionData.type,
        mentionTitle: mentionData.title,
        measuredWidth: width,
        innerHTML: tempElement.innerHTML
      })
      measurementContainer.value?.removeChild(tempElement)
      resolve(width)
    })
  })
}

// Function to measure individual character widths precisely
function measureCharWidth(): Promise<number> {
  return new Promise((resolve) => {
    if (!measurementContainer.value) {
      console.warn('⚠️ measurementContainer not available for char width, using fallback')
      resolve(8.4) // fallback
      return
    }
    
    const testChars = ['m', 'i', 'l', 'j', '.', ' ', '|']
    const measurements: Record<string, number> = {}
    
    // Create elements for each character
    const elements = testChars.map(char => {
      const element = document.createElement('span')
      element.textContent = char
      element.style.position = 'absolute'
      element.style.visibility = 'hidden'
      element.style.whiteSpace = 'nowrap'
      element.style.fontFamily = 'Monaco, Menlo, Ubuntu Mono, monospace'
      element.style.fontSize = '14px'
      element.style.lineHeight = '1.5'
      measurementContainer.value!.appendChild(element)
      return { char, element }
    })
    
    nextTick(() => {
      // Measure each character individually
      elements.forEach(({ char, element }) => {
        measurements[char] = element.getBoundingClientRect().width
      })
      
      console.log('📐 Individual character measurements:', measurements)
      
      // Find the best characters for our system
      const mainWidth = measurements['m']
      let thinWidth = measurements['i']
      let spaceWidth = measurements[' ']
      
      // Try to find a thinner character if 'i' is same width as 'm'
      if (Math.abs(thinWidth - mainWidth) < 0.1) {
        // Try other narrow characters
        const narrowChars = ['l', 'j', '|', '.']
        for (const char of narrowChars) {
          if (measurements[char] < mainWidth * 0.8) {
            thinWidth = measurements[char]
            console.log(`🔍 Found thinner character '${char}' with width ${thinWidth}px`)
            break
          }
        }
      }
      
      console.log('📏 Selected character widths:', {
        mainChar: 'm',
        mainWidth,
        thinChar: thinWidth < mainWidth * 0.8 ? 'l' : 'i', // Use 'l' if it's thinner
        thinWidth,
        spaceChar: 'space',
        spaceWidth
      })
      
      CHAR_WIDTH_PX = mainWidth
      THIN_CHAR_WIDTH_PX = thinWidth
      SPACE_WIDTH_PX = spaceWidth
      
      // Update the thin character if we found a better one
      if (thinWidth < mainWidth * 0.8) {
        // Find which character has this width
        for (const [char, width] of Object.entries(measurements)) {
          if (Math.abs(width - thinWidth) < 0.1) {
            // Update the global thin char
            const originalThinChar = THIN_CHAR
            THIN_CHAR = char // Update to use the thinner character
            console.log(`💡 Updated THIN_CHAR from '${originalThinChar}' to '${char}' (${width}px)`)
            break
          }
        }
      }
      
      // Clean up elements
      elements.forEach(({ element }) => {
        measurementContainer.value?.removeChild(element)
      })
      
      resolve(mainWidth)
    })
  })
}

// Function to generate surrogate text for a mention
async function generateSurrogateText(originalMention: string, mentionData: any): Promise<string> {
  console.log('🔧 Starting surrogate text generation for:', mentionData.title)
  
  // Measure actual component width
  const componentWidth = await measureComponentWidth(mentionData)
  
  // Calculate remaining width after fixed ID
  const fixedIdWidth = FIXED_ID_TOTAL_WIDTH * CHAR_WIDTH_PX
  const remainingWidth = componentWidth - fixedIdWidth
  
  // Generate unique fixed-width ID for this mention
  const surrogateId = generateFixedWidthId()
  
  // Store mapping
  mentionMapping.value[surrogateId] = originalMention
  // Store target width for precise scaling
  mentionWidthMapping.value[surrogateId] = componentWidth
  
  // Build surrogate characters to match remaining width as closely as possible
  // Since all characters have the same width in monospace fonts, we use optimal rounding
  
  // Calculate exact number of characters needed for remaining width
  const exactCharsNeeded = remainingWidth / CHAR_WIDTH_PX
  
  // Use smart rounding: if we're very close to the next integer, round up
  const threshold = 0.8 // Round up if we're 80% or more to the next character
  const optimalChars = exactCharsNeeded % 1 >= threshold 
    ? Math.ceil(exactCharsNeeded) 
    : Math.floor(exactCharsNeeded)
  
  const surrogateChars = SURROGATE_CHAR.repeat(Math.max(0, optimalChars))
  const currentWidth = optimalChars * CHAR_WIDTH_PX
  
  const result = `${ID_PREFIX}${surrogateId}${ID_SUFFIX}${surrogateChars}`
  const totalCalculatedWidth = fixedIdWidth + currentWidth
  
  console.log('🧮 Optimal surrogate calculation:', {
    componentWidth,
    fixedIdWidth,
    remainingWidth,
    exactCharsNeeded,
    optimalChars,
    fractionalPart: exactCharsNeeded % 1,
    roundingDecision: exactCharsNeeded % 1 >= threshold ? 'ROUND_UP' : 'ROUND_DOWN',
    surrogateChars: JSON.stringify(surrogateChars),
    totalCalculatedWidth,
    widthDifference: Math.abs(componentWidth - totalCalculatedWidth),
    result,
    surrogateScaleFactor: componentWidth / totalCalculatedWidth
  })
  
  // Verify the calculation by measuring the surrogate text
  if (measurementContainer.value) {
    const verifyElement = document.createElement('span')
    verifyElement.textContent = result
    verifyElement.style.position = 'absolute'
    verifyElement.style.visibility = 'hidden'
    verifyElement.style.whiteSpace = 'nowrap'
    verifyElement.style.fontFamily = 'Monaco, Menlo, Ubuntu Mono, monospace'
    verifyElement.style.fontSize = '14px'
    verifyElement.style.lineHeight = '1.5'
    
    measurementContainer.value.appendChild(verifyElement)
    
    nextTick(() => {
      const actualSurrogateWidth = verifyElement.getBoundingClientRect().width
      const widthDifference = Math.abs(componentWidth - actualSurrogateWidth)
      
      console.log('🔍 Width verification:', {
        componentWidth,
        actualSurrogateWidth,
        widthDifference,
        isAccurate: widthDifference < 2, // Within 2px tolerance
        surrogateText: result
      })
      
      if (widthDifference > 2) {
        console.warn('⚠️ Width mismatch detected! Component and surrogate widths differ by', widthDifference, 'px')
      }
      
      measurementContainer.value?.removeChild(verifyElement)
    })
  }
  
  return result
}

// Function to restore original text from rendered text
function restoreOriginalText(rendered: string): string {
  let restored = rendered
  
  // Replace surrogate patterns with original mentions
  Object.entries(mentionMapping.value).forEach(([id, originalMention]) => {
    const pattern = new RegExp(`\\[${id}\\]m*`, 'g')
    restored = restored.replace(pattern, originalMention)
  })
  
  return restored
}

// Function to convert original text to rendered text with surrogates
function generateRenderedText(original: string): string {
  let rendered = original
  
  // Find all mention URIs and replace with surrogates
  rendered = rendered.replace(
    /mention:\/\/([^\/\s]+)\/([^\s]+)/g,
    (match, type, id) => {
      const mentionData = mockMentionData.find(item => item.id === id && item.type === type)
      if (mentionData) {
        return generateSurrogateText(match, mentionData)
      }
      return match // Keep original if not found
    }
  )
  
  return rendered
}

// Function to render mention components as HTML with optional width adjustment
function renderMentionComponent(mentionData: any, targetWidth?: number) {
  const { type, icon, title, subtitle, id } = mentionData
  
  // Calculate scale factor if target width is provided
  let scaleStyle = ''
  if (targetWidth) {
    // We'll measure the component and apply a scale transform if needed
    scaleStyle = ` data-target-width="${targetWidth}"`
  }
  
  switch (type) {
    case 'custom_field':
      return `<span class="mention mention-custom-field" data-id="${id}"${scaleStyle}><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-value">${subtitle}</span></span></span>`
      
    case 'observable':
      return `<span class="mention mention-observable" data-id="${id}"${scaleStyle}><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'note':
      return `<span class="mention mention-note" data-id="${id}"${scaleStyle}><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'attachment':
      return `<span class="mention mention-attachment" data-id="${id}"${scaleStyle}><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'event':
      return `<span class="mention mention-event" data-id="${id}"${scaleStyle}><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    case 'case':
      return `<span class="mention mention-case" data-id="${id}"${scaleStyle}><span class="mention-icon">${icon}</span><span class="mention-content"><span class="mention-title">${title}</span><span class="mention-subtitle">${subtitle}</span></span></span>`
      
    default:
      return `<span class="mention mention-unknown" data-id="${id}"${scaleStyle}><span class="mention-icon">❓</span><span class="mention-content"><span class="mention-title">Unknown: ${title}</span></span></span>`
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

async function selectMention(item: any) {
  const ta = textarea.value
  if (!ta) return

  // Create mention URI
  const mentionURI = `mention://${item.type}/${item.id}`
  
  // Generate surrogate text for this mention (now async with real measurements)
  const surrogateText = await generateSurrogateText(mentionURI, item)
  
  // Replace @ with the surrogate text in rendered text
  const beforeMention = renderedText.value.substring(0, mentionStartPosition.value)
  const afterCaret = renderedText.value.substring(ta.selectionStart)
  
  renderedText.value = beforeMention + surrogateText + afterCaret
  
  // Update original text
  originalText.value = restoreOriginalText(renderedText.value)
  
  // Move caret to end of inserted mention
  nextTick(() => {
    const newPosition = beforeMention.length + surrogateText.length
    ta.setSelectionRange(newPosition, newPosition)
    ta.focus()
    
    // Apply precision adjustments after content is rendered
    setTimeout(() => {
      applyPrecisionScaling()
      adjustSurrogateSpacing()
    }, 10)
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
onMounted(async () => {
  console.log('🚀 Component mounted, initializing measurements...')
  syncScroll()
  updateMentionPopupPosition()
  
  // Initialize character width measurement
  console.log('📐 Measuring character width...')
  await measureCharWidth()
  console.log('✅ Character width measurement complete:', CHAR_WIDTH_PX)
  
  // Add keyboard event listeners
  document.addEventListener('keydown', handleKeydown)
})

// Watch for text changes to update overlay
watch(text, () => {
  nextTick(() => {
    syncScroll()
    applyPrecisionScaling()
  })
})

// Function to apply precision scaling to mention components
function applyPrecisionScaling() {
  if (!overlay.value) {
    console.warn('⚠️ No overlay element for precision scaling')
    return
  }
  
  // Find all mention components with target width data
  const mentionElements = overlay.value.querySelectorAll('.mention[data-target-width]')
  console.log('🔍 Found mention elements for scaling:', mentionElements.length)
  
  mentionElements.forEach((element: Element, index: number) => {
    const targetWidth = parseFloat(element.getAttribute('data-target-width') || '0')
    if (targetWidth > 0) {
      const actualWidth = element.getBoundingClientRect().width
      const scaleFactor = targetWidth / actualWidth
      const difference = targetWidth - actualWidth
      
      console.log(`🎯 Scaling analysis for element ${index}:`, {
        targetWidth,
        actualWidth,
        scaleFactor,
        difference,
        shouldScale: Math.abs(difference) > 0.5
      })
      
      // Only apply scaling if the difference is significant (> 0.5px)
      if (Math.abs(difference) > 0.5) {
        (element as HTMLElement).style.transform = `scaleX(${scaleFactor})`
        (element as HTMLElement).style.transformOrigin = 'left center'
        (element as HTMLElement).style.display = 'inline-block' // Ensure transform works
        
        console.log('✅ Applied precision scaling:', {
          targetWidth,
          actualWidth,
          scaleFactor,
          difference,
          newWidth: actualWidth * scaleFactor
        })
        
        // Verify the scaling worked
        setTimeout(() => {
          const newWidth = element.getBoundingClientRect().width
          console.log('🔍 Scaling verification:', {
            originalWidth: actualWidth,
            targetWidth,
            newWidth,
            finalDifference: targetWidth - newWidth
          })
        }, 50)
      } else {
        console.log('ℹ️ Scaling not needed, difference too small:', difference)
      }
    }
  })
}

// Function to adjust surrogate text spacing for precision
function adjustSurrogateSpacing() {
  console.log('🔧 Starting surrogate spacing adjustment...')
  
  if (!textarea.value || !overlay.value) {
    console.warn('⚠️ Missing textarea or overlay for spacing adjustment')
    return
  }
  
  // Find mention components and calculate average width difference
  const mentionElements = overlay.value.querySelectorAll('.mention[data-target-width]')
  console.log('📏 Found mentions for spacing adjustment:', mentionElements.length)
  
  if (mentionElements.length === 0) {
    console.log('ℹ️ No mentions found for spacing adjustment')
    return
  }
  
  let totalWidthDifference = 0
  let totalSurrogateChars = 0
  
  mentionElements.forEach((element: Element) => {
    const targetWidth = parseFloat(element.getAttribute('data-target-width') || '0')
    const actualComponentWidth = element.getBoundingClientRect().width
    
    console.log('📊 Analyzing mention element:', {
      targetWidth,
      actualComponentWidth,
      componentDifference: targetWidth - actualComponentWidth
    })
    
    // The component is already perfect width, so we need to stretch the surrogate text
    // to match the component width. The difference we need to make up is between
    // the surrogate text width and the component width.
    
    // Get the surrogate ID to find the surrogate text
    const surrogateId = element.getAttribute('data-id')
    if (surrogateId && mentionMapping.value[surrogateId]) {
      // Find the surrogate pattern in the rendered text
      const surrogatePattern = renderedText.value.match(new RegExp(`\\[${surrogateId}\\](m*)`, 'g'))
      if (surrogatePattern && surrogatePattern[0]) {
        const mChars = surrogatePattern[0].match(/m/g)?.length || 0
        const surrogateTextWidth = surrogatePattern[0].length * CHAR_WIDTH_PX
        const widthDifference = actualComponentWidth - surrogateTextWidth
        
        console.log('📏 Surrogate analysis:', {
          surrogatePattern: surrogatePattern[0],
          mChars,
          surrogateTextWidth,
          actualComponentWidth,
          widthDifference
        })
        
        totalSurrogateChars += mChars
        totalWidthDifference += widthDifference
      }
    }
  })
  
  if (totalSurrogateChars > 0 && Math.abs(totalWidthDifference) > 0.5) {
    // Calculate letter spacing needed to distribute the width difference
    const letterSpacing = totalWidthDifference / totalSurrogateChars
    
    console.log('📏 Adjusting surrogate spacing:', {
      totalWidthDifference,
      totalSurrogateChars,
      letterSpacing
    })
    
    // Apply letter spacing to textarea
    textarea.value.style.setProperty('--surrogate-letter-spacing', `${letterSpacing}px`)
    textarea.value.classList.add('precise-spacing')
  }
}

// Expose methods for parent components if needed
defineExpose({
  focusTextarea: () => textarea.value?.focus(),
  insertText: (textToInsert: string) => {
    if (!textarea.value) return
    const start = textarea.value.selectionStart
    const end = textarea.value.selectionEnd
    const beforeCaret = renderedText.value.substring(0, start)
    const afterCaret = renderedText.value.substring(end)
    renderedText.value = beforeCaret + textToInsert + afterCaret
    originalText.value = restoreOriginalText(renderedText.value)
    nextTick(() => {
      const newPosition = start + textToInsert.length
      textarea.value?.setSelectionRange(newPosition, newPosition)
    })
  },
  getOriginalText: () => originalText.value,
  getRenderedText: () => renderedText.value,
  getMentionMapping: () => mentionMapping.value
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

/* Letter spacing adjustment for surrogate text precision */
.tq-textarea.precise-spacing {
  letter-spacing: var(--surrogate-letter-spacing, 0px);
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
  
  /* Apply scaling if data-target-width is present */
  &[data-target-width] {
    transform-origin: left center;
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

/* Hidden measurement container */
.measurement-container {
  position: absolute;
  top: -9999px;
  left: -9999px;
  visibility: hidden;
  pointer-events: none;
  z-index: -1;
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
  max-width: 100%;
  overflow-x: auto;
  
  p {
    margin: 4px 0;
  }
  
  .debug-texts {
    margin-top: 8px;
    
    div {
      margin: 8px 0;
      padding: 4px;
      background: #fff;
      border: 1px solid #ddd;
      border-radius: 3px;
      
      code {
        display: block;
        white-space: pre-wrap;
        word-break: break-all;
        font-size: 10px;
        color: #333;
      }
    }
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
