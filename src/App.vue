<template>
  <div class="bg-[#0a0a0f] text-white h-dvh max-h-dvh flex flex-col overflow-hidden">
    <!-- ── Compact Navbar ── -->
    <Navbar
      :has-custom-background="hasCustomBackground"
      @mobile-menu="onNavMobileMenu"
      @upload-bg="onCustomBgUpload"
      @overlay-active="onNavbarOverlayActive"
    />

    <!-- ── Main viewer area ── -->
    <main class="relative flex-1 min-h-0">
      <SpineViewer
        ref="viewerRef"
        :mobile-overlay-active="overlayActive"
        @animations="animations = $event"
        @skins="skins = $event"
      />

      <!-- Keyboard hint toast for layer selection -->
      <Transition name="toast">
        <div
          v-if="showLayerSelectionHint"
          class="absolute top-16 left-1/2 -translate-x-1/2 z-50 pointer-events-none"
        >
          <div class="rounded-xl border border-amber-500/20 bg-black/80 px-4 py-2.5 text-xs font-medium text-amber-200/90 shadow-lg shadow-amber-500/5 backdrop-blur-lg whitespace-nowrap">
            Tap a part to select · <kbd class="mx-0.5 rounded bg-amber-500/20 px-1.5 py-0.5 text-amber-300">H</kbd> hide · <kbd class="rounded bg-amber-500/20 px-1.5 py-0.5 text-amber-300">U</kbd> undo · <kbd class="rounded bg-amber-500/20 px-1.5 py-0.5 text-amber-300">Esc</kbd> reset
          </div>
        </div>
      </Transition>

      <!-- ── Mobile: Quick action chips floating top ── -->
      <div class="absolute top-3 left-3 right-3 z-40 lg:hidden flex items-center gap-1.5">
        <button
          v-show="!overlayActive"
          class="glass-card btn-press rounded-xl px-3 py-2 text-xs font-semibold flex items-center gap-1.5 whitespace-nowrap"
          @click="openSheet('characters')"
        >
          <span class="text-amber-400">◆</span>
          {{ currentCharShortName }}
        </button>

        <!-- Layers toggle + layer actions row -->
        <div
          v-show="!overlayActive"
          class="glass-card rounded-xl flex items-center gap-0.5"
        >
          <button
            class="btn-press rounded-lg px-2.5 py-2 text-xs font-semibold flex items-center gap-1 whitespace-nowrap"
            :class="store.layerSelectionEnabled ? 'text-amber-300 bg-amber-500/15' : 'text-gray-300'"
            @click="store.layerSelectionEnabled = !store.layerSelectionEnabled"
          >
            <span>{{ store.layerSelectionEnabled ? '⬡' : '⬢' }}</span>
            Layers
          </button>
          <template v-if="store.layerSelectionEnabled">
            <span class="w-px h-5 bg-white/10" />
            <button
              class="btn-press px-2 py-2 rounded-lg text-[10px] font-semibold text-red-300/80 hover:text-red-300 hover:bg-red-500/15 transition-colors"
              @click="hideSelectedLayer"
              :disabled="!store.selectedLayerName"
              :class="!store.selectedLayerName ? 'opacity-30' : ''"
            >H</button>
            <button
              class="btn-press px-2 py-2 rounded-lg text-[10px] font-semibold text-amber-300/80 hover:text-amber-300 hover:bg-amber-500/15 transition-colors"
              @click="undoHideLayerAction"
              :disabled="store.hiddenLayerStack.length === 0"
              :class="store.hiddenLayerStack.length === 0 ? 'opacity-30' : ''"
            >U</button>
            <button
              class="btn-press px-2 py-2 rounded-lg text-[10px] font-semibold text-gray-400 hover:text-gray-300 hover:bg-white/10 transition-colors"
              @click="clearAllLayers"
              :disabled="store.hiddenLayerStack.length === 0"
              :class="store.hiddenLayerStack.length === 0 ? 'opacity-30' : ''"
            >
              <span class="text-xs leading-none">⟳</span>
            </button>
          </template>
        </div>

        <!-- Layer name display -->
        <Transition name="toast">
          <div
            v-if="store.layerSelectionEnabled && store.selectedLayerName"
            class="text-xs font-semibold text-amber-300 truncate max-w-[120px] bg-white/5 rounded-lg px-3 py-2 border border-amber-500/20"
          >
            {{ store.selectedLayerName }}
          </div>
        </Transition>
      </div>

      <!-- ── Mobile: Zoom controls (top-right corner, always visible) ── -->
      <div
        v-show="!overlayActive"
        class="absolute right-3 top-3 z-40 lg:hidden flex flex-col gap-1"
      >
        <button
          aria-label="Zoom in"
          class="glass-card btn-press w-9 h-9 rounded-xl flex items-center justify-center"
          @click="onZoomIn"
        >
          <PlusIcon class="w-4 h-4 text-gray-300" />
        </button>
        <button
          aria-label="Zoom out"
          class="glass-card btn-press w-9 h-9 rounded-xl flex items-center justify-center"
          @click="onZoomOut"
        >
          <MinusIcon class="w-4 h-4 text-gray-300" />
        </button>
      </div>

      <!-- ── Mobile: Bottom floating toolbar ── -->
      <div
        v-show="!overlayActive"
        class="absolute bottom-3 left-3 right-3 z-40 lg:hidden"
      >
        <div class="bottom-toolbar rounded-2xl px-3 py-2 flex items-center justify-between gap-1">
          <!-- Play/Pause -->
          <button class="btn-press flex flex-col items-center justify-center w-12 h-11 rounded-xl" @click="store.playing = !store.playing">
            <PauseIcon v-if="store.playing" class="w-4 h-4 text-amber-400" />
            <PlayIcon v-else class="w-4 h-4 text-amber-400" />
            <span class="text-[9px] text-gray-400 mt-0.5">{{ store.playing ? 'Pause' : 'Play' }}</span>
          </button>

          <!-- Category -->
          <button
            class="btn-press flex flex-col items-center justify-center w-12 h-11 rounded-xl"
            @click="cycleCategory"
            :disabled="!canCycleCategory"
          >
            <span class="text-xs font-bold" :class="categoryIconColor">{{ categoryIcon }}</span>
            <span class="text-[9px] text-gray-400 mt-0.5 truncate">{{ categoryLabel }}</span>
          </button>

          <!-- Animations / skin picker -->
          <button class="btn-press flex flex-col items-center justify-center w-12 h-11 rounded-xl" @click="openSheet('animations')">
            <span class="text-xs text-amber-400">▶</span>
            <span class="text-[9px] text-gray-400 mt-0.5">Anims</span>
          </button>

          <!-- Speed -->
          <button class="btn-press flex flex-col items-center justify-center w-11 h-11 rounded-xl" @click="cycleSpeed">
            <span class="text-xs font-bold text-amber-400">{{ store.animationSpeed.toFixed(1) }}×</span>
            <span class="text-[9px] text-gray-400 mt-0.5">Speed</span>
          </button>

          <!-- Zoom out -->
          <button class="btn-press flex flex-col items-center justify-center w-10 h-11 rounded-xl" @click="onZoomOut">
            <MinusIcon class="w-3.5 h-3.5 text-gray-300" />
            <span class="text-[9px] text-gray-400 mt-0.5">Out</span>
          </button>

          <!-- Zoom in -->
          <button class="btn-press flex flex-col items-center justify-center w-10 h-11 rounded-xl" @click="onZoomIn">
            <PlusIcon class="w-3.5 h-3.5 text-gray-300" />
            <span class="text-[9px] text-gray-400 mt-0.5">In</span>
          </button>

          <!-- Menu / More -->
          <button class="btn-press flex flex-col items-center justify-center w-11 h-11 rounded-xl" @click="openSheet('more')">
            <span class="text-base text-amber-400">⋯</span>
            <span class="text-[9px] text-gray-400 mt-0.5">More</span>
          </button>
        </div>
      </div>
    </main>

    <!-- ═══ DESKTOP sidebars (unchanged layout logic) ═══ -->
    <div class="hidden lg:flex flex-row h-full min-h-0 absolute left-0 top-[52px] bottom-0 z-30 pointer-events-none">
      <div class="pointer-events-auto">
        <AnimationSidebar
          :animations="animations"
          :skins="skins"
          :exporting="isExporting"
          :screenshotting="isScreenshotting"
          @select="onSelectAnimation"
          @reset-camera="onResetCamera"
          @screenshot="onScreenshot"
          @export-animation="onExportAnimation"
          @category-change="onCategoryChange"
          class="w-64"
        />
      </div>
    </div>
    <div class="hidden lg:flex flex-row h-full min-h-0 absolute right-0 top-[52px] bottom-0 z-30 pointer-events-none">
      <div class="pointer-events-auto">
        <CharacterSidebar @select="onSelectCharacter" class="w-72" />
      </div>
    </div>

    <!-- ═══ MOBILE BOTTOM SHEET: Characters ═══ -->
    <Transition name="sheet-backdrop">
      <div
        v-if="activeSheet === 'characters'"
        class="fixed inset-0 z-50 bg-black/60 backdrop-blur-sm lg:hidden"
        @click.self="closeSheet"
      >
        <Transition name="sheet">
          <div
            v-if="activeSheet === 'characters'"
            class="absolute bottom-0 left-0 right-0 max-h-[75dvh] rounded-t-2xl overflow-hidden flex flex-col"
            style="background: linear-gradient(180deg, rgba(18,18,26,0.98) 0%, rgba(10,10,15,1) 100%); border-top: 1px solid rgba(255,255,255,0.08);"
          >
            <div class="sheet-handle" @click="closeSheet" />
            <div class="px-4 pt-2 pb-1">
              <input
                v-model="charFilter"
                type="text"
                placeholder="Search characters..."
                class="w-full bg-white/5 border border-white/10 rounded-xl px-4 py-3 text-sm text-white placeholder:text-gray-500 outline-none focus:border-amber-500/40 transition-colors"
              />
            </div>
            <div class="flex-1 overflow-y-auto px-2 pb-safe sidebar-scroll">
              <div class="grid grid-cols-3 gap-2">
                <button
                  v-for="char in filteredMobileChars"
                  :key="char.id"
                  class="char-chip flex flex-col items-center gap-1 p-2 rounded-xl border"
                  :class="[
                    char.id === store.selectedCharacterId
                      ? 'active border-amber-500/40 bg-amber-500/8'
                      : 'border-transparent bg-white/[0.03] hover:bg-white/[0.06]'
                  ]"
                  @click="selectCharAndClose(char.id)"
                >
                  <img
                    :src="icons[char.icon] || icons['unknown']"
                    :alt="char.costumeName"
                    class="w-14 h-14 rounded-full object-cover border-2"
                    :class="char.id === store.selectedCharacterId ? 'border-amber-500/60 shadow-lg shadow-amber-500/20' : 'border-white/10'"
                    loading="lazy"
                  />
                  <span class="text-[10px] leading-tight text-gray-300 line-clamp-2 text-center">{{ char.charName }}</span>
                  <div class="flex gap-0.5">
                    <span v-if="char.dating" class="text-[8px] bg-blue-500/20 text-blue-400 px-1 rounded">FG</span>
                    <span v-if="char.cutscene" class="text-[8px] bg-purple-500/20 text-purple-400 px-1 rounded">ULT</span>
                  </div>
                </button>
              </div>
            </div>
          </div>
        </Transition>
      </div>
    </Transition>

    <!-- ═══ MOBILE BOTTOM SHEET: Animations & Skins ═══ -->
    <Transition name="sheet-backdrop">
      <div
        v-if="activeSheet === 'animations'"
        class="fixed inset-0 z-50 bg-black/60 backdrop-blur-sm lg:hidden"
        @click.self="closeSheet"
      >
        <Transition name="sheet">
          <div
            v-if="activeSheet === 'animations'"
            class="absolute bottom-0 left-0 right-0 max-h-[70dvh] rounded-t-2xl overflow-hidden flex flex-col"
            style="background: linear-gradient(180deg, rgba(18,18,26,0.98) 0%, rgba(10,10,15,1) 100%); border-top: 1px solid rgba(255,255,255,0.08);"
          >
            <div class="sheet-handle" @click="closeSheet" />

            <!-- Skin selector -->
            <div class="px-4 pt-3 pb-2">
              <span class="text-[10px] uppercase tracking-widest text-gray-500">Skin</span>
              <div class="flex gap-2 mt-1.5 flex-wrap">
                <button
                  v-for="skin in skins"
                  :key="skin"
                  class="category-pill px-3 py-1.5 rounded-lg text-xs font-medium border border-white/10"
                  :class="skin === store.selectedSkin ? 'active' : ''"
                  @click="store.selectedSkin = skin"
                >
                  {{ skin }}
                </button>
              </div>
            </div>

            <!-- Animation list -->
            <div class="px-4 pt-1 pb-2 flex items-center justify-between">
              <span class="text-[10px] uppercase tracking-widest text-gray-500">Animations</span>
              <span class="text-[10px] text-gray-500">{{ animations.length }} total</span>
            </div>
            <div class="flex-1 overflow-y-auto px-3 pb-safe sidebar-scroll">
              <div class="grid grid-cols-2 gap-2">
                <button
                  v-for="name in sortedAnimations"
                  :key="name"
                  class="text-left px-3 py-2.5 rounded-xl text-xs font-medium transition-all border"
                  :class="
                    name === store.selectedAnimation
                      ? 'bg-amber-500/15 border-amber-500/30 text-amber-200'
                      : 'bg-white/[0.03] border-transparent text-gray-400 hover:text-white hover:bg-white/[0.06]'
                  "
                  @click="selectAnimAndClose(name)"
                >
                  {{ name }}
                </button>
              </div>
            </div>
          </div>
        </Transition>
      </div>
    </Transition>

    <!-- ═══ MOBILE BOTTOM SHEET: More options ═══ -->
    <Transition name="sheet-backdrop">
      <div
        v-if="activeSheet === 'more'"
        class="fixed inset-0 z-50 bg-black/60 backdrop-blur-sm lg:hidden"
        @click.self="closeSheet"
      >
        <Transition name="sheet">
          <div
            v-if="activeSheet === 'more'"
            class="absolute bottom-0 left-0 right-0 rounded-t-2xl overflow-hidden flex flex-col"
            style="background: linear-gradient(180deg, rgba(18,18,26,0.98) 0%, rgba(10,10,15,1) 100%); border-top: 1px solid rgba(255,255,255,0.08);"
          >
            <div class="sheet-handle" @click="closeSheet" />

            <div class="px-4 pt-3 pb-1">
              <span class="text-[10px] uppercase tracking-widest text-gray-500">More options</span>
            </div>

            <div class="px-4 py-2 space-y-2 pb-safe">
              <!-- Reset view -->
              <button
                class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm"
                @click="onResetCamera(); closeSheet();"
              >
                <CameraResetIcon class="w-4 h-4 text-amber-400" />
                Reset view
              </button>

              <!-- Toggle background -->
              <button
                v-if="store.characters.find(c => c.id === store.selectedCharacterId)?.datingHasNoBg && store.animationCategory === 'dating'"
                class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm"
                @click="store.showDatingBg = !store.showDatingBg"
              >
                <BgToggleIcon :active="store.showDatingBg" class="w-4 h-4" />
                {{ store.showDatingBg ? 'Hide dating BG' : 'Show dating BG' }}
              </button>

              <!-- Screenshot -->
              <button
                class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm"
                :disabled="isScreenshotting"
                @click="onScreenshotMenu(); closeSheet();"
              >
                <LoadingIcon v-if="isScreenshotting" class="w-4 h-4 animate-spin" />
                <span v-else class="text-base">📸</span>
                Screenshot
              </button>

              <!-- Export as video -->
              <button
                class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm"
                :disabled="isExporting"
                @click="onExportAnimation({ format: 'video', transparent: transparentBg }); closeSheet();"
              >
                <LoadingIcon v-if="isExporting" class="w-4 h-4 animate-spin" />
                <span v-else class="text-base">🎬</span>
                Export as video
              </button>

              <!-- Export as frames -->
              <button
                class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm"
                :disabled="isExporting"
                @click="onExportAnimation({ format: 'frames', transparent: transparentBg }); closeSheet();"
              >
                <LoadingIcon v-if="isExporting" class="w-4 h-4 animate-spin" />
                <span v-else class="text-base">🖼️</span>
                Export as frames (ZIP)
              </button>

              <!-- BG Color picker -->
              <label class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm cursor-pointer">
                <span class="text-base">🎨</span>
                Background color
                <input
                  type="color"
                  :value="store.backgroundColor"
                  @input="onColorChange"
                  class="ml-auto w-8 h-8 rounded-lg border border-white/20 bg-transparent cursor-pointer"
                />
              </label>

              <!-- Transparent toggle -->
              <label class="w-full flex items-center justify-between px-3 py-3 rounded-xl bg-white/[0.03] text-sm">
                <span>Transparent exports</span>
                <input type="checkbox" v-model="transparentBg" class="rounded accent-amber-500" />
              </label>

              <!-- Use current camera -->
              <label class="w-full flex items-center justify-between px-3 py-3 rounded-xl bg-white/[0.03] text-sm">
                <span>Use current camera</span>
                <input type="checkbox" v-model="store.useCurrentCamera" class="rounded accent-amber-500" />
              </label>

              <!-- Upload custom -->
              <button
                class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm"
                @click="closeSheet(); showUploadModal = true;"
              >
                <span class="text-base">📦</span>
                Upload custom Spine
              </button>

              <button
                class="w-full flex items-center gap-3 px-3 py-3 rounded-xl bg-white/[0.03] hover:bg-white/[0.06] transition-colors text-sm"
                @click="closeSheet(); showBgModal = true;"
              >
                <span class="text-base">🖼️</span>
                Upload background
              </button>
            </div>
          </div>
        </Transition>
      </div>
    </Transition>

    <!-- ═══ MOBILE BOTTOM SHEET: Layer management ═══ -->
    <Transition name="sheet-backdrop">
      <div
        v-if="activeSheet === 'layers'"
        class="fixed inset-0 z-50 bg-black/60 backdrop-blur-sm lg:hidden"
        @click.self="closeSheet"
      >
        <Transition name="sheet">
          <div
            v-if="activeSheet === 'layers'"
            class="absolute bottom-0 left-0 right-0 max-h-[60dvh] rounded-t-2xl overflow-hidden flex flex-col"
            style="background: linear-gradient(180deg, rgba(18,18,26,0.98) 0%, rgba(10,10,15,1) 100%); border-top: 1px solid rgba(255,255,255,0.08);"
          >
            <div class="sheet-handle" @click="closeSheet" />
            <div class="px-4 pt-2 pb-1 flex items-center justify-between">
              <span class="text-[10px] uppercase tracking-widest text-gray-500">Layers</span>
              <div class="flex gap-2">
                <button
                  v-if="store.hiddenLayerStack.length > 0"
                  class="text-[10px] text-amber-400 font-medium px-2 py-0.5 rounded bg-amber-500/10"
                  @click="undoHideLayerAction"
                >
                  Undo ({{ store.hiddenLayerStack.length }})
                </button>
                <button
                  v-if="store.hiddenLayerStack.length > 0"
                  class="text-[10px] text-gray-400 font-medium px-2 py-0.5 rounded bg-white/5"
                  @click="clearAllLayers"
                >
                  Reset all
                </button>
              </div>
            </div>
            <div class="px-4 pb-2">
              <input
                v-model="layerFilterText"
                type="text"
                placeholder="Filter layers..."
                class="w-full bg-white/5 border border-white/10 rounded-lg px-3 py-2 text-xs text-white placeholder:text-gray-500 outline-none focus:border-amber-500/40 transition-colors"
              />
            </div>
            <div class="flex-1 overflow-y-auto px-3 pb-safe sidebar-scroll">
              <label
                v-for="layer in filteredLayers"
                :key="layer.key"
                class="flex items-center gap-3 px-3 py-2.5 rounded-xl layer-toggle cursor-pointer"
                :class="isLayerVisible(layer.key) ? 'hover:bg-white/[0.04]' : 'bg-red-500/5'"
              >
                <input
                  type="checkbox"
                  :checked="isLayerVisible(layer.key)"
                  @change="toggleLayer(layer.key)"
                  class="rounded accent-amber-500 w-4 h-4"
                />
                <span class="text-xs truncate flex-1" :class="isLayerVisible(layer.key) ? 'text-gray-300' : 'text-red-400/70 line-through'">
                  {{ layer.label }}
                </span>
                <span v-if="!isLayerVisible(layer.key)" class="text-[8px] text-red-400/60 font-medium">HIDDEN</span>
              </label>
              <div v-if="filteredLayers.length === 0" class="text-center text-gray-500 text-xs py-8">
                No layers match your filter
              </div>
            </div>
          </div>
        </Transition>
      </div>
    </Transition>

    <!-- ═══ Modals (shared between mobile/desktop) ═══ -->
    <UploadSpineModal v-if="showUploadModal" @close="showUploadModal = false" />
    <ChangelogModal v-if="showChangelog" @close="showChangelog = false" />
    <UploadBackgroundModal
      v-if="showBgModal"
      :show-reset="hasCustomBackground"
      @close="showBgModal = false"
      @upload-bg="handleBgUpload"
      @reset-bg="handleBgReset"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, watchEffect, onBeforeUnmount } from 'vue'
