<script setup lang="ts">
import { computed, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { CdxButton, CdxIcon } from '@wikimedia/codex'
import { cdxIconExpand, cdxIconUserAvatar } from '@wikimedia/codex-icons'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Revision history',
    description:
      'Mobile prototype mirroring the current Minerva history page, surfaced from the admin moderation dashboard.',
  },
})

const route = useRoute()
const router = useRouter()

const pageTitle = computed(() => {
  const t = route.query.title
  if (typeof t === 'string' && t.trim()) return t.trim()
  return 'Apple1'
})

type Module = 'protection' | 'speedy'
const moduleParam = computed<Module>(() => {
  const m = route.query.module
  if (m === 'speedy') return 'speedy'
  return 'protection'
})

// Canonical "return to dashboard" URL — module is the one the admin is viewing.
const fallbackDashboardRoute = computed(() => ({
  path: '/admin-moderation-dashboard',
  query: {
    variant: 'prototype-v2',
    direction: 'cards-attention',
    protection: 'stale',
    speedy: 'stale',
    module: moduleParam.value,
  },
}))

function goBackToDashboard(event: Event) {
  event.preventDefault()
  router.push(fallbackDashboardRoute.value)
}

// Protect CTA strip — surfaced when admin came from the protection queue.
// Gate on the explicit query param, not the computed default, so a no-module
// URL doesn't accidentally show it.
const showProtectCta = computed(() => route.query.module === 'protection')

function goToProtectForm(event?: Event) {
  if (event) event.preventDefault()
  router.push({
    path: '/admin-moderation-dashboard/protect',
    query: { title: pageTitle.value, module: 'protection' },
  })
}

interface Revision {
  time: string
  user: string
  sizeDelta: number
  summary: string
}

interface DateGroup {
  date: string
  revisions: Revision[]
}

interface ArticleFixture {
  groups: DateGroup[]
}

const articleFixtures: Record<string, ArticleFixture> = {
  Apple1: {
    groups: [
      {
        date: '27 March 2026',
        revisions: [
          {
            time: '15:15',
            user: 'Alice',
            sizeDelta: 516,
            summary:
              'Created page with "The \'\'\'orange\'\'\', also called \'\'\'sweet orange\'\'\' to distinguish it fro...',
          },
        ],
      },
    ],
  },
  'Smith Industries Holdings Ltd': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '11:56',
            user: 'NewPagePatroller',
            sizeDelta: 38,
            summary: 'Tagged for speedy deletion under G11 (advertising)',
          },
          {
            time: '11:42',
            user: 'SmithCorp_Comms',
            sizeDelta: 4218,
            summary:
              'Created page with "\'\'\'Smith Industries Holdings Ltd\'\'\' is a leading global provider of innovative solutions across multiple sectors...',
          },
        ],
      },
    ],
  },
  'Regional innovation timeline': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '12:18',
            user: 'AfCBot',
            sizeDelta: 64,
            summary: 'Tagged for speedy deletion under G15 (unreviewed LLM-generated content)',
          },
          {
            time: '11:02',
            user: 'TimelineDraftBot',
            sizeDelta: 6044,
            summary:
              'Created page with "The \'\'\'Regional innovation timeline\'\'\' presents a comprehensive overview of key milestones..."',
          },
        ],
      },
    ],
  },
  'Henrik Olsen (footballer)': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '12:30',
            user: 'PatrolBot',
            sizeDelta: 28,
            summary: 'Tagged for speedy deletion under A1 (no context)',
          },
          {
            time: '12:28',
            user: 'NewEditor2026',
            sizeDelta: 38,
            summary: 'Created page with "\'\'\'Henrik Olsen\'\'\' is a footballer."',
          },
        ],
      },
    ],
  },
  'Talk:Discontinued template (orphan)': {
    groups: [
      {
        date: '5 May 2026',
        revisions: [
          {
            time: '08:11',
            user: 'HousekeepingBot',
            sizeDelta: 35,
            summary: 'Tagged for speedy deletion under G6 (orphan talk page; template deleted)',
          },
        ],
      },
      {
        date: '5 May 2025',
        revisions: [
          {
            time: '04:55',
            user: 'OldHand',
            sizeDelta: 412,
            summary: 'Discussion archive for {{tl|Discontinued}}',
          },
        ],
      },
    ],
  },
  'Eurovision Song Contest Asia': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '14:02',
            user: '203.0.113.42',
            sizeDelta: -312,
            summary: 'rv',
          },
          {
            time: '13:50',
            user: 'NewVandalAcct',
            sizeDelta: 312,
            summary: '(no edit summary)',
          },
          {
            time: '12:14',
            user: 'AdminBeta',
            sizeDelta: 0,
            summary: 'Reverted edits by 198.51.100.7 to last revision by RegularEditor',
          },
        ],
      },
      {
        date: '5 May 2026',
        revisions: [
          {
            time: '22:48',
            user: 'RegularEditor',
            sizeDelta: 184,
            summary: 'Updated 2026 lineup details',
          },
        ],
      },
    ],
  },
  'Talk:Blue Whale Challenge': {
    groups: [
      {
        date: '5 May 2026',
        revisions: [
          {
            time: '06:30',
            user: 'NewAccountBWC',
            sizeDelta: 612,
            summary: 'New section: invitation thread',
          },
        ],
      },
      {
        date: '21 April 2026',
        revisions: [
          {
            time: '02:12',
            user: 'AdminGamma',
            sizeDelta: -812,
            summary: 'Removed off-topic re-post',
          },
        ],
      },
    ],
  },
  'Narcissistic personality disorder': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '09:21',
            user: '198.51.100.55',
            sizeDelta: 142,
            summary: 'Added unsourced diagnosis claim about [redacted]',
          },
        ],
      },
    ],
  },
  'Cardig Air': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '10:55',
            user: 'CardigSocketAcct',
            sizeDelta: 480,
            summary: 'Added uncited fleet expansion details',
          },
        ],
      },
    ],
  },
  'Groundhog Day (film)': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '08:14',
            user: 'PlotEditor1',
            sizeDelta: 240,
            summary: 'Reworded plot ending — see talk',
          },
          {
            time: '07:51',
            user: 'PlotEditor2',
            sizeDelta: -240,
            summary: 'Reverted plot rewording',
          },
        ],
      },
    ],
  },
  '2025-26 Persian Gulf Pro League': {
    groups: [
      {
        date: '6 May 2026',
        revisions: [
          {
            time: '13:45',
            user: '203.0.113.99',
            sizeDelta: -428,
            summary: 'replaced standings with junk',
          },
        ],
      },
    ],
  },
}

