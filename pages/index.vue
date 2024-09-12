<template>
  <div class="flex align-middle justify-center w-2/3">
    <div class="flex flex-wrap">
      <div
        v-for="item, index in 513"
        :key="index"
        class="h-10 w-10 border-2 border-slate-700"
        :class="[{ ['bg-cyan-700']: toggled.includes(index) }, { ['bg-stone-700']: start === index }, { ['bg-slate-700']: end === index }]"
        @click="setVariables(index)"
      />
    </div>
    <UButton
      :disabled="start === null || end === null"
      @click="startAlgorithm"
    >
      Start
    </UButton>
  </div>
</template>

<script setup lang="ts">
const toggled = ref<number[]>([])

const start = ref<number | null>(null)
const end = ref<number | null>(null)

const setVariables = (index: number) => {
  if (start.value === null) {
    start.value = index
  }
  else if (start.value === index) {
    start.value = null
  }
  else if (end.value === null) {
    end.value = index
  }
  else if (end.value === index) {
    end.value = null
  }
}

// I dont think flex wrap is the best way to do this, we need it to be a more structured grid we need a real 100 by 100 or 130 by 130
const startAlgorithm = () => {
  console.log("Start algorithm")
  // This code needs to take the start and end variables and run a search and populate other squares as it goes
  // The squares should be populated in the order they are visited
  // The squares should be populated in a different color
}
</script>
