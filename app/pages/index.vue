<template>
  <div class="mx-auto max-w-6xl px-4 py-10">
    <header class="mb-8 flex flex-col items-center text-center">
      <h1 class="text-4xl font-bold tracking-tight sm:text-5xl">
        Algorithm <span class="text-primary-500">Visualizer</span>
      </h1>
      <p class="mt-3 max-w-xl text-muted">
        Place start and end, draw walls and weighted terrain, then run BFS or Dijkstra and compare.
        BFS minimizes <em>steps</em>; Dijkstra minimizes <em>total cost</em> — so weighted cells
        make the two diverge.
      </p>
    </header>

    <UCard class="mb-4">
      <div class="mb-3 flex flex-wrap items-center gap-2">
        <span class="text-xs uppercase tracking-wide text-muted mr-1">Click to place</span>
        <UButton
          v-for="opt in tools"
          :key="opt.value"
          size="xs"
          :color="tool === opt.value ? 'primary' : 'neutral'"
          :variant="tool === opt.value ? 'solid' : 'subtle'"
          :icon="opt.icon"
          @click="tool = opt.value"
        >
          {{ opt.label }}
        </UButton>
        <span class="ml-2 text-xs text-muted">
          {{ toolHint }}
        </span>
      </div>

      <div class="flex justify-center overflow-x-auto">
        <div
          class="grid gap-px rounded-md bg-accented p-2 select-none"
          :style="gridStyle"
          @mouseleave="painting = null"
        >
          <button
            v-for="[c, r] in cells"
            :key="`${c}-${r}`"
            type="button"
            class="h-7 w-7 rounded-sm transition-colors duration-150"
            :class="squareClass(c, r)"
            :aria-label="cellLabel(c, r)"
            @mousedown.prevent="onCellMouseDown(c, r)"
            @mouseenter="onCellMouseEnter(c, r)"
            @mouseup="painting = null"
          />
        </div>
      </div>
    </UCard>

    <UCard>
      <div class="flex flex-wrap items-center justify-between gap-4">
        <div class="flex flex-wrap items-center gap-3">
          <UButton
            color="primary"
            icon="i-lucide-play"
            :disabled="start === null || end === null || processing"
            @click="startAlgorithm"
          >
            Start
          </UButton>
          <UButton
            color="neutral"
            variant="outline"
            icon="i-lucide-eraser"
            :disabled="processing"
            @click="clearSearch"
          >
            Clear search
          </UButton>
          <UButton
            color="neutral"
            variant="outline"
            icon="i-lucide-rotate-ccw"
            :disabled="processing"
            @click="reset"
          >
            Reset board
          </UButton>
          <USelect
            v-model="algorithm"
            :items="algorithms"
            class="w-40"
          />
        </div>
        <div class="flex flex-wrap items-center gap-x-4 gap-y-2 text-sm">
          <span class="flex items-center gap-2">
            <span class="size-3 rounded-sm bg-primary-500" />
            <span class="text-muted">Start</span>
          </span>
          <span class="flex items-center gap-2">
            <span class="size-3 rounded-sm bg-rose-500" />
            <span class="text-muted">End</span>
          </span>
          <span class="flex items-center gap-2">
            <span class="size-3 rounded-sm bg-neutral-700 dark:bg-neutral-300" />
            <span class="text-muted">Wall</span>
          </span>
          <span class="flex items-center gap-2">
            <span class="size-3 rounded-sm bg-amber-300 dark:bg-amber-600" />
            <span class="text-muted">Weighted (cost {{ weightCost }})</span>
          </span>
          <span class="flex items-center gap-2">
            <span class="size-3 rounded-sm bg-primary-300 dark:bg-primary-700" />
            <span class="text-muted">Visited</span>
          </span>
          <span class="flex items-center gap-2">
            <span class="size-3 rounded-sm bg-emerald-400" />
            <span class="text-muted">Path</span>
          </span>
        </div>
      </div>

      <div class="mt-5 grid grid-cols-2 gap-3 sm:grid-cols-4">
        <div class="rounded-md border border-default p-3">
          <div class="text-xs uppercase tracking-wide text-muted">Status</div>
          <div class="mt-1 text-sm font-medium">{{ statusText }}</div>
        </div>
        <div class="rounded-md border border-default p-3">
          <div class="text-xs uppercase tracking-wide text-muted">Visited</div>
          <div class="mt-1 text-sm font-medium">{{ visited.length }}</div>
        </div>
        <div class="rounded-md border border-default p-3">
          <div class="text-xs uppercase tracking-wide text-muted">Path length</div>
          <div class="mt-1 text-sm font-medium">{{ path.length || "—" }}</div>
        </div>
        <div class="rounded-md border border-default p-3">
          <div class="text-xs uppercase tracking-wide text-muted">Path cost</div>
          <div class="mt-1 text-sm font-medium">{{ pathCost || "—" }}</div>
        </div>
      </div>
    </UCard>
  </div>
