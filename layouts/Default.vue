<template>
  <div>
    <UHorizontalNavigation
      :links="links"
      class="border-b border-gray-200 dark:border-gray-800"
    />
    <slot />
  </div>
</template>

<script setup lang="ts">
const links = [{
  label: "Algorithm Visualizer",
  to: "/",
}, {
  label: "About",
  to: "/about",
}]
</script>
