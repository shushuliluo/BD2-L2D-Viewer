<!-- eslint-disable vue/multi-word-component-names -->
<template>
  <div>
    <nav class="flex items-center justify-between bg-black text-white px-3 lg:px-4 h-[44px] lg:h-[52px] border-b border-white/[0.06]">
      <!-- Brand (abbreviated on mobile) -->
      <div class="text-base lg:text-xl font-bold tracking-tight flex items-center gap-1.5">
        <span class="text-amber-400 hidden sm:inline">⬡</span>
        <span class="hidden sm:inline">BD2 Viewer</span>
        <span class="sm:hidden text-sm">BD2 L2D</span>
      </div>

      <!-- Desktop actions -->
      <div class="hidden md:flex items-center gap-3">
        <button class="cursor-pointer text-gray-400 hover:text-white transition-colors" @click="showUploadModal = true" title="Upload custom Spine model">
          <PlusIcon class="w-5 h-5" />
        </button>
        <button
          class="cursor-pointer text-gray-400 hover:text-white transition-colors"
          @click="openBackgroundModal(false)"
          aria-label="Upload background"
          title="Upload background image"
        >
          <BgUploadIcon class="w-5 h-5" />
        </button>
        <button
          v-if="hasCustomBackground"
          class="cursor-pointer text-gray-400 hover:text-white transition-colors"
          @click="resetBackground"
          aria-label="Reset background"
          title="Reset background image"
        >
          <BgResetIcon class="w-5 h-5" />
        </button>
        <a
          href="https://ko-fi.com/jelosus1"
          target="_blank"
          rel="noopener"
          class="relative text-gray-400 hover:text-white transition-colors"
          title="Support on Ko-fi"
        >
          <KoFiIcon class="w-5 h-5" />
          <div
            v-if="showKofiTooltip"
            :class="[
              'absolute top-full z-10 mt-2 left-1/2 -translate-x-1/2 whitespace-nowrap bg-white text-black text-xs rounded px-2 py-1 shadow transition-opacity duration-500',
              kofiTooltipHidden ? 'opacity-0' : 'opacity-100'
            ]"
          >
            If you like the work consider supporting!
            <span class="absolute left-1/2 -top-2 -translate-x-1/2 border-4 border-transparent border-b-white"></span>
          </div>
        </a>
        <a
          href="https://www.patreon.com/cw/jelosus1"
          target="_blank"
          rel="noopener"
          class="text-gray-400 hover:text-white transition-colors"
          title="Support on Patreon"
        >
          <PatreonIcon class="w-5 h-5" />
        </a>
        <button class="cursor-pointer text-gray-400 hover:text-white transition-colors" @click="showChangelog = true" title="Changelog">
          <ChangelogIcon class="w-5 h-5" />
        </button>
        <a href="https://github.com/Jelosus2/BD2-L2D-Viewer" target="_blank" class="text-gray-400 hover:text-white transition-colors" title="Open GitHub repository">
          <GithubIcon class="w-5 h-5" />
        </a>
      </div>

      <!-- Mobile hamburger -->
      <button
        class="md:hidden cursor-pointer text-gray-400 hover:text-white transition-colors p-1"
        @click="openMobileMenu()"
        aria-label="Menu"
      >
        <MenuIcon class="w-5 h-5" />
      </button>
    </nav>

    <!-- Mobile slide-out menu -->
    <Transition name="mobile-menu">
      <div
        v-if="mobileMenuOpen"
        ref="mobileMenu"
        tabindex="-1"
        class="fixed inset-0 z-50 md:hidden"
      >
        <div class="absolute inset-0 bg-black/60 backdrop-blur-sm" @click="closeMobileMenu" />
        <div class="absolute right-0 top-0 bottom-0 w-64 bg-[#0d0d14] border-l border-white/[0.06] flex flex-col shadow-2xl shadow-black/50">
          <div class="flex items-center justify-between px-4 py-3 border-b border-white/[0.06]">
            <span class="text-sm font-semibold text-gray-400">Menu</span>
            <button class="text-gray-400 hover:text-white text-lg" @click="closeMobileMenu" aria-label="Close menu">✕</button>
          </div>
          <div class="flex-1 flex flex-col gap-0.5 px-2 py-3">
            <button class="flex items-center gap-3 px-3 py-3 rounded-xl hover:bg-white/[0.04] transition-colors text-sm text-gray-300" @click="() => { showUploadModal = true; closeMobileMenu() }">
              <PlusIcon class="w-4 h-4 text-amber-400/80" /><span>Upload Spine</span>
            </button>
            <button class="flex items-center gap-3 px-3 py-3 rounded-xl hover:bg-white/[0.04] transition-colors text-sm text-gray-300" @click="openBackgroundModal(true)">
              <BgUploadIcon class="w-4 h-4 text-amber-400/80" /><span>Upload Background</span>
            </button>
            <button v-if="hasCustomBackground" class="flex items-center gap-3 px-3 py-3 rounded-xl hover:bg-white/[0.04] transition-colors text-sm text-gray-300" @click="() => { resetBackground(); closeMobileMenu() }">
              <BgResetIcon class="w-4 h-4 text-amber-400/80" /><span>Reset Background</span>
            </button>
            <div class="border-t border-white/[0.06] my-2" />
            <button class="flex items-center gap-3 px-3 py-3 rounded-xl hover:bg-white/[0.04] transition-colors text-sm text-gray-300" @click="() => { showChangelog = true; closeMobileMenu() }">
              <ChangelogIcon class="w-4 h-4 text-amber-400/80" /><span>Changelog</span>
            </button>
            <a href="https://github.com/Jelosus2/BD2-L2D-Viewer" target="_blank" rel="noopener" class="flex items-center gap-3 px-3 py-3 rounded-xl hover:bg-white/[0.04] transition-colors text-sm text-gray-300" @click="closeMobileMenu">
              <GithubIcon class="w-4 h-4 text-amber-400/80" /><span>GitHub</span>
            </a>
            <a href="https://ko-fi.com/jelosus1" target="_blank" rel="noopener" class="relative flex items-center gap-3 px-3 py-3 rounded-xl hover:bg-white/[0.04] transition-colors text-sm text-gray-300" @click="closeMobileMenu">
              <KoFiIcon class="w-4 h-4 text-amber-400/80" /><span>Support</span>
            </a>
          </div>
        </div>
      </div>
    </Transition>

    <UploadSpineModal v-if="showUploadModal" @close="showUploadModal = false" />
    <ChangelogModal v-if="showChangelog" @close="showChangelog = false" />
    <UploadBackgroundModal
      v-if="showBackgroundModal"
      :show-reset="hasCustomBackground"
      @close="showBackgroundModal = false"
      @upload-bg="handleBackgroundUpload"
      @reset-bg="handleBackgroundReset"
    />
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'
import ChangelogModal from '@/components/ChangelogModal.vue'
import UploadSpineModal from '@/components/UploadSpineModal.vue'
import UploadBackgroundModal from '@/components/UploadBackgroundModal.vue'

