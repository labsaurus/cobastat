<script setup lang="ts">
import { ref, defineEmits, defineProps } from 'vue';

const props = defineProps<{
  currentData?: any[]
  animationInProgress?: boolean
}>()

const emit = defineEmits<{
  (e: 'file-uploaded', data: any): void
  (e: 'data-updated', data: any[]): void
  (e: 'open-json-editor'): void
}>();

const fileInput = ref<HTMLInputElement | null>(null);
const errorMessage = ref('');
const isLoading = ref(false);

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (!target.files || target.files.length === 0) return;
  
  const file = target.files[0];
  if (file.type !== 'application/json') {
    errorMessage.value = 'Please upload a JSON file';
    return;
  }
  
  errorMessage.value = '';
  isLoading.value = true;
  
  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const result = e.target?.result as string;
      const jsonData = JSON.parse(result);
      
      // Validate JSON structure
      if (!jsonData.title || !jsonData.data || !Array.isArray(jsonData.data)) {
        errorMessage.value = 'Invalid JSON format. Please ensure it has title and data array.';
        isLoading.value = false;
        return;
      }
      
      // Validate data items
      const isValid = jsonData.data.every((item: any) => 
        typeof item.name === 'string' && 
        (typeof item.value === 'number' || typeof item.value === 'string')
      );
      
      if (!isValid) {
        errorMessage.value = 'Invalid data format. Each item must have name (string) and value (string or number).';
        isLoading.value = false;
        return;
      }
      
      emit('file-uploaded', jsonData);
      isLoading.value = false;
    } catch (error) {
      errorMessage.value = 'Error parsing JSON file';
      isLoading.value = false;
    }
  };
  
  reader.onerror = () => {
    errorMessage.value = 'Error reading file';
    isLoading.value = false;
  };
  
  reader.readAsText(file);
};

const triggerFileInput = () => {
  fileInput.value?.click();
};

const useSampleData = () => {
  const sampleData = {
    title: "Top 10 Countries by Population",
    data: [
      { name: "China", value: 1444.2 },
      { name: "India", value: 1393.4 },
      { name: "USA", value: 332.9 },
      { name: "Indonesia", value: 276.4 },
      { name: "Pakistan", value: 225.2 },
      { name: "Brazil", value: 213.9 },
      { name: "Nigeria", value: 211.4 },
      { name: "Bangladesh", value: 166.3 },
      { name: "Russia", value: 145.9 },
      { name: "Mexico", value: 130.3 }
    ]
  };
  
  emit('file-uploaded', sampleData);
};
</script>

<template>
  <div class="file-uploader">
    <h3>Data Source</h3>
    <div class="upload-controls">
      <button class="upload-button" @click="triggerFileInput">
        <span v-if="isLoading">Loading...</span>
        <span v-else>Upload JSON</span>
      </button>
      <button class="sample-button" @click="useSampleData">Use Sample Data</button>
      <button class="json-editor-button" @click="emit('open-json-editor')" :disabled="animationInProgress">JSON Editor</button>
      <input 
        ref="fileInput"
        type="file" 
        accept=".json,application/json" 
        @change="handleFileChange"
        style="display: none"
      >
    </div>
    <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
    <div class="json-format-info">
      <p class="format-title">Required JSON format:</p>
      <pre>{
  "title": "Your Chart Title",
  "data": [
    { "name": "Item 1", "value": "100" },
    { "name": "Item 2", "value": "200" }
  ]
}</pre>
      <a href="/sample-data.json" download="sample-data.json" class="download-link">Download Sample JSON</a>
    </div>
  </div>
</template>

<style scoped>
.file-uploader {
  background: rgba(0, 0, 0, 0.1);
  padding: 0.75rem 1rem;
  border-radius: 8px;
  min-width: 200px;
  max-width: 100%;
}

h3 {
  margin-top: 0;
  margin-bottom: 0.5rem;
  font-size: 1rem;
  text-align: center;
  font-weight: 500;
}

.upload-controls {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.upload-button, .sample-button, .json-editor-button {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 0.9rem;
}

.upload-button {
  background-color: #646cff;
  color: white;
  box-shadow: 0 2px 4px rgba(100, 108, 255, 0.2);
}

.upload-button:hover {
  background-color: #535bf2;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(100, 108, 255, 0.3);
}

.sample-button {
  background-color: rgba(255, 255, 255, 0.1);
  color: white;
}

.sample-button:hover {
  background-color: rgba(255, 255, 255, 0.2);
  transform: translateY(-2px);
}

.json-editor-button {
  background-color: #42b883;
  color: white;
  box-shadow: 0 2px 4px rgba(66, 184, 131, 0.2);
}

.json-editor-button:hover:not(:disabled) {
  background-color: #369870;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(66, 184, 131, 0.3);
}

.json-editor-button:disabled {
  background-color: #666;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.error-message {
  color: #ff4d4d;
  font-size: 0.8rem;
  margin: 0.5rem 0;
  text-align: center;
}

.json-format-info {
  margin-top: 0.5rem;
  font-size: 0.7rem;
  background: rgba(0, 0, 0, 0.2);
  padding: 0.5rem;
  border-radius: 4px;
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease, padding 0.3s ease, margin 0.3s ease;
}

.file-uploader:hover .json-format-info {
  max-height: 200px;
  padding: 0.5rem;
  margin-top: 0.5rem;
}

.format-title {
  margin: 0 0 0.25rem 0;
  font-weight: bold;
}

pre {
  margin: 0;
  white-space: pre-wrap;
  font-family: monospace;
  font-size: 0.7rem;
  color: #aaa;
}

.download-link {
  display: block;
  margin-top: 0.5rem;
  font-size: 0.8rem;
  color: #646cff;
  text-decoration: underline;
  text-align: center;
}

.download-link:hover {
  color: #535bf2;
}
</style>