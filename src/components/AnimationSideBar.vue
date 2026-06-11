<template>
  <div class="w-full lg:w-64 h-full flex flex-col min-h-0"
       style="background: linear-gradient(180deg, rgba(18,18,26,0.96) 0%, rgba(10,10,15,1) 100%); border-right: 1px solid rgba(255,255,255,0.06);">
    <!-- Tabs -->
    <div class="px-2 pt-3 pb-1">
      <div class="inline-flex bg-white/[0.04] rounded-lg p-0.5 gap-0.5 w-full">
        <button
          class="flex-1 px-3 py-1.5 rounded-md text-xs font-medium transition-colors"
          :class="sidebarTab === 'controls' ? 'bg-amber-500/20 text-amber-200' : 'text-gray-400 hover:text-gray-300'"
          @click="sidebarTab = 'controls'"
        >Controls</button>
        <button
          class="flex-1 px-3 py-1.5 rounded-md text-xs font-medium transition-colors"
          :class="sidebarTab === 'layers' ? 'bg-amber-500/20 text-amber-200' : 'text-gray-400 hover:text-gray-300'"
          @click="sidebarTab = 'layers'"
        >Layers</button>
      </div>
    </div>

    <template v-if="sidebarTab === 'controls'">
      <div class="flex flex-col gap-2 flex-1 min-h-0 px-2 pt-1.5">
        <!-- Skin -->
        <div>
          <span class="text-[10px] uppercase tracking-widest text-gray-500">Skin</span>
          <select
            v-model="store.selectedSkin"
            class="bg-white/[0.04] border border-white/10 rounded-lg text-white text-xs px-2 py-1.5 mt-1 w-full outline-none focus:border-amber-500/40"
          >
            <option v-for="skin in skins" :key="skin" :value="skin">{{ skin }}</option>
          </select>
        </div>

        <!-- Animations -->
        <div class="flex items-center justify-between">
          <span class="text-[10px] uppercase tracking-widest text-gray-500">Animations</span>
          <span class="text-[9px] text-gray-600">{{ animations.length }}</span>
        </div>
        <div class="overflow-y-auto sidebar-scroll flex-1 min-h-0 -mx-2">
          <div
            v-for="name in sortedAnimations"
            :key="name"
            class="px-3 py-2 mx-1 cursor-pointer rounded-lg text-xs transition-all border mb-0.5"
            :class="name === store.selectedAnimation
              ? 'bg-amber-500/12 border-amber-500/25 text-amber-200'
              : 'border-transparent text-gray-400 hover:text-white hover:bg-white/[0.03]'"
            @click="select(name)"
          >
            {{ name }}
          </div>
          <div v-if="sortedAnimations.length === 0" class="text-center text-gray-500 text-xs py-4">
            No animations
          </div>
        </div>
      </div>
    </template>

    <!-- Layers tab -->
    <template v-else>
      <div class="flex flex-col gap-2 flex-1 min-h-0 px-2 pt-1.5">
        <div class="flex items-center justify-between">
          <span class="text-[10px] uppercase tracking-widest text-gray-500">Layers</span>
          <div class="flex gap-1.5">
            <button
              v-if="store.hiddenLayerStack.length > 0"
              class="text-[9px] text-amber-400 font-medium px-2 py-0.5 rounded bg-amber-500/10 hover:bg-amber-500/20 transition-colors"
              @click="undoLast"
            >Undo</button>
            <button
              v-if="store.hiddenLayerStack.length > 0"
              class="text-[9px] text-gray-400 px-2 py-0.5 rounded bg-white/5 hover:bg-white/10 transition-colors"
              @click="resetAll"
            >Reset</button>
          </div>
        </div>
        <input
          v-model="layerFilter"
          type="text"
          placeholder="Filter layers..."
          class="w-full bg-white/5 border border-white/10 rounded-lg px-3 py-1.5 text-xs text-white placeholder:text-gray-500 outline-none focus:border-amber-500/40 transition-colors"
        />
        <div class="overflow-y-auto sidebar-scroll flex-1 min-h-0 -mx-2">
          <label
            v-for="layer in filteredLayers"
            :key="layer.key"
            class="flex items-center gap-2.5 px-3 py-1.5 mx-1 rounded-lg cursor-pointer layer-toggle transition-colors"
            :class="isLayerVisible(layer.key) ? 'hover:bg-white/[0.03]' : 'bg-red-500/[0.06]'"
          >
            <input
              type="checkbox"
              :checked="isLayerVisible(layer.key)"
              @change="toggleLayer(layer.key)"
              class="rounded accent-amber-500 w-3.5 h-3.5"
            />
            <span class="truncate text-xs flex-1" :class="isLayerVisible(layer.key) ? 'text-gray-300' : 'text-red-400/60 line-through'">
              {{ layer.label }}
            </span>
          </label>
          <div v-if="filteredLayers.length === 0" class="text-center text-gray-500 text-xs py-4">
            No layers match
          </div>
        </div>
      </div>
    </template>

    <!-- Footer controls -->
    <div class="px-2 py-2 space-y-2 border-t border-white/[0.04] mt-auto">
      <!-- Category -->
      <div v-if="!currentChar?.customFiles">
        <span class="text-[10px] uppercase tracking-widest text-gray-500">Category</span>
        <select v-model="store.animationCategory" class="bg-white/[0.04] border border-white/10 rounded-lg text-white text-xs px-2 py-1.5 mt-1 w-full outline-none focus:border-amber-500/40">
          <option value="character">Character</option>
          <option value="ultimate" :disabled="!currentChar?.cutscene">Ultimate</option>
          <option value="dating" :disabled="!currentChar?.dating">Fated Guest</option>
        </select>
      </div>

      <!-- Speed -->
      <div>
        <div class="flex items-center justify-between">
          <span class="text-[10px] uppercase tracking-widest text-gray-500">Speed</span>
          <span class="text-[10px] text-amber-400/80 font-mono">{{ store.animationSpeed.toFixed(2) }}×</span>
        </div>
        <input
          type="range" min="0.1" max="2" step="0.05"
          v-model.number="store.animationSpeed"
          class="w-full mt-1 accent-amber-500 h-1"
        />
      </div>

      <!-- Action buttons -->
      <div class="flex gap-2">
        <button class="flex-1 bg-white/[0.04] hover:bg-white/[0.08] text-white rounded-lg text-xs py-2 transition-colors border border-white/[0.06]" @click="emit('reset-camera')">
          Reset View
        </button>
        <button class="flex-1 bg-amber-500/15 hover:bg-amber-500/25 text-amber-300 rounded-lg text-xs py-2 transition-colors border border-amber-500/20 font-medium" @click="store.playing = !store.playing">
          {{ store.playing ? 'Pause' : 'Play' }}
        </button>
      </div>

      <!-- BG Color -->
      <button class="w-full bg-white/[0.04] hover:bg-white/[0.08] text-white rounded-lg text-xs py-2 transition-colors border border-white/[0.06]" @click="colorInput?.click()">
        BG Color
      </button>
      <input ref="colorInput" type="color" class="hidden" :value="store.backgroundColor" @input="onColorChange" />

      <!-- Screenshot row -->
      <div class="flex gap-2 items-center">
        <button class="flex-1 bg-white/[0.04] hover:bg-white/[0.08] text-white rounded-lg text-xs py-2 transition-colors border border-white/[0.06]" :disabled="screenshotting" @click="onScreenshot">
          <LoadingIcon v-if="screenshotting" class="w-3.5 h-3.5 animate-spin mx-auto" />
          <span v-else>Screenshot</span>
        </button>
        <label class="flex items-center gap-1 text-[10px] text-gray-400 cursor-pointer">
          <input type="checkbox" v-model="transparentBg" class="rounded accent-amber-500 w-3 h-3" />
          <span>Transparent</span>
        </label>
      </div>

      <!-- Export -->
      <div class="relative" ref="desktopExportRef">
        <button class="w-full bg-white/[0.04] hover:bg-white/[0.08] text-white rounded-lg text-xs py-2 transition-colors border border-white/[0.06]" :disabled="exporting" @click="showExportMenu = !showExportMenu">
          <LoadingIcon v-if="exporting" class="w-3.5 h-3.5 animate-spin mx-auto" />
          <span v-else>Export Animation</span>
        </button>
        <div v-if="showExportMenu" class="absolute left-2 right-2 bottom-full mb-1 bg-[#0d0d14] border border-white/[0.08] rounded-lg shadow-xl shadow-black/40 z-10 overflow-hidden">
          <button class="block w-full text-left px-3 py-2 hover:bg-white/[0.04] transition-colors text-xs text-gray-300" @click="onExport('video')">Export as WebM</button>
          <button class="block w-full text-left px-3 py-2 hover:bg-white/[0.04] transition-colors text-xs text-gray-300" @click="onExport('frames')">Export as Frames (ZIP)</button>
        </div>
      </div>

      <!-- Use camera -->
      <label class="flex items-center gap-1.5 text-[10px] text-gray-400 cursor-pointer">
        <input type="checkbox" v-model="store.useCurrentCamera" class="rounded accent-amber-500 w-3 h-3" />
        <span>Use current camera</span>
      </label>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, toRefs, ref, watch, onMounted, onUnmounted } from 'vue'
