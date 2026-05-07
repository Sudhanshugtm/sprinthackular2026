<script setup lang="ts">
import { computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Request',
    description:
      'Mobile prototype mirroring the wiki page where this request lives — RFPP entry for protection cards, talk discussion for speedy cards.',
  },
})

const route = useRoute()
const router = useRouter()

const pageTitle = computed(() => {
  const t = route.query.title
  if (typeof t === 'string' && t.trim()) return t.trim()
  return 'Apple1'
})

type Context = 'protection' | 'speedy'
const contextParam = computed<Context>(() => {
  const c = route.query.context
  if (c === 'speedy') return 'speedy'
  return 'protection'
})

// Canonical "return to dashboard" URL — module mirrors the request context.
const fallbackDashboardRoute = computed(() => ({
  path: '/admin-moderation-dashboard',
  query: {
    variant: 'prototype-v2',
    direction: 'cards-attention',
    protection: 'stale',
    speedy: 'stale',
    module: contextParam.value,
  },
}))

function goBackToDashboard(event: Event) {
  event.preventDefault()
  router.push(fallbackDashboardRoute.value)
}

interface RequestEntry {
  /** Bold lead, e.g. "Temporary semi-protection:" or "Speedy deletion nomination" */
  lead: string
  /** Free-text body the requester / tagger wrote. */
  body: string
  /** Username that signed the request. */
  signedBy: string
  /** Pre-formatted UTC timestamp. */
  signedAt: string
}

interface ProtectionFixture {
  context: 'protection'
  /** Hosting page title at the top — defaults to RFPP. */
  hostTitle: string
  intro: string
  entry: RequestEntry
}

interface SpeedyFixture {
  context: 'speedy'
  /** Talk page title at the top, e.g. "Talk:Smith Industries Holdings Ltd". */
  hostTitle: string
  /** Section heading (the discussion title). */
  sectionTitle: string
  entry: RequestEntry
}

type ArticleFixture = ProtectionFixture | SpeedyFixture

const protectionFixtures: Record<string, RequestEntry> = {
  'Eurovision Song Contest Asia': {
    lead: 'Semi-protection requested for 1 week:',
    body: 'Repeated disruptive edits by multiple new and anonymous editors. Last revert 12 minutes ago. Five distinct accounts active in the past six hours. Diffs in talk.',
    signedBy: 'Reporter1',
    signedAt: '06:30, 6 May 2026 (UTC)',
  },
  'Talk:Blue Whale Challenge': {
    lead: 'Talk-page semi-protection requested indefinitely:',
    body: 'Off-topic suicide-encouragement re-posts after we cleared the page two weeks ago. Same thread keeps coming back via fresh accounts.',
    signedBy: 'Reporter2',
    signedAt: '18:14, 5 May 2026 (UTC)',
  },
  'Narcissistic personality disorder': {
    lead: 'Indefinite semi-protection requested:',
    body: 'Repeated diagnosis-claims about named living people. Diffs supplied below. BLP risk if these revisions remain visible while we clean up.',
    signedBy: 'Reporter3',
    signedAt: '21:22, 5 May 2026 (UTC)',
  },
  'Cardig Air': {
    lead: 'Temporary semi-protection requested:',
    body: 'Persistent IP socking adds unsourced material after each warning. Temp account already blocked but disruption continues from new IPs.',
    signedBy: 'Reporter4',
    signedAt: '14:18, 5 May 2026 (UTC)',
  },
  'Groundhog Day (film)': {
    lead: 'Temporary full protection requested:',
    body: "Talk-page dispute over plot wording. Multiple editors won't accept the consensus version. Brief full protection while we close the discussion.",
    signedBy: 'Reporter5',
    signedAt: '03:11, 6 May 2026 (UTC)',
  },
  '2025-26 Persian Gulf Pro League': {
    lead: 'Semi-protection requested:',
    body: 'Vandalism resumed immediately after the previous protection expired. Same pattern as last cycle (replacing standings with junk on match days).',
    signedBy: 'Reporter6',
    signedAt: '10:14, 6 May 2026 (UTC)',
  },
}

const speedyFixtures: Record<string, RequestEntry> = {
  'Smith Industries Holdings Ltd': {
    lead: 'Speedy deletion nomination — G11 (advertising):',
    body: 'Promotional tone throughout; reads like marketing copy with no neutral coverage. No independent sources cited. Author appears connected to the subject.',
    signedBy: 'NewPagePatroller',
    signedAt: '11:56, 6 May 2026 (UTC)',
  },
  'Regional innovation timeline': {
    lead: 'Speedy deletion nomination — G15 (unreviewed LLM-generated content):',
    body: 'Generated text patterns; references hallucinated; submitted via AfC by author. Tagging because the article is in mainspace despite being clearly LLM-drafted.',
    signedBy: 'AfCBot',
    signedAt: '12:18, 6 May 2026 (UTC)',
  },
  'Henrik Olsen (footballer)': {
    lead: 'Speedy deletion nomination — A1 (no context):',
    body: "Sub-stub: no context, no claims of notability, no sources. Tagged 2 minutes after creation; flagging here so an admin can decide whether to wait for the author to expand.",
    signedBy: 'PatrolBot',
    signedAt: '12:30, 6 May 2026 (UTC)',
  },
  'Talk:Discontinued template (orphan)': {
    lead: 'Speedy deletion nomination — G6 (housekeeping):',
    body: 'Talk page for a template that was deleted seven days ago. Orphaned with no useful discussion to preserve. Routine cleanup.',
    signedBy: 'HousekeepingBot',
    signedAt: '08:11, 5 May 2026 (UTC)',
  },
  Apple1: {
    lead: 'Speedy deletion nomination — G6 (housekeeping):',
    body: 'Test article from local mwdd; placeholder so the prototype renders something for Apple1.',
    signedBy: 'PatrolBot',
    signedAt: '12:00, 6 May 2026 (UTC)',
  },
}

