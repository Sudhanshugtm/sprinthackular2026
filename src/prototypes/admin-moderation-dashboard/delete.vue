<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { CdxButton, CdxCheckbox, CdxSelect, CdxTextInput } from '@wikimedia/codex'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Delete page',
    description:
      'Mobile prototype mirroring the current Minerva delete-page form, surfaced from the admin moderation dashboard.',
  },
})

const route = useRoute()
const router = useRouter()

const pageTitle = computed(() => {
  const t = route.query.title
  if (typeof t === 'string' && t.trim()) return t.trim()
  return 'Apple1'
})

// Mirrors the local mwdd Minerva delete form's "Reason" dropdown verbatim.
// Auto-fidelity baseline; speedy-criterion options (G6, G11, …) are intentionally
// absent in this round so reviewers can feel the current state.
const reasonOptions = [
  { value: 'other', label: 'Other reason' },
  { value: 'Spam', label: 'Spam' },
  { value: 'Vandalism', label: 'Vandalism' },
  { value: 'Copyright violation', label: 'Copyright violation' },
  { value: 'Author request', label: 'Author request' },
  { value: 'Broken redirect', label: 'Broken redirect' },
]

interface ArticleFixture {
  /** Username shown in the page creation log. */
  creator: string
  /** Pre-formatted timestamp shown in the page creation log. */
  createdAt: string
  /** Wikitext snippet MW auto-fills into "Other/additional reason". */
  contentSnippet: string
}

// Per-article fixtures. Keys mirror the speedy-deletion card titles in index.vue
// (and the Apple1 page in our local mwdd that the original walkthrough used).
// Anything not listed falls back to a generic placeholder built at runtime.
const articleFixtures: Record<string, ArticleFixture> = {
  Apple1: {
    creator: 'Alice',
    createdAt: '15:15, 23 March 2026',
    contentSnippet:
      "The '''orange''', also called '''sweet orange''' to distinguish it from the bitter orange (''Citrus × aurantium''), is the fruit of a tree in the family Rutaceae...",
  },
  'Smith Industries Holdings Ltd': {
    creator: 'SmithCorp_Comms',
    createdAt: '11:42, 6 May 2026',
    contentSnippet:
      "'''Smith Industries Holdings Ltd''' is a leading global provider of innovative solutions across multiple sectors. Founded with a vision to deliver world-class value, the company has consistently set industry benchmarks for excellence and customer satisfaction...",
  },
  'Regional innovation timeline': {
    creator: 'TimelineDraftBot',
    createdAt: '09:18, 6 May 2026',
    contentSnippet:
      "The '''Regional innovation timeline''' presents a comprehensive overview of key milestones in regional innovation. In 1998, several pioneering initiatives were launched. By 2003, the ecosystem had expanded considerably. The 2010s saw rapid acceleration, and by 2024 the region was recognized as a global leader...",
  },
  'Henrik Olsen (footballer)': {
    creator: 'NewEditor2026',
    createdAt: '12:28, 6 May 2026',
    contentSnippet: "'''Henrik Olsen''' is a footballer.",
  },
  'Talk:Discontinued template (orphan)': {
    creator: 'OldHand',
    createdAt: '04:55, 5 May 2025',
    contentSnippet:
      'Discussion archive for {{tl|Discontinued}}. The template was retired following the consolidation discussion at [[Wikipedia talk:WikiProject Templates/Archive 12]]...',
  },
}

const fixture = computed<ArticleFixture>(() => {
  return (
    articleFixtures[pageTitle.value] ?? {
      creator: 'PageCreator',
      createdAt: '12:00, 6 May 2026',
      contentSnippet: `Stub content for ${pageTitle.value}; auto-summary placeholder shown when an article fixture is not provided.`,
    }
  )
})

const reason = ref<string>('other')
// MW currently auto-fills: "content was: <wikitext snippet>, and the only
// contributor was <user>". Recomputed when the title changes so each article
// shows its own snippet rather than the orange/citrus default.
const otherReason = ref<string>('')
watch(
  fixture,
  (f) => {
    otherReason.value = `content was: "${f.contentSnippet}", and the only contributor was "[[Special:Contributions/${f.creator}|${f.creator}]]" ([[User talk:${f.creator}|talk]])`
  },
  { immediate: true },
)
const watchPage = ref<boolean>(false)

// Truncated wikitext for the "Created page with …" footnote in the log.
const creationLogPreview = computed(() => {
  const max = 90
  const s = fixture.value.contentSnippet
  return s.length > max ? `${s.slice(0, max).trimEnd()}…` : s
})

const fallbackDashboardRoute = {
  path: '/admin-moderation-dashboard',
  query: {
    variant: 'current',
    direction: 'cards-attention',
    module: 'speedy',
  },
}

function hasDashboardHistory() {
  const previousPath = window.history.state?.back
  return (
    typeof previousPath === 'string' &&
    (previousPath === fallbackDashboardRoute.path ||
      previousPath.startsWith(`${fallbackDashboardRoute.path}?`))
  )
}

function onConfirm(event: Event) {
  event.preventDefault()
  router.push({
    path: '/admin-moderation-dashboard',
    query: {
      variant: 'current',
      direction: 'cards-attention',
      module: 'speedy',
      deleted: pageTitle.value,
    },
  })
}

function goBackToDashboard(event: Event) {
  event.preventDefault()
  if (hasDashboardHistory()) {
    router.back()
  } else {
    router.push(fallbackDashboardRoute)
  }
}