</template>

<script setup lang="ts">
type Cell = [number, number]
type Tool = "endpoints" | "wall" | "weight"

const columns = 30
const rows = 15
const weightCost = 5

const cells = computed<Cell[]>(() => {
  const out: Cell[] = []
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < columns; c++) {
      out.push([c, r])
    }
  }
  return out
})

const gridStyle = computed(() => ({
  gridTemplateColumns: `repeat(${columns}, minmax(0, 1fr))`,
}))

const start = ref<Cell | null>(null)
const end = ref<Cell | null>(null)
const walls = reactive<Set<string>>(new Set())
const weights = reactive<Set<string>>(new Set())
const visited = reactive<Cell[]>([])
const path = reactive<Cell[]>([])
const pathCost = ref(0)
const processing = ref(false)
const found = ref(false)

const algorithms = ["bfs", "dijkstra"]
const algorithm = ref(algorithms[0])

const tools: { value: Tool, label: string, icon: string }[] = [
  { value: "endpoints", label: "Start / End", icon: "i-lucide-flag" },
  { value: "wall", label: "Wall", icon: "i-lucide-square" },
  { value: "weight", label: "Weight", icon: "i-lucide-weight" },
]
const tool = ref<Tool>("endpoints")
const painting = ref<"add" | "remove" | null>(null)

const toolHint = computed(() => {
  if (tool.value === "endpoints") return "click to set start, then end. Click again to clear."
  if (tool.value === "wall") return "click and drag — impassable to both algorithms."
  return `click and drag — passable but costs ${weightCost} (BFS ignores cost, Dijkstra routes around).`
})

const statusText = computed(() => {
  if (processing.value) return path.length ? "Drawing path…" : "Searching…"
  if (found.value) return "Path found"
  if (start.value && end.value && visited.length && !found.value) return "No path"
  if (!start.value) return "Click to set start"
  if (!end.value) return "Click to set end"
  return "Ready"
})

const cellKey = (c: number, r: number) => `${c},${r}`
const eq = (a: Cell | null, b: Cell | null) =>
  !!a && !!b && a[0] === b[0] && a[1] === b[1]

const visitedSet = computed(() => new Set(visited.map(([c, r]) => cellKey(c, r))))
const pathSet = computed(() => new Set(path.map(([c, r]) => cellKey(c, r))))

const cellLabel = (c: number, r: number) => {
  if (eq(start.value, [c, r])) return "Start"
  if (eq(end.value, [c, r])) return "End"
  const key = cellKey(c, r)
  if (walls.has(key)) return `Wall at ${c}, ${r}`
  if (weights.has(key)) return `Weighted at ${c}, ${r}`
  return `Cell ${c}, ${r}`
}

const squareClass = (c: number, r: number) => {
  const key = cellKey(c, r)
  const isStart = eq(start.value, [c, r])
  const isEnd = eq(end.value, [c, r])
  const onPath = pathSet.value.has(key)
  const visitedHere = visitedSet.value.has(key)
  const isWall = walls.has(key)
  const isWeight = weights.has(key)

  if (isStart) return "bg-primary-500 shadow shadow-primary-500/40"
  if (isEnd) return "bg-rose-500 shadow shadow-rose-500/40"
  if (isWall) return "bg-neutral-700 dark:bg-neutral-300"
  if (onPath) return "bg-emerald-400 path-pop"
  if (visitedHere && isWeight) return "bg-amber-500/80 square-pop"
  if (visitedHere) return "bg-primary-300 dark:bg-primary-700 square-pop"
  if (isWeight) return "bg-amber-300 dark:bg-amber-600 hover:brightness-110 cursor-pointer"
  return "bg-default hover:bg-elevated cursor-pointer"
}

// ---- Painting ----

const placeEndpoint = (cell: Cell) => {
  const key = cellKey(cell[0], cell[1])
  if (walls.has(key) || weights.has(key)) return
  if (start.value === null) {
    if (eq(end.value, cell)) end.value = null
    start.value = cell
  }
  else if (eq(start.value, cell)) {
    start.value = null
  }
  else if (end.value === null) {
    end.value = cell
  }
  else if (eq(end.value, cell)) {
    end.value = null
  }
  else {
    start.value = cell
    end.value = null
  }
}

const paintTerrain = (cell: Cell, mode: "add" | "remove") => {
  if (eq(start.value, cell) || eq(end.value, cell)) return
  const key = cellKey(cell[0], cell[1])
  const target = tool.value === "wall" ? walls : weights
  const other = tool.value === "wall" ? weights : walls
  if (mode === "add") {
    other.delete(key)
    target.add(key)
  }
  else {
    target.delete(key)
  }
}