const fixture = computed<ArticleFixture>(() => {
  return articleFixtures[pageTitle.value] ?? { groups: [] }
})

const filterOpen = ref(false)

function formatDelta(d: number) {
  if (d > 0) return `+${d}`
  if (d < 0) return `${d}`
  return '0'
}
function deltaClass(d: number) {
  if (d > 0) return 'history__delta--positive'
  if (d < 0) return 'history__delta--negative'
  return 'history__delta--neutral'
}
</script>

<template>
  <ChromeWrapper skin="mobile">
    <article class="history" :class="{ 'history--has-cta': showProtectCta }">
      <h1 class="history__title">{{ pageTitle }}: Revision history</h1>

      <p class="history__back">
        &larr; <a href="#" @click="goBackToDashboard">{{ pageTitle }}</a>
      </p>

      <nav class="history__tabs" aria-label="Page tabs">
        <a class="history__tab" href="#" @click.prevent>Page</a>
        <a class="history__tab history__tab--inactive" href="#" @click.prevent>Discussion</a>
      </nav>

      <p class="history__logs">
        <a href="#" @click.prevent>View logs for this page</a>
        (<a href="#" @click.prevent>view abuse log</a>)
      </p>

      <button
        class="history__filter-bar"
        type="button"
        :aria-expanded="filterOpen"
        @click="filterOpen = !filterOpen"
      >
        <CdxIcon :icon="cdxIconExpand" size="medium" />
        <span class="history__filter-label">Filter revisions</span>
      </button>

      <div v-if="fixture.groups.length === 0" class="history__empty">
        No revisions to show.
      </div>

      <section
        v-for="group in fixture.groups"
        :key="group.date"
        class="history__date-group"
      >
        <header class="history__date-header">{{ group.date }}</header>
        <ul class="history__revisions">
          <li
            v-for="(rev, idx) in group.revisions"
            :key="idx"
            class="history__revision"
          >
            <div class="history__revision-meta">
              <span class="history__time">{{ rev.time }}</span>
              <CdxIcon :icon="cdxIconUserAvatar" size="medium" />
              <a class="history__user" href="#" @click.prevent>{{ rev.user }}</a>
            </div>
            <div class="history__revision-body">
              <span class="history__delta" :class="deltaClass(rev.sizeDelta)">{{
                formatDelta(rev.sizeDelta)
              }}</span>
              <span class="history__summary">{{ rev.summary }}</span>
            </div>
          </li>
        </ul>
      </section>

      <!--
        Protect CTA strip — only when the admin arrived from the protection
        queue. Sits below the revision list so the admin reads the history first
        and acts after.
      -->
      <section
        v-if="showProtectCta"
        class="history__protect-cta"
        role="region"
        aria-label="Protection actions"
      >
        <CdxButton
          class="history__protect-cta-primary"
          weight="primary"
          action="progressive"
          size="medium"
          @click="goToProtectForm"
        >
          Protect this page
        </CdxButton>
      </section>
    </article>
  </ChromeWrapper>
