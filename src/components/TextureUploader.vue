<script setup lang="ts">
import { ref, defineEmits } from 'vue';

const emit = defineEmits<{
  (e: 'texture-uploaded', textureUrl: string): void
}>();

const fileInput = ref<HTMLInputElement | null>(null);
const errorMessage = ref('');
const isLoading = ref(false);
const previewUrl = ref('');

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (!target.files || target.files.length === 0) return;
  
  const file = target.files[0];
  
  // Check if file is an image
  if (!file.type.startsWith('image/')) {
    errorMessage.value = 'Please upload an image file (PNG, JPG, JPEG, GIF, etc.)';
    return;
  }
  
  // Check file size (max 5MB)
  if (file.size > 5 * 1024 * 1024) {
    errorMessage.value = 'File size must be less than 5MB';
    return;
  }
  
  errorMessage.value = '';
  isLoading.value = true;
  
  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const result = e.target?.result as string;
      previewUrl.value = result;
      emit('texture-uploaded', result);
      isLoading.value = false;
    } catch (error) {
      errorMessage.value = 'Error reading image file';
      isLoading.value = false;
    }
  };
  
  reader.onerror = () => {
    errorMessage.value = 'Error reading file';
    isLoading.value = false;
  };
  
  reader.readAsDataURL(file);
};

const triggerFileInput = () => {
  fileInput.value?.click();
};

const clearTexture = () => {
  previewUrl.value = '';
  emit('texture-uploaded', '');
  if (fileInput.value) {
    fileInput.value.value = '';
  }
};
</script>

<template>
  <div class="texture-uploader">
    <h3>Bar Texture</h3>
    <div class="upload-controls">
      <button class="upload-button" @click="triggerFileInput">
        <span v-if="isLoading">Loading...</span>
        <span v-else>Upload Image</span>
      </button>
      <button v-if="previewUrl" class="clear-button" @click="clearTexture">
        Clear
      </button>
      <input 
        ref="fileInput"
        type="file" 
        accept="image/*" 
        @change="handleFileChange"
        style="display: none"
      >
    </div>
    <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
    <div v-if="previewUrl" class="preview-container">
      <img :src="previewUrl" alt="Texture Preview" class="texture-preview" />
      <p class="preview-label">Texture Preview</p>
    </div>
    <div class="texture-info">
      <p class="format-title">Supported formats:</p>
      <p class="format-details">PNG, JPG, JPEG, GIF (max 5MB)</p>
      <p class="usage-tip">Gambar akan menjadi background utuh pada bar dengan pengulangan vertikal untuk bar tinggi</p>
    </div>
  </div>
</template>

<style scoped>
.texture-uploader {
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

.upload-button, .clear-button {
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

.clear-button {
  background-color: #ff4d4d;
  color: white;
}

.clear-button:hover {
  background-color: #ff3333;
  transform: translateY(-2px);
}

.error-message {
  color: #ff4d4d;
  font-size: 0.8rem;
  margin: 0.5rem 0;
  text-align: center;
}

.preview-container {
  text-align: center;
  margin: 0.5rem 0;
}

.texture-preview {
  max-width: 100px;
  max-height: 100px;
  border-radius: 4px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  object-fit: cover;
}

.preview-label {
  font-size: 0.7rem;
  margin: 0.25rem 0 0 0;
  color: rgba(255, 255, 255, 0.8);
}

.texture-info {
  margin-top: 0.5rem;
  font-size: 0.7rem;
  background: rgba(0, 0, 0, 0.2);
  padding: 0.5rem;
  border-radius: 4px;
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease, padding 0.3s ease, margin 0.3s ease;
}

.texture-uploader:hover .texture-info {
  max-height: 150px;
  padding: 0.5rem;
  margin-top: 0.5rem;
}

.format-title {
  margin: 0 0 0.25rem 0;
  font-weight: 500;
}

.format-details {
  margin: 0 0 0.25rem 0;
  color: rgba(255, 255, 255, 0.8);
}

.usage-tip {
  margin: 0;
  color: rgba(255, 255, 255, 0.6);
  font-style: italic;
}
</style>