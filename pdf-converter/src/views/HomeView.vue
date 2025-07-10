<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import jsPDF from 'jspdf'
import html2canvas from 'html2canvas'
import { saveAs } from 'file-saver'
import * as pdfjsLib from 'pdfjs-dist/legacy/build/pdf'
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.js'
import * as ffmpegModule from '@ffmpeg/ffmpeg'
let createFFmpeg: any, fetchFile: any, ffmpeg: any;
const selectedFile = ref<File | null>(null)
const targetFormat = ref('pdf')
const isConverting = ref(false)
const previewContent = ref('')
const dragOver = ref(false)
const videoFile = ref<File | null>(null)
const videoUrl = ref<string | null>(null)
const outputUrl = ref<string | null>(null)
const isVideoConverting = ref(false)
const ffmpegLoaded = ref(false)
const activeTab = ref('file') // 'file' or 'video'

const supportedFormats = [
  { value: 'pdf', label: 'PDF', icon: '📄' },
  { value: 'png', label: 'PNG', icon: '🖼️' },
  { value: 'jpeg', label: 'JPEG', icon: '🖼️' },
  { value: 'svg', label: 'SVG', icon: '🎨' },
  { value: 'webp', label: 'WEBP', icon: '🖼️' },
  { value: 'gif', label: 'GIF', icon: '🖼️' },
  { value: 'bmp', label: 'BMP', icon: '🖼️' },
  { value: 'ico', label: 'ICO', icon: '🖼️' }
]

const isPdf = computed(() => selectedFile.value && selectedFile.value.type === 'application/pdf')

const availableFormats = computed(() => {
  if (isPdf.value) {
    // PDF yüklenirse sadece görsel formatlar aktif
    return supportedFormats.filter(f => ['png', 'jpeg', 'svg', 'webp', 'gif', 'bmp', 'ico'].includes(f.value))
  } else {
    // Diğer dosyalar için tüm formatlar
    return supportedFormats
  }
})

const categories = [
  { name: 'Görüntü', key: 'Görüntü' },
  { name: 'Belge', key: 'Belge' },
  { name: 'EBook', key: 'EBook' },
  { name: 'Yazı tipi', key: 'Yazı tipi' },
  { name: 'Vektör', key: 'Vektör' },
  { name: 'CAD', key: 'CAD' },
]

const showUnsupportedWarning = ref(false)
const unsupportedLabel = ref('')

const handleFormatSelect = (format: any) => {
  if (!format.supported) {
    showUnsupportedWarning.value = true
    unsupportedLabel.value = format.label
    return
  }
  targetFormat.value = format.value
  showUnsupportedWarning.value = false
}

const handleFileSelect = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    selectedFile.value = target.files[0]
    previewFile(target.files[0])
    // PDF yüklendiyse formatı otomatik PNG yap
    if (target.files[0].type === 'application/pdf') {
      targetFormat.value = 'png'
    }
  }
}

const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  dragOver.value = false
  
  if (event.dataTransfer?.files && event.dataTransfer.files[0]) {
    selectedFile.value = event.dataTransfer.files[0]
    previewFile(event.dataTransfer.files[0])
    if (event.dataTransfer.files[0].type === 'application/pdf') {
      targetFormat.value = 'png'
    }
  }
}

const handleDragOver = (event: DragEvent) => {
  event.preventDefault()
  dragOver.value = true
}

const handleDragLeave = () => {
  dragOver.value = false
}

const previewFile = (file: File) => {
  if (file.type.startsWith('image/')) {
    const reader = new FileReader()
    reader.onload = (e) => {
      previewContent.value = `<img src="${e.target?.result}" style="max-width: 100%; height: auto;" />`
    }
    reader.readAsDataURL(file)
  } else if (file.type === 'application/pdf') {
    // PDF için ilk sayfanın önizlemesini göster
    renderPdfPreview(file)
  } else if (file.type === 'text/html') {
    const reader = new FileReader()
    reader.onload = (e) => {
      previewContent.value = e.target?.result as string
    }
    reader.readAsText(file)
  } else if (file.type === 'text/plain') {
    const reader = new FileReader()
    reader.onload = (e) => {
      previewContent.value = `<pre style="white-space: pre-wrap; font-family: monospace;">${e.target?.result}</pre>`
    }
    reader.readAsText(file)
  }
}

const renderPdfPreview = async (file: File) => {
  const arrayBuffer = await file.arrayBuffer()
  const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise
  const page = await pdf.getPage(1)
  const viewport = page.getViewport({ scale: 1 })
  const canvas = document.createElement('canvas')
  const context = canvas.getContext('2d')!
  canvas.width = viewport.width
  canvas.height = viewport.height
  await page.render({ canvasContext: context, viewport }).promise
  previewContent.value = `<img src="${canvas.toDataURL()}" style="max-width: 100%; height: auto;" />`
}