import { useCharacterStore } from '@/stores/characterStore'
import { buildUrl } from '@/utils/urlSync'
import icons from '@/utils/charIcons'

import Navbar from '@/components/Navbar.vue'
import CharacterSidebar from '@/components/CharacterSideBar.vue'
import AnimationSidebar from '@/components/AnimationSideBar.vue'
import SpineViewer from '@/components/SpineViewer.vue'
import UploadSpineModal from '@/components/UploadSpineModal.vue'
import ChangelogModal from '@/components/ChangelogModal.vue'
import UploadBackgroundModal from '@/components/UploadBackgroundModal.vue'

import CameraResetIcon from '@/components/icons/CameraResetIcon.vue'
import PauseIcon from '@/components/icons/PauseIcon.vue'
import PlayIcon from '@/components/icons/PlayIcon.vue'
import BgToggleIcon from '@/components/icons/BgToggleIcon.vue'
import MinusIcon from '@/components/icons/MinusIcon.vue'
import PlusIcon from '@/components/icons/PlusIcon.vue'
import LoadingIcon from '@/components/icons/LoadingIcon.vue'

const LAYER_SOURCE_SEPARATOR = ' > '

const store = useCharacterStore()

const animations = ref<string[]>([])
const skins = ref<string[]>([])
const viewerRef = ref<InstanceType<typeof SpineViewer> | null>(null)
const isExporting = ref(false)
const isScreenshotting = ref(false)
const navMobileMenuOpen = ref(false)
const navbarOverlayActive = ref(false)
const showLayerSelectionHint = ref(false)
const transparentBg = ref(false)
let layerSelectionHintTimeout: number | null = null

