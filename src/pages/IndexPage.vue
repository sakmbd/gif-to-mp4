<template>
  <q-page padding>
    <q-card class="q-pa-md q-mx-auto" style="max-width: 600px">
      <q-card-section>
        <div class="text-h6 text-primary">
          <q-icon name="mdi-movie-open" size="md" class="q-mr-sm" />
          GIF to MP4 Converter
        </div>
        <div class="text-subtitle2">Select multiple .gif files to convert offline</div>
      </q-card-section>

      <q-card-section>
        <div
          class="dropzone rounded-borders"
          @dragover.prevent
          @drop="handleDrop"
          @click="$refs.fileInput.click()"
        >
          <input
            ref="fileInput"
            type="file"
            multiple
            accept=".gif"
            hidden
            @change="handleFileSelect"
          />
          <div class="dropzone-content text-center">
            <q-icon name="mdi-cloud-upload" size="xl" class="animated-upload-icon" />
            <div class="text-h6 q-mt-md">Drop GIFs Here</div>
            <div class="text-caption">or click to browse</div>
            <q-linear-progress v-if="isConverting" indeterminate color="primary" class="q-mt-md" />
          </div>
        </div>

        <div class="col-12 col-sm-auto q-mt-md">
          <q-btn
            label="Add from URL"
            color="accent"
            icon="mdi-link"
            @click="showUrlDialog = true"
            outline
            class="full-width"
          />
        </div>

        <div v-if="gifFiles.length" class="q-mt-md">
          <q-chip
            v-for="(file, i) in gifFiles"
            :key="i"
            removable
            color="primary"
            text-color="white"
            @remove="removeFile(i)"
          >
            {{ truncateName(file.name, 12) }}
          </q-chip>
        </div>
      </q-card-section>

      <q-card-section class="text-center">
        <div class="row q-col-gutter-sm justify-center">
          <div class="col-12 col-sm-auto" v-if="gifFiles.length || convertedFiles.length">
            <q-btn
              label="Clear All"
              color="negative"
              icon="clear"
              @click="clearAll"
              :disable="isConverting"
              class="full-width"
            />
          </div>

          <div class="col-12 col-sm-auto">
            <q-btn
              label="Convert & Download"
              color="primary"
              icon="mdi-autorenew"
              @click="convertGifs"
              :loading="isConverting"
              :disable="!gifFiles.length || isConverting"
              class="full-width"
            />
          </div>
          <div class="col-12 col-sm-auto" v-if="convertedFiles.length">
            <q-btn
              label="Download All"
              color="secondary"
              icon="download"
              @click="downloadAllFiles"
              :disable="isConverting"
              class="full-width"
            />
          </div>
        </div>
      </q-card-section>

      <q-card-section v-if="convertedFiles.length">
        <div class="text-caption text-grey q-mb-sm">
          Converted {{ convertedFiles.length }} file(s)
        </div>
        <q-list bordered separator>
          <q-item v-for="(file, i) in convertedFiles" :key="i">
            <q-item-section>
              <div>{{ truncateName(file.name, 15) }}</div>
            </q-item-section>
            <q-item-section avatar>
              <q-icon name="check" color="green" size="sm" />
            </q-item-section>
            <q-item-section side>
              <q-btn flat round icon="download" @click="downloadFile(file)" />
            </q-item-section>
          </q-item>
        </q-list>
      </q-card-section>

      <q-inner-loading :showing="isConverting">
        <q-spinner-gears size="50px" color="primary" />
        <div class="q-mt-sm">Converting {{ gifFiles.length }} files...</div>
      </q-inner-loading>
    </q-card>

    <q-dialog v-model="showUrlDialog">
      <q-card style="min-width: 325px">
        <q-card-section>
          <div class="text-h6">Add GIF from URL</div>
        </q-card-section>

        <q-card-section class="q-pt-none">
          <q-input
            v-model="gifUrl"
            label="GIF URL"
            outlined
            :rules="[
              (val) => !!val || 'URL is required',
              (val) => isValidUrl(val) || 'Invalid URL',
            ]"
            lazy-rules
            :disable="isFetchingUrl"
          />
        </q-card-section>

        <q-inner-loading :showing="isFetchingUrl">
          <q-spinner-gears size="40px" color="primary" />
          <div class="q-mt-sm">Fetching GIF from URL...</div>
        </q-inner-loading>

        <q-card-actions align="right" class="q-pt-none">
          <q-btn flat label="Cancel" color="primary" v-close-popup :disable="isFetchingUrl" />
          <q-btn
            flat
            label="Add GIF"
            color="primary"
            @click="addGifFromUrl"
            :loading="isFetchingUrl"
          />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup>
import { ref, onBeforeUnmount } from 'vue'
import { createFFmpeg, fetchFile } from '@ffmpeg/ffmpeg'

const gifFiles = ref([])
const isConverting = ref(false)
const convertedFiles = ref([])

