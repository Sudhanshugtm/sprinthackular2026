<script setup lang="ts">
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { CdxIcon } from '@wikimedia/codex'
import { cdxIconArrowNext } from '@wikimedia/codex-icons'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Action complete',
    description:
      "Mobile prototype destination after delete / decline / protect submits — replaces today's MediaWiki 'Action complete · Return to Main Page' cul-de-sac with a queue-aware confirmation surface.",
  },
})

const route = useRoute()
const router = useRouter()

// Only `deleted` lands here. Protect goes to article.vue with a banner; decline was retired.
function readQueryString(key: string): string {
  const v = route.query[key]
  return typeof v === 'string' ? v : ''
}

const titleParam = computed(() => readQueryString('title') || 'Untitled')
const criterionParam = computed(() => readQueryString('criterion'))

// Headline copy mirrors Wikipedia's native voice — "Page deleted", not "Deleted".
const headline = 'Page deleted'
const confirmationLogLabel = 'View deletion log'
const dashboardModule = 'speedy' as const

interface NextItem {
  title: string
  criterion?: string
  age: string
  contested?: boolean
  flagLabel?: string
  quote: string
}

// Per-criterion next-in-queue mock + remaining-count. Fixture data, not derived
// from index.vue (the brief asks for hardcoded mocks here).
const speedyMocks: Record<string, { count: number; next: NextItem }> = {
  G15: {
    count: 12,
    next: {
      title: 'Smith Industries Holdings Ltd',
      criterion: 'G15',
      age: '2h ago',
      contested: true,
      flagLabel: 'contested',
      quote: 'Generated text patterns; references hallucinated…',
    },
  },
  G11: {
    count: 8,
    next: {
      title: 'Foundry Capital Partners',
      criterion: 'G11',
      age: '1h ago',
      quote: 'Promotional tone throughout; reads like marketing copy with no neutral coverage…',
    },
  },
  A1: {
    count: 4,
    next: {
      title: 'Marius Sundgot (footballer)',
      criterion: 'A1',
      age: '5m ago',
      contested: true,
      flagLabel: 'too new',
      quote: 'Sub-stub: no context, no claims of notability, no sources…',
    },
  },
  G6: {
    count: 17,
    next: {
      title: 'Talk:Decommissioned helper template',
      criterion: 'G6',
      age: '1d ago',
      quote: 'Orphan talk page; target template deleted seven days ago…',
    },
  },
}

const queueData = computed<{ count: number; next: NextItem | null }>(() => {
  const fixture = speedyMocks[criterionParam.value]
  return fixture ?? { count: 0, next: null }
})

const isQueueEmpty = computed(() => queueData.value.next === null)

const nextSectionLabel = computed(
  () => `Next in queue · ${queueData.value.count} left`,
)

// Canonical "return to dashboard" URL — module is fixed since post-action only
// handles delete (= speedy module).
function dashboardRoute() {
  return {
    path: '/admin-moderation-dashboard',
    query: {
      variant: 'prototype-v2',
      direction: 'cards-attention',
      protection: 'stale',
      speedy: 'stale',
      module: dashboardModule,
    },
  }
}

function goToDashboard(event?: Event) {
  if (event) event.preventDefault()
  router.push(dashboardRoute())
}

function openNext(event?: Event) {
  if (event) event.preventDefault()
  const next = queueData.value.next
  if (!next) return
  router.push({
    path: '/admin-moderation-dashboard/article',
    query: {
      title: next.title,
      module: dashboardModule,
    },
  })
}
</script>