// Sheet state
const activeSheet = ref<'characters' | 'animations' | 'layers' | 'more' | null>(null)
const charFilter = ref('')
const layerFilterText = ref('')
const showUploadModal = ref(false)
const showBgModal = ref(false)
const showChangelog = ref(false)

const overlayActive = computed(
  () => !!activeSheet.value || navMobileMenuOpen.value || navbarOverlayActive.value,
)

const hasCustomBackground = computed(() => !!store.customBackgroundImage)

// Derived
const currentChar = computed(() => store.characters.find(c => c.id === store.selectedCharacterId))
const currentCharShortName = computed(() => {
  const c = currentChar.value
  if (!c) return 'Select'
  const parts = c.charName.split(' ')
  return parts[parts.length - 1] || c.charName
})

const categoryIcon = computed(() => {
  switch (store.animationCategory) {
    case 'ultimate': return '⚡'
    case 'dating': return '💕'
    default: return '👤'
  }
})
const categoryIconColor = computed(() => {
  switch (store.animationCategory) {
    case 'ultimate': return 'text-purple-400'
    case 'dating': return 'text-blue-400'
    default: return 'text-amber-400'
  }
})
const categoryLabel = computed(() => {
  switch (store.animationCategory) {
    case 'ultimate': return 'Ult'
    case 'dating': return 'Date'
    default: return 'Char'
  }
})