const convertFile = async () => {
  if (!selectedFile.value) return
  isConverting.value = true
  try {
    if (isPdf.value) {
      await convertPdfToImage()
    } else {
      switch (targetFormat.value) {
        case 'pdf':
          await convertToPdf()
          break
        case 'svg':
          await convertToSvg()
          break
        case 'png':
          await convertToPng()
          break
        case 'jpeg':
          await convertToJpg()
          break
      }
    }
  } catch (error) {
    console.error('Dönüştürme hatası:', error)
    alert('Dönüştürme sırasında bir hata oluştu!')
  } finally {
    isConverting.value = false
  }
}

const convertPdfToImage = async () => {
  if (!selectedFile.value) return
  const arrayBuffer = await selectedFile.value.arrayBuffer()
  const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise
  const page = await pdf.getPage(1)
  const viewport = page.getViewport({ scale: 2 })
  const canvas = document.createElement('canvas')
  const context = canvas.getContext('2d')!
  canvas.width = viewport.width
  canvas.height = viewport.height
  await page.render({ canvasContext: context, viewport }).promise
  const fileName = selectedFile.value.name.replace(/\.[^/.]+$/, '')
  switch (targetFormat.value) {
    case 'png':
      canvas.toBlob(blob => blob && saveAs(blob, `${fileName}.png`), 'image/png')
      break
    case 'jpeg':
      canvas.toBlob(blob => blob && saveAs(blob, `${fileName}.jpg`), 'image/jpeg', 0.95)
      break
    case 'webp':
      canvas.toBlob(blob => blob && saveAs(blob, `${fileName}.webp`), 'image/webp')
      break
    case 'gif':
      // Tarayıcıda doğrudan canvas'tan GIF üretmek zordur, ek kütüphane gerekir
      alert('GIF formatı için ek kütüphane gerekir. Şu an desteklenmiyor.')
      break
    case 'bmp':
      // Tarayıcıda doğrudan canvas'tan BMP üretmek zordur, ek kütüphane gerekir
      alert('BMP formatı için ek kütüphane gerekir. Şu an desteklenmiyor.')
      break
    case 'ico':
      // Tarayıcıda doğrudan canvas'tan ICO üretmek zordur, ek kütüphane gerekir
      alert('ICO formatı için ek kütüphane gerekir. Şu an desteklenmiyor.')
      break
    case 'svg':
      // Canvas'tan SVG'ye doğrudan dönüştürme yoktur
      alert('SVG formatı için doğrudan dönüşüm desteklenmiyor.')
      break
  }
}

const convertToPdf = async () => {
  const pdf = new jsPDF()
  
  if (selectedFile.value?.type.startsWith('image/')) {
    const img = new Image()
    img.src = URL.createObjectURL(selectedFile.value)
    
    await new Promise((resolve) => {
      img.onload = () => {
        const canvas = document.createElement('canvas')
        const ctx = canvas.getContext('2d')!
        canvas.width = img.width
        canvas.height = img.height
        ctx.drawImage(img, 0, 0)
        
        const imgData = canvas.toDataURL('image/jpeg')
        const pdfWidth = pdf.internal.pageSize.getWidth()
        const pdfHeight = (img.height * pdfWidth) / img.width
        
        pdf.addImage(imgData, 'JPEG', 0, 0, pdfWidth, pdfHeight)
        pdf.save(`${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.pdf`)
        resolve(true)
      }
    })
  } else {
    // HTML veya metin için
    const element = document.createElement('div')
    element.innerHTML = previewContent.value
    element.style.width = '800px'
    element.style.padding = '20px'
    element.style.fontFamily = 'Arial, sans-serif'
    element.style.fontSize = '12px'
    element.style.lineHeight = '1.5'
    
    document.body.appendChild(element)
    
    const canvas = await html2canvas(element, {
      width: 800,
      height: element.scrollHeight,
      scale: 2
    })
    
    document.body.removeChild(element)
    
    const imgData = canvas.toDataURL('image/jpeg')
    const pdfWidth = pdf.internal.pageSize.getWidth()
    const pdfHeight = (canvas.height * pdfWidth) / canvas.width
    
    pdf.addImage(imgData, 'JPEG', 0, 0, pdfWidth, pdfHeight)
    pdf.save(`${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.pdf`)
  }
}