import { useCharacterStore } from '@/stores/characterStore'
import LoadingIcon from '@/components/icons/LoadingIcon.vue'

const LAYER_SOURCE_SEPARATOR = ' > '

const props = defineProps<{ animations: string[]; skins: string[]; exporting: boolean; screenshotting: boolean }>()
const { animations, skins, exporting, screenshotting } = toRefs(props)

const store = useCharacterStore()
const colorInput = ref<HTMLInputElement | null>(null)
const transparentBg = ref(false)
const showExportMenu = ref(false)
const desktopExportRef = ref<HTMLElement | null>(null)
const sidebarTab = ref<'controls' | 'layers'>('controls')
const layerFilter = ref('')

const emit = defineEmits(['select', 'reset-camera', 'screenshot', 'export-animation', 'category-change'])

const sortedAnimations = computed(() => [...animations.value].sort((a, b) => a.localeCompare(b)))
const currentChar = computed(() => store.characters.find(c => c.id === store.selectedCharacterId))

// Layer full names (keep for desktop)
const layerNames = computed(() => [...store.layerNames].sort((a, b) => a.localeCompare(b)))
const layerItems = computed(() => {
  const baseCounts = new Map<string, number>()
  layerNames.value.forEach(name => {
    const baseName = name.includes(LAYER_SOURCE_SEPARATOR)
      ? name.slice(name.indexOf(LAYER_SOURCE_SEPARATOR) + LAYER_SOURCE_SEPARATOR.length)
      : name
    baseCounts.set(baseName, (baseCounts.get(baseName) ?? 0) + 1)
  })
  const seenDups = new Map<string, number>()
  return layerNames.value.map(name => {
    const baseName = name.includes(LAYER_SOURCE_SEPARATOR)
      ? name.slice(name.indexOf(LAYER_SOURCE_SEPARATOR) + LAYER_SOURCE_SEPARATOR.length)
      : name
    const dupCount = baseCounts.get(baseName) ?? 0
    if (dupCount <= 1) return { key: name, label: baseName }
    const seen = seenDups.get(baseName) ?? 0
    seenDups.set(baseName, seen + 1)
    return { key: name, label: seen === 0 ? baseName : `${baseName} (${seen === 1 ? 'extra' : `extra ${seen}`})` }
  })
})
const filteredLayers = computed(() => {
  const q = layerFilter.value.trim().toLowerCase()
  if (!q) return layerItems.value
  return layerItems.value.filter(l => l.key.toLowerCase().includes(q) || l.label.toLowerCase().includes(q))
})