const canCycleCategory = computed(() => {
  const c = currentChar.value
  if (!c) return false
  return !!(c.cutscene || c.dating)
})

const sortedAnimations = computed(() => [...animations.value].sort((a, b) => a.localeCompare(b)))

const filteredMobileChars = computed(() => {
  const q = charFilter.value.trim().toLowerCase()
  if (!q) return store.characters
  return store.characters.filter(c =>
    `${c.charName} ${c.costumeName}`.toLowerCase().includes(q),
  )
})

// Layer helpers
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
  const q = layerFilterText.value.trim().toLowerCase()
  if (!q) return layerItems.value
  return layerItems.value.filter(l => l.key.toLowerCase().includes(q) || l.label.toLowerCase().includes(q))
})

// Sheet management
function openSheet(sheet: 'characters' | 'animations' | 'layers' | 'more') {
  if (activeSheet.value === sheet) {
    closeSheet()
    return
  }
  activeSheet.value = sheet
  document.body.classList.add('sheet-open')
}

function closeSheet() {
  activeSheet.value = null
  document.body.classList.remove('sheet-open')
}

function selectCharAndClose(id: string) {
  onSelectCharacter(id)
  closeSheet()
}

function selectAnimAndClose(name: string) {
  onSelectAnimation(name)
  closeSheet()
}

