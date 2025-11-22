<template>
  <div class="potree_container">
    <div class="url-input-container fixed bottom-4 left-4 bg-white/90 rounded-lg shadow-md p-3 flex gap-2 w-80 max-w-[calc(100vw-2rem)]">
      <input
        v-model="pointCloudUrl"
        placeholder="Вставьте ссылку на облако точек"
        class="url-input w-full px-3 py-2 border border-gray-300 rounded-md text-sm"
      />
      <button
        @click="loadPointCloudFromUrl"
        class="load-button uppercase w-full px-4 py-2 bg-orange-500 hover:bg-orange-600 text-white text-sm font-medium rounded-md disabled:bg-gray-400 disabled:cursor-not-allowed"
        :disabled="isButtonDisabled"
      >
        Посмотреть облако
      </button>
      <button
        @click="copyUrlToClipboard"
        class="copy-button uppercase w-full px-4 py-2 bg-orange-500 hover:bg-orange-600 text-white text-sm font-medium rounded-md disabled:bg-gray-400 disabled:cursor-not-allowed"
        :disabled="isButtonDisabled"
      >
        Скопировать ссылку
      </button>
    </div>
    <div id="potree_render_area"></div>
    <div id="potree_sidebar_container"></div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, computed } from 'vue'

32 | // Declare global objects
declare global {
  interface Window {
    $: any;
    jQuery: any;
    Potree: any;
    loadPointCloudFromUrl: (url: string) => Promise<void>;
  }
}

const pointCloudUrl = ref('')
let viewerInstance: any = null

// URL validation function
const isValidUrl = (urlString: string) => {
  try {
    new URL(urlString);
    return true;
  } catch (err) {
    return false;
  }
};

// Computed property for button disabled state
const isButtonDisabled = computed(() => {
  return !pointCloudUrl.value || !isValidUrl(pointCloudUrl.value);
});

// Store viewer instance when it's created
const setViewerInstance = (viewer: any) => {
  viewerInstance = viewer
}

// Expose function to global scope so it can be called from outside
window.loadPointCloudFromUrl = async (url: string) => {
  if (!url || !viewerInstance) return
  
  try {
    // @ts-ignore
    const pointcloud = await window.Potree.loadPointCloud(
      url,
      'pointcloud'
    )
    
    // Clear existing point clouds
    // Remove all existing point clouds from the scene
    while (viewerInstance.scene.pointclouds.length > 0) {
      const pointcloud = viewerInstance.scene.pointclouds[0];
      viewerInstance.scene.scenePointCloud.remove(pointcloud);
      viewerInstance.scene.pointclouds.splice(0, 1);
    }
    
    // Add new point cloud to scene
    viewerInstance.scene.addPointCloud(pointcloud.pointcloud)
    
    // Update meta tags with point cloud information
    updateMetaTags(pointCloudUrl.value);
    
    // Fit camera to point cloud
    viewerInstance.fitToScreen()
    
    console.log('Point cloud loaded successfully')
  } catch (error) {
    console.error('Failed to load point cloud:', error)
  }
}

const loadPointCloudFromUrl = async () => {
  window.loadPointCloudFromUrl(pointCloudUrl.value)
  
  // Update URL with query parameter
  const url = new URL(window.location.href);
  url.searchParams.set('pointcloud', pointCloudUrl.value);
  window.history.pushState({}, '', url);
}

// Function to update meta tags dynamically
const updateMetaTags = (pointCloudUrl: string) => {
  // Update title
  document.title = `Potree Cloud Viewer - ${pointCloudUrl}`;
  
  // Update description
  const description = `Viewing 3D point cloud: ${pointCloudUrl}`;
  const descriptionMeta = document.querySelector('meta[name="description"]');
  if (descriptionMeta) {
    descriptionMeta.setAttribute('content', description);
  }
  
  // Update Open Graph tags
  const ogTitle = document.querySelector('meta[property="og:title"]');
  if (ogTitle) {
    ogTitle.setAttribute('content', `Potree Cloud Viewer - ${pointCloudUrl}`);
  }
  
  const ogDescription = document.querySelector('meta[property="og:description"]');
  if (ogDescription) {
    ogDescription.setAttribute('content', description);
  }
  
  // Update Twitter tags
  const twitterTitle = document.querySelector('meta[name="twitter:title"]');
  if (twitterTitle) {
    twitterTitle.setAttribute('content', `Potree Cloud Viewer - ${pointCloudUrl}`);
  }
  
  const twitterDescription = document.querySelector('meta[name="twitter:description"]');
  if (twitterDescription) {
    twitterDescription.setAttribute('content', description);
  }
};