const convertToSvg = async () => {
  if (selectedFile.value?.type.startsWith('image/')) {
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')!
    const img = new Image()
    
    img.onload = () => {
      canvas.width = img.width
      canvas.height = img.height
      ctx.drawImage(img, 0, 0)
      
      // Canvas'ı SVG'ye dönüştür
      const svgData = canvas.toDataURL('image/svg+xml')
      const link = document.createElement('a')
      link.href = svgData
      link.download = `${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.svg`
      link.click()
    }
    
    img.src = URL.createObjectURL(selectedFile.value)
  } else {
    // HTML'i SVG'ye dönüştür
    const element = document.createElement('div')
    element.innerHTML = previewContent.value
    element.style.width = '800px'
    element.style.padding = '20px'
    
    document.body.appendChild(element)
    
    const canvas = await html2canvas(element, {
      width: 800,
      height: element.scrollHeight,
      scale: 2
    })
    
    document.body.removeChild(element)
    
    const svgData = canvas.toDataURL('image/svg+xml')
    const link = document.createElement('a')
    link.href = svgData
    link.download = `${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.svg`
    link.click()
  }
}

const convertToPng = async () => {
  if (selectedFile.value?.type.startsWith('image/')) {
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')!
    const img = new Image()
    
    img.onload = () => {
      canvas.width = img.width
      canvas.height = img.height
      ctx.drawImage(img, 0, 0)
      
      canvas.toBlob((blob) => {
        if (blob) {
          saveAs(blob, `${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.png`)
        }
      })
    }
    
    img.src = URL.createObjectURL(selectedFile.value)
  } else {
    const element = document.createElement('div')
    element.innerHTML = previewContent.value
    element.style.width = '800px'
    element.style.padding = '20px'
    
    document.body.appendChild(element)
    
    const canvas = await html2canvas(element, {
      width: 800,
      height: element.scrollHeight,
      scale: 2
    })
    
    document.body.removeChild(element)
    
    canvas.toBlob((blob) => {
      if (blob) {
        saveAs(blob, `${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.png`)
      }
    })
  }
}

const convertToJpg = async () => {
  if (selectedFile.value?.type.startsWith('image/')) {
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')!
    const img = new Image()
    
    img.onload = () => {
      canvas.width = img.width
      canvas.height = img.height
      ctx.drawImage(img, 0, 0)
      
      canvas.toBlob((blob) => {
        if (blob) {
          saveAs(blob, `${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.jpg`)
        }
      }, 'image/jpeg', 0.9)
    }
    
    img.src = URL.createObjectURL(selectedFile.value)
  } else {
    const element = document.createElement('div')
    element.innerHTML = previewContent.value
    element.style.width = '800px'
    element.style.padding = '20px'
    
    document.body.appendChild(element)
    
    const canvas = await html2canvas(element, {
      width: 800,
      height: element.scrollHeight,
      scale: 2
    })
    
    document.body.removeChild(element)
    
    canvas.toBlob((blob) => {
      if (blob) {
        saveAs(blob, `${selectedFile.value?.name.replace(/\.[^/.]+$/, '')}.jpg`)
      }
    }, 'image/jpeg', 0.9)
  }
}

const handleVideoSelect = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    videoFile.value = target.files[0]
    videoUrl.value = URL.createObjectURL(target.files[0])
    outputUrl.value = null
  }
}

const handleVideoDrop = (event: DragEvent) => {
  event.preventDefault()
  dragOver.value = false
  
  if (event.dataTransfer?.files && event.dataTransfer.files[0]) {
    videoFile.value = event.dataTransfer.files[0]
    videoUrl.value = URL.createObjectURL(event.dataTransfer.files[0])
    outputUrl.value = null
  }
}



const waitForFfmpeg = () =>
  new Promise<void>((resolve, reject) => {
    let attempts = 0;
    const maxAttempts = 50; // 5 saniye
    
    const check = () => {
      attempts++;
      // @ts-ignore
      console.log(`FFmpeg check attempt ${attempts}:`, window.FFmpegWASM);
      
      // @ts-ignore
      if (window.FFmpegWASM && window.FFmpegWASM.FFmpeg) {
        console.log('FFmpeg found!');
        resolve();
      } else if (attempts >= maxAttempts) {
        console.error('FFmpeg not found after 5 seconds');
        reject(new Error('FFmpeg not loaded'));
      } else {
        setTimeout(check, 100);
      }
    };
    check();
  });

