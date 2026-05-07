<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { CdxButton, CdxIcon } from '@wikimedia/codex'
import { cdxIconClose, cdxIconNext } from '@wikimedia/codex-icons'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Decline request',
    description:
      'Mobile prototype of the deep-linked section editor with the decline template pre-pasted.',
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

interface SectionFixture {
  /** Section heading rendered in the editor (between === === for protection, ==== for speedy talk). */
  heading: string
  /** Body wikitext for that section. */
  body: string
}

// Same per-article content as request.vue, kept in sync so the editor view matches
// what an admin would see when they open the section in the wiki source editor.
const protectionSections: Record<string, SectionFixture> = {
  'Eurovision Song Contest Asia': {
    heading: '[[Eurovision Song Contest Asia]]',
    body: "'''Semi-protection requested for 1 week:''' Repeated disruptive edits by multiple new and anonymous editors. Last revert 12 minutes ago. Five distinct accounts active in the past six hours. Diffs in talk. — User:Reporter1 06:30, 6 May 2026 (UTC)",
  },
  'Talk:Blue Whale Challenge': {
    heading: '[[Talk:Blue Whale Challenge]]',
    body: "'''Talk-page semi-protection requested indefinitely:''' Off-topic suicide-encouragement re-posts after we cleared the page two weeks ago. Same thread keeps coming back via fresh accounts. — User:Reporter2 18:14, 5 May 2026 (UTC)",
  },
  'Narcissistic personality disorder': {
    heading: '[[Narcissistic personality disorder]]',
    body: "'''Indefinite semi-protection requested:''' Repeated diagnosis-claims about named living people. Diffs supplied below. BLP risk if these revisions remain visible while we clean up. — User:Reporter3 21:22, 5 May 2026 (UTC)",
  },
  'Cardig Air': {
    heading: '[[Cardig Air]]',
    body: "'''Temporary semi-protection requested:''' Persistent IP socking adds unsourced material after each warning. Temp account already blocked but disruption continues from new IPs. — User:Reporter4 14:18, 5 May 2026 (UTC)",
  },
  'Groundhog Day (film)': {
    heading: '[[Groundhog Day (film)]]',
    body: "'''Temporary full protection requested:''' Talk-page dispute over plot wording. Multiple editors won't accept the consensus version. Brief full protection while we close the discussion. — User:Reporter5 03:11, 6 May 2026 (UTC)",
  },
  '2025-26 Persian Gulf Pro League': {
    heading: '[[2025-26 Persian Gulf Pro League]]',
    body: "'''Semi-protection requested:''' Vandalism resumed immediately after the previous protection expired. Same pattern as last cycle (replacing standings with junk on match days). — User:Reporter6 10:14, 6 May 2026 (UTC)",
  },
}

const speedySections: Record<string, SectionFixture> = {
  'Smith Industries Holdings Ltd': {
    heading: 'Speedy deletion nomination',
    body: "'''G11 (advertising):''' Promotional tone throughout; reads like marketing copy with no neutral coverage. No independent sources cited. Author appears connected to the subject. — User:NewPagePatroller 11:56, 6 May 2026 (UTC)",
  },
  'Regional innovation timeline': {
    heading: 'Speedy deletion nomination',
    body: "'''G15 (unreviewed LLM-generated content):''' Generated text patterns; references hallucinated; submitted via AfC by author. Tagging because the article is in mainspace despite being clearly LLM-drafted. — User:AfCBot 12:18, 6 May 2026 (UTC)",
  },
  'Henrik Olsen (footballer)': {
    heading: 'Speedy deletion nomination',
    body: "'''A1 (no context):''' Sub-stub: no context, no claims of notability, no sources. Tagged 2 minutes after creation; flagging here so an admin can decide whether to wait for the author to expand. — User:PatrolBot 12:30, 6 May 2026 (UTC)",
  },
  'Talk:Discontinued template (orphan)': {
    heading: 'Speedy deletion nomination',
    body: "'''G6 (housekeeping):''' Talk page for a template that was deleted seven days ago. Orphaned with no useful discussion to preserve. Routine cleanup. — User:HousekeepingBot 08:11, 5 May 2026 (UTC)",
  },
  Apple1: {
    heading: 'Speedy deletion nomination',
    body: "'''G6 (housekeeping):''' Test article from local mwdd; placeholder so the prototype renders something. — User:PatrolBot 12:00, 6 May 2026 (UTC)",
  },
}

interface DeclinePageState {
  /** Title shown in the editor toolbar. */
  hostTitle: string
  /** Heading depth: '===' for RFPP sections, '==' for talk-page subjects. */
  headingMarker: string
  /** Pre-pasted decline-template line that appears above the section. */
  pastedTemplate: string
  /** Section content, mirrored from request.vue. */
  section: SectionFixture
  /** Saved-edit toast / confirmation copy used after Save. */
  savedToastNoun: string
}

const state = computed<DeclinePageState>(() => {
  if (contextParam.value === 'protection') {
    const section = protectionSections[pageTitle.value] ?? {
      heading: `[[${pageTitle.value}]]`,
      body: `Placeholder request body for ${pageTitle.value}.`,
    }
    return {
      hostTitle: 'Project:Requests for page protection',
      headingMarker: '===',
      pastedTemplate: '{{RFPP|d|<your reason here>}} ~~~~',
      section,
      savedToastNoun: 'protection request',
    }
  }
  const section = speedySections[pageTitle.value] ?? {
    heading: 'Speedy deletion nomination',
    body: `Placeholder nomination body for ${pageTitle.value}.`,
  }
  return {
    hostTitle: `Talk:${pageTitle.value}`,
    headingMarker: '==',
    pastedTemplate: "'''Speedy declined.''' <your reason here> ~~~~",
    section,
    savedToastNoun: 'speedy nomination',
  }
})

