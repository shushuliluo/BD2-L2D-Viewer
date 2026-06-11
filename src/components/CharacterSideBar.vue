<template>
  <div class="w-full lg:w-72 h-full flex flex-col min-h-0"
       style="background: linear-gradient(180deg, rgba(18,18,26,0.96) 0%, rgba(10,10,15,1) 100%); border-left: 1px solid rgba(255,255,255,0.06);">
    <div class="px-3 pt-3 pb-2">
      <input
        v-model="filter"
        type="text"
        placeholder="Search characters..."
        class="w-full bg-white/5 border border-white/10 rounded-xl px-4 py-2.5 text-sm text-white placeholder:text-gray-500 outline-none focus:border-amber-500/40 transition-colors"
      />
    </div>
    <div class="overflow-y-auto flex-1 px-2 pb-1 sidebar-scroll">
      <div
        v-for="char in filteredCharacters"
        :key="char.id"
        class="flex items-center gap-3 px-2 py-2 cursor-pointer rounded-xl mb-1 transition-all border"
        :class="[
          char.id === store.selectedCharacterId
            ? 'border-amber-500/30 bg-amber-500/8'
            : 'border-transparent hover:bg-white/[0.03]'
        ]"
        @click="select(char.id)"
      >
        <div class="relative flex-shrink-0">
          <img
            :src="icons[char.icon] || icons['unknown']"
            :alt="char.costumeName"
            class="w-11 h-11 object-cover rounded-full border-2"
            :class="char.id === store.selectedCharacterId ? 'border-amber-500/60 shadow-md shadow-amber-500/20' : 'border-white/10'"
            loading="lazy"
          />
          <div v-if="char.id === store.selectedCharacterId" class="absolute -bottom-0.5 -right-0.5 w-3.5 h-3.5 bg-amber-500 rounded-full border-2 border-black flex items-center justify-center">
            <span class="text-[6px] text-black font-bold">✓</span>
          </div>
        </div>
        <div class="flex-grow min-w-0">
          <div class="text-xs font-medium text-gray-200 truncate">{{ char.charName }}</div>
          <div class="text-[10px] text-gray-500 truncate">{{ char.costumeName }}</div>
        </div>
        <div class="flex flex-shrink-0 gap-1">
          <div v-if="char.dating" class="w-auto h-5 px-1.5 bg-blue-500/15 text-blue-400 flex items-center justify-center text-[8px] font-bold rounded border border-blue-500/20">FG</div>
          <div v-if="char.cutscene" class="w-auto h-5 px-1.5 bg-purple-500/15 text-purple-400 flex items-center justify-center text-[8px] font-bold rounded border border-purple-500/20">ULT</div>
        </div>
      </div>
      <div v-if="filteredCharacters.length === 0" class="text-center text-gray-500 text-xs py-8">
        No characters match
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import icons from '@/utils/charIcons'
import { ref, computed, onMounted } from 'vue'
import { useCharacterStore } from '@/stores/characterStore'

const emit = defineEmits(['select'])
const store = useCharacterStore()

const filter = ref('')

const filteredCharacters = computed(() =>
  store.characters.filter((c) =>
    (c.charName + ' ' + c.costumeName)
      .toLowerCase()
      .includes(filter.value.toLowerCase())
  )
)

function select(id: string) {
  if (id === store.selectedCharacterId) return
  emit('select', id)
  store.selectedCharacterId = id
}

onMounted(() => {
  emit('select', store.selectedCharacterId)
})
</script>
