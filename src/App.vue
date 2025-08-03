<script setup lang="ts">
import { ref, onMounted } from 'vue'
import RankingChart from './components/RankingChart.vue'
import BackgroundUploader from './components/BackgroundUploader.vue'
import FileUploader from './components/FileUploader.vue'
import TextureUploader from './components/TextureUploader.vue'
import JsonEditor from './components/JsonEditor.vue'

const jsonData = ref<any>(null)
const customBackgrounds = ref({
  sky: '/sky.jpg',
  floor: '/floor.jpg'
})
const barTexture = ref<string>('')
const animationInProgress = ref(false)
const showJsonEditor = ref(false)

const handleFileUpload = (data: any) => {
  jsonData.value = data
}

const handleBackgroundChange = (type: 'sky' | 'floor', imageUrl: string) => {
  customBackgrounds.value[type] = imageUrl
}

const handleTextureUpload = (textureUrl: string) => {
  barTexture.value = textureUrl
}

const handleDataUpdate = (newData: any[]) => {
  jsonData.value = {
    title: jsonData.value?.title || "Updated Data",
    data: newData
  }
}

const handleOpenJsonEditor = () => {
  showJsonEditor.value = true
}

const handleCloseJsonEditor = () => {
  showJsonEditor.value = false
}

// Sample data for initial display
onMounted(() => {
  jsonData.value = {
    title: "Top 10 Countries by Population",
    data: [
      { name: "China", ranking: 10, value: "1444.2", url_image: "https://flagcdn.com/w320/cn.png" },
      { name: "India", ranking: 9, value: "1393.4", url_image: "https://flagcdn.com/w320/in.png" },
      { name: "USA", ranking: 8, value: "332.9", url_image: "https://flagcdn.com/w320/us.png" },
      { name: "Indonesia", ranking: 7, value: "276.4", url_image: "https://flagcdn.com/w320/id.png" },
      { name: "Pakistan", ranking: 6, value: "225.2", url_image: "https://flagcdn.com/w320/pk.png" },
      { name: "Brazil", ranking: 5, value: "213.9", url_image: "https://flagcdn.com/w320/br.png" },
      { name: "Nigeria", ranking: 4, value: "211.4", url_image: "https://flagcdn.com/w320/ng.png" },
      { name: "Bangladesh", ranking: 3, value: "166.3", url_image: "https://flagcdn.com/w320/bd.png" },
      { name: "Russia", ranking: 2, value: "145.9", url_image: "https://flagcdn.com/w320/ru.png" },
      { name: "Mexico", ranking: 1, value: "130.3", url_image: "https://flagcdn.com/w320/mx.png" }
    ]
  }
})
</script>

<template>
  <div class="app-container">
    <h1 class="app-title">3D Ranking Animation</h1>
    
    <div class="chart-wrapper">
      <div class="chart-container">
        <RankingChart 
          v-if="jsonData" 
          :data="jsonData" 
          :custom-backgrounds="customBackgrounds"
          :bar-texture="barTexture"
          @animation-status="(status) => animationInProgress = status"
        />
        <div v-else class="loading">Loading chart...</div>
      </div>
    </div>
    
    <div class="controls">
      <FileUploader 
        :current-data="jsonData?.data || []"
        :animation-in-progress="animationInProgress"
        @file-uploaded="handleFileUpload" 
        @open-json-editor="handleOpenJsonEditor"
      />
      <TextureUploader @texture-uploaded="handleTextureUpload" />
      <BackgroundUploader @background-changed="handleBackgroundChange" />
    </div>
    
    <!-- JSON Editor Modal -->
    <div v-if="showJsonEditor" class="modal-overlay" @click="handleCloseJsonEditor">
      <div class="modal-content" @click.stop>
        <JsonEditor 
          :current-data="jsonData?.data || []"
          :animation-in-progress="animationInProgress"
          @data-updated="handleDataUpdate"
        />
        <button class="close-button" @click="handleCloseJsonEditor">Close</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  height: 100vh;
  padding: 1rem;
  box-sizing: border-box;
  max-width: 100vw;
  margin: 0 auto;
}

.app-title {
  margin-bottom: 1rem;
  font-size: 1.8rem;
  text-align: center;
}

.chart-wrapper {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 1rem;
  margin-bottom: 1.5rem;
}

.controls {
  display: flex;
  flex-direction: row;
  justify-content: center;
  gap: 1rem;
  width: 100%;
  margin-top: 1rem;
}

.chart-container {
  width: 100%;
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  /* Ensure 16:9 aspect ratio for landscape mode */
  aspect-ratio: 16/9;
  max-width: 1200px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
}

.loading {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  font-size: 1.2rem;
  color: #888;
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background: #1a1a1a;
  border-radius: 12px;
  padding: 1.5rem;
  max-width: 90vw;
  max-height: 90vh;
  overflow-y: auto;
  position: relative;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}

.close-button {
  position: absolute;
  top: 1rem;
  right: 1rem;
  background-color: #ff4d4d;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 0.5rem 1rem;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
}

.close-button:hover {
  background-color: #ff3333;
  transform: translateY(-2px);
}

@media (max-width: 767px) {
  .controls {
    flex-direction: column;
    align-items: center;
    gap: 0.5rem;
  }
  
  .chart-container {
    aspect-ratio: 16/9;
    max-height: calc(100vh - 250px);
  }
  
  .app-title {
    font-size: 1.5rem;
    margin-bottom: 0.5rem;
  }
  
  .modal-content {
    max-width: 95vw;
    max-height: 95vh;
    padding: 1rem;
  }
}
</style>