<template>
  <ChromeWrapper skin="mobile">
    <article class="post-action">
      <h1 class="post-action__headline">{{ headline }}</h1>

      <p class="post-action__confirmation">
        <strong>&ldquo;{{ titleParam }}&rdquo;</strong> was deleted.
        <a class="post-action__link" href="#" @click.prevent>{{ confirmationLogLabel }}</a>
      </p>

      <div class="post-action__divider" />

      <section class="post-action__next" :aria-label="nextSectionLabel">
        <p class="post-action__eyebrow">{{ nextSectionLabel }}</p>

        <template v-if="!isQueueEmpty && queueData.next">
          <a
            class="post-action__next-row"
            href="#"
            @click="openNext"
          >
            <span class="post-action__next-title">{{ queueData.next.title }}</span>
            <CdxIcon
              class="post-action__next-arrow"
              :icon="cdxIconArrowNext"
              size="medium"
            />
          </a>
          <p class="post-action__next-meta">
            <template v-if="queueData.next.criterion">
              <strong class="post-action__next-criterion">{{ queueData.next.criterion }}</strong>
              <span class="post-action__sep"> · </span>
            </template>
            <span>{{ queueData.next.age }}</span>
            <template v-if="queueData.next.flagLabel">
              <span class="post-action__sep"> · </span>
              <span class="post-action__next-flag">{{ queueData.next.flagLabel }}</span>
            </template>
          </p>
          <p class="post-action__next-quote">&ldquo;{{ queueData.next.quote }}&rdquo;</p>
        </template>

        <template v-else>
          <p class="post-action__empty">
            Queue is clear<template v-if="criterionParam"> · 0 {{ criterionParam }} pages waiting</template>.
          </p>
        </template>
      </section>

      <div class="post-action__divider" />

      <p class="post-action__return">
        Or return to your
        <a class="post-action__link" href="#" @click="goToDashboard">Personal Dashboard</a>
        or the
        <a class="post-action__link" href="#" @click.prevent>main page</a>.
      </p>
    </article>
  </ChromeWrapper>
</template>

<style scoped>
.post-action {
  box-sizing: border-box;
  padding: var(--spacing-100, 16px);
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
  animation: post-action-fade 220ms ease-out;
}

@keyframes post-action-fade {
  from {
    opacity: 0;
    transform: translateY(4px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.post-action__headline {
  margin: 0 0 var(--spacing-75, 12px);
  padding-bottom: var(--spacing-50, 8px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.875rem;
  font-weight: 400;
  line-height: 1.2;
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.post-action__confirmation {
  margin: 0 0 var(--spacing-150, 24px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.6);
  color: var(--color-base, #202122);
}

.post-action__link {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.post-action__link:hover {
  text-decoration: underline;
}

.post-action__divider {
  height: 1px;
  margin: var(--spacing-100, 16px) 0;
  background-color: var(--border-color-subtle, #eaecf0);
}

.post-action__eyebrow {
  margin: 0 0 var(--spacing-75, 12px);
  font-size: var(--font-size-x-small, 0.75rem);
  font-weight: var(--font-weight-bold, 700);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--color-subtle, #54595d);
}

.post-action__next-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-50, 8px);
  margin: 0 0 var(--spacing-50, 8px);
  text-decoration: none;
  color: var(--color-progressive, #36c);
}

.post-action__next-row:hover .post-action__next-title {
  text-decoration: underline;
}

.post-action__next-title {
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.375rem;
  font-weight: 400;
  line-height: 1.3;
  color: var(--color-progressive, #36c);
  word-break: break-word;
}

.post-action__next-arrow {
  flex-shrink: 0;
  color: var(--color-base, #202122);
}

.post-action__next-meta {
  margin: 0 0 var(--spacing-50, 8px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-subtle, #54595d);
}

.post-action__next-criterion {
  font-weight: var(--font-weight-bold, 700);
  color: var(--color-base, #202122);
}

.post-action__sep {
  color: var(--color-subtle, #54595d);
}

.post-action__next-flag {
  color: #ac6600;
}

.post-action__next-quote {
  margin: 0;
  font-style: italic;
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-subtle, #54595d);
  word-break: break-word;
}

.post-action__empty {
  margin: 0;
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-subtle, #54595d);
}

.post-action__return {
  margin: 0;
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.6);
  color: var(--color-subtle, #54595d);
}
</style>