import GithubIcon from '@/components/icons/GithubIcon.vue'
import ChangelogIcon from '@/components/icons/ChangelogIcon.vue'
import PlusIcon from '@/components/icons/PlusIcon.vue'
import MenuIcon from '@/components/icons/MenuIcon.vue'
import KoFiIcon from '@/components/icons/KoFiIcon.vue'
import PatreonIcon from '@/components/icons/PatreonIcon.vue'
import BgUploadIcon from '@/components/icons/BgUploadIcon.vue'
import BgResetIcon from '@/components/icons/BgResetIcon.vue'

const props = defineProps<{ hasCustomBackground?: boolean }>()
const hasCustomBackground = computed(() => !!props.hasCustomBackground)

const showChangelog = ref(false)
const showUploadModal = ref(false)
const showBackgroundModal = ref(false)
const showKofiTooltip = ref(false)
const kofiTooltipHidden = ref(false)
const mobileMenuOpen = ref(false)
const mobileMenu = ref<HTMLElement | null>(null)
let closeMenuAfterBgUpload = false

const emit = defineEmits<{
  (e: 'mobile-menu', open: boolean): void
  (e: 'upload-bg', dataUrl: string | null): void
  (e: 'overlay-active', active: boolean): void
}>()

const openMobileMenu = () => {
  mobileMenuOpen.value = true
  emit('mobile-menu', true)
}

const closeMobileMenu = () => {
  mobileMenuOpen.value = false
  emit('mobile-menu', false)
}

const openBackgroundModal = (fromMobile: boolean) => {
  closeMenuAfterBgUpload = fromMobile
  if (fromMobile && mobileMenuOpen.value) closeMobileMenu()
  showBackgroundModal.value = true
}

const resetBackground = () => {
  closeMenuAfterBgUpload = false
  emit('upload-bg', null)
  showBackgroundModal.value = false
  if (mobileMenuOpen.value) closeMobileMenu()
}

const handleBackgroundUpload = (dataUrl: string | null) => {
  if (dataUrl) emit('upload-bg', dataUrl)
  if (closeMenuAfterBgUpload && mobileMenuOpen.value) closeMobileMenu()
  closeMenuAfterBgUpload = false
  showBackgroundModal.value = false
}

const handleBackgroundReset = () => {
  closeMenuAfterBgUpload = false
  emit('upload-bg', null)
  showBackgroundModal.value = false
}

watch(
  [showChangelog, showUploadModal, showBackgroundModal],
  () => {
    const active = showChangelog.value || showUploadModal.value || showBackgroundModal.value
    emit('overlay-active', active)
  },
  { immediate: true },
)

onMounted(() => {
  if (!localStorage.getItem('kofiPromptSeen') && !window.matchMedia('(max-width: 767px)').matches) {
    showKofiTooltip.value = true
    setTimeout(() => {
      kofiTooltipHidden.value = true
      setTimeout(() => {
        showKofiTooltip.value = false
        localStorage.setItem('kofiPromptSeen', '1')
      }, 500)
    }, 5000)
  }
})
</script>

<style scoped>
.mobile-menu-enter-active { transition: all 0.3s cubic-bezier(0.32, 0.72, 0, 1); }
.mobile-menu-leave-active { transition: all 0.2s ease-in; }
.mobile-menu-enter-from,
.mobile-menu-leave-to { opacity: 0; }
</style>
