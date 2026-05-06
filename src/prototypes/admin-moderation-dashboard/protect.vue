<script setup lang="ts">
import { computed, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { CdxButton, CdxCheckbox, CdxSelect, CdxTextInput } from '@wikimedia/codex'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Change protection settings',
    description:
      'Mobile prototype mirroring the current Minerva protect-page form, surfaced from the admin moderation dashboard.',
  },
})

const route = useRoute()
const router = useRouter()

const pageTitle = computed(() => {
  const t = route.query.title
  if (typeof t === 'string' && t.trim()) return t.trim()
  return 'Apple1'
})

// Mirrors local mwdd Minerva protect form's dropdowns verbatim.
const levelOptions = [
  { value: '', label: 'Allow all users' },
  { value: 'autoconfirmed', label: 'Allow only autoconfirmed users' },
  { value: 'sysop', label: 'Allow only administrators' },
]

const expiryOptions = [
  { value: 'othertime', label: 'other time' },
  { value: '1 hour', label: '1 hour' },
  { value: '1 day', label: '1 day' },
  { value: '1 week', label: '1 week' },
  { value: '2 weeks', label: '2 weeks' },
  { value: '1 month', label: '1 month' },
  { value: '3 months', label: '3 months' },
  { value: '6 months', label: '6 months' },
  { value: '1 year', label: '1 year' },
  { value: 'infinite', label: 'infinite' },
]

const reasonOptions = [
  { value: 'other', label: 'Other reason' },
  { value: 'Excessive vandalism', label: 'Excessive vandalism' },
  { value: 'Excessive spamming', label: 'Excessive spamming' },
  { value: 'Edit warring', label: 'Edit warring' },
  { value: 'High traffic page', label: 'High traffic page' },
]

interface ProtectionLogEntry {
  timestamp: string
  admin: string
  action: string
}

interface ArticleFixture {
  /** Past protection events, oldest at the bottom (MW orders newest first). */
  protectionLog: ProtectionLogEntry[]
}

// Per-article fixtures keyed by the protection-card titles in index.vue
// (and Apple1 for the local-mwdd fallback). Anything else gets an empty log.
const articleFixtures: Record<string, ArticleFixture> = {
  Apple1: {
    protectionLog: [],
  },
  'Eurovision Song Contest Asia': {
    protectionLog: [
      {
        timestamp: '08:42, 11 March 2026',
        admin: 'AdminBeta',
        action:
          'changed protection settings of <strong>Eurovision Song Contest Asia</strong> [Edit=Allow only autoconfirmed users] (expires 18 March 2026 08:42 UTC) [Move=Allow only autoconfirmed users] (expires 18 March 2026 08:42 UTC) (semi: vandalism after broadcast)',
      },
      {
        timestamp: '14:10, 17 May 2025',
        admin: 'AdminAlpha',
        action:
          'protected <strong>Eurovision Song Contest Asia</strong> [Edit=Allow only autoconfirmed users] (expires 24 May 2025 14:10 UTC) (vandalism)',
      },
    ],
  },
  'Talk:Blue Whale Challenge': {
    protectionLog: [
      {
        timestamp: '02:15, 21 April 2026',
        admin: 'AdminGamma',
        action:
          'changed protection settings of <strong>Talk:Blue Whale Challenge</strong> [Edit=Allow only autoconfirmed users] (expires 28 April 2026 02:15 UTC) (off-topic re-posting)',
      },
    ],
  },
  'Narcissistic personality disorder': {
    protectionLog: [
      {
        timestamp: '19:38, 02 February 2026',
        admin: 'AdminDelta',
        action:
          'changed protection settings of <strong>Narcissistic personality disorder</strong> [Edit=Allow only autoconfirmed users] (expires 02 March 2026 19:38 UTC) (BLP-violation diffs)',
      },
    ],
  },
  'Cardig Air': {
    protectionLog: [],
  },
  'Groundhog Day (film)': {
    protectionLog: [
      {
        timestamp: '11:05, 28 January 2024',
        admin: 'AdminEpsilon',
        action:
          'protected <strong>Groundhog Day (film)</strong> [Edit=Allow only administrators] (expires 04 February 2024 11:05 UTC) (edit warring on plot)',
      },
    ],
  },
  '2025-26 Persian Gulf Pro League': {
    protectionLog: [
      {
        timestamp: '17:22, 28 March 2026',
        admin: 'AdminZeta',
        action:
          'changed protection settings of <strong>2025-26 Persian Gulf Pro League</strong> [Edit=Allow only autoconfirmed users] (expires 04 April 2026 17:22 UTC) (vandalism after match days)',
      },
    ],
  },
}