</template>

<style scoped>
.history {
  box-sizing: border-box;
  padding: var(--spacing-100, 16px);
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
}

.history__title {
  margin: 0 0 var(--spacing-50, 8px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.75rem;
  font-weight: 400;
  line-height: 1.25;
  border: none;
}

.history__back {
  margin: 0 0 var(--spacing-100, 16px);
  font-size: var(--font-size-medium, 1rem);
}

.history__back a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.history__back a:hover {
  text-decoration: underline;
}

.history__tabs {
  display: flex;
  gap: var(--spacing-150, 24px);
  margin: 0 0 var(--spacing-100, 16px);
  padding-bottom: var(--spacing-25, 4px);
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
  font-size: var(--font-size-medium, 1rem);
  font-weight: var(--font-weight-bold, 700);
}

.history__tab {
  color: var(--color-base, #202122);
  text-decoration: none;
  padding-bottom: var(--spacing-25, 4px);
  border-bottom: 2px solid var(--color-base, #202122);
}

.history__tab--inactive {
  color: var(--color-destructive, #bf3c2c);
  border-bottom-color: transparent;
}

.history__logs {
  margin: 0 0 var(--spacing-100, 16px);
  font-size: var(--font-size-medium, 1rem);
}

.history__logs a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.history__logs a:hover {
  text-decoration: underline;
}

.history__filter-bar {
  display: flex;
  align-items: center;
  gap: var(--spacing-75, 12px);
  width: 100%;
  margin: 0 0 var(--spacing-100, 16px);
  padding: var(--spacing-75, 12px) var(--spacing-100, 16px);
  background: var(--background-color-base, #fff);
  border: 1px solid var(--border-color-base, #c8ccd1);
  border-radius: var(--border-radius-base, 2px);
  cursor: pointer;
  font: inherit;
  text-align: left;
}

.history__filter-label {
  font-weight: var(--font-weight-bold, 700);
}

.history__empty {
  margin: var(--spacing-150, 24px) 0;
  color: var(--color-subtle, #54595d);
}

.history__date-group {
  margin-bottom: var(--spacing-100, 16px);
}

.history__date-header {
  margin-inline: calc(-1 * var(--spacing-100, 16px));
  padding: var(--spacing-50, 8px) var(--spacing-100, 16px);
  background: var(--background-color-neutral-subtle, #f8f9fa);
  font-weight: var(--font-weight-bold, 700);
  color: var(--color-subtle, #54595d);
}

.history__revisions {
  list-style: none;
  margin: 0;
  padding: 0;
}

.history__revision {
  padding: var(--spacing-75, 12px) 0;
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.history__revision-meta {
  display: flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  margin-bottom: var(--spacing-25, 4px);
  font-size: var(--font-size-medium, 1rem);
}

.history__time {
  font-variant-numeric: tabular-nums;
  color: var(--color-base, #202122);
}

.history__user {
  color: var(--color-base, #202122);
  text-decoration: none;
}

.history__revision-body {
  display: flex;
  align-items: baseline;
  gap: var(--spacing-75, 12px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
  padding-left: var(--spacing-200, 32px);
}

.history__delta {
  font-variant-numeric: tabular-nums;
  font-weight: var(--font-weight-bold, 700);
  flex-shrink: 0;
  min-width: 3em;
}

.history__delta--positive {
  color: var(--color-success, #14866d);
}

.history__delta--negative {
  color: var(--color-destructive, #bf3c2c);
}

.history__delta--neutral {
  color: var(--color-subtle, #54595d);
}

.history__summary {
  color: var(--color-base, #202122);
  word-break: break-word;
}

/*
 * Protect CTA strip — pinned to the viewport bottom while module=protection so
 * the button stays in reach as the admin scrolls through the revision list.
 * Article body gets bottom padding (.history--has-cta) to keep the last
 * revision row from sliding under the strip.
 */
.history__protect-cta {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 50;
  display: flex;
  flex-direction: column;
  gap: var(--spacing-75, 12px);
  padding: var(--spacing-100, 16px);
  padding-bottom: calc(var(--spacing-100, 16px) + env(safe-area-inset-bottom, 0));
  background-color: var(--background-color-base, #fff);
  border-top: 1px solid var(--border-color-subtle, #eaecf0);
  box-shadow: 0 -2px 8px rgba(0, 0, 0, 0.06);
}

.history__protect-cta-primary {
  align-self: stretch;
}

.history--has-cta {
  padding-bottom: calc(var(--spacing-200, 32px) + 64px + env(safe-area-inset-bottom, 0));
}
</style>
