<script setup lang="ts">
import { computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'

definePage({
  meta: {
    title: 'Decline (parked)',
    description:
      "Decline as a discrete admin action was retired — declining a speedy is just removing the {{db-…}} template, which is a regular page edit. This route now silently forwards to the article view so any leftover links don't dead-end.",
  },
})

const route = useRoute()
const router = useRouter()

const titleParam = computed(() => {
  const t = route.query.title
  return typeof t === 'string' && t.trim() ? t.trim() : 'Apple1'
})

type Module = 'protection' | 'speedy'
const moduleParam = computed<Module>(() => {
  // decline.vue used to take ?context=… so honour either spelling on the way out.
  const ctx = route.query.context
  const mod = route.query.module
  const value = typeof ctx === 'string' ? ctx : typeof mod === 'string' ? mod : ''
  return value === 'protection' ? 'protection' : 'speedy'
})

onMounted(() => {
  router.replace({
    path: '/admin-moderation-dashboard/article',
    query: {
      title: titleParam.value,
      module: moduleParam.value,
    },
  })
})
</script>

<template>
  <p class="decline-stub">Redirecting to article view&hellip;</p>
</template>

<style scoped>
.decline-stub {
  margin: 0;
  padding: var(--spacing-100, 16px);
  font-size: var(--font-size-small, 0.875rem);
  color: var(--color-subtle, #54595d);
}
</style>
