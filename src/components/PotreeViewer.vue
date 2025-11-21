<template>
  <div class="potree_container">
    <div class="url-input-container fixed bottom-4 left-4 bg-white/90 rounded-lg shadow-md p-3 flex flex-col gap-2 w-80 max-w-[calc(100vw-2rem)]">
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

// Declare global function
declare global {
  interface Window {
    loadPointCloudFromUrl: (url: string) => Promise<void>;
  }
}

// Import CSS dependencies
import '/src/assets/potree/potree.css'
import '/src/assets/libs/jquery-ui/jquery-ui.min.css'
import '/src/assets/libs/openlayers3/ol.css'
import '/src/assets/libs/spectrum/spectrum.css'
import '/src/assets/libs/jstree/themes/mixed/style.css'

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
      // Create script elements for dependencies
      const loadScript = (src: string) => {
        return new Promise((resolve, reject) => {
          const script = document.createElement('script')
          script.src = src
          script.onload = resolve
          script.onerror = reject
          document.head.appendChild(script)
        })
      }
      
      // Load jQuery and UI dependencies first
      await loadScript('/src/assets/libs/jquery/jquery-3.1.1.min.js')
      await loadScript('/src/assets/libs/jquery-ui/jquery-ui.min.js')
      await loadScript('/src/assets/libs/spectrum/spectrum.js')
      
      // Load other dependencies
      await loadScript('/src/assets/libs/other/BinaryHeap.js')
      await loadScript('/src/assets/libs/tween/tween.min.js')
      await loadScript('/src/assets/libs/d3/d3.js')
      await loadScript('/src/assets/libs/proj4/proj4.js')
      await loadScript('/src/assets/libs/openlayers3/ol.js')
      await loadScript('/src/assets/libs/i18next/i18next.js')
      await loadScript('/src/assets/libs/jstree/jstree.js')
      
      // Load Potree
      await loadScript('/src/assets/potree/potree.js')
      
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
.potree_container {
  position: absolute;
  width: 100%;
  height: 100%;
  height: -webkit-fill-available;
  left: 0px;
  top: 0px;
}

.url-input-container {
  position: absolute;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10; /* Reduced z-index to prevent covering menu toggle */
  display: flex;
  gap: 10px;
  background: rgba(255, 255, 255, 0.9);
  padding: 10px;
  border-radius: 5px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.0);
  
  /* Responsive design for smaller screens */
  flex-wrap: wrap;
  justify-content: center;
  width: 90%;
  max-width: 600px;
}

/* Move to bottom for narrow screens */
@media (max-width: 480px) {
  .url-input-container {
    top: auto;
    bottom: 10px;
    left: 10px;
    transform: none;
    width: calc(100% - 20px);
    max-width: none;
    padding: 5px;
    background: rgba(255, 255, 255, 0.9);
    border-radius: 6px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    flex-direction: column;
  }
  
  .url-input {
    width: 100%;
    margin-bottom: 5px;
  }
  
  .load-button, .copy-button {
    width: 100%;
    margin-bottom: 3px;
  }
}

.url-input {
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  min-width: 200px;
  flex: 1;
  
  /* Responsive design */
  min-width: 150px;
}

.load-button, .copy-button {
  padding: 8px 16px;
  background: orange;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  text-transform: uppercase;
  white-space: nowrap;
  
  /* Responsive design */
  flex: 1;
  min-width: 120px;
}

.load-button:hover, .copy-button:hover {
  background: #ff8c00;
}

.load-button:disabled, .copy-button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}

/* Responsive design for small screens */
@media (max-width: 768px) {
  .url-input-container {
    top: 10px;
    width: 95%;
    padding: 8px;
  }
  
  .url-input {
    padding: 6px;
    min-width: 120px;
    font-size: 14px;
  }
  
  .load-button, .copy-button {
    padding: 6px 12px;
    font-size: 12px;
    min-width: 100px;
  }
}

@media (max-width: 480px) {
  .url-input-container {
    flex-direction: column;
    align-items: center;
    gap: 5px;
    padding: 6px;
  }
  
  .url-input {
    width: 100%;
    min-width: auto;
  }
  
  .load-button, .copy-button {
    width: 100%;
    min-width: auto;
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
  background-image: url('/src/assets/build/potree/resources/images/background.jpg');
  
  /* Responsive design */
  transition: all 0.3s ease;
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
}
</style>