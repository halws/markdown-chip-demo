<template>
  <div class="markdown-editor-wrapper" ref="wrapper">
    <!-- Edit Mode: Simple textarea -->
    <div v-if="!readonly" class="edit-mode">
      <textarea
        ref="textarea"
        v-model="originalText"
        class="markdown-textarea"
        :style="{ height: textareaHeight + 'px' }"
        placeholder="Type your markdown here... Use mention://type/id for object references"
        @input="onInput"
      />

    </div>

    <!-- Readonly Mode: Rendered markdown with custom components -->
    <div v-else class="readonly-mode">
      <div 
        class="markdown-content"
        v-html="renderedMarkdown"
      />
    </div>

    <!-- Note Tooltip -->
    <div 
      v-if="showTooltip && tooltipData" 
      class="note-tooltip"
      :style="{ 
        left: tooltipPosition.x + 'px', 
        top: tooltipPosition.y + 'px' 
      }"
    >
      <v-card max-width="400" elevation="8">
        <v-card-title class="text-h6 pb-2">
          <v-icon class="mr-2">mdi-note-text</v-icon>
          {{ tooltipData.title }}
        </v-card-title>
        <v-card-text>
          <p class="mb-3">{{ tooltipData.content }}</p>
          <v-divider class="mb-3"></v-divider>
          <div class="text-caption">
            <div class="mb-1">
              <v-icon size="small" class="mr-1">mdi-account-plus</v-icon>
              <strong>Created:</strong> {{ tooltipData.createdAt }} by {{ tooltipData.creator }}
            </div>
            <div v-if="tooltipData.editor">
              <v-icon size="small" class="mr-1">mdi-account-edit</v-icon>
              <strong>Last edited:</strong> {{ tooltipData.editedAt }} by {{ tooltipData.editor }}
            </div>
          </div>
        </v-card-text>
      </v-card>
    </div>

    <!-- Debug info (optional) -->
    <div v-if="showDebugInfo" class="debug-info">
      <p><strong>Mode:</strong> {{ readonly ? 'Readonly' : 'Edit' }}</p>
      <p><strong>Original Text Length:</strong> {{ originalText.length }}</p>
      <div class="debug-texts">
        <div><strong>Original Text:</strong> <code>{{ originalText }}</code></div>
        <div><strong>Rendered Markdown:</strong> <code>{{ renderedMarkdown.substring(0, 200) }}...</code></div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
import { marked } from 'marked'
import DOMPurify from 'dompurify'

// Props
interface Props {
  modelValue?: string
  initialText?: string
  showDebugInfo?: boolean
  readonly?: boolean
}

// Emits
const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

const props = withDefaults(defineProps<Props>(), {
  initialText: `# Markdown Editor with Mentions

Type your **markdown** here with mention URIs like mention://custom_field/cf1

Try hovering over notes in readonly mode!`,
  showDebugInfo: false,
  readonly: false
})

// Reactive state - use modelValue if provided, otherwise fall back to initialText
const originalText = ref(props.modelValue || props.initialText)

// DOM refs
const textarea = ref<HTMLTextAreaElement | null>(null)
const wrapper = ref<HTMLDivElement | null>(null)

// UI state
const textareaHeight = ref(300)

// Hover tooltip state
const showTooltip = ref(false)
const tooltipData = ref<any>(null)
const tooltipPosition = ref({ x: 0, y: 0 })

