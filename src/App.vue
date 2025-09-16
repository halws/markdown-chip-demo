<script setup lang="ts">
import { ref, watch } from 'vue'
import MarkdownTextareaOverlay from './components/MarkdownTextareaOverlay.vue'
import { demoText } from './data/demoContent'

const showDebugInfo = ref(false)
const currentText = ref(demoText) // Reactive text that both components will share

// Watch for changes in currentText
watch(currentText, (newValue) => {
  console.log('🏠 App.vue: currentText changed, length:', newValue.length)
})
</script>

<template>
  <v-app>
    <v-app-bar color="primary" dark>
      <v-app-bar-title>
        <v-icon class="mr-2">mdi-robot</v-icon>
        Markdown Editor with Mentions
      </v-app-bar-title>
      <v-spacer></v-spacer>
      <v-switch
        v-model="showDebugInfo"
        label="Debug Info"
        color="white"
        hide-details
        class="mr-4"
      ></v-switch>
    </v-app-bar>

    <v-main>
      <v-container fluid class="pa-6">
        <v-card class="mb-6" elevation="2">
          <v-card-title class="text-h5">
            <v-icon class="mr-2">mdi-map</v-icon>
            Mention URI Mapping Guide
          </v-card-title>
          <v-card-text>
            <v-row>
              <v-col cols="12" md="6">
                <v-card variant="outlined" class="h-100">
                  <v-card-title class="text-h6">
                    <v-icon class="mr-2">mdi-tag</v-icon>
                    Custom Fields
                  </v-card-title>
                  <v-card-text>
                    <v-list density="compact">
                      <v-list-item>
                        <template v-slot:prepend>
                          <code class="text-caption">mention://custom_field/cf1</code>
                        </template>
                        <template v-slot:append>
                          <v-chip color="pink" variant="elevated" size="small">@ comm_method • Slack</v-chip>
                        </template>
                        <v-icon class="mx-2">mdi-arrow-right</v-icon>
                      </v-list-item>
                      <v-list-item>
                        <template v-slot:prepend>
                          <code class="text-caption">mention://custom_field/cf2</code>
                        </template>
                        <template v-slot:append>
                          <v-chip color="pink" variant="elevated" size="small">@ executor • John Doe</v-chip>
                        </template>
                        <v-icon class="mx-2">mdi-arrow-right</v-icon>
                      </v-list-item>
                      <v-list-item>
                        <template v-slot:prepend>
                          <code class="text-caption">mention://custom_field/cf3</code>
                        </template>
                        <template v-slot:append>
                          <v-chip color="pink" variant="elevated" size="small">@ priority_level • Critical</v-chip>
                        </template>
                        <v-icon class="mx-2">mdi-arrow-right</v-icon>
                      </v-list-item>
                    </v-list>
                  </v-card-text>
                </v-card>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>

        <!-- Editor Section -->
        <v-row class="mt-6">
          <v-col cols="12" md="6">
            <v-card elevation="2">
              <v-card-title class="text-h6">
                <v-icon class="mr-2">mdi-pencil</v-icon>
                Edit Mode (Raw Markdown)
              </v-card-title>
              <v-card-text>
                <MarkdownTextareaOverlay 
                  v-model="currentText"
                  :show-debug-info="showDebugInfo"
                  :readonly="false"
                />
              </v-card-text>
            </v-card>
          </v-col>
          
          <v-col cols="12" md="6">
            <v-card elevation="2">
              <v-card-title class="text-h6">
                <v-icon class="mr-2">mdi-eye</v-icon>
                Readonly Mode (Rendered Output)
              </v-card-title>
              <v-card-text>
                <MarkdownTextareaOverlay 
                  v-model="currentText"
                  :readonly="true"
                />
              </v-card-text>
            </v-card>
          </v-col>
        </v-row>

        <!-- Usage Tips -->
        <v-card class="mt-6" color="blue-grey-lighten-5">
          <v-card-title class="text-h6">
            <v-icon class="mr-2">mdi-lightbulb</v-icon>
            Usage Tips
          </v-card-title>
          <v-card-text>
            <v-list density="compact">
              <v-list-item>
                <template v-slot:prepend>
                  <v-icon>mdi-content-copy</v-icon>
                </template>
                Copy any URI from above and paste it into the editor
              </v-list-item>
              <v-list-item>
                <template v-slot:prepend>
                  <v-icon>mdi-at</v-icon>
                </template>
                Type <kbd>@</kbd> to open the mention selector
              </v-list-item>
              <v-list-item>
                <template v-slot:prepend>
                  <v-icon>mdi-mouse</v-icon>
                </template>
                Hover over notes in readonly mode to see details
              </v-list-item>
            </v-list>
          </v-card-text>
        </v-card>
      </v-container>
    </v-main>
  </v-app>
</template>

<style scoped>
/* Minimal custom styles - Vuetify handles most styling */
kbd {
  background: #f5f5f5;
  border: 1px solid #ddd;
  border-radius: 3px;
  padding: 0.1em 0.3em;
  font-size: 0.85em;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}
</style>