const onCellMouseDown = (c: number, r: number) => {
  if (processing.value) return
  if (visited.length || path.length) clearSearch()

  const cell: Cell = [c, r]
  if (tool.value === "endpoints") {
    placeEndpoint(cell)
    return
  }

  const key = cellKey(c, r)
  const target = tool.value === "wall" ? walls : weights
  painting.value = target.has(key) ? "remove" : "add"
  paintTerrain(cell, painting.value)
}

const onCellMouseEnter = (c: number, r: number) => {
  if (!painting.value || processing.value) return
  if (tool.value === "endpoints") return
  paintTerrain([c, r], painting.value)
}

// ---- Algorithms ----

const sleep = (ms: number) => new Promise(resolve => setTimeout(resolve, ms))
const directions: Cell[] = [[0, 1], [1, 0], [0, -1], [-1, 0]]

const reconstruct = (parents: Map<string, Cell>): Cell[] => {
  if (!end.value || !start.value) return []
  const trail: Cell[] = []
  let cur: Cell | undefined = end.value
  while (cur && !eq(cur, start.value)) {
    trail.push(cur)
    cur = parents.get(cellKey(cur[0], cur[1]))
  }
  if (cur) trail.push(cur)
  trail.reverse()
  return trail
}

const drawPath = async (trail: Cell[]) => {
  let cost = 0
  for (const step of trail) {
    await sleep(35)
    path.push(step)
    if (eq(step, start.value)) continue
    cost += weights.has(cellKey(step[0], step[1])) ? weightCost : 1
  }
  pathCost.value = cost
}

const doBFS = async () => {
  if (!start.value || !end.value) return
  const queue: Cell[] = [start.value]
  const parents = new Map<string, Cell>()
  const seen = new Set<string>([cellKey(start.value[0], start.value[1])])

  while (queue.length > 0) {
    await sleep(6)
    const current = queue.shift() as Cell
    visited.push(current)

    if (eq(current, end.value)) {
      found.value = true
      break
    }

    for (const [dc, dr] of directions) {
      const next: Cell = [current[0] + dc, current[1] + dr]
      if (next[0] < 0 || next[0] >= columns || next[1] < 0 || next[1] >= rows) continue
      const key = cellKey(next[0], next[1])
      if (seen.has(key) || walls.has(key)) continue
      seen.add(key)
      parents.set(key, current)
      queue.push(next)
    }
  }

  if (found.value) await drawPath(reconstruct(parents))
}

const doDijkstra = async () => {
  if (!start.value || !end.value) return
  const startKey = cellKey(start.value[0], start.value[1])
  const dist = new Map<string, number>([[startKey, 0]])
  const parents = new Map<string, Cell>()
  const frontier: { cell: Cell, cost: number }[] = [{ cell: start.value, cost: 0 }]
  const settled = new Set<string>()

  while (frontier.length > 0) {
    // Linear-scan extract-min — grid is small, no need for a heap.
    let minIdx = 0
    for (let i = 1; i < frontier.length; i++) {
      if (frontier[i]!.cost < frontier[minIdx]!.cost) minIdx = i
    }
    const { cell: current, cost } = frontier.splice(minIdx, 1)[0]!
    const key = cellKey(current[0], current[1])
    if (settled.has(key)) continue
    settled.add(key)

    await sleep(6)
    visited.push(current)

    if (eq(current, end.value)) {
      found.value = true
      break
    }

    for (const [dc, dr] of directions) {
      const next: Cell = [current[0] + dc, current[1] + dr]
      if (next[0] < 0 || next[0] >= columns || next[1] < 0 || next[1] >= rows) continue
      const nKey = cellKey(next[0], next[1])
      if (walls.has(nKey) || settled.has(nKey)) continue

      const step = weights.has(nKey) ? weightCost : 1
      const tentative = cost + step
      const known = dist.get(nKey)
      if (known === undefined || tentative < known) {
        dist.set(nKey, tentative)
        parents.set(nKey, current)
        frontier.push({ cell: next, cost: tentative })
      }
    }
  }

  if (found.value) await drawPath(reconstruct(parents))
}

const startAlgorithm = async () => {
  if (processing.value) return
  clearSearch()
  processing.value = true
  if (algorithm.value === "bfs") await doBFS()
  else if (algorithm.value === "dijkstra") await doDijkstra()
  processing.value = false
}

const clearSearch = () => {
  visited.splice(0, visited.length)
  path.splice(0, path.length)
  pathCost.value = 0
  found.value = false
}

const reset = () => {
  start.value = null
  end.value = null
  walls.clear()
  weights.clear()
  clearSearch()
  processing.value = false
}
</script>