// Category cycling
function cycleCategory() {
  const c = currentChar.value
  if (!c) return
  const categories: Array<'character' | 'ultimate' | 'dating'> = ['character']
  if (c.cutscene) categories.push('ultimate')
  if (c.dating) categories.push('dating')
  const idx = categories.indexOf(store.animationCategory)
  store.animationCategory = categories[(idx + 1) % categories.length]
}

// Speed cycling
const SPEEDS = [0.25, 0.5, 0.75, 1, 1.25, 1.5, 1.75, 2]
function cycleSpeed() {
  const current = store.animationSpeed
  const idx = SPEEDS.findIndex(s => Math.abs(s - current) < 0.001)
  store.animationSpeed = SPEEDS[(idx + 1) % SPEEDS.length]
}

// Layer management
function toggleLayer(key: string) {
  store.layerVisibility[key] = !(store.layerVisibility[key] !== false)
}
function isLayerVisible(key: string) {
  return store.layerVisibility[key] !== false
}
function hideSelectedLayer() {
  if (store.selectedLayerName) {
    store.layerVisibility[store.selectedLayerName] = false
    store.hiddenLayerStack.push(store.selectedLayerName)
    store.selectedLayerName = null
  }
}
function undoHideLayerAction() {
  if (store.hiddenLayerStack.length > 0) {
    const last = store.hiddenLayerStack.pop()!
    store.layerVisibility[last] = true
    store.selectedLayerName = last
  }
}
function clearAllLayers() {
  while (store.hiddenLayerStack.length > 0) {
    const n = store.hiddenLayerStack.pop()!
    store.layerVisibility[n] = true
  }
  store.selectedLayerName = null
}