const copyUrlToClipboard = () => {
  navigator.clipboard.writeText(window.location.href)
    .then(() => {
      console.log('Ссылка скопирована в буфер обмена');
      // Add visual feedback
      alert('Ссылка скопирована в буфер обмена');
    })
    .catch(err => {
      console.error('Failed to copy URL: ', err);
      // Add error feedback
      alert('Ошибка копирования ссылки в буфер обмена');
    });
}

onMounted(() => {
  // Load Potree dependencies and initialize viewer
  const initPotree = async () => {
    try {
      // Load jQuery and UI dependencies first
      // Load jQuery and UI dependencies first
      // @ts-ignore
      await import('/libs/jquery/jquery-3.1.1.min.js')
      // @ts-ignore
      await import('/libs/jquery-ui/jquery-ui.min.js')
      // @ts-ignore
176 |       await import('/libs/spectrum/spectrum.js')
      
      
      // Load other dependencies
      // @ts-ignore
      await import('/libs/other/BinaryHeap.js')
      // @ts-ignore
      await import('/libs/tween/tween.min.js')
      // @ts-ignore
      await import('/libs/d3/d3.js')
      // @ts-ignore
      await import('/libs/proj4/proj4.js')
      // @ts-ignore
      await import('/libs/openlayers3/ol.js')
      // @ts-ignore
      await import('/libs/i18next/i18next.js')
      // @ts-ignore
192 |       await import('/libs/jstree/jstree.js')
      
      // Load Potree
      // @ts-ignore
196 |       await import('/potree/potree.js')
      // @ts-ignore
      const Potree = window.Potree
      
      if (!Potree) {
        console.error('Potree not found in window')
        return
      }
      
      // Initialize Potree viewer
      // @ts-ignore
      const viewer = new Potree.Viewer(document.getElementById("potree_render_area"))
      
      // Store viewer instance for use in loadPointCloudFromUrl
      setViewerInstance(viewer)
      
      viewer.setEDLEnabled(true)
      viewer.setFOV(60)
      viewer.setPointBudget(1_000_000)
      viewer.loadSettingsFromURL()
      
      viewer.setDescription("")
      
      viewer.loadGUI(() => {
        viewer.setLanguage('en')
        // @ts-ignore
        const appearanceMenu = document.getElementById("menu_appearance")
        if (appearanceMenu && appearanceMenu.nextSibling) {
          // @ts-ignore
          const nextSibling = appearanceMenu.nextSibling;
          if (nextSibling && (nextSibling as HTMLElement).style) {
            (nextSibling as HTMLElement).style.display = 'block'
          }
        }
      })
      
      console.log('Potree viewer initialized successfully')
            
            // Only load default point cloud if none is specified in URL
            const urlParams = new URLSearchParams(window.location.search)
            const defaultPointCloud = urlParams.get('pointcloud')
            
            if (!pointCloudUrl.value && defaultPointCloud) {
              pointCloudUrl.value = defaultPointCloud
              // Load the point cloud from the URL parameter
              window.loadPointCloudFromUrl(pointCloudUrl.value)
            }
            // Note: We're intentionally not loading a default point cloud here
            // to allow the user to start with a clean viewer
    } catch (error) {
      console.error('Failed to initialize Potree:', error)
    }
  }
  
  initPotree()
})
</script>

<style scoped>
@import '/potree/potree.css';
@import '/libs/jquery-ui/jquery-ui.min.css';
@import '/libs/openlayers3/ol.css';
@import '/libs/spectrum/spectrum.css';
@import '/libs/jstree/themes/mixed/style.css';