const showUrlDialog = ref(false)
const gifUrl = ref('')
const isFetchingUrl = ref(false)

const isValidUrl = (url) => {
  try {
    new URL(url)
    return true
  } catch {
    return false
  }
}

const addGifFromUrl = async () => {
  if (!gifUrl.value || !isValidUrl(gifUrl.value)) {
    return
  }

  isFetchingUrl.value = true

  try {
    const response = await fetch(gifUrl.value)
    if (!response.ok) throw new Error('Failed to fetch GIF')

    const blob = await response.blob()
    if (!blob.type.includes('gif')) throw new Error('URL does not point to a GIF file')

    const file = new File([blob], `gif-from-url-${Date.now()}.gif`, { type: blob.type })
    gifFiles.value.push(file)
    showUrlDialog.value = false
    gifUrl.value = ''
  } catch (error) {
    alert(`Error: ${error.message}`)
  } finally {
    isFetchingUrl.value = false
  }
}

// Initialize FFmpeg with local core
const ffmpeg = createFFmpeg({
  log: true,
  corePath: '/ffmpeg/ffmpeg-core.js',
  progress: ({ ratio }) => {
    console.log(`Progress: ${(ratio * 100).toFixed(2)}%`)
  },
})

const setupFFmpeg = async () => {
  if (!ffmpeg.isLoaded()) {
    try {
      await ffmpeg.load()
    } catch (error) {
      console.error('FFmpeg load error:', error)
      throw new Error('Failed to load FFmpeg')
    }
  }
}

const convertGifs = async () => {
  isConverting.value = true
  convertedFiles.value = []

  try {
    await setupFFmpeg()

    for (const [index, file] of gifFiles.value.entries()) {
      const inputName = `input_${index}.gif`
      const outputName = file.name.replace(/\.gif$/, '.mp4')

      ffmpeg.FS('writeFile', inputName, await fetchFile(file))
      await ffmpeg.run(
        '-i',
        inputName,
        '-movflags',
        'faststart',
        '-pix_fmt',
        'yuv420p',
        '-vf',
        'scale=trunc(iw/2)*2:trunc(ih/2)*2',
        outputName,
      )

      const data = ffmpeg.FS('readFile', outputName)
      const blob = new Blob([data.buffer], { type: 'video/mp4' })
      const url = URL.createObjectURL(blob)

      convertedFiles.value.push({
        name: outputName,
        url,
        blob,
      })

      ffmpeg.FS('unlink', inputName)
      ffmpeg.FS('unlink', outputName)
    }
  } catch (err) {
    console.error('Conversion error:', err)
    alert(`Conversion failed: ${err.message}`)
  } finally {
    isConverting.value = false
  }
}

const downloadFile = (file) => {
  const a = document.createElement('a')
  a.href = file.url
  a.download = file.name
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
}

const downloadAllFiles = () => {
  convertedFiles.value.forEach((file, index) => {
    setTimeout(() => downloadFile(file), index * 1000)
  })
}

// New clear all function
const clearAll = () => {
  // Clean up object URLs before clearing
  convertedFiles.value.forEach((file) => {
    URL.revokeObjectURL(file.url)
  })
  gifFiles.value = []
  convertedFiles.value = []
}

onBeforeUnmount(() => {
  clearAll()
})

const removeFile = (index) => {
  gifFiles.value.splice(index, 1)
}

const handleFileSelect = (e) => {
  gifFiles.value = Array.from(e.target.files)
  convertedFiles.value = []
}

const handleDrop = (e) => {
  e.preventDefault()
  gifFiles.value = Array.from(e.dataTransfer.files)
  convertedFiles.value = []
}

const truncateName = (name, limit = 12) =>
  name.length > limit ? name.slice(0, limit) + '...' : name
</script>

<style scoped>
.dropzone {
  border: 2px dashed var(--q-primary);
  padding: 2rem;
  background: linear-gradient(135deg, #f5f7fa 0%, #e4f0ff 100%);
  transition: all 0.3s;
  cursor: pointer;
}

.dropzone:hover {
  background: linear-gradient(135deg, #e4f0ff 0%, #d0e3ff 100%);
  transform: translateY(-2px);
}

.animated-upload-icon {
  animation: float 3s ease-in-out infinite;
}

@keyframes float {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}
.floating-input {
  display: inline-block;
  padding: 2rem;
  border-radius: 50%;
  background: rgba(var(--q-primary-rgb), 0.1);
  transition: all 0.3s;
  cursor: pointer;
}
.floating-input:hover {
  transform: scale(1.05);
  background: rgba(var(--q-primary-rgb), 0.2);
}
.q-btn ::v-deep(.q-icon) {
  margin-right: 10px;
}

@media (max-width: 600px) {
  .q-card {
    border-radius: 0;
    box-shadow: none;
  }

  .q-btn {
    padding: 0.8rem;
  }

  .q-item {
    padding-right: 0;
    height: 8px;
  }
}
</style>