// Mock data for mentions (in real app, this would come from API)
const mockMentionData = [
  // Custom Fields - Following the spec format: @ [field name] • [field value]
  { id: 'cf1', type: 'custom_field', icon: '🏷️', title: 'comm_method', subtitle: 'Slack', category: 'Custom Fields' },
  { id: 'cf2', type: 'custom_field', icon: '🏷️', title: 'executor', subtitle: 'John Doe', category: 'Custom Fields' },
  { id: 'cf3', type: 'custom_field', icon: '🏷️', title: 'priority_level', subtitle: 'Critical', category: 'Custom Fields' },
  { id: 'cf4', type: 'custom_field', icon: '🏷️', title: 'source_system', subtitle: 'SIEM-01', category: 'Custom Fields' },
  { id: 'cf5', type: 'custom_field', icon: '🏷️', title: 'case_status', subtitle: 'In Progress', category: 'Custom Fields' },
  { id: 'cf6', type: 'custom_field', icon: '🏷️', title: 'threat_level', subtitle: 'High', category: 'Custom Fields' },
  { id: 'cf7', type: 'custom_field', icon: '🏷️', title: 'investigation_type', subtitle: 'Malware Analysis', category: 'Custom Fields' },
  { id: 'cf8', type: 'custom_field', icon: '🏷️', title: 'assigned_team', subtitle: 'SOC Team Alpha', category: 'Custom Fields' },
  
  // Observables
  { id: 'obs1', type: 'observable', icon: '🔍', title: '192.168.1.100', subtitle: 'IP Address', category: 'Observables' },
  { id: 'obs2', type: 'observable', icon: '🔍', title: 'malware.exe', subtitle: 'File Hash', category: 'Observables' },
  { id: 'obs3', type: 'observable', icon: '🔍', title: 'evil.com', subtitle: 'Domain', category: 'Observables' },
  
  // Notes
  { id: 'note1', type: 'note', icon: '📝', title: 'Investigation Summary', subtitle: 'Created 2h ago', category: 'Notes', 
    content: 'Initial analysis reveals sophisticated malware with persistence mechanisms. The attack vector appears to be a spear-phishing email targeting the finance department.',
    creator: 'Sarah Johnson', createdAt: '2024-01-15 14:30', editor: 'Mike Chen', editedAt: '2024-01-15 16:45' },
  { id: 'note2', type: 'note', icon: '📝', title: 'Analyst Comments', subtitle: 'Updated 1h ago', category: 'Notes',
    content: 'Found additional IOCs in network logs. Recommend expanding scope to include all workstations in the finance subnet. Potential lateral movement detected.',
    creator: 'Mike Chen', createdAt: '2024-01-15 15:20', editor: 'Mike Chen', editedAt: '2024-01-15 17:15' },
  { id: 'note3', type: 'note', icon: '📝', title: 'Containment Actions', subtitle: 'Created 30m ago', category: 'Notes',
    content: 'Isolated affected systems and implemented network segmentation. All suspicious processes terminated. Forensic imaging in progress.',
    creator: 'Alex Rodriguez', createdAt: '2024-01-15 17:30', editor: null, editedAt: null },
  { id: 'note4', type: 'note', icon: '📝', title: 'Timeline Analysis', subtitle: 'Updated 15m ago', category: 'Notes',
    content: 'Reconstructed attack timeline: Initial compromise at 09:15, privilege escalation at 09:45, data exfiltration attempt at 10:30. Total dwell time: ~1.5 hours.',
    creator: 'Sarah Johnson', createdAt: '2024-01-15 16:00', editor: 'Sarah Johnson', editedAt: '2024-01-15 17:45' },
  
  // Attachments
  { id: 'att1', type: 'attachment', icon: '📎', title: 'screenshot.png', subtitle: 'Image file', category: 'Attachments' },
  { id: 'att2', type: 'attachment', icon: '📎', title: 'report.pdf', subtitle: 'PDF document', category: 'Attachments' },
  
  // Events
  { id: 'evt1', type: 'event', icon: '⚡', title: 'Login Attempt', subtitle: 'Failed authentication', category: 'Events' },
  { id: 'evt2', type: 'event', icon: '⚡', title: 'File Access', subtitle: 'Suspicious activity', category: 'Events' },
  
  // Cases
  { id: 'case1', type: 'case', icon: '📋', title: 'CASE-2024-001', subtitle: 'Malware Investigation', category: 'Cases' },
  { id: 'case2', type: 'case', icon: '📋', title: 'CASE-2024-002', subtitle: 'Data Breach', category: 'Cases' }
]


// Computed properties
const renderedMarkdown = computed(() => {
  const textContent = originalText.value || ''
  console.log('🔄 Computing rendered markdown, text length:', textContent.length, 'readonly:', props.readonly)
  
  // Parse markdown first
  let htmlContent = marked.parse(textContent) as string
  
  // Replace mention:// URIs with custom components after markdown parsing
  htmlContent = htmlContent.replace(
    /mention:\/\/([^\/\s]+)\/([^\s<]+)/g,
    (match, type, id) => {
      const mentionData = mockMentionData.find(item => item.id === id && item.type === type)
      if (mentionData) {
        return renderMentionComponent(mentionData)
      }
      return `<span class="mention-error">Unknown ${type}: ${id}</span>`
    }
  )
  
  // Sanitize the HTML
  return DOMPurify.sanitize(htmlContent, { 
    FORBID_TAGS: ['style'],
    ALLOW_TAGS: ['span', 'mark', 'strong', 'em', 'code', 'pre', 'a', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'p', 'ul', 'ol', 'li', 'blockquote', 'br'],
    ALLOW_ATTR: ['class', 'data-type', 'data-id', 'href', 'target', 'rel']
  })
})