const loadFFmpeg = async () => {
  try {
    console.log('Waiting for FFmpeg...');
    await waitForFfmpeg();
    console.log('FFmpeg found in window');
    // @ts-ignore
    const FFmpeg = window.FFmpegWASM.FFmpeg;
    createFFmpeg = () => new FFmpeg();
    fetchFile = async (file: File) => {
      const arrayBuffer = await file.arrayBuffer();
      return new Uint8Array(arrayBuffer);
    };
    
    ffmpeg = createFFmpeg();
    ffmpeg.on('log', ({ message }: { message: string }) => console.log('FFmpeg:', message));
    ffmpeg.on('progress', ({ progress }: { progress: number }) => console.log('Progress:', progress));
    
    console.log('FFmpeg loading...');
    await ffmpeg.load();
    console.log('FFmpeg loaded successfully');
    ffmpegLoaded.value = true;
  } catch (error) {
    console.error('FFmpeg loading error:', error);
  }
};

const convertMovToMp4 = async () => {
  if (!videoFile.value) {
    alert('Lütfen bir video dosyası seçin!');
    return;
  }
  
  console.log('Video conversion started');
  console.log('FFmpeg loaded status:', ffmpegLoaded.value);
  
  isVideoConverting.value = true;
  outputUrl.value = null;
  
  try {
    if (!ffmpegLoaded.value) {
      console.log('Loading FFmpeg...');
      await loadFFmpeg();
    }
    
    console.log('Starting video conversion...');
    const fileData = await fetchFile(videoFile.value);
    const fileExtension = videoFile.value.name.split('.').pop()?.toLowerCase();
    const inputFileName = `input.${fileExtension || 'mp4'}`;
    console.log('Input filename:', inputFileName);
    console.log('File size:', fileData.length);
    console.log('File type:', videoFile.value.type);
    
    if (fileData.length < 1000) {
      alert('Dosya çok küçük, geçerli bir video dosyası değil!');
      return;
    }
    
    ffmpeg.writeFile(inputFileName, fileData);
    console.log('File written, starting conversion...');
    
    // Convert with more options
    await ffmpeg.exec(['-i', inputFileName, '-c:v', 'libx264', '-preset', 'ultrafast', '-crf', '23', '-y', 'output.mp4']);
    console.log('Conversion finished!');
    
    try {
      const data = await ffmpeg.readFile('output.mp4');
      console.log('Output file data:', data);
      console.log('Output file type:', typeof data);
      
      if (data && data.length > 0) {
        outputUrl.value = URL.createObjectURL(new Blob([data], { type: 'video/mp4' }));
        console.log('Video URL created successfully');
      } else {
        // Try to list files to see what's available
        const files = await ffmpeg.listDir('.');
        console.log('Available files:', files);
        throw new Error('Output file is empty or undefined');
      }
    } catch (readError) {
      console.error('Error reading output file:', readError);
      throw new Error('Dönüştürülen dosya okunamadı');
    }
    
    // Clean up
    try {
      ffmpeg.deleteFile(inputFileName);
      ffmpeg.deleteFile('output.mp4');
    } catch (e) {
      console.log('Cleanup error:', e);
    }
    
  } catch (error) {
    console.error('Video conversion error:', error);
    alert('Video dönüştürme sırasında bir hata oluştu! Dosya bozuk olabilir.');
  } finally {
    isVideoConverting.value = false;
  }
};

onMounted(() => {
  // Load FFmpeg when component mounts
  loadFFmpeg();
});
</script>