const fixture = computed<ArticleFixture>(() => {
  return articleFixtures[pageTitle.value] ?? { protectionLog: [] }
})

const editLevel = ref<string>('')
const editExpiry = ref<string>('infinite')
const editOtherTime = ref<string>('')

const showFurtherOptions = ref<boolean>(false)

const moveLevel = ref<string>('')
const moveExpiry = ref<string>('infinite')
const moveOtherTime = ref<string>('')

const cascading = ref<boolean>(false)
const reason = ref<string>('other')
const otherReason = ref<string>('')
const watchPage = ref<boolean>(true)

const fallbackDashboardRoute = {
  path: '/admin-moderation-dashboard',
  query: {
    variant: 'current',
    direction: 'cards-attention',
    module: 'protection',
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
      module: 'protection',
      protected: pageTitle.value,
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
    <article class="protect-page">
      <h1 class="protect-page__title">
        Change protection settings for &ldquo;{{ pageTitle }}&rdquo;
      </h1>

      <p class="protect-page__back">
        &larr; <a href="#" @click.prevent="goBackToDashboard">{{ pageTitle }}</a>
      </p>

      <p class="protect-page__intro">
        Here you may view and change the protection settings for the page <strong>{{ pageTitle }}</strong>.
      </p>

      <form class="protect-form" @submit="onConfirm">
        <fieldset class="protect-form__fieldset">
          <legend class="protect-form__legend">Confirm protection</legend>

          <h3 class="protect-form__sub-heading">Edit</h3>
          <div class="protect-form__field">
            <CdxSelect
              :menu-items="levelOptions"
              :selected="editLevel"
              @update:selected="(v: string) => (editLevel = v)"
            />
          </div>
          <div class="protect-form__field">
            <label class="protect-form__label" for="protect-edit-expiry">Expires:</label>
            <CdxSelect
              id="protect-edit-expiry"
              :menu-items="expiryOptions"
              :selected="editExpiry"
              @update:selected="(v: string) => (editExpiry = v)"
            />
          </div>
          <div class="protect-form__field">
            <label class="protect-form__label" for="protect-edit-othertime">Other time:</label>
            <CdxTextInput id="protect-edit-othertime" v-model="editOtherTime" />
          </div>

          <div class="protect-form__field protect-form__field--inline">
            <CdxCheckbox v-model="showFurtherOptions">Unlock further protect options</CdxCheckbox>
          </div>

          <h3 class="protect-form__sub-heading">Move</h3>
          <div class="protect-form__field">
            <CdxSelect
              :menu-items="levelOptions"
              :selected="moveLevel"
              :disabled="!showFurtherOptions"
              @update:selected="(v: string) => (moveLevel = v)"
            />
          </div>
          <div class="protect-form__field">
            <label class="protect-form__label" for="protect-move-expiry">Expires:</label>
            <CdxSelect
              id="protect-move-expiry"
              :menu-items="expiryOptions"
              :selected="moveExpiry"
              :disabled="!showFurtherOptions"
              @update:selected="(v: string) => (moveExpiry = v)"
            />
          </div>
          <div class="protect-form__field">
            <label class="protect-form__label" for="protect-move-othertime">Other time:</label>
            <CdxTextInput
              id="protect-move-othertime"
              v-model="moveOtherTime"
              :disabled="!showFurtherOptions"
            />
          </div>

          <div class="protect-form__field protect-form__field--inline">
            <CdxCheckbox v-model="cascading">
              Protect pages included in this page (cascading protection)
            </CdxCheckbox>
          </div>

          <div class="protect-form__field">
            <label class="protect-form__label" for="protect-reason-list">Reason:</label>
            <CdxSelect
              id="protect-reason-list"
              :menu-items="reasonOptions"
              :selected="reason"
              @update:selected="(v: string) => (reason = v)"
            />
          </div>

          <div class="protect-form__field">
            <label class="protect-form__label" for="protect-reason-other"
              >Other/additional reason:</label
            >
            <CdxTextInput id="protect-reason-other" v-model="otherReason" :maxlength="500" />
          </div>

          <div class="protect-form__field protect-form__field--inline">
            <CdxCheckbox v-model="watchPage">Watch this page</CdxCheckbox>
          </div>

          <div class="protect-form__actions">
            <CdxButton action="progressive" weight="primary" type="submit">Confirm</CdxButton>
            <CdxButton weight="quiet" @click="onCancel">Cancel</CdxButton>
          </div>
        </fieldset>
      </form>

      <p class="protect-page__edit-reasons">
        <a href="#" @click.prevent>Edit protection reasons</a>
      </p>

      <section class="protect-page__log">
        <h2 class="protect-page__log-heading">Protection log</h2>
        <p v-if="fixture.protectionLog.length === 0" class="protect-page__log-empty">
          No matching items in log.
        </p>
        <ul v-else class="protect-page__log-list">
          <li v-for="(entry, idx) in fixture.protectionLog" :key="idx">
            <strong>{{ entry.timestamp }}</strong>
            <a href="#" @click.prevent>{{ entry.admin }}</a>
            (<a href="#" @click.prevent>talk</a> ·
            <a href="#" @click.prevent>contribs</a> ·
            <a href="#" @click.prevent>block</a>)
            <span v-html="entry.action" />
          </li>
        </ul>
      </section>
    </article>
  </ChromeWrapper>
</template>

<style scoped>
.protect-page {
  box-sizing: border-box;
  padding: var(--spacing-100, 16px);
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
}

.protect-page__title {
  margin: 0 0 var(--spacing-50, 8px);
  padding: 0;
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.625rem;
  font-weight: 400;
  line-height: 1.3;
  border: none;
}

.protect-page__back {
  margin: 0 0 var(--spacing-100, 16px);
  font-size: var(--font-size-medium, 1rem);
}

.protect-page__back a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.protect-page__back a:hover {
  text-decoration: underline;
}

.protect-page__intro {
  margin: 0 0 var(--spacing-150, 24px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
}

.protect-form__fieldset {
  margin: 0 0 var(--spacing-75, 12px);
  padding: var(--spacing-100, 16px);
  border: 1px solid var(--border-color-base, #c8ccd1);
  border-radius: var(--border-radius-base, 2px);
  background-color: var(--background-color-base, #fff);
}

.protect-form__legend {
  padding: 0 var(--spacing-25, 4px);
  font-weight: var(--font-weight-bold, 700);
  color: var(--color-base, #202122);
}

.protect-form__sub-heading {
  margin: var(--spacing-100, 16px) 0 var(--spacing-50, 8px);
  padding: 0;
  font-size: var(--font-size-medium, 1rem);
  font-weight: var(--font-weight-bold, 700);
  border: none;
}

.protect-form__sub-heading:first-of-type {
  margin-top: 0;
}

.protect-form__field {
  margin-bottom: var(--spacing-75, 12px);
}

.protect-form__field--inline {
  display: flex;
  align-items: center;
}

.protect-form__label {
  display: block;
  margin-bottom: var(--spacing-25, 4px);
  font-size: var(--font-size-medium, 1rem);
  color: var(--color-base, #202122);
}

.protect-form__actions {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-75, 12px);
  margin-top: var(--spacing-100, 16px);
}

.protect-page__edit-reasons {
  margin: var(--spacing-50, 8px) 0 var(--spacing-200, 32px);
  font-size: var(--font-size-small, 0.875rem);
  text-align: right;
}

.protect-page__edit-reasons a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.protect-page__edit-reasons a:hover {
  text-decoration: underline;
}

.protect-page__log {
  margin: 0 0 var(--spacing-150, 24px);
}

.protect-page__log-heading {
  margin: 0 0 var(--spacing-50, 8px);
  padding-bottom: var(--spacing-25, 4px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.375rem;
  font-weight: 400;
  line-height: 1.25;
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.protect-page__log-empty {
  margin: 0;
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-medium, 1rem);
}

.protect-page__log-list {
  margin: 0;
  padding-left: var(--spacing-150, 24px);
  font-size: var(--font-size-small, 0.875rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-base, #202122);
}

.protect-page__log-list li {
  margin-bottom: var(--spacing-50, 8px);
}

.protect-page__log-list a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}
</style>