// Build the initial wikitext that lives in the editor textarea. Decline template
// pre-pasted at the top, then a blank line, then the original section.
const editorText = ref<string>('')
watch(
  state,
  (s) => {
    editorText.value = `${s.pastedTemplate}\n\n${s.headingMarker} ${s.section.heading} ${s.headingMarker}\n${s.section.body}\n`
  },
  { immediate: true },
)

const fallbackDashboardRoute = computed(() => ({
  path: '/admin-moderation-dashboard',
  query: {
    variant: 'current',
    direction: 'cards-attention',
    module: contextParam.value,
  },
}))

function isDashboardPath(path: unknown): path is string {
  return (
    typeof path === 'string' &&
    (path === fallbackDashboardRoute.value.path ||
      path.startsWith(`${fallbackDashboardRoute.value.path}?`))
  )
}

function getDashboardBackPath() {
  const previousPath = window.history.state?.back
  return isDashboardPath(previousPath) ? previousPath : null
}

function getDashboardReturnPath() {
  const returnPath = window.history.state?.dashboardReturnTo
  return isDashboardPath(returnPath) ? returnPath : null
}

function goBackToDashboard(event?: Event) {
  if (event) event.preventDefault()
  if (getDashboardBackPath()) {
    router.back()
  } else if (getDashboardReturnPath()) {
    router.push(getDashboardReturnPath() as string)
  } else {
    router.push(fallbackDashboardRoute.value)
  }
}

function onSave(event: Event) {
  event.preventDefault()
  router.push({
    ...fallbackDashboardRoute.value,
    query: {
      ...fallbackDashboardRoute.value.query,
      declined: pageTitle.value,
    },
  })
}
</script>

<template>
  <ChromeWrapper skin="mobile">
    <article class="decline">
      <header class="decline__editor-bar">
        <button
          class="decline__editor-bar-btn"
          type="button"
          aria-label="Cancel and close editor"
          @click="goBackToDashboard"
        >
          <CdxIcon :icon="cdxIconClose" size="medium" />
        </button>
        <span class="decline__editor-bar-title">
          <span class="decline__editor-bar-title-prefix">Editing</span>
          {{ state.hostTitle }}
        </span>
        <button
          class="decline__editor-bar-btn decline__editor-bar-btn--disabled"
          type="button"
          aria-label="Next section"
          disabled
        >
          <CdxIcon :icon="cdxIconNext" size="medium" />
        </button>
      </header>

      <p class="decline__hint">
        Decline template added at the top of the section. Edit the reason between the pipes, then
        Save.
      </p>

      <form class="decline__form" @submit="onSave">
        <textarea
          class="decline__textarea"
          v-model="editorText"
          rows="14"
          spellcheck="false"
          autofocus
        />

        <div class="decline__actions">
          <CdxButton action="progressive" weight="primary" type="submit">Save</CdxButton>
          <CdxButton weight="quiet" @click="goBackToDashboard">Cancel</CdxButton>
        </div>
      </form>
    </article>
  </ChromeWrapper>
</template>

<style scoped>
.decline {
  box-sizing: border-box;
  padding: 0;
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
}

.decline__editor-bar {
  display: flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  padding: var(--spacing-75, 12px) var(--spacing-100, 16px);
  background-color: var(--background-color-base, #fff);
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.decline__editor-bar-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  padding: 0;
  background: transparent;
  border: none;
  border-radius: var(--border-radius-base, 2px);
  cursor: pointer;
  color: var(--color-base, #202122);
}

.decline__editor-bar-btn:hover {
  background-color: var(--background-color-neutral-subtle, #f8f9fa);
}

.decline__editor-bar-btn--disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.decline__editor-bar-title {
  flex: 1;
  min-width: 0;
  font-size: var(--font-size-medium, 1rem);
  font-weight: var(--font-weight-bold, 700);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.decline__editor-bar-title-prefix {
  margin-right: var(--spacing-25, 4px);
  font-weight: var(--font-weight-normal, 400);
  color: var(--color-base, #202122);
}

.decline__hint {
  margin: 0;
  padding: var(--spacing-75, 12px) var(--spacing-100, 16px);
  font-size: var(--font-size-small, 0.875rem);
  color: var(--color-subtle, #54595d);
  background-color: var(--background-color-notice, #eaf3ff);
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.decline__form {
  padding: var(--spacing-100, 16px);
}

.decline__textarea {
  display: block;
  width: 100%;
  min-height: 320px;
  padding: var(--spacing-75, 12px);
  font-family: ui-monospace, 'SFMono-Regular', Menlo, Consolas, monospace;
  font-size: var(--font-size-medium, 1rem);
  line-height: 1.45;
  color: var(--color-base, #202122);
  background-color: var(--background-color-base, #fff);
  border: 1px solid var(--border-color-base, #c8ccd1);
  border-radius: var(--border-radius-base, 2px);
  resize: vertical;
  box-sizing: border-box;
}

.decline__textarea:focus {
  outline: 2px solid var(--color-progressive, #36c);
  outline-offset: -1px;
}

.decline__actions {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-75, 12px);
  margin-top: var(--spacing-100, 16px);
}
</style>
