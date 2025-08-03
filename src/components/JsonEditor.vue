<template>
  <div class="json-editor">
    <div class="modal-header">
      <h2>JSON Data Editor</h2>
    </div>
    
    <div class="modal-body">
      <!-- Import Section -->
      <div class="section">
        <h3>Import JSON</h3>
        <div class="import-controls">
          <input 
            type="file" 
            @change="handleFileImport" 
            accept=".json"
            ref="fileInput"
            class="file-input"
          >
          <button @click="loadSampleData" class="btn-secondary">
            Load Sample Data
          </button>
        </div>
      </div>

      <!-- Editor Section -->
      <div class="section">
        <h3>Edit JSON Data</h3>
        <div class="editor-container">
          <textarea 
            v-model="jsonText" 
            @input="validateJson"
            class="json-textarea"
            placeholder="Paste or import your JSON data here..."
          ></textarea>
          <div v-if="jsonError" class="error-message">
            ❌ {{ jsonError }}
          </div>
        </div>
      </div>

      <!-- Preview Section -->
      <div class="section" v-if="parsedData.length > 0">
        <h3>Data Preview & Image Validation</h3>
        
        <!-- Title Editor -->
        <div v-if="hasTitle" class="title-section">
          <label for="title-input">Title:</label>
          <input 
            id="title-input"
            v-model="jsonTitle" 
            @input="updateJsonText"
            class="title-input"
            placeholder="Enter title..."
          >
        </div>
        
        <div class="preview-container">
          <div 
            v-for="(item, index) in parsedData" 
            :key="index"
            class="data-item"
          >
            <div class="item-info">
              <strong>{{ item.name }}</strong>
              <span class="value">Value: {{ item.value }}</span>
              <span class="ranking">Ranking: {{ item.ranking }}</span>
            </div>
            
            <div class="image-section">
              <div class="image-url">
                <label>Image URL:</label>
                <input 
                  v-model="item.url_image" 
                  @input="validateImage(index)"
                  class="url-input"
                  placeholder="Enter image URL..."
                >
              </div>
              
              <div class="image-preview">
                <div v-if="imageStatus[index] === 'loading'" class="loading">
                  ⏳ Loading...
                </div>
                <div v-else-if="imageStatus[index] === 'error'" class="error">
                  ❌ Image failed to load
                </div>
                <div v-else-if="imageStatus[index] === 'success'" class="success">
                  ✅ <img :src="item.url_image" alt="Preview" class="preview-img">
                </div>
                <div v-else class="no-url">
                  📷 No image URL
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="modal-footer">
      <button @click="exportJson" class="btn-primary" :disabled="!!jsonError">
        💾 Export JSON
      </button>
      <button @click="applyChanges" class="btn-success" :disabled="!!jsonError">
        ✅ Apply Changes
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, watch } from 'vue'

interface DataItem {
  name: string
  value: string
  ranking: number
  url_image: string
}

interface Props {
  currentData: DataItem[]
  animationInProgress: boolean
}

interface Emits {
  'data-updated': [data: DataItem[]]
}

const emit = defineEmits<Emits>()
const props = defineProps<Props>()

const jsonText = ref('')
const jsonError = ref('')
const parsedData = ref<DataItem[]>([])
const jsonTitle = ref('')
const hasTitle = ref(false)
const imageStatus = reactive<Record<number, 'loading' | 'success' | 'error' | 'none'>>({})
const fileInput = ref<HTMLInputElement>()

const handleFileImport = (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (file && file.type === 'application/json') {
    const reader = new FileReader()
    reader.onload = (e) => {
      jsonText.value = e.target?.result as string
      validateJson()
    }
    reader.readAsText(file)
  }
}

const loadSampleData = async () => {
  try {
    const response = await fetch('/sample-data.json')
    const data = await response.json()
    jsonText.value = JSON.stringify(data, null, 2)
    validateJson()
  } catch (error) {
    jsonError.value = 'Failed to load sample data'
  }
}

const validateJson = () => {
  try {
    const parsed = JSON.parse(jsonText.value)
    
    let dataArray: DataItem[]
    
    if (parsed.title && parsed.data && Array.isArray(parsed.data)) {
      dataArray = parsed.data
      jsonTitle.value = parsed.title
      hasTitle.value = true
    } else if (Array.isArray(parsed)) {
      dataArray = parsed
      jsonTitle.value = ''
      hasTitle.value = false
    } else {
      throw new Error('JSON must be an array or an object with title and data fields')
    }
    
    for (let i = 0; i < dataArray.length; i++) {
      const item = dataArray[i]
      if (!item.name || !item.value || typeof item.ranking !== 'number') {
        throw new Error(`Item ${i + 1} missing required fields (name, value, ranking)`)
      }
      if (!item.url_image) {
        item.url_image = ''
      }
    }
    
    parsedData.value = dataArray
    jsonError.value = ''
    
    dataArray.forEach((_, index) => {
      validateImage(index)
    })
    
  } catch (error) {
    jsonError.value = error instanceof Error ? error.message : 'Invalid JSON'
    parsedData.value = []
    jsonTitle.value = ''
    hasTitle.value = false
  }
}

const validateImage = (index: number) => {
  const item = parsedData.value[index]
  if (!item.url_image) {
    imageStatus[index] = 'none'
    return
  }
  
  imageStatus[index] = 'loading'
  
  const img = new Image()
  img.onload = () => {
    imageStatus[index] = 'success'
  }
  img.onerror = () => {
    imageStatus[index] = 'error'
  }
  img.src = item.url_image
}