<template>
  <div class="converter">
    <!-- Top Ad Banner -->
    <div class="ad-banner top-ad">
      <ins class="adsbygoogle"
           style="display:block"
           data-ad-client="ca-pub-6592065784254031"
           data-ad-slot="1234567890"
           data-ad-format="auto"
           data-full-width-responsive="true"></ins>
      <script>
        (adsbygoogle = window.adsbygoogle || []).push({});
      </script>
    </div>

    <div class="header">
      <h1>Dosya Dönüştürücü</h1>
      <p>Dosyalarınızı ve videolarınızı istediğiniz formata dönüştürün</p>
    </div>

    <!-- Tab Navigation -->
    <div class="tab-navigation">
      <button 
        class="tab-btn"
        :class="{ active: activeTab === 'file' }"
        @click="activeTab = 'file'"
      >
        📄 Dosya Dönüştürücü
      </button>
      <button 
        class="tab-btn"
        :class="{ active: activeTab === 'video' }"
        @click="activeTab = 'video'"
      >
        🎥 Video Dönüştürücü
      </button>
    </div>

    <!-- File Converter Tab -->
    <div v-if="activeTab === 'file'" class="converter-container">
      <!-- Dosya Yükleme Alanı -->
      <div 
        class="file-upload"
        :class="{ 'drag-over': dragOver }"
        @drop="handleDrop"
        @dragover="handleDragOver"
        @dragleave="handleDragLeave"
      >
        <div class="upload-content">
          <div class="upload-icon">📁</div>
          <h3>Dosyanızı buraya sürükleyin veya seçin</h3>
          <p>Desteklenen formatlar: PDF, HTML, TXT, JPG, PNG, SVG</p>
          <input 
            type="file" 
            @change="handleFileSelect"
            accept=".pdf,.html,.txt,.jpg,.jpeg,.png,.svg"
            class="file-input"
          />
          <button class="select-btn">Dosya Seç</button>
        </div>
      </div>

      <!-- Seçilen Dosya Bilgisi -->
      <div v-if="selectedFile" class="file-info">
        <div class="file-details">
          <span class="file-name">{{ selectedFile.name }}</span>
          <span class="file-size">{{ (selectedFile.size / 1024).toFixed(1) }} KB</span>
        </div>
      </div>

      <!-- Format Seçimi -->
      <div class="format-selection">
        <h3>Hangi formata dönüştürmek istiyorsunuz?</h3>
        <div class="format-options">
          <label 
            v-for="format in availableFormats" 
            :key="format.value"
            class="format-option"
            :class="{ active: targetFormat === format.value }"
            @click="targetFormat = format.value"
          >
            <input 
              type="radio" 
              :value="format.value" 
              v-model="targetFormat"
              class="format-radio"
            />
            <span class="format-icon">{{ format.icon }}</span>
            <span class="format-label">{{ format.label }}</span>
          </label>
        </div>
      </div>
      <div v-if="showUnsupportedWarning" class="unsupported-warning">
        <span>{{ unsupportedLabel }} formatı şu anda tarayıcıda desteklenmiyor.</span>
      </div>

      <!-- Dönüştürme Butonu -->
      <button 
        @click="convertFile"
        :disabled="!selectedFile || isConverting"
        class="convert-btn"
        :class="{ converting: isConverting }"
      >
        <span v-if="isConverting">Dönüştürülüyor...</span>
        <span v-else>Dönüştür</span>
      </button>

      <!-- Middle Ad Banner -->
      <div class="ad-banner middle-ad">
        <ins class="adsbygoogle"
             style="display:block"
             data-ad-client="ca-pub-6592065784254031"
             data-ad-slot="1122334455"
             data-ad-format="auto"
             data-full-width-responsive="true"></ins>
        <script>
          (adsbygoogle = window.adsbygoogle || []).push({});
        </script>
      </div>

      <!-- Önizleme -->
      <div v-if="previewContent" class="preview">
        <h3>Önizleme</h3>
        <div class="preview-content" v-html="previewContent"></div>
      </div>
    </div>

    <!-- Video Converter Tab -->
    <div v-if="activeTab === 'video'" class="video-converter-container">
      <div class="video-converter">
        <h2>🎥 Video Dönüştürücü</h2>
        <p class="video-description">MOV, MP4, AVI ve diğer video formatlarını MP4'e dönüştürün</p>
        


        <!-- Video Upload Area -->
        <div 
          class="video-upload"
          :class="{ 'drag-over': dragOver }"
          @drop="handleVideoDrop"
          @dragover="handleDragOver"
          @dragleave="handleDragLeave"
        >
          <div class="upload-content">
            <div class="upload-icon">🎬</div>
            <h3>Video dosyanızı buraya sürükleyin veya seçin</h3>
            <p>Desteklenen formatlar: MOV, MP4, AVI, MKV, WMV</p>
            <input 
              type="file" 
              accept=".mov,.mp4,.avi,.mkv,.wmv,video/*" 
              @change="handleVideoSelect"
              class="file-input"
            />
            <button class="select-btn">Video Seç</button>
          </div>
        </div>

        <!-- Video Preview -->
        <div v-if="videoUrl" class="video-preview">
          <h4>📹 Yüklenen Video</h4>
          <video :src="videoUrl" controls class="video-player" />
          <div class="video-info">
            <span class="video-name">{{ videoFile?.name }}</span>
            <span class="video-size">{{ videoFile ? (videoFile.size / (1024 * 1024)).toFixed(1) + ' MB' : '' }}</span>
          </div>
        </div>

        <!-- Convert Button -->
        <button 
          @click="convertMovToMp4" 
          :disabled="!videoFile || isVideoConverting"
          class="convert-btn video-convert-btn"
          :class="{ converting: isVideoConverting }"
        >
          <span v-if="isVideoConverting">🔄 Dönüştürülüyor...</span>
          <span v-else>🎬 MP4'e Dönüştür</span>
        </button>

        <!-- Video Middle Ad Banner -->
        <div class="ad-banner video-middle-ad">
          <ins class="adsbygoogle"
               style="display:block"
               data-ad-client="ca-pub-6592065784254031"
               data-ad-slot="5566778899"
               data-ad-format="auto"
               data-full-width-responsive="true"></ins>
          <script>
            (adsbygoogle = window.adsbygoogle || []).push({});
          </script>
        </div>

        <!-- Output Video -->
        <div v-if="outputUrl" class="video-preview output-preview">
          <h4>✅ Dönüştürülen Video (MP4)</h4>
          <video :src="outputUrl" controls class="video-player" />
        </div>
        
        <!-- Download Button -->
        <div v-if="outputUrl" class="download-section">
          <a :href="outputUrl" download="converted_video.mp4" class="download-btn">
            💾 MP4 Olarak İndir
          </a>
        </div>
      </div>
    </div>

    <!-- Bottom Ad Banner -->
    <div class="ad-banner bottom-ad">
      <ins class="adsbygoogle"
           style="display:block"
           data-ad-client="ca-pub-6592065784254031"
           data-ad-slot="0987654321"
           data-ad-format="auto"
           data-full-width-responsive="true"></ins>
      <script>
        (adsbygoogle = window.adsbygoogle || []).push({});
      </script>
    </div>
  </div>