function select(name: string) {
  emit('select', name)
  store.selectedAnimation = name
}

function isLayerVisible(key: string) {
  return store.layerVisibility[key] !== false
}

function toggleLayer(key: string) {
  store.layerVisibility[key] = !(store.layerVisibility[key] !== false)
}

function undoLast() {
  if (store.hiddenLayerStack.length > 0) {
    const last = store.hiddenLayerStack.pop()!
    store.layerVisibility[last] = true
    store.selectedLayerName = last
  }
}

function resetAll() {
  while (store.hiddenLayerStack.length > 0) {
    const n = store.hiddenLayerStack.pop()!
    store.layerVisibility[n] = true
  }
}

function onColorChange(e: Event) {
  const input = e.target as HTMLInputElement
  store.backgroundColor = input.value
}

function onScreenshot() {
  emit('screenshot', transparentBg.value)
}

function onExport(format: 'video' | 'frames') {
  emit('export-animation', { format, transparent: transparentBg.value })
  showExportMenu.value = false
}

function handleClickOutside(e: MouseEvent) {
  const target = e.target as Node
  if (desktopExportRef.value?.contains(target)) return
  showExportMenu.value = false
}

watch(() => store.animationCategory, () => {
  emit('category-change')
})

onMounted(() => document.addEventListener('click', handleClickOutside))
onUnmounted(() => document.removeEventListener('click', handleClickOutside))
</script>