function onCancel(event: Event) {
  goBackToDashboard(event)
}
</script>

<template>
  <ChromeWrapper skin="mobile">
    <article class="delete-page">
      <h1 class="delete-page__title">Delete &ldquo;{{ pageTitle }}&rdquo;</h1>

      <p class="delete-page__back">
        &larr; <a href="#" @click.prevent="goBackToDashboard">{{ pageTitle }}</a>
      </p>

      <p class="delete-page__intro">
        You are about to delete a page along with all of its history. Please confirm that you
        intend to do this, that you understand the consequences, and that you are doing this in
        accordance with the
        <a href="#" @click.prevent>policy</a>.
      </p>

      <form class="delete-form" @submit="onConfirm">
        <fieldset class="delete-form__fieldset">
          <legend class="delete-form__legend">Delete</legend>

          <div class="delete-form__field">
            <label class="delete-form__label" for="delete-reason-list">Reason:</label>
            <CdxSelect
              id="delete-reason-list"
              :menu-items="reasonOptions"
              :selected="reason"
              @update:selected="(v: string) => (reason = v)"
            />
          </div>

          <div class="delete-form__field">
            <label class="delete-form__label" for="delete-reason-other"
              >Other/additional reason:</label
            >
            <CdxTextInput
              id="delete-reason-other"
              v-model="otherReason"
              :maxlength="500"
            />
          </div>

          <div class="delete-form__field delete-form__field--inline">
            <CdxCheckbox v-model="watchPage">Watch this page</CdxCheckbox>
          </div>

          <div class="delete-form__actions">
            <CdxButton action="destructive" weight="primary" type="submit"
              >Delete page</CdxButton
            >
            <CdxButton weight="quiet" @click="onCancel">Cancel</CdxButton>
          </div>
        </fieldset>
      </form>

      <p class="delete-page__edit-reasons">
        <a href="#" @click.prevent>Edit deletion reasons</a>
      </p>

      <section class="delete-page__log">
        <h2 class="delete-page__log-heading">Deletion log</h2>
        <p class="delete-page__log-empty">No matching items in log.</p>
      </section>

      <section class="delete-page__log">
        <h2 class="delete-page__log-heading">Page creation log</h2>
        <ul class="delete-page__log-list">
          <li>
            <strong>{{ fixture.createdAt }}</strong> {{ fixture.creator }}
            (<a href="#" @click.prevent>talk</a> ·
            <a href="#" @click.prevent>contribs</a> ·
            <a href="#" @click.prevent>block</a>)
            created page <strong>{{ pageTitle }}</strong>
            <em>(Created page with "{{ creationLogPreview }}")</em>
          </li>
        </ul>
      </section>
    </article>
  </ChromeWrapper>
</template>

<style scoped>
.delete-page {
  box-sizing: border-box;
  padding: var(--spacing-100, 16px);
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
}

.delete-page__title {
  margin: 0 0 var(--spacing-50, 8px);
  padding: 0;
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.75rem;
  font-weight: 400;
  line-height: 1.25;
  border: none;
}

.delete-page__back {
  margin: 0 0 var(--spacing-100, 16px);
  font-size: var(--font-size-medium, 1rem);
}

.delete-page__back a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.delete-page__back a:hover {
  text-decoration: underline;
}

.delete-page__intro {
  margin: 0 0 var(--spacing-150, 24px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
}

.delete-page__intro a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.delete-page__intro a:hover {
  text-decoration: underline;
}

.delete-form__fieldset {
  margin: 0 0 var(--spacing-75, 12px);
  padding: var(--spacing-100, 16px);
  border: 1px solid var(--border-color-base, #c8ccd1);
  border-radius: var(--border-radius-base, 2px);
  background-color: var(--background-color-base, #fff);
}

.delete-form__legend {
  padding: 0 var(--spacing-25, 4px);
  font-weight: var(--font-weight-bold, 700);
  color: var(--color-base, #202122);
}

.delete-form__field {
  margin-bottom: var(--spacing-100, 16px);
}

.delete-form__field:last-of-type {
  margin-bottom: 0;
}

.delete-form__field--inline {
  display: flex;
  align-items: center;
}

.delete-form__label {
  display: block;
  margin-bottom: var(--spacing-25, 4px);
  font-size: var(--font-size-medium, 1rem);
  color: var(--color-base, #202122);
}

.delete-form__actions {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-75, 12px);
  margin-top: var(--spacing-100, 16px);
}

.delete-page__edit-reasons {
  margin: var(--spacing-50, 8px) 0 var(--spacing-200, 32px);
  font-size: var(--font-size-small, 0.875rem);
  text-align: right;
}

.delete-page__edit-reasons a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.delete-page__edit-reasons a:hover {
  text-decoration: underline;
}

.delete-page__log {
  margin: 0 0 var(--spacing-150, 24px);
}

.delete-page__log-heading {
  margin: 0 0 var(--spacing-50, 8px);
  padding-bottom: var(--spacing-25, 4px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.375rem;
  font-weight: 400;
  line-height: 1.25;
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.delete-page__log-empty {
  margin: 0;
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-medium, 1rem);
}

.delete-page__log-list {
  margin: 0;
  padding-left: var(--spacing-150, 24px);
  font-size: var(--font-size-small, 0.875rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-subtle, #54595d);
}

.delete-page__log-list a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.delete-page__log-list em {
  color: var(--color-subtle, #54595d);
}
</style>
