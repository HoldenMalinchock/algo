<template>
  <div class="flex align-middle justify-center w-2/3 flex-row">
    <div class="flex">
      <div
        v-for="row, index in squares"
        :key="processed + index"
      >
        <div
          v-for="square, secondary in row"
          :key="square"
          class="h-10 w-10 border-2 border-slate-700"
          :class="[{ ['bg-slate-200 animate-pulse']: visited.some(location => arraysEqual([index, secondary], location)) && !arraysEqual(start, [index, secondary]) && !arraysEqual(end, [index, secondary]) }, { ['bg-primary-700']: arraysEqual(start, [index, secondary]) }, { ['bg-blue-500']: arraysEqual(end, [index, secondary]) }]"
          @click="setVariables(index, secondary)"
        />
      </div>
    </div>
    <div class="">
      <UCard>
        <UButton
          :disabled="start === null || end === null || processing"
          @click="startAlgorithm"
        >
          Start
        </UButton>
      </UCard>
    </div>
  </div>
</template>

<script setup lang="ts">
// We are going to make a 2D array and update this to render the squares for our table so we can more easily set the data as expected
const rows = 15
const columns = 15
const squares = Array.from(Array(rows), () => new Array(columns).fill(0))

const start = ref<number[] | null>(null)
const end = ref<number[] | null>(null)

const visited = []
const processed = ref(1000)
const processing = ref(false)

const setVariables = (index: number, secondary: number) => {
  if (start.value === null) {
    start.value = [index, secondary]
  }
  else if (arraysEqual(start.value, [index, secondary])) {
    start.value = null
  }
  else if (end.value === null) {
    end.value = [index, secondary]
  }
  else if (arraysEqual(end.value, [index, secondary])) {
    end.value = null
  }
}

const arraysEqual = (a: number[] | null, b: number[] | null) => {
  if (a === b) return true
  if (a == null || b == null) return false
  if (a.length !== b.length) return false

  // If you don't care about the order of the elements inside
  // the array, you should sort both arrays here.
  // Please note that calling sort on an array will modify that array.
  // you might want to clone your array first.

  for (let i = 0; i < a.length; ++i) {
    if (a[i] !== b[i]) return false
  }
  return true
}

const sleep = delay => new Promise(resolve => setTimeout(resolve, delay))

// We need to think about the data structure like a 2D array and render that
// I dont think flex wrap is the best way to do this, we need it to be a more structured grid we need a real 100 by 100 or 130 by 130
const startAlgorithm = async () => {
  // Perform dijkstra's algorithm on the 2d array of squares from start to end
  // We need to make sure that we are able to update the squares array with the correct values

  // Since all directions have a location of 1 we are going to look in all 4 directions
  const queue = []
  const directions = [[0, 1], [1, 0], [0, -1], [-1, 0]]
  processing.value = true

  // Ok the logic here is take the starting location is to add it to the queue
  // Then process the node, processing the node checks if we are at the end location
  // If we are at the end location we are done
  // If we are not at the end location we need to add the neighbors to the queue
  // We need to make sure that we are not adding the same node to the queue
  // We need to make sure that we are not adding the same node to the visited array

  queue.push(start.value)

  // We need to optimize this a little for the display as it is really slow to display
  while (queue.length > 0) {
    await sleep(1)
    const currentLocation = queue.shift()
    visited.push(currentLocation)
    processed.value++
    // found the end
    if (arraysEqual(currentLocation, end.value)) {
      processing.value = false
      break
    }

    for (const direction of directions) {
      const newLocation = [currentLocation[0] + direction[0], currentLocation[1] + direction[1]]

      if (newLocation[0] < 0 || newLocation[0] >= rows || newLocation[1] < 0 || newLocation[1] >= columns) {
        continue
      }

      if (visited.some(location => arraysEqual(location, newLocation))) {
        continue
      }

      queue.push(newLocation)
      processed.value++
    }
  }
}
</script>
