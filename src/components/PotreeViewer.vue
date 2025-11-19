<template>
  <div class="potree_container">
    <div id="potree_render_area"></div>
    <div id="potree_sidebar_container"></div>
  </div>
</template>

<script setup lang="ts">
import { onMounted } from 'vue'

// Import CSS dependencies
import '/src/assets/build/potree/potree.css'
import '/src/assets/libs/jquery-ui/jquery-ui.min.css'
import '/src/assets/libs/openlayers3/ol.css'
import '/src/assets/libs/spectrum/spectrum.css'
import '/src/assets/libs/jstree/themes/mixed/style.css'

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
      await loadScript('/src/assets/build/potree/potree.js')
      
      // @ts-ignore
      const Potree = window.Potree
      
      if (!Potree) {
        console.error('Potree not found in window')
        return
      }
      
      // Initialize Potree viewer
      // @ts-ignore
      const viewer = new Potree.Viewer(document.getElementById("potree_render_area"))
      
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
          appearanceMenu.nextSibling.style.display = 'block'
        }
      })
      
      console.log('Potree viewer initialized successfully')
      // Load point cloud
      try {
        // @ts-ignore
        const pointcloud = await Potree.loadPointCloud(
          'https://storage.yandexcloud.net/pointclouds.nova-engineering.pro/mars_melnitsa/cloud.js',
          'pointcloud'
        )
        
        // Add point cloud to scene
        viewer.scene.addPointCloud(pointcloud.pointcloud)
        
        // Fit camera to point cloud
        viewer.fitToScreen()
        
        console.log('Point cloud loaded successfully')
      } catch (error) {
        console.error('Failed to load point cloud:', error)
      }
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
  left: 0px;
  top: 0px;
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
}

#potree_sidebar_container {
  position: absolute;
  z-index: 0;
  width: 350px;
  height: 100%;
  overflow-y: scroll;
  font-size: 85%;
  border-right: 1px solid black;
}
</style>