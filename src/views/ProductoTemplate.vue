<template>
  <div class="app-container">
    <div class="bg-glow"></div>

    <header class="header">
      <div class="brand-tag">Family Web</div>
      <h1>Generador de Productos <span class="gradient-text">HD</span></h1>
      <p class="subtitle">Sube fotos, remueve el fondo con IA y genera las plantillas listas para publicar.</p>
    </header>

    <main class="main-card">
      <div class="options-bar">
        <label class="toggle-card">
          <input type="checkbox" v-model="removeBackground" />
          <span class="slider"></span>
          <span class="toggle-info">
            <span class="toggle-title">Remover fondo (IA)</span>
            <span class="toggle-sub">Elimina el fondo blanco o ruidoso del producto</span>
          </span>
        </label>
      </div>

      <input 
        type="file" 
        accept="image/*" 
        multiple 
        @change="onFilesChange" 
        id="file-input"
        :disabled="loading"
      />
      
      <label for="file-input" class="upload-zone" :class="{ 'disabled': loading }">
        <div class="icon-pulse">
          <svg xmlns="http://www.w3.org/2000/svg" width="36" height="36" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
            <polyline points="17 8 12 3 7 8"/>
            <line x1="12" y1="3" x2="12" y2="15"/>
          </svg>
        </div>
        <div class="upload-text-group">
          <span class="upload-title">Arrastra tus fotos o <span class="highlight">explora</span></span>
          <span class="upload-hint">Puedes cargar múltiples imágenes a la vez (.jpg, .png, .webp)</span>
        </div>
      </label>
    </main>

    <transition name="fade">
      <div v-if="loading" class="status-card">
        <div class="spinner-wrapper">
          <div class="spinner"></div>
        </div>
        <div class="status-info">
          <div class="status-header">
            <span class="status-title">Procesando lote</span>
            <span class="status-counter">{{ progress.current }} / {{ progress.total }}</span>
          </div>
          <p class="status-step">{{ currentStepText }}</p>
        </div>
      </div>
    </transition>

    <section v-if="results.length > 0" class="results-section">
      <div class="results-header">
        <h2>Listos para descargar</h2>
        <span class="count-pill">{{ results.length }} creadas</span>
      </div>
      
      <div class="cards-grid">
        <article v-for="(item, index) in results" :key="index" class="product-card">
          <div class="card-media">
            <img :src="item.dataUrl" :alt="item.code" class="preview-img" />
            <span class="quality-badge">2K HD</span>
          </div>
          
          <div class="card-body">
            <div class="code-info">
              <span class="meta-label">Código</span>
              <span class="code-badge">{{ item.code }}</span>
            </div>

            <button class="btn-download" @click="downloadSingleImage(item)">
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
                <polyline points="7 10 12 15 17 10"/>
                <line x1="12" y1="15" x2="12" y2="3"/>
              </svg>
              Descargar Imagen
            </button>
          </div>
        </article>
      </div>
    </section>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { removeBackground as removeBg } from '@imgly/background-removal'
import templateUrl from '@/assets/plantillablanco.png'

const SCALE_FACTOR = 2
const BASE_PRODUCT_AREA = {
  x: 100,
  y: 160,
  width: 880,
  height: 720,
}

const loading = ref(false)
const removeBackground = ref(true)
const currentStepText = ref('')
const progress = ref({ current: 0, total: 0 })
const results = ref([])

function loadImage(src) {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.crossOrigin = 'anonymous'
    img.onload = () => resolve(img)
    img.onerror = reject
    img.src = src
  })
}

function getProductCode(fileName) {
  return fileName.replace(/\.[^/.]+$/, '')
}

function getContainParams(img, area) {
  const imgRatio = img.width / img.height
  const areaRatio = area.width / area.height

  let drawWidth, drawHeight

  if (imgRatio > areaRatio) {
    drawWidth = area.width
    drawHeight = drawWidth / imgRatio
  } else {
    drawHeight = area.height
    drawWidth = drawHeight * imgRatio
  }

  const offsetX = (area.width - drawWidth) / 2
  const offsetY = (area.height - drawHeight) / 2

  return { drawWidth, drawHeight, offsetX, offsetY }
}