</template>

<style scoped>
.converter {
  min-height: 100vh;
  width: 100vw;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: stretch;
  padding: 0;
  background: #fff;
}

/* Ad Banner Styles */
.ad-banner {
  width: 100%;
  text-align: center;
  margin: 1rem 0;
  min-height: 90px;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #f8f9fa;
  border-radius: 8px;
  overflow: hidden;
}

.top-ad {
  margin-top: 0;
  margin-bottom: 1rem;
}

.bottom-ad {
  margin-top: 2rem;
  margin-bottom: 0;
}

.middle-ad {
  margin: 2rem 0;
}

.video-middle-ad {
  margin: 2rem 0;
}

/* Ad placeholder for when ads are loading */
.ad-banner:empty::before {
  content: "Reklam yükleniyor...";
  color: #7f8c8d;
  font-size: 0.9rem;
  font-style: italic;
}

.header {
  text-align: center;
  margin: 2rem 0 1.5rem 0;
  width: 100%;
}

.header h1 {
  font-size: clamp(2rem, 5vw, 3rem);
  color: #2c3e50;
  margin-bottom: 1rem;
}

.header p {
  font-size: clamp(1rem, 2.5vw, 1.2rem);
  color: #7f8c8d;
}

.converter-container {
  width: 100%;
  max-width: 100vw;
  margin: 0 auto;
  padding: 0 2vw;
  box-shadow: none;
  border-radius: 0;
  background: none;
  max-width: 800px;
}

.file-upload {
  border: 3px dashed #bdc3c7;
  border-radius: 16px;
  padding: 3rem 2rem;
  text-align: center;
  transition: all 0.3s ease;
  position: relative;
  margin-bottom: 2rem;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  width: 100%;
}