const exportJson = () => {
  let exportData
  if (hasTitle.value) {
    exportData = {
      title: jsonTitle.value,
      data: parsedData.value
    }
  } else {
    exportData = parsedData.value
  }
  
  const dataStr = JSON.stringify(exportData, null, 2)
  const dataBlob = new Blob([dataStr], { type: 'application/json' })
  const url = URL.createObjectURL(dataBlob)
  const link = document.createElement('a')
  link.href = url
  link.download = 'edited-data.json'
  link.click()
  URL.revokeObjectURL(url)
}

const applyChanges = () => {
  emit('data-updated', parsedData.value)
}

const updateJsonText = () => {
  if (parsedData.value.length > 0) {
    let updateData
    if (hasTitle.value) {
      updateData = {
        title: jsonTitle.value,
        data: parsedData.value
      }
    } else {
      updateData = parsedData.value
    }
    jsonText.value = JSON.stringify(updateData, null, 2)
  }
}

// Initialize editor when component mounts
const initializeEditor = () => {
  if (props.currentData && props.currentData.length > 0) {
    jsonText.value = JSON.stringify(props.currentData, null, 2)
  } else {
    jsonText.value = '[]'
  }
  validateJson()
}

watch(parsedData, (newData) => {
  if (newData.length > 0) {
    let updateData
    if (hasTitle.value) {
      updateData = {
        title: jsonTitle.value,
        data: newData
      }
    } else {
      updateData = newData
    }
    jsonText.value = JSON.stringify(updateData, null, 2)
  }
}, { deep: true })

// Initialize on mount
initializeEditor()

// Watch for changes in currentData
watch(() => props.currentData, () => {
  initializeEditor()
}, { deep: true })
</script>

<style scoped>
.json-editor {
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
  max-width: 1000px;
  width: 90vw;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  padding: 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 12px 12px 0 0;
}

.modal-header h2 {
  margin: 0;
  font-size: 24px;
  font-weight: 600;
}

.modal-body {
  padding: 20px;
  overflow-y: auto;
  flex: 1;
}

.section {
  margin-bottom: 25px;
  padding: 20px;
  border: 1px solid #e1e8ed;
  border-radius: 8px;
  background: #f8f9fa;
}

.section h3 {
  margin: 0 0 15px 0;
  color: #333;
  font-size: 18px;
  font-weight: 600;
}

.import-controls {
  display: flex;
  gap: 15px;
  align-items: center;
  flex-wrap: wrap;
}

.file-input {
  padding: 8px;
  border: 2px solid #ddd;
  border-radius: 6px;
  background: white;
  cursor: pointer;
}

.file-input:focus {
  outline: none;
  border-color: #667eea;
}

.editor-container {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.json-textarea {
  width: 100%;
  min-height: 200px;
  padding: 15px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-family: 'Courier New', monospace;
  font-size: 14px;
  line-height: 1.5;
  resize: vertical;
  background: white;
  color: #000;
}

.json-textarea:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.error-message {
  color: #e74c3c;
  background: #fdf2f2;
  padding: 10px;
  border-radius: 6px;
  margin-top: 10px;
  border-left: 4px solid #e74c3c;
}

.preview-container {
  max-height: 400px;
  overflow-y: auto;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #f8f9fa;
}

.data-item {
  padding: 15px;
  border-bottom: 1px solid #eee;
  display: flex;
  gap: 20px;
}

.data-item:last-child {
  border-bottom: none;
}

.item-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.item-info strong {
  font-size: 16px;
  color: #333;
}

.value, .ranking {
  font-size: 14px;
  color: #666;
}

.image-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.image-url label {
  display: block;
  font-weight: 600;
  margin-bottom: 5px;
  color: #333;
}

.url-input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.url-input:focus {
  outline: none;
  border-color: #667eea;
}

.image-preview {
  min-height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: white;
  padding: 10px;
}

.preview-img {
  max-width: 100px;
  max-height: 50px;
  object-fit: contain;
  margin-left: 10px;
}

.loading, .error, .success, .no-url {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
}

.error {
  color: #e74c3c;
}

.success {
  color: #27ae60;
}

.loading {
  color: #f39c12;
}

.no-url {
  color: #95a5a6;
}

.modal-footer {
  padding: 20px;
  background: #f8f9fa;
  border-top: 1px solid #ddd;
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.btn-primary, .btn-success, .btn-secondary {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.3s ease;
}

.btn-primary {
  background: #3498db;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #2980b9;
}

.btn-success {
  background: #27ae60;
  color: white;
}

.btn-success:hover:not(:disabled) {
  background: #229954;
}

.btn-secondary {
  background: #95a5a6;
  color: white;
}

.btn-secondary:hover {
  background: #7f8c8d;
}

.btn-primary:disabled, .btn-success:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.title-section {
  margin-bottom: 20px;
  padding: 15px;
  background: #f0f8ff;
  border: 1px solid #b3d9ff;
  border-radius: 8px;
}

.title-section label {
  display: block;
  font-weight: 600;
  margin-bottom: 8px;
  color: #333;
  font-size: 14px;
}

.title-input {
  width: 100%;
  padding: 10px;
  border: 2px solid #ddd;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 600;
  background: white;
  color: #333;
}

.title-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}
</style>