async function processSingleFile(file, templateImg) {
  const productCode = getProductCode(file.name)
  let imageSource = URL.createObjectURL(file)

  if (removeBackground.value) {
    currentStepText.value = `Removiendo fondo de "${file.name}"...`
    const blobClean = await removeBg(file)
    imageSource = URL.createObjectURL(blobClean)
  }

  currentStepText.value = `Montando imagen de "${file.name}"...`
  const productImg = await loadImage(imageSource)

  const canvas = document.createElement('canvas')
  const scale = SCALE_FACTOR

  canvas.width = templateImg.width * scale
  canvas.height = templateImg.height * scale

  const ctx = canvas.getContext('2d')
  ctx.imageSmoothingEnabled = true
  ctx.imageSmoothingQuality = 'high'
  ctx.scale(scale, scale)

  // 1. Plantilla
  ctx.drawImage(templateImg, 0, 0, templateImg.width, templateImg.height)

  // 2. Producto
  const { drawWidth, drawHeight, offsetX, offsetY } = getContainParams(productImg, BASE_PRODUCT_AREA)
  ctx.drawImage(
    productImg,
    BASE_PRODUCT_AREA.x + offsetX,
    BASE_PRODUCT_AREA.y + offsetY,
    drawWidth,
    drawHeight
  )

  // 3. Código Vertical
  ctx.save()
  ctx.translate(45, 270)
  ctx.rotate(-Math.PI / 2)
  ctx.font = 'bold 28px Arial'
  ctx.fillStyle = '#1a1a1a'
  ctx.textAlign = 'left'
  ctx.fillText(productCode, 0, 0)
  ctx.restore()

  return {
    code: productCode,
    fileName: `producto-${productCode}.png`,
    dataUrl: canvas.toDataURL('image/png', 1.0)
  }
}

async function onFilesChange(event) {
  const files = Array.from(event.target.files)
  if (!files.length) return

  loading.value = true
  results.value = []
  progress.value = { current: 0, total: files.length }

  try {
    const templateImg = await loadImage(templateUrl)

    for (let i = 0; i < files.length; i++) {
      progress.value.current = i + 1
      const processed = await processSingleFile(files[i], templateImg)
      results.value.push(processed)
    }
  } catch (err) {
    console.error('Error procesando imágenes:', err)
    alert('Ocurrió un error al procesar las imágenes.')
  } finally {
    loading.value = false
    currentStepText.value = ''
  }
}

function downloadSingleImage(item) {
  const link = document.createElement('a')
  link.download = item.fileName
  link.href = item.dataUrl
  link.click()
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;600;700;800&display=swap');

/* Variables Locales de Color */
.app-container {
  --primary: #ef4444;
  --primary-hover: #dc2626;
  --primary-light: #fef2f2;
  --bg-app: #f8fafc;
  --card-bg: #ffffff;
  --text-main: #0f172a;
  --text-muted: #64748b;
  --border-color: #e2e8f0;
  
  position: relative;
  max-width: 1140px;
  margin: 0 auto;
  padding: 3rem 1.5rem;
  font-family: 'Plus Jakarta Sans', system-ui, -apple-system, sans-serif;
  color: var(--text-main);
  min-height: 100vh;
}

/* Fondo decorativo (Glow Effect) */
.bg-glow {
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 600px;
  height: 250px;
  background: radial-gradient(circle, rgba(239, 68, 68, 0.12) 0%, rgba(255, 255, 255, 0) 70%);
  z-index: -1;
  pointer-events: none;
}

/* Header */
.header {
  text-align: center;
  margin-bottom: 3rem;
}

.brand-tag {
  display: inline-block;
  background: rgba(239, 68, 68, 0.08);
  color: var(--primary);
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  padding: 0.4rem 0.8rem;
  border-radius: 99px;
  margin-bottom: 0.75rem;
}

.header h1 {
  font-size: 2.5rem;
  font-weight: 800;
  letter-spacing: -0.03em;
  color: var(--text-main);
  margin: 0 0 0.5rem 0;
}

.gradient-text {
  background: linear-gradient(135deg, #ef4444 0%, #b91c1c 100%);
  -webkit-background-clip: text;
  background-clip: text;
  -webkit-text-fill-color: transparent;
}

.subtitle {
  color: var(--text-muted);
  font-size: 1.05rem;
  max-width: 550px;
  margin: 0 auto;
}

/* Tarjeta Principal */
.main-card {
  background: var(--card-bg);
  border-radius: 20px;
  border: 1px solid var(--border-color);
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.03), 0 8px 10px -6px rgba(0, 0, 0, 0.01);
  padding: 1.5rem;
  transition: all 0.3s ease;
}

/* Toggle Option Bar */
.options-bar {
  margin-bottom: 1.25rem;
}

.toggle-card {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: var(--bg-app);
  padding: 0.85rem 1.25rem;
  border-radius: 12px;
  cursor: pointer;
  border: 1px solid transparent;
  transition: all 0.2s ease;
}

.toggle-card:hover {
  border-color: #cbd5e1;
}

.toggle-card input {
  display: none;
}

.slider {
  position: relative;
  width: 46px;
  height: 26px;
  background-color: #cbd5e1;
  border-radius: 99px;
  transition: 0.25s ease;
  flex-shrink: 0;
}

.slider::before {
  content: "";
  position: absolute;
  height: 20px;
  width: 20px;
  left: 3px;
  bottom: 3px;
  background-color: white;
  border-radius: 50%;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  transition: 0.25s ease;
}

input:checked + .slider {
  background-color: var(--primary);
}

input:checked + .slider::before {
  transform: translateX(20px);
}

.toggle-info {
  display: flex;
  flex-direction: column;
}

.toggle-title {
  font-weight: 700;
  font-size: 0.95rem;
  color: var(--text-main);
}

.toggle-sub {
  font-size: 0.8rem;
  color: var(--text-muted);
}

/* Zone de Carga Drag & Drop */
input[type="file"] {
  display: none;
}

.upload-zone {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border: 2px dashed #cbd5e1;
  border-radius: 14px;
  padding: 3.5rem 2rem;
  cursor: pointer;
  background-color: #fafafa;
  transition: all 0.25s ease;
}

.upload-zone:hover:not(.disabled) {
  border-color: var(--primary);
  background-color: var(--primary-light);
}

.upload-zone:hover:not(.disabled) .icon-pulse {
  transform: translateY(-4px);
  color: var(--primary);
  background: white;
  box-shadow: 0 10px 15px -3px rgba(239, 68, 68, 0.2);
}

.upload-zone.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.icon-pulse {
  background: #ffffff;
  color: #64748b;
  padding: 1.1rem;
  border-radius: 16px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05), 0 2px 4px -2px rgba(0, 0, 0, 0.05);
  margin-bottom: 1.25rem;
  transition: all 0.25s ease;
}

