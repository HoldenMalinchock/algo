<template>
  <div class="min-h-screen bg-default text-default">
    <header class="border-b border-default">
      <div class="flex items-center justify-between px-4">
        <UNavigationMenu :items="links" />
        <ClientOnly>
          <UButton
            :icon="isDark ? 'i-lucide-sun' : 'i-lucide-moon'"
            color="neutral"
            variant="ghost"
            :aria-label="isDark ? 'Switch to light mode' : 'Switch to dark mode'"
            @click="toggleColorMode"
          />
          <template #fallback>
            <div class="size-8" />
          </template>
        </ClientOnly>
      </div>
    </header>
    <slot />
  </div>
</template>

<script setup lang="ts">
const links = [
  { label: "Algorithm Visualizer", to: "/" },
  { label: "About", to: "/about" },
]

const colorMode = useColorMode()
const isDark = computed(() => colorMode.value === "dark")
const toggleColorMode = () => {
  colorMode.preference = isDark.value ? "light" : "dark"
}
</script>