.file-upload.drag-over {
  border-color: #3498db;
  background: linear-gradient(135deg, #ecf0f1 0%, #d5dbdb 100%);
  transform: scale(1.02);
}

.upload-content {
  position: relative;
  width: 100%;
}

.upload-icon {
  font-size: clamp(3rem, 8vw, 4rem);
  margin-bottom: 1rem;
}

.upload-content h3 {
  color: #2c3e50;
  margin-bottom: 0.5rem;
  font-size: clamp(1.1rem, 3vw, 1.3rem);
}

.upload-content p {
  color: #7f8c8d;
  margin-bottom: 1.5rem;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
}

.file-input {
  position: absolute;
  opacity: 0;
  width: 100%;
  height: 100%;
  cursor: pointer;
  top: 0;
  left: 0;
}

.select-btn {
  background: #3498db;
  color: white;
  border: none;
  padding: clamp(0.6rem, 2vw, 0.8rem) clamp(1.5rem, 4vw, 2rem);
  border-radius: 8px;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 600;
}

.select-btn:hover {
  background: #2980b9;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(52, 152, 219, 0.3);
}

.file-info {
  background: #fff;
  padding: 1.5rem;
  border-radius: 12px;
  margin-bottom: 2rem;
  box-shadow: 0 4px 20px rgba(44,62,80,0.1);
  width: 100%;
}

.file-details {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.file-name {
  font-weight: bold;
  color: #2c3e50;
  word-break: break-all;
}

.file-size {
  color: #7f8c8d;
  font-size: 0.9rem;
  white-space: nowrap;
}

.format-selection {
  margin-bottom: 2rem;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(44,62,80,0.1);
  padding: 2rem;
  width: 100%;
}

.format-selection h3 {
  color: #2c3e50;
  margin-bottom: 1rem;
  font-size: clamp(1rem, 2.5vw, 1.2rem);
  text-align: center;
}

.format-options {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
  gap: 1rem;
}

.format-option {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: clamp(0.8rem, 2vw, 1rem);
  border: 2px solid #ecf0f1;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  min-height: 100px;
  justify-content: center;
  background: none;
}

.format-option:hover {
  border-color: #3498db;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(52, 152, 219, 0.2);
}

.format-option.active {
  border-color: #3498db;
  background-color: #ecf0f1;
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(52, 152, 219, 0.3);
}

.format-option.unsupported {
  opacity: 0.5;
  cursor: not-allowed;
}

.format-radio {
  display: none;
}

.format-icon {
  font-size: clamp(1.5rem, 4vw, 2rem);
  margin-bottom: 0.5rem;
}

.format-label {
  font-weight: bold;
  color: #2c3e50;
  font-size: clamp(0.8rem, 2vw, 1rem);
  text-align: center;
}

.unsupported-text {
  color: #e74c3c;
  font-size: 0.8em;
  margin-left: 0.3em;
}

.unsupported-warning {
  color: #e74c3c;
  background: #fdecea;
  border: 1px solid #e74c3c;
  border-radius: 6px;
  padding: 0.7em 1em;
  margin: 1em 0;
  font-size: 1em;
  text-align: center;
}

.convert-btn {
  width: 100%;
  background: linear-gradient(135deg, #27ae60 0%, #229954 100%);
  color: white;
  border: none;
  padding: 1rem 2rem;
  border-radius: 12px;
  font-size: 1.2rem;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-bottom: 2rem;
  min-height: 50px;
  box-shadow: 0 4px 12px rgba(39, 174, 96, 0.3);
}

.convert-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #229954 0%, #1e8449 100%);
  transform: translateY(-3px);
  box-shadow: 0 8px 25px rgba(39, 174, 96, 0.4);
}

.convert-btn:disabled {
  background: #bdc3c7;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.convert-btn.converting {
  background: #f39c12;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.02); }
  100% { transform: scale(1); }
}

.preview {
  margin: 2rem auto 0 auto;
  max-width: 100%;
  background: #fff;
  border: 1px solid #e0e0e0;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(44,62,80,0.1);
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  min-height: 80px;
  width: 100%;
}

.preview h3 {
  color: #2c3e50;
  margin-bottom: 0.7rem;
  font-size: 1.1rem;
  font-weight: 600;
  letter-spacing: 0.01em;
  text-align: center;
  width: 100%;
}

.preview-content {
  width: 100%;
  background: #f8f9fa;
  border: 1px solid #dee2e6;
  border-radius: 8px;
  padding: 0.7rem;
  max-height: 220px;
  overflow-y: auto;
  font-size: 0.97rem;
  box-sizing: border-box;
  margin: 0 auto;
  display: block;
}

.preview-content img {
  display: block;
  max-width: 100%;
  max-height: 200px;
  height: auto;
  width: auto;
  object-fit: contain;
  margin: 0 auto;
}

/* Tab Navigation */
.tab-navigation {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 2rem;
  padding: 0 2vw;
}

.tab-btn {
  background: #ecf0f1;
  border: 2px solid transparent;
  padding: 1rem 2rem;
  border-radius: 12px;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  color: #7f8c8d;
}

.tab-btn:hover {
  background: #d5dbdb;
  transform: translateY(-2px);
}

.tab-btn.active {
  background: #3498db;
  color: white;
  border-color: #2980b9;
  box-shadow: 0 4px 12px rgba(52, 152, 219, 0.3);
}

/* Video Converter Container */
.video-converter-container {
  width: 100%;
  max-width: 100vw;
  margin: 0 auto;
  padding: 0 2vw;
}

.video-converter {
  max-width: 800px;
  margin: 0 auto;
  background: #fff;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(44,62,80,0.1);
  padding: 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.video-converter h2 {
  color: #2c3e50;
  margin-bottom: 0.5rem;
  font-size: 2rem;
}

.video-description {
  color: #7f8c8d;
  margin-bottom: 2rem;
  text-align: center;
  font-size: 1.1rem;
}

/* Video Upload Area */
.video-upload {
  border: 3px dashed #bdc3c7;
  border-radius: 16px;
  padding: 3rem 2rem;
  text-align: center;
  transition: all 0.3s ease;
  position: relative;
  margin-bottom: 2rem;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  width: 100%;
}

.video-upload.drag-over {
  border-color: #3498db;
  background: linear-gradient(135deg, #ecf0f1 0%, #d5dbdb 100%);
  transform: scale(1.02);
}

/* Video Preview */
.video-preview {
  margin: 2rem 0;
  width: 100%;
  text-align: center;
  background: #f8f9fa;
  border-radius: 12px;
  padding: 1.5rem;
}

.video-preview h4 {
  color: #2c3e50;
  margin-bottom: 1rem;
  font-size: 1.2rem;
}

.video-player {
  max-width: 100%;
  max-height: 400px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.video-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

.video-name {
  font-weight: 600;
  color: #2c3e50;
  word-break: break-all;
}

.video-size {
  color: #7f8c8d;
  font-size: 0.9rem;
  white-space: nowrap;
}

.output-preview {
  background: linear-gradient(135deg, #d4edda 0%, #c3e6cb 100%);
  border: 2px solid #28a745;
}

/* Video Convert Button */
.video-convert-btn {
  background: linear-gradient(135deg, #3498db 0%, #2980b9 100%);
  font-size: 1.2rem;
  padding: 1rem 2rem;
  border-radius: 12px;
  margin: 1rem 0;
  min-width: 200px;
}

.video-convert-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #2980b9 0%, #1f5f8b 100%);
  transform: translateY(-3px);
  box-shadow: 0 8px 25px rgba(52, 152, 219, 0.4);
}



/* Download Section */
.download-section {
  margin: 1rem 0;
  text-align: center;
  width: 100%;
}

/* Download Button */
.download-btn {
  display: inline-block;
  background: linear-gradient(135deg, #27ae60 0%, #229954 100%);
  color: #fff;
  padding: 1rem 2rem;
  border-radius: 12px;
  text-decoration: none;
  font-weight: bold;
  font-size: 1.1rem;
  transition: all 0.3s ease;
  box-shadow: 0 4px 12px rgba(39, 174, 96, 0.3);
}

.download-btn:hover {
  background: linear-gradient(135deg, #229954 0%, #1e8449 100%);
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(39, 174, 96, 0.4);
}
.video-converter h2 {
  margin-bottom: 1rem;
  color: #2c3e50;
}
.video-preview {
  margin: 1rem 0;
  width: 100%;
  text-align: center;
}
.download-btn {
  display: inline-block;
  margin-top: 1rem;
  background: #3498db;
  color: #fff;
  padding: 0.7rem 1.5rem;
  border-radius: 8px;
  text-decoration: none;
  font-weight: bold;
  transition: background 0.2s;
}
.download-btn:hover {
  background: #217dbb;
}

@media (max-width: 768px) {
  .converter {
    padding: 0;
  }
  .converter-container {
    padding: 0 1vw;
  }
  .file-upload {
    min-height: 150px;
    padding: 1.5rem 1rem;
  }
  .format-options {
    grid-template-columns: repeat(2, 1fr);
    gap: 0.8rem;
  }
  .format-option {
    min-height: 80px;
    padding: 0.8rem 0.5rem;
  }
  .file-details {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.3rem;
  }
  .preview {
    max-width: 98vw;
    padding: 0.7rem 0.5rem 1rem 0.5rem;
  }
  .preview-content {
    max-height: 140px;
    font-size: 0.92rem;
  }
  .preview-content img {
    max-height: 100px;
  }
}

@media (max-width: 768px) {
  .tab-navigation {
    flex-direction: column;
    gap: 0.5rem;
  }
  
  .tab-btn {
    padding: 0.8rem 1.5rem;
    font-size: 1rem;
  }
  
  .video-converter {
    padding: 1.5rem;
  }
  

  
  .video-upload {
    padding: 2rem 1rem;
    min-height: 150px;
  }
  
  .video-player {
    max-height: 300px;
  }
  
  .video-info {
    flex-direction: column;
    gap: 0.5rem;
    text-align: center;
  }
  
  .ad-banner {
    min-height: 60px;
    margin: 0.5rem 0;
  }
}

@media (max-width: 480px) {
  .converter-container {
    padding: 0 0.5vw;
  }
  .file-upload {
    padding: 1rem;
    min-height: 120px;
  }
  .format-options {
    grid-template-columns: 1fr;
  }
  .format-option {
    flex-direction: row;
    justify-content: flex-start;
    gap: 1rem;
    min-height: 60px;
  }
  .format-icon {
    margin-bottom: 0;
  }
  
  .video-converter {
    padding: 1rem;
  }
  
  .video-upload {
    padding: 1.5rem 0.5rem;
    min-height: 120px;
  }
  
  .video-player {
    max-height: 200px;
  }
  
  .download-btn {
    padding: 0.8rem 1.5rem;
    font-size: 1rem;
  }
}
</style>