// Original handlers (keep same logic)
function onSelectCharacter(id: string) {
  if (id === store.selectedCharacterId) return
  store.selectedCharacterId = id
  store.selectedAnimation = ''
  store.selectedSkin = ''
  animations.value = []
  skins.value = []
}

function onSelectAnimation(name: string) {
  store.selectedAnimation = name
}

function onResetCamera() {
  viewerRef.value?.resetCamera()
}

function onZoomIn() {
  viewerRef.value?.zoomIn()
}

function onZoomOut() {
  viewerRef.value?.zoomOut()
}

function onScreenshot(value: boolean) {
  if (!viewerRef.value) return
  closeSheet()
  isScreenshotting.value = true
  viewerRef.value.saveScreenshot(value)
  isScreenshotting.value = false
}

function onScreenshotMenu() {
  onScreenshot(transparentBg.value)
}

async function onExportAnimation({ format, transparent }: { format: 'video' | 'frames'; transparent: boolean }) {
  if (!viewerRef.value) return
  closeSheet()
  isExporting.value = true
  if (format === 'frames') {
    await viewerRef.value.exportAnimationFrames(transparent)
  } else {
    await viewerRef.value.exportAnimation(transparent)
  }
  isExporting.value = false
}

function onCategoryChange() {
  /* desktop sidebar */
}