const fixture = computed<ArticleFixture>(() => {
  if (contextParam.value === 'protection') {
    const entry =
      protectionFixtures[pageTitle.value] ??
      ({
        lead: 'Protection requested:',
        body: `Placeholder request body for ${pageTitle.value}.`,
        signedBy: 'Reporter',
        signedAt: '12:00, 6 May 2026 (UTC)',
      } as RequestEntry)
    return {
      context: 'protection',
      hostTitle: 'Project:Requests for page protection',
      intro:
        'This page is for requesting that an administrator protect or unprotect a page to prevent vandalism, edit warring, or other disruption.',
      entry,
    }
  }
  const entry =
    speedyFixtures[pageTitle.value] ??
    ({
      lead: 'Speedy deletion nomination:',
      body: `Placeholder nomination body for ${pageTitle.value}.`,
      signedBy: 'Tagger',
      signedAt: '12:00, 6 May 2026 (UTC)',
    } as RequestEntry)
  return {
    context: 'speedy',
    hostTitle: `Talk:${pageTitle.value}`,
    sectionTitle: 'Speedy deletion nomination',
    entry,
  }
})
</script>

<template>
  <ChromeWrapper skin="mobile">
    <article class="request">
      <h1 class="request__title">{{ fixture.hostTitle }}</h1>

      <p class="request__back">
        &larr; <a href="#" @click="goBackToDashboard">{{ pageTitle }}</a>
      </p>

      <nav class="request__tabs" aria-label="Page tabs">
        <a
          class="request__tab"
          :class="{ 'request__tab--active': fixture.context === 'protection' }"
          href="#"
          @click.prevent
          >Page</a
        >
        <a
          class="request__tab"
          :class="{ 'request__tab--active': fixture.context === 'speedy' }"
          href="#"
          @click.prevent
          >Discussion</a
        >
      </nav>

      <p v-if="fixture.context === 'protection'" class="request__intro">
        {{ fixture.intro }}
      </p>

      <h2 class="request__section-heading">
        <template v-if="fixture.context === 'protection'">{{ pageTitle }}</template>
        <template v-else>{{ fixture.sectionTitle }}</template>
      </h2>

      <div class="request__entry">
        <p>
          <strong>{{ fixture.entry.lead }}</strong>
          {{ fixture.entry.body }}
          <span class="request__signature">
            — <a href="#" @click.prevent>{{ fixture.entry.signedBy }}</a>
            {{ fixture.entry.signedAt }}
          </span>
        </p>
      </div>
    </article>
  </ChromeWrapper>
</template>

<style scoped>
.request {
  box-sizing: border-box;
  padding: var(--spacing-100, 16px);
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
}

.request__title {
  margin: 0 0 var(--spacing-50, 8px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.625rem;
  font-weight: 400;
  line-height: 1.25;
  border: none;
  word-break: break-word;
}

.request__back {
  margin: 0 0 var(--spacing-100, 16px);
  font-size: var(--font-size-medium, 1rem);
}

.request__back a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.request__back a:hover {
  text-decoration: underline;
}

.request__tabs {
  display: flex;
  gap: var(--spacing-150, 24px);
  margin: 0 0 var(--spacing-100, 16px);
  padding-bottom: var(--spacing-25, 4px);
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
  font-size: var(--font-size-medium, 1rem);
  font-weight: var(--font-weight-bold, 700);
}

.request__tab {
  color: var(--color-destructive, #bf3c2c);
  text-decoration: none;
  padding-bottom: var(--spacing-25, 4px);
  border-bottom: 2px solid transparent;
}

.request__tab--active {
  color: var(--color-base, #202122);
  border-bottom-color: var(--color-base, #202122);
}

.request__intro {
  margin: 0 0 var(--spacing-150, 24px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
}

.request__section-heading {
  margin: var(--spacing-100, 16px) 0 var(--spacing-75, 12px);
  padding: 0 0 var(--spacing-25, 4px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.375rem;
  font-weight: 400;
  line-height: 1.3;
  border: none;
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.request__entry {
  margin: 0;
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
}

.request__entry p {
  margin: 0 0 var(--spacing-100, 16px);
}

.request__entry a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.request__entry a:hover {
  text-decoration: underline;
}

.request__signature {
  display: inline;
  margin-left: var(--spacing-25, 4px);
  color: var(--color-subtle, #54595d);
}
</style>