.upload-text-group {
  text-align: center;
}

.upload-title {
  display: block;
  font-size: 1.1rem;
  font-weight: 700;
  color: var(--text-main);
  margin-bottom: 0.35rem;
}

.upload-title .highlight {
  color: var(--primary);
}

.upload-hint {
  font-size: 0.875rem;
  color: var(--text-muted);
}

/* Card de Estado de Carga */
.status-card {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  background: var(--card-bg);
  border: 1px solid var(--border-color);
  padding: 1.25rem 1.5rem;
  border-radius: 16px;
  margin-top: 1.5rem;
  box-shadow: 0 10px 15px -3px rgba(0,0,0,0.03);
}

.spinner-wrapper {
  position: relative;
}

.spinner {
  width: 32px;
  height: 32px;
  border: 3px solid var(--primary-light);
  border-top-color: var(--primary);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.status-info {
  flex-grow: 1;
}

.status-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.2rem;
}

.status-title {
  font-weight: 700;
  font-size: 0.95rem;
}

.status-counter {
  font-size: 0.85rem;
  font-weight: 700;
  color: var(--primary);
  background: var(--primary-light);
  padding: 0.15rem 0.6rem;
  border-radius: 99px;
}

.status-step {
  margin: 0;
  font-size: 0.85rem;
  color: var(--text-muted);
}

/* Resultados */
.results-section {
  margin-top: 3.5rem;
}

.results-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1.5rem;
}

.results-header h2 {
  font-size: 1.35rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  margin: 0;
}

.count-pill {
  background: #0f172a;
  color: white;
  font-size: 0.8rem;
  font-weight: 700;
  padding: 0.3rem 0.8rem;
  border-radius: 99px;
}

/* Grid & Cards de Producto */
.cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1.5rem;
}

.product-card {
  background: var(--card-bg);
  border-radius: 18px;
  border: 1px solid var(--border-color);
  overflow: hidden;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
}

.product-card:hover {
  transform: translateY(-6px);
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.08);
  border-color: #cbd5e1;
}

.card-media {
  position: relative;
  background-color: #f1f5f9;
  aspect-ratio: 1;
  overflow: hidden;
}

.preview-img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.quality-badge {
  position: absolute;
  top: 12px;
  right: 12px;
  background: rgba(15, 23, 42, 0.75);
  backdrop-filter: blur(8px);
  color: #ffffff;
  font-size: 0.65rem;
  font-weight: 800;
  letter-spacing: 0.05em;
  padding: 0.25rem 0.5rem;
  border-radius: 6px;
}

.card-body {
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}

.code-info {
  display: flex;
  flex-direction: column;
  margin-bottom: 1.25rem;
}

.meta-label {
  font-size: 0.7rem;
  font-weight: 700;
  text-transform: uppercase;
  color: #94a3b8;
  letter-spacing: 0.05em;
  margin-bottom: 0.2rem;
}

.code-badge {
  font-size: 1.05rem;
  font-weight: 800;
  color: var(--text-main);
  word-break: break-all;
}

.btn-download {
  margin-top: auto;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  background-color: var(--primary);
  color: #ffffff;
  border: none;
  border-radius: 10px;
  padding: 0.8rem 1rem;
  font-weight: 700;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 6px -1px rgba(239, 68, 68, 0.2);
}

.btn-download:hover {
  background-color: var(--primary-hover);
  box-shadow: 0 8px 12px -2px rgba(239, 68, 68, 0.3);
}

.btn-download:active {
  transform: scale(0.98);
}

/* Transiciones de Animación */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
</style>