function onNavMobileMenu(open: boolean) {
  navMobileMenuOpen.value = open
}

function onCustomBgUpload(image: string | null) {
  if (image && image === store.customBackgroundImage) {
    store.customBackgroundImage = null
  }
  store.customBackgroundImage = image
}

function onNavbarOverlayActive(active: boolean) {
  navbarOverlayActive.value = active
}

function handleBgUpload(dataUrl: string | null) {
  if (dataUrl) onCustomBgUpload(dataUrl)
  showBgModal.value = false
}

function handleBgReset() {
  onCustomBgUpload(null)
  showBgModal.value = false
}

function onColorChange(e: Event) {
  const input = e.target as HTMLInputElement
  store.backgroundColor = input.value
}

// Layer selection hint
function clearLayerHint() {
  if (layerSelectionHintTimeout !== null) {
    clearTimeout(layerSelectionHintTimeout)
    layerSelectionHintTimeout = null
  }
}

watch(
  () => store.layerSelectionEnabled,
  enabled => {
    clearLayerHint()
    if (!enabled) {
      showLayerSelectionHint.value = false
      return
    }
    showLayerSelectionHint.value = true
    layerSelectionHintTimeout = window.setTimeout(() => {
      showLayerSelectionHint.value = false
      layerSelectionHintTimeout = null
    }, 4000)
  },
)

onBeforeUnmount(clearLayerHint)

watchEffect(() => {
  const query = buildUrl(store)
  history.replaceState(null, '', `${window.location.pathname}?${query}`)
})
</script>

<style scoped>
/* Transition: toast */
.toast-enter-active { transition: all 0.3s cubic-bezier(0.32, 0.72, 0, 1); }
.toast-leave-active { transition: all 0.2s ease-in; }
.toast-enter-from { opacity: 0; transform: translateY(-8px); }
.toast-leave-to { opacity: 0; transform: translateY(-8px); }
</style>