// Computed property to expose the original text for external use
const text = computed(() => originalText.value)

// Function to render mention component HTML
function renderMentionComponent(mentionData: any) {
  const { type, icon, title, subtitle, id } = mentionData
  
  // Custom fields follow the format: @ [field name] • [field value]
  if (type === 'custom_field') {
    return `<span class="mention" data-type="${type}" data-id="${id}">
      <span class="mention-icon">@</span>
      <span class="mention-text">${title} • ${subtitle}</span>
    </span>`
  }
  
  // Notes have hover functionality for showing details
  if (type === 'note') {
    return `<span class="mention mention-hoverable" data-type="${type}" data-id="${id}">
      <span class="mention-icon">${icon}</span>
      <span class="mention-text">${title}</span>
    </span>`
  }
  
  // Other object types use the standard format
  return `<span class="mention" data-type="${type}" data-id="${id}">
    <span class="mention-icon">${icon}</span>
    <span class="mention-text">${title}</span>
  </span>`
}

// Watch for modelValue changes from parent
watch(() => props.modelValue, (newValue) => {
  if (newValue !== undefined && newValue !== originalText.value) {
    console.log('📥 Received modelValue update from parent:', newValue.substring(0, 50) + '...')
    originalText.value = newValue
  }
})

// Watch for internal text changes and emit to parent
watch(originalText, (newValue) => {
  console.log('📤 Emitting text change to parent:', newValue.substring(0, 50) + '...')
  emit('update:modelValue', newValue)
})

// Tooltip functions for notes
function showNoteTooltip(noteId: string, event: MouseEvent) {
  const noteData = mockMentionData.find(item => item.id === noteId && item.type === 'note')
  if (noteData && wrapper.value) {
    tooltipData.value = noteData
    
    // Calculate position relative to the wrapper
    const wrapperRect = wrapper.value.getBoundingClientRect()
    const x = event.clientX - wrapperRect.left + 10 // Small offset to avoid blocking the cursor
    const y = event.clientY - wrapperRect.top - 10
    
    tooltipPosition.value = { x, y }
    showTooltip.value = true
    console.log('📝 Showing tooltip for note:', noteId, 'at position:', { x, y })
  }
}

function hideNoteTooltip() {
  showTooltip.value = false
  tooltipData.value = null
  console.log('📝 Hiding tooltip')
}

// Set up event listeners for hoverable mentions
function setupHoverListeners() {
  if (!wrapper.value) return
  
  const hoverableElements = wrapper.value.querySelectorAll('.mention-hoverable')
  
  hoverableElements.forEach((element) => {
    const noteId = element.getAttribute('data-id')
    if (noteId) {
      element.addEventListener('mouseenter', (event) => {
        showNoteTooltip(noteId, event as MouseEvent)
      })
      element.addEventListener('mouseleave', hideNoteTooltip)
    }
  })
  
  console.log('🔧 Set up hover listeners for', hoverableElements.length, 'hoverable mentions')
}

// Simple input handler
function onInput() {
  console.log('⌨️ Input event triggered, current text length:', originalText.value.length)
}

// Watch for rendered markdown changes to re-setup event listeners
watch(renderedMarkdown, () => {
  if (props.readonly) {
    nextTick(() => {
      setupHoverListeners()
    })
  }
})

// Setup event listeners on mount
onMounted(() => {
  if (props.readonly) {
    nextTick(() => {
      setupHoverListeners()
    })
  }
})

onUnmounted(() => {
  // Event listeners are automatically cleaned up when elements are removed
  // No manual cleanup needed
})

// Expose methods for parent components
defineExpose({
  text
})
</script>

<style scoped lang="scss">
/* Modern CSS Reset */
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.markdown-editor-wrapper {
  position: relative;
  width: 100%;
  min-height: 300px;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
}

/* Edit Mode Styles */
.edit-mode {
  position: relative;
  width: 100%;
}

.markdown-textarea {
  width: 100%;
  min-height: 300px;
  padding: 16px;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  font-size: 14px;
  line-height: 1.5;
  border: 1px solid #e1e5e9;
  border-radius: 8px;
  background: #ffffff;
  color: #24292f;
  resize: vertical;
  outline: none;
  transition: border-color 0.2s ease;

  &:focus {
    border-color: #0969da;
    box-shadow: 0 0 0 3px rgba(9, 105, 218, 0.1);
  }

  &::placeholder {
    color: #656d76;
  }
}

/* Readonly Mode Styles */
.readonly-mode {
  width: 100%;
  min-height: 300px;
}

.markdown-content {
  padding: 16px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Noto Sans', Helvetica, Arial, sans-serif;
  font-size: 14px;
  line-height: 1.6;
  color: #24292f;
  background: #ffffff;
  border: 1px solid #e1e5e9;
  border-radius: 8px;

  /* Markdown styling */
  :deep(h1), :deep(h2), :deep(h3), :deep(h4), :deep(h5), :deep(h6) {
    margin-top: 24px;
    margin-bottom: 16px;
    font-weight: 600;
    line-height: 1.25;
  }

  :deep(h1) { font-size: 2em; }
  :deep(h2) { font-size: 1.5em; }
  :deep(h3) { font-size: 1.25em; }

  :deep(p) {
    margin-bottom: 16px;
  }

  :deep(strong) {
    font-weight: 600;
  }

  :deep(em) {
    font-style: italic;
  }

  :deep(code) {
    padding: 0.2em 0.4em;
    margin: 0;
    font-size: 85%;
    background-color: rgba(175, 184, 193, 0.2);
    border-radius: 6px;
    font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  }

  :deep(pre) {
    padding: 16px;
    overflow: auto;
    font-size: 85%;
    line-height: 1.45;
    background-color: #f6f8fa;
    border-radius: 6px;
    margin-bottom: 16px;
  }

  :deep(ul), :deep(ol) {
    padding-left: 2em;
    margin-bottom: 16px;
  }

  :deep(li) {
    margin-bottom: 0.25em;
  }

  :deep(blockquote) {
    padding: 0 1em;
    color: #656d76;
    border-left: 0.25em solid #d1d9e0;
    margin-bottom: 16px;
  }

  /* Mention component styling */
  :deep(.mention) {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    padding: 2px 6px;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border-radius: 12px;
    font-size: 12px;
    font-weight: 500;
    text-decoration: none;
    cursor: pointer;
    transition: all 0.2s ease;
    vertical-align: baseline;
    line-height: 1.5;
    max-height: 1.5em;
    white-space: nowrap;

    &:hover {
      transform: translateY(-1px);
      box-shadow: 0 2px 8px rgba(102, 126, 234, 0.3);
    }

    &.mention-hoverable {
      cursor: help;
      
      &:hover {
        background: linear-gradient(135deg, #764ba2 0%, #667eea 100%);
        box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
      }
    }

    .mention-icon {
      font-size: 10px;
      line-height: 1;
    }

    .mention-text {
      font-size: 11px;
      line-height: 1;
      font-weight: 500;
    }
  }

  :deep(.mention-error) {
    color: #d1242f;
    background: #fff1f3;
    padding: 2px 6px;
    border-radius: 4px;
    font-size: 12px;
  }
}

/* Note Tooltip Styles */
.note-tooltip {
  position: absolute;
  z-index: 1000;
  pointer-events: none; /* Allow mouse events to pass through to prevent conflicts */
  transition: opacity 0.2s ease;
}

/* Debug Info Styles */
.debug-info {
  margin-top: 16px;
  padding: 16px;
  background: #f6f8fa;
  border: 1px solid #e1e5e9;
  border-radius: 8px;
  font-size: 12px;
  color: #24292f;

  p {
    margin-bottom: 8px;
  }

  .debug-texts {
    margin-top: 12px;

    > div {
      margin-bottom: 8px;
      word-break: break-all;

      code {
        background: white;
        padding: 4px 8px;
        border-radius: 4px;
        font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
      }
    }
  }
}
</style>