.potree_container {
  position: absolute;
  width: 100%;
  height: 100%;
  height: -webkit-fill-available;
  left: 0px;
  top: 0px;
}

.url-input-container {
  position: fixed;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  gap: 0.5rem;
  background: rgba(255, 255, 255, 0.9);
  padding: 0.75rem;
  border-radius: 0.5rem;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  width: auto;
  max-width: calc(100vw - 2rem);
  min-width: 25rem;
}

@media (min-width: 768px) {
  .url-input-container {
    flex-direction: row;
    min-width: 35rem;
  }
}

@media (min-width: 768px) and (max-width: 1140px) {
  .url-input-container {
    min-width: 35rem;
  }
  
  .url-input {
    flex: 2;
  }
}

@media (min-width: 1140px) {
  .url-input-container {
    min-width: 40rem;
  }
}

/* Move to bottom for narrow screens */
@media (max-width: 768px) {
  .url-input-container {
    flex-direction: column;
    align-items: stretch;
    gap: 0.5rem;
    padding: 0.75rem;
    background: rgba(255, 255, 255, 0.9);
    border-radius: 0.5rem;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  }
  
  .url-input {
    width: 100%;
    margin-bottom: 0;
  }
  
  .load-button, .copy-button {
    width: 100%;
    margin-bottom: 0;
  }
}

.url-input {
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 0.25rem;
  flex: 1;
  font-size: 0.875rem;
  min-width: 0;
}

.load-button, .copy-button {
  padding: 0.5rem 1rem;
  background: #f97316;
  color: white;
  border: none;
  border-radius: 0.25rem;
  cursor: pointer;
  font-size: 0.875rem;
  text-transform: uppercase;
  white-space: nowrap;
  flex: 1;
  min-width: max-content;
}

.load-button:hover, .copy-button:hover {
  background: #ea580c;
}

.load-button:disabled, .copy-button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}

/* Responsive design for small screens */
@media (max-width: 768px) {
  .url-input-container {
    width: calc(100vw - 2rem);
    padding: 0.5rem;
  }
  
  .url-input {
    padding: 0.5rem;
    font-size: 0.875rem;
  }
  
  .load-button, .copy-button {
    padding: 0.5rem;
    font-size: 0.75rem;
  }
}

@media (max-width: 480px) {
  .url-input-container {
    width: calc(100vw - 2rem);
    padding: 0.5rem;
  }
  
  .url-input {
    padding: 0.5rem;
    font-size: 0.875rem;
  }
  
  .load-button, .copy-button {
    padding: 0.5rem;
    font-size: 0.75rem;
  }
}

#potree_render_area {
  position: absolute;
  top: 0px;
  bottom: 0px;
  left: 0px;
  right: 0px;
  overflow: hidden;
  z-index: 1;
  background-image: url('/potree/resources/images/background.jpg');
  height: 100vh !important;
  width: 100vw !important;
  max-width: 100% !important;
  max-height: 100% !important;
  
  /* Responsive design */
  transition: all 0.3s ease;
}

#potree_render_area > canvas {
  height: 100vh !important;
  width: 100% !important;
  max-width: 100% !important;
  max-height: 100% !important;
}

#potree_sidebar_container {
  position: absolute;
  z-index: 0;
  width: 350px;
  height: 100%;
  overflow-y: scroll;
  font-size: 85%;
  border-right: 1px solid black;
  
  /* Responsive design */
  transition: all 0.3s ease;
}

/* Responsive sidebar for smaller screens */
@media (max-width: 768px) {
  #potree_sidebar_container {
    width: 280px;
  }
}

@media (max-width: 480px) {
  #potree_sidebar_container {
    width: 100%;
    height: 40%;
    bottom: 0;
    top: auto;
    border-right: none;
    border-top: 1px solid black;
  }
  
  #potree_render_area {
    height: 60%;
  }
  
  #potree_render_area > canvas {
    height: 100vh !important;
    width: 100% !important;
  }
}
</style>