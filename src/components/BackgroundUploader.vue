<script setup lang="ts">
import { ref, defineEmits } from 'vue';

const emit = defineEmits<{
  (e: 'background-changed', type: 'sky' | 'floor', imageUrl: string): void
}>();

const skyInput = ref<HTMLInputElement | null>(null);
const floorInput = ref<HTMLInputElement | null>(null);
const skyPreview = ref<string>('');
const floorPreview = ref<string>('');

const handleSkyUpload = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];
  
  if (file && file.type.startsWith('image/')) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const imageUrl = e.target?.result as string;
      skyPreview.value = imageUrl;
      emit('background-changed', 'sky', imageUrl);
    };
    reader.readAsDataURL(file);
  }
};

const handleFloorUpload = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];
  
  if (file && file.type.startsWith('image/')) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const imageUrl = e.target?.result as string;
      floorPreview.value = imageUrl;
      emit('background-changed', 'floor', imageUrl);
    };
    reader.readAsDataURL(file);
  }
};

const triggerSkyUpload = () => {
  skyInput.value?.click();
};

const triggerFloorUpload = () => {
  floorInput.value?.click();
};

const resetSky = () => {
  skyPreview.value = '';
  if (skyInput.value) skyInput.value.value = '';
  emit('background-changed', 'sky', '/sky.jpg'); // Reset to default
};

const resetFloor = () => {
  floorPreview.value = '';
  if (floorInput.value) floorInput.value.value = '';
  emit('background-changed', 'floor', '/floor.jpg'); // Reset to default
};
</script>

<template>
  <div class="background-uploader">
    <h3>Custom Background</h3>
    
    <div class="upload-section">
      <div class="upload-item">
        <label>Sky Background:</label>
        <div class="upload-controls">
          <button @click="triggerSkyUpload" class="upload-btn">
            📁 Browse Sky
          </button>
          <button v-if="skyPreview" @click="resetSky" class="reset-btn">
            🔄 Reset
          </button>
        </div>
        <div v-if="skyPreview" class="preview">
          <img :src="skyPreview" alt="Sky Preview" />
        </div>
        <input 
          ref="skyInput"
          type="file" 
          accept="image/*" 
          @change="handleSkyUpload"
          style="display: none;"
        />
      </div>
      
      <div class="upload-item">
        <label>Floor Texture:</label>
        <div class="upload-controls">
          <button @click="triggerFloorUpload" class="upload-btn">
            📁 Browse Floor
          </button>
          <button v-if="floorPreview" @click="resetFloor" class="reset-btn">
            🔄 Reset
          </button>
        </div>
        <div v-if="floorPreview" class="preview">
          <img :src="floorPreview" alt="Floor Preview" />
        </div>
        <input 
          ref="floorInput"
          type="file" 
          accept="image/*" 
          @change="handleFloorUpload"
          style="display: none;"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
.background-uploader {
  background: rgba(0, 0, 0, 0.1);
  padding: 0.75rem 1rem;
  border-radius: 8px;
  min-width: 250px;
  max-width: 100%;
}

h3 {
  margin-top: 0;
  margin-bottom: 0.75rem;
  font-size: 1rem;
  text-align: center;
  font-weight: 500;
}

.upload-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.upload-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

label {
  font-size: 0.9rem;
  font-weight: 500;
  color: #fff;
}

.upload-controls {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.upload-btn {
  padding: 0.5rem 0.75rem;
  border: none;
  border-radius: 4px;
  background: rgba(100, 108, 255, 0.8);
  color: white;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.85rem;
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.upload-btn:hover {
  background: rgba(100, 108, 255, 1);
  transform: translateY(-1px);
}

.reset-btn {
  padding: 0.5rem 0.75rem;
  border: none;
  border-radius: 4px;
  background: rgba(255, 100, 100, 0.8);
  color: white;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.85rem;
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.reset-btn:hover {
  background: rgba(255, 100, 100, 1);
  transform: translateY(-1px);
}

.preview {
  margin-top: 0.5rem;
}

.preview img {
  width: 60px;
  height: 40px;
  object-fit: cover;
  border-radius: 4px;
  border: 2px solid rgba(255, 255, 255, 0.3);
}

@media (max-width: 767px) {
  .background-uploader {
    min-width: 200px;
  }
  
  .upload-controls {
    flex-direction: column;
    align-items: stretch;
  }
  
  .upload-btn, .reset-btn {
    justify-content: center;
  }
}
</style>