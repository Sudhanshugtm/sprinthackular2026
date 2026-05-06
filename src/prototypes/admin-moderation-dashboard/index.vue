<script setup lang="ts">
import { computed, ref } from 'vue'

import { CdxButton, CdxIcon, CdxSelect } from '@wikimedia/codex'
import {
  cdxIconArticle,
  cdxIconBell,
  cdxIconCheck,
  cdxIconClock,
  cdxIconClose,
  cdxIconEdit,
  cdxIconEye,
  cdxIconFunnel,
  cdxIconHistory,
  cdxIconInfo,
  cdxIconLabFlask,
  cdxIconLock,
  cdxIconMenu,
  cdxIconNext,
  cdxIconPrevious,
  cdxIconRecentChanges,
  cdxIconRobot,
  cdxIconSearch,
  cdxIconSettings,
  cdxIconSpeechBubble,
  cdxIconTrash,
  cdxIconTray,
  cdxIconUserAvatar,
  cdxIconUserAvatarOutline,
  type Icon,
} from '@wikimedia/codex-icons'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Admin moderation dashboard',
    description: 'Mobile logged-in dashboard prototype for admin moderation triage.',
  },
})

type TaskType = 'protection' | 'deletion'
type TaskStatus = 'needs-review' | 'in-discussion' | 'already-handled'
type DashboardVariantId = 'base' | 'option-a' | 'option-b' | 'current'

interface Task {
  id: string
  type: TaskType
  title: string
  source: string
  age: string
  status: TaskStatus
  priority: 'High' | 'Medium' | 'Low'
  request: string
  reason: string
  judgment: string
  signals: string[]
  nextSurface: string
}

interface DashboardVariant {
  id: DashboardVariantId
  label: string
  heading: string
  description: string
  lensLabel: string
  lensCopy: string
}

interface ProtectionRequestCard {
  id: string
  title: string
  visibility: string
  requestSummary: string
  signalLabel: string
  signalLine: string
  status: string
  activeScore: number
  actorCount: number
  ageHours: number
}

interface SpeedyDeletionCard {
  id: string
  title: string
  criterion: string
  visibility: string
  requestSummary: string
  signalLabel: string
  signalLine: string
  status: string
  ageMinutes: number
  needsDecision: boolean
  decisionScore: number
  criterionOrder: number
}

const moderator = {
  name: 'ModeratorDemo',
  rights: 'Administrator',
  wiki: 'English Wikipedia',
}

const variants: DashboardVariant[] = [
  {
    id: 'base',
    label: 'Current dashboard',
    heading: 'Current Personal Dashboard',
    description: 'Matches the local PersonalDashboard treatment before the protection-request patch.',
    lensLabel: 'Baseline',
    lensCopy: 'Shows the current modules and visual hierarchy without admin recommendation logic.',
  },
  {
    id: 'option-a',
    label: 'Option A',
    heading: 'Current Personal Dashboard',
    description: 'A copy of the current PersonalDashboard baseline for design iteration.',
    lensLabel: 'Option A',
    lensCopy: 'Starts from the current dashboard modules and visual hierarchy.',
  },
  {
    id: 'option-b',
    label: 'Prototype v0',
    heading: 'Current Personal Dashboard',
    description: 'Previous prototype version with two admin backlog cards.',
    lensLabel: 'Prototype v0',
    lensCopy: 'Per-backlog stacking: each admin queue gets its own card in the same pattern.',
  },
  {
    id: 'current',
    label: 'Prototype v1 (latest)',
    heading: 'Current Personal Dashboard',
    description: 'Latest prototype version with admin backlog cards and clarified queue metrics.',
    lensLabel: 'Prototype v1',
    lensCopy: 'Latest iteration: current actionability is separated from incoming volume.',
  },
]

const summaryByVariant = {
  base: [
    { icon: cdxIconBell, value: '0', label: 'alerts' },
    { icon: cdxIconTray, value: '1', label: 'notice', strong: true },
    { icon: cdxIconRecentChanges, value: '4', label: 'changes' },
  ],
  'option-a': [
    { icon: cdxIconBell, value: '0', label: 'alerts' },
    { icon: cdxIconTray, value: '1', label: 'notice', strong: true },
    { icon: cdxIconRecentChanges, value: '4', label: 'changes' },
  ],
  'option-b': [
    { icon: cdxIconBell, value: '0', label: 'alerts' },
    { icon: cdxIconTray, value: '1', label: 'notice', strong: true },
    { icon: cdxIconRecentChanges, value: '4', label: 'changes' },
  ],
  current: [
    { icon: cdxIconBell, value: '0', label: 'alerts' },
    { icon: cdxIconTray, value: '1', label: 'notice', strong: true },
    { icon: cdxIconRecentChanges, value: '4', label: 'changes' },
  ],
} satisfies Record<DashboardVariantId, { icon: Icon; value: string; label: string; strong?: boolean }[]>

const tasks = ref<Task[]>([
  {
    id: 'rfpp-eurovision',
    type: 'protection',
    title: 'Eurovision Song Contest Asia',
    source: 'Requests for page protection',
    age: '22h',
    status: 'needs-review',
    priority: 'High',
    request: 'Semi-protection requested',
    reason: 'Repeated disruptive edits by multiple new and anonymous editors.',
    judgment: 'Decide whether disruption is broad enough for page protection, or whether user-level action is enough.',
    signals: [
      'No admin reply on RFPP thread',
      'Multiple recent editors in history',
      'Protection not currently active',
    ],
    nextSurface: 'Open page history',
  },
  {
    id: 'csd-llm',
    type: 'deletion',
    title: 'Draft:Regional innovation timeline',
    source: 'Speedy deletion candidates',
    age: '3h',
    status: 'needs-review',
    priority: 'Medium',
    request: 'Speedy deletion requested',
    reason: 'Tagged as unreviewed LLM-generated content with no sources.',
    judgment: 'Decide whether the speedy deletion criterion applies, or remove the template with an explanation.',
    signals: ['Tagged with deletion template', 'Category-backed queue', 'No return-to-noticeboard step'],
    nextSurface: 'Review tagged page',
  },
  {
    id: 'rfpp-blue-whale',
    type: 'protection',
    title: 'Talk:Blue Whale Challenge',
    source: 'Requests for page protection',
    age: '1d',
    status: 'in-discussion',
    priority: 'Medium',
    request: 'Protection requested',
    reason: 'Editors are discussing whether the disruption is frequent enough for protection.',
    judgment: 'Decide whether discussion has enough evidence for action, or whether to wait.',
    signals: ['5 comments', '3 participants', 'Talk-page protection has limited options'],
    nextSurface: 'Read discussion',
  },
])

const selectedTaskId = ref(tasks.value[0].id)
const variantSwitcherOpen = ref(false)
const activeVariantId = ref<DashboardVariantId>(getInitialVariant())
const optionAProtectionViewOpen = ref(getInitialVariant() === 'option-a' && getInitialModule() === 'protection')
const optionBProtectionViewOpen = ref(isAdminPrototypeVariantId(getInitialVariant()) && getInitialModule() === 'protection')
const optionBSpeedyViewOpen = ref(isAdminPrototypeVariantId(getInitialVariant()) && getInitialModule() === 'speedy')
const selectedTask = computed(() => tasks.value.find((task) => task.id === selectedTaskId.value) ?? tasks.value[0])
const activeVariant = computed(() => variants.find((variant) => variant.id === activeVariantId.value) ?? variants[0])
const usesCurrentDashboardTreatment = computed(() => activeVariant.value.id === 'base' || activeVariant.value.id === 'option-a' || isAdminPrototypeVariantId(activeVariant.value.id))
const isOptionAVariant = computed(() => activeVariant.value.id === 'option-a')
const isOptionBVariant = computed(() => isAdminPrototypeVariantId(activeVariant.value.id))
const isOptionAProtectionDetail = computed(() => isOptionAVariant.value && optionAProtectionViewOpen.value)
const isOptionBProtectionDetail = computed(() => isOptionBVariant.value && optionBProtectionViewOpen.value)
const isOptionBSpeedyDetail = computed(() => isOptionBVariant.value && optionBSpeedyViewOpen.value)
const isAnyDetailOpen = computed(() => isOptionAProtectionDetail.value || isOptionBProtectionDetail.value || isOptionBSpeedyDetail.value)
const summaryItems = computed(() => summaryByVariant[activeVariant.value.id])

const visibleTasks = computed(() => tasks.value.filter((task) => task.status !== 'already-handled'))

const protectionSortItems = [
  { value: 'hot', label: 'Active disruption first' },
  { value: 'unanswered', label: 'No admin reply first' },
  { value: 'multi', label: 'Most editors involved' },
  { value: 'oldest', label: 'Waiting longest' },
  { value: 'newest', label: 'Recently requested' },
]
const protectionSortValue = ref('hot')

const optionBProtectionRequests: ProtectionRequestCard[] = [
  {
    id: 'eurovision',
    title: 'Eurovision Song Contest Asia',
    visibility: '12k views/day · ↑ 4× this week',
    requestSummary: 'Semi-protection requested for 1 week',
    signalLabel: 'Active disruption',
    signalLine: 'Last revert 12m ago · 5 actors · 24 reverts in 6h',
    status: 'No admin reply',
    activeScore: 100,
    actorCount: 5,
    ageHours: 6,
  },
  {
    id: 'blue-whale',
    title: 'Talk:Blue Whale Challenge',
    visibility: '80 views/day · stable',
    requestSummary: 'Talk-page semi-protection requested indefinitely',
    signalLabel: 'Discussion quiet',
    signalLine: 'Last activity 18h ago · 3 participants · 5 comments in 1d',
    status: 'Waiting on requester',
    activeScore: 10,
    actorCount: 3,
    ageHours: 18,
  },
  {
    id: 'npd',
    title: 'Narcissistic personality disorder',
    visibility: '9.4k views/day · steady',
    requestSummary: 'Indefinite semi-protection requested',
    signalLabel: 'Recurring vandalism',
    signalLine: 'Repeated diagnosis claims · recent diffs supplied',
    status: 'No admin reply',
    activeScore: 70,
    actorCount: 3,
    ageHours: 14,
  },
  {
    id: 'cardig-air',
    title: 'Cardig Air',
    visibility: '120 views/day · low traffic',
    requestSummary: 'Temporary semi-protection requested',
    signalLabel: 'Sourcing disruption',
    signalLine: 'Unsourced additions · blocked temp account involved',
    status: 'No admin reply',
    activeScore: 45,
    actorCount: 2,
    ageHours: 22,
  },
  {
    id: 'groundhog-day',
    title: 'Groundhog Day (film)',
    visibility: '5.1k views/day · stable',
    requestSummary: 'Temporary full protection requested',
    signalLabel: 'Edit warring',
    signalLine: 'Talk-page dispute · multiple editors oppose changes',
    status: 'Discussion active',
    activeScore: 55,
    actorCount: 6,
    ageHours: 9,
  },
  {
    id: 'persian-gulf-pro-league',
    title: '2025-26 Persian Gulf Pro League',
    visibility: '900 views/day · rising',
    requestSummary: 'Semi-protection requested',
    signalLabel: 'Protection expired',
    signalLine: 'Vandalism resumed immediately after expiry',
    status: 'No admin reply',
    activeScore: 85,
    actorCount: 4,
    ageHours: 2,
  },
]

const sortedOptionBProtectionRequests = computed(() => {
  const requests = [...optionBProtectionRequests]
  const byActive = (a: ProtectionRequestCard, b: ProtectionRequestCard) => b.activeScore - a.activeScore

  switch (protectionSortValue.value) {
    case 'unanswered':
      return requests.sort((a, b) =>
        Number(b.status === 'No admin reply') - Number(a.status === 'No admin reply') ||
        byActive(a, b)
      )
    case 'multi':
      return requests.sort((a, b) => b.actorCount - a.actorCount || byActive(a, b))
    case 'oldest':
      return requests.sort((a, b) => b.ageHours - a.ageHours || byActive(a, b))
    case 'newest':
      return requests.sort((a, b) => a.ageHours - b.ageHours || byActive(a, b))
    case 'hot':
    default:
      return requests.sort(byActive)
  }
})

const speedySortItems = [
  { value: 'decision', label: 'Needs decision first' },
  { value: 'newest', label: 'Recently tagged' },
  { value: 'oldest', label: 'Waiting longest' },
  { value: 'criterion', label: 'By speedy reason' },
]
const speedySortValue = ref('decision')

const optionBSpeedyDeletionRequests: SpeedyDeletionCard[] = [
  {
    id: 'smith-industries',
    title: 'Smith Industries Holdings Ltd',
    criterion: 'G11 · Advertising',
    visibility: '42 views in last hour',
    requestSummary: 'Tagged 14m ago · company-style article',
    signalLabel: 'Admin question',
    signalLine: 'Is this only promotion, or is a neutral rewrite plausible?',
    status: 'Needs decision',
    ageMinutes: 14,
    needsDecision: true,
    decisionScore: 90,
    criterionOrder: 2,
  },
  {
    id: 'regional-innovation',
    title: 'Regional innovation timeline',
    criterion: 'G15 · Unreviewed LLM content',
    visibility: '90 views · contested',
    requestSummary: 'Tagged 3h ago · disputed on talk page',
    signalLabel: 'Admin question',
    signalLine: 'Does the speedy criterion still apply, or should this move to discussion?',
    status: 'Contested tag',
    ageMinutes: 180,
    needsDecision: true,
    decisionScore: 80,
    criterionOrder: 3,
  },
  {
    id: 'henrik-olsen',
    title: 'Henrik Olsen (footballer)',
    criterion: 'A1 · No context',
    visibility: '4 views · new page',
    requestSummary: 'Tagged 2m after page creation',
    signalLabel: 'Admin caution',
    signalLine: 'Creator may still be editing; deletion could be premature.',
    status: 'Too new to judge',
    ageMinutes: 2,
    needsDecision: false,
    decisionScore: 30,
    criterionOrder: 1,
  },
  {
    id: 'orphan-template',
    title: 'Talk:Discontinued template (orphan)',
    criterion: 'G6 · Housekeeping',
    visibility: 'Housekeeping page · low traffic',
    requestSummary: 'Tagged 1d ago · target template deleted',
    signalLabel: 'Routine check',
    signalLine: 'Confirm no useful talk history before deleting.',
    status: 'Housekeeping cleanup',
    ageMinutes: 1440,
    needsDecision: false,
    decisionScore: 10,
    criterionOrder: 4,
  },
]

const sortedOptionBSpeedyDeletionRequests = computed(() => {
  const requests = [...optionBSpeedyDeletionRequests]

  switch (speedySortValue.value) {
    case 'decision':
      return requests.sort((a, b) =>
        Number(b.needsDecision) - Number(a.needsDecision) ||
        b.decisionScore - a.decisionScore ||
        a.ageMinutes - b.ageMinutes
      )
    case 'oldest':
      return requests.sort((a, b) => b.ageMinutes - a.ageMinutes)
    case 'criterion':
      return requests.sort((a, b) =>
        a.criterionOrder - b.criterionOrder ||
        a.ageMinutes - b.ageMinutes
      )
    case 'newest':
    default:
      return requests.sort((a, b) => a.ageMinutes - b.ageMinutes)
  }
})

function taskIcon(task: Task) {
  return task.type === 'protection' ? cdxIconLock : cdxIconTrash
}

function isDashboardVariant(value: string | null): value is DashboardVariantId {
  return value === 'base' || value === 'option-a' || value === 'option-b' || value === 'current'
}

function isAdminPrototypeVariantId(value: DashboardVariantId) {
  return value === 'option-b' || value === 'current'
}

function getInitialVariant(): DashboardVariantId {
  if (typeof window === 'undefined') {
    return 'base'
  }

  const variant = new URLSearchParams(window.location.search).get('variant')
  return isDashboardVariant(variant) ? variant : 'base'
}

function getInitialModule() {
  if (typeof window === 'undefined') {
    return ''
  }

  return new URLSearchParams(window.location.search).get('module') ?? ''
}

function selectVariant(variantId: DashboardVariantId) {
  activeVariantId.value = variantId
  variantSwitcherOpen.value = false
  optionAProtectionViewOpen.value = false
  optionBProtectionViewOpen.value = false
  optionBSpeedyViewOpen.value = false

  if (typeof window === 'undefined') {
    return
  }

  const url = new URL(window.location.href)
  url.searchParams.set('variant', variantId)
  url.searchParams.delete('module')
  window.history.replaceState({}, '', url)
}

function openOptionAProtectionRequests(event: MouseEvent) {
  if (!isOptionAVariant.value) {
    return
  }

  event.preventDefault()
  optionAProtectionViewOpen.value = true
  syncModuleParam('protection')
}

function closeOptionAProtectionRequests() {
  optionAProtectionViewOpen.value = false
  syncModuleParam(null)
}

function openOptionBProtection(event: MouseEvent) {
  if (!isOptionBVariant.value) {
    return
  }

  event.preventDefault()
  optionBProtectionViewOpen.value = true
  syncModuleParam('protection')
}

function openOptionBSpeedy(event: MouseEvent) {
  if (!isOptionBVariant.value) {
    return
  }

  event.preventDefault()
  optionBSpeedyViewOpen.value = true
  syncModuleParam('speedy')
}

function closeOptionBModule() {
  optionBProtectionViewOpen.value = false
  optionBSpeedyViewOpen.value = false
  syncModuleParam(null)
}

function syncModuleParam(value: string | null) {
  if (typeof window === 'undefined') {
    return
  }

  const url = new URL(window.location.href)
  if (value === null) {
    url.searchParams.delete('module')
  } else {
    url.searchParams.set('module', value)
  }
  window.history.replaceState({}, '', url)
  window.scrollTo({ top: 0 })
}

function variantOptionLabel(variantId: DashboardVariantId) {
  switch (variantId) {
    case 'base':
      return 'Current dashboard'
    case 'option-a':
      return 'Option A'
    case 'option-b':
      return 'Prototype v0'
    case 'current':
      return 'Prototype v1 (latest)'
  }
}
</script>

<template>
  <ChromeWrapper skin="mobile" lang="en" dir="ltr">
    <template #header>
      <header class="mock-minerva-header">
        <CdxButton weight="quiet" aria-label="Main menu">
          <CdxIcon :icon="cdxIconMenu" />
        </CdxButton>
        <a class="mock-minerva-header__brand" href="#">mwdd-en</a>
        <div class="mock-minerva-header__actions">
          <CdxButton weight="quiet" aria-label="Search">
            <CdxIcon :icon="cdxIconSearch" />
          </CdxButton>
          <button class="mock-minerva-header__notification-count" type="button" aria-label="Notifications">
            1
          </button>
          <CdxButton weight="quiet" aria-label="User menu">
            <CdxIcon :icon="cdxIconUserAvatarOutline" />
          </CdxButton>
        </div>
      </header>
    </template>

    <template #footer>
      <footer class="mock-minerva-footer">
        <strong>mwdd-en</strong>
        <nav aria-label="Footer">
          <a href="#">Privacy policy</a>
          <span aria-hidden="true">·</span>
          <a href="#">Mobile view</a>
          <span aria-hidden="true">·</span>
          <a href="#">Developers</a>
          <span aria-hidden="true">·</span>
          <a href="#">Statistics</a>
          <span aria-hidden="true">·</span>
          <a href="#">Cookie statement</a>
        </nav>
      </footer>
    </template>

    <div
      class="moderator-dashboard"
      :class="[
        `moderator-dashboard--${activeVariant.id}`,
        {
          'moderator-dashboard--current-treatment': usesCurrentDashboardTreatment,
          'moderator-dashboard--detail-page': isAnyDetailOpen,
        },
      ]"
    >
      <template v-if="usesCurrentDashboardTreatment">
        <template v-if="isOptionAProtectionDetail">
          <section class="personal-dashboard-detail" aria-labelledby="option-a-protection-heading">
            <header class="personal-dashboard-detail__header">
              <CdxButton
                aria-label="Back to dashboard"
                weight="quiet"
                @click="closeOptionAProtectionRequests"
              >
                <CdxIcon :icon="cdxIconPrevious" />
              </CdxButton>
              <h1 id="option-a-protection-heading">Protection requests</h1>
            </header>

            <div class="personal-dashboard-detail__body">
              <div class="personal-dashboard-detail__toolbar">
                <p class="personal-dashboard-detail__lede-line">
                  <strong>17</strong> unresolved · <strong>6</strong> no admin reply
                </p>
                <div class="personal-dashboard-detail__sort-row">
                  <span class="personal-dashboard-detail__sort-label">Sort</span>
                  <CdxSelect
                    class="personal-dashboard-detail__sort-select"
                    :menu-items="protectionSortItems"
                    :selected="protectionSortValue"
                    aria-label="Sort by"
                    @update:selected="protectionSortValue = $event"
                  />
                </div>
              </div>

              <div class="personal-dashboard-detail__requests">
                <article class="personal-dashboard-detail__request">
                  <h2 class="personal-dashboard-detail__title-line">Eurovision Song Contest Asia</h2>
                  <p class="personal-dashboard-detail__importance">
                    <span class="personal-dashboard-detail__importance-item">
                      <CdxIcon :icon="cdxIconEye" size="small" />
                      12k views/day · ↑ 4× this week
                    </span>
                  </p>
                  <p class="personal-dashboard-detail__request-summary">
                    Semi-protection requested for 1 week
                  </p>
                  <p class="personal-dashboard-detail__signal-label">
                    Active disruption
                  </p>
                  <p class="personal-dashboard-detail__signal-line">
                    Last revert 12m ago · 5 actors · 24 reverts in 6h
                  </p>
                  <p class="personal-dashboard-detail__status-line">
                    No admin reply
                  </p>
                </article>

                <article class="personal-dashboard-detail__request">
                  <h2 class="personal-dashboard-detail__title-line">Talk:Blue Whale Challenge</h2>
                  <p class="personal-dashboard-detail__importance">
                    <span class="personal-dashboard-detail__importance-item">
                      <CdxIcon :icon="cdxIconEye" size="small" />
                      80 views/day · stable
                    </span>
                  </p>
                  <p class="personal-dashboard-detail__request-summary">
                    Talk-page semi-protection requested indefinitely
                  </p>
                  <p class="personal-dashboard-detail__signal-label">
                    Discussion quiet
                  </p>
                  <p class="personal-dashboard-detail__signal-line">
                    Last activity 18h ago · 3 participants · 5 comments in 1d
                  </p>
                  <p class="personal-dashboard-detail__status-line">
                    Waiting on requester
                  </p>
                </article>

              </div>
            </div>
          </section>
        </template>

        <template v-else-if="isOptionBProtectionDetail">
          <section class="personal-dashboard-detail" aria-labelledby="option-b-protection-detail-heading">
            <header class="personal-dashboard-detail__header">
              <CdxButton
                aria-label="Back to dashboard"
                weight="quiet"
                @click="closeOptionBModule"
              >
                <CdxIcon :icon="cdxIconPrevious" />
              </CdxButton>
              <h1 id="option-b-protection-detail-heading">Pages for protection</h1>
            </header>

            <div class="personal-dashboard-detail__body">
              <div class="personal-dashboard-detail__toolbar">
                <p class="personal-dashboard-detail__lede-line">
                  <strong>17</strong> unresolved · <strong>6</strong> no admin reply
                </p>
                <div class="personal-dashboard-detail__sort-row">
                  <span class="personal-dashboard-detail__sort-label">Sort</span>
                  <CdxSelect
                    class="personal-dashboard-detail__sort-select"
                    :menu-items="protectionSortItems"
                    :selected="protectionSortValue"
                    aria-label="Sort by"
                    @update:selected="protectionSortValue = $event"
                  />
                </div>
              </div>

              <div class="personal-dashboard-detail__requests">
                <article
                  v-for="request in sortedOptionBProtectionRequests"
                  :key="request.id"
                  class="personal-dashboard-detail__request"
                >
                  <h2 class="personal-dashboard-detail__title-line">{{ request.title }}</h2>
                  <p class="personal-dashboard-detail__importance">
                    <span class="personal-dashboard-detail__importance-item">
                      <CdxIcon :icon="cdxIconEye" size="small" />
                      {{ request.visibility }}
                    </span>
                  </p>
                  <p class="personal-dashboard-detail__request-summary">
                    {{ request.requestSummary }}
                  </p>
                  <p class="personal-dashboard-detail__signal-label">
                    {{ request.signalLabel }}
                  </p>
                  <p class="personal-dashboard-detail__signal-line">
                    {{ request.signalLine }}
                  </p>
                  <p class="personal-dashboard-detail__status-line">
                    {{ request.status }}
                  </p>
                </article>
              </div>
            </div>
          </section>
        </template>

        <template v-else-if="isOptionBSpeedyDetail">
          <section class="personal-dashboard-detail" aria-labelledby="option-b-speedy-detail-heading">
            <header class="personal-dashboard-detail__header">
              <CdxButton
                aria-label="Back to dashboard"
                weight="quiet"
                @click="closeOptionBModule"
              >
                <CdxIcon :icon="cdxIconPrevious" />
              </CdxButton>
              <h1 id="option-b-speedy-detail-heading">Pages for speedy deletion</h1>
            </header>

            <div class="personal-dashboard-detail__body">
              <div class="personal-dashboard-detail__toolbar">
                <p class="personal-dashboard-detail__lede-line">
                  <strong>4</strong> awaiting review · <strong>2</strong> need decision
                </p>
                <div class="personal-dashboard-detail__sort-row">
                  <span class="personal-dashboard-detail__sort-label">Sort</span>
                  <CdxSelect
                    class="personal-dashboard-detail__sort-select"
                    :menu-items="speedySortItems"
                    :selected="speedySortValue"
                    aria-label="Sort by"
                    @update:selected="speedySortValue = $event"
                  />
                </div>
              </div>

              <div class="personal-dashboard-detail__requests">
                <article
                  v-for="request in sortedOptionBSpeedyDeletionRequests"
                  :key="request.id"
                  class="personal-dashboard-detail__request"
                >
                  <span class="personal-dashboard-detail__criterion">{{ request.criterion }}</span>
                  <h2 class="personal-dashboard-detail__title-line">{{ request.title }}</h2>
                  <p class="personal-dashboard-detail__importance">
                    <span class="personal-dashboard-detail__importance-item">
                      <CdxIcon :icon="cdxIconEye" size="small" />
                      {{ request.visibility }}
                    </span>
                  </p>
                  <p class="personal-dashboard-detail__request-summary">
                    {{ request.requestSummary }}
                  </p>
                  <p class="personal-dashboard-detail__signal-label">
                    {{ request.signalLabel }}
                  </p>
                  <p class="personal-dashboard-detail__signal-line">
                    {{ request.signalLine }}
                  </p>
                  <p class="personal-dashboard-detail__status-line">
                    {{ request.status }}
                  </p>
                </article>
              </div>
            </div>
          </section>
        </template>

        <template v-else>
        <header class="personal-dashboard-baseline__hero">
          <h1>Hello, {{ moderator.name }}!</h1>
          <div class="personal-dashboard-baseline__survey">
            <a href="#">Share feedback</a>
            <span class="personal-dashboard-baseline__beta">
              <CdxIcon :icon="cdxIconLabFlask" size="small" />
              Beta
            </span>
          </div>
        </header>

        <template v-if="isOptionBVariant">
          <a
            class="personal-dashboard-card personal-dashboard-card--linked"
            href="#"
            aria-labelledby="option-b-protection-heading"
            @click="openOptionBProtection"
          >
            <header class="personal-dashboard-card__header">
              <h2 id="option-b-protection-heading">Pages for protection</h2>
              <CdxIcon :icon="cdxIconNext" />
            </header>
            <ul class="personal-dashboard-baseline__protection-metrics" aria-label="Pages for protection details">
              <li aria-label="17 unresolved protection requests">
                <CdxIcon :icon="cdxIconSpeechBubble" />
                <strong>17</strong>
                <span>unresolved</span>
              </li>
              <li aria-label="6 protection requests have no admin reply">
                <CdxIcon :icon="cdxIconHistory" />
                <strong>6</strong>
                <span>no admin reply</span>
              </li>
              <li aria-label="Longest unresolved protection request has waited 22 hours">
                <CdxIcon :icon="cdxIconClock" />
                <strong>22h</strong>
                <span>longest wait</span>
              </li>
            </ul>
          </a>

          <a
            class="personal-dashboard-card personal-dashboard-card--linked"
            href="#"
            aria-labelledby="option-b-speedy-heading"
            @click="openOptionBSpeedy"
          >
            <header class="personal-dashboard-card__header">
              <h2 id="option-b-speedy-heading">Pages for speedy deletion</h2>
              <CdxIcon :icon="cdxIconNext" />
            </header>
            <ul class="personal-dashboard-baseline__protection-metrics" aria-label="Pages for speedy deletion details">
              <li aria-label="4 speedy deletion pages are awaiting admin review">
                <CdxIcon :icon="cdxIconTrash" />
                <strong>4</strong>
                <span>awaiting review</span>
              </li>
              <li aria-label="2 speedy deletion pages need an admin decision">
                <CdxIcon :icon="cdxIconHistory" />
                <strong>2</strong>
                <span>need decision</span>
              </li>
              <li aria-label="Longest speedy deletion page has waited 3 hours">
                <CdxIcon :icon="cdxIconClock" />
                <strong>3h</strong>
                <span>longest wait</span>
              </li>
            </ul>
          </a>
        </template>

        <section v-if="!isOptionBVariant" class="personal-dashboard-card" aria-labelledby="baseline-review-heading">
          <a class="personal-dashboard-card__header" href="#">
            <h2 id="baseline-review-heading">Review changes</h2>
            <CdxIcon :icon="cdxIconNext" />
          </a>
          <div class="personal-dashboard-baseline__review-summary">
            <CdxIcon :icon="cdxIconEdit" />
            <p>NewContributor edited the Page protection demo May 5 article</p>
          </div>
          <CdxButton action="progressive" weight="primary" class="personal-dashboard-baseline__full-button">
            View more edits
          </CdxButton>
        </section>

        <section v-if="!isOptionBVariant && !isOptionAVariant" class="personal-dashboard-card" aria-labelledby="baseline-impact-heading">
          <h2 id="baseline-impact-heading">Your impact</h2>
          <div class="personal-dashboard-baseline__impact">
            <div>
              <CdxIcon :icon="cdxIconSpeechBubble" />
              <strong>0</strong>
              <span>Thanks sent</span>
            </div>
            <div>
              <CdxIcon :icon="cdxIconCheck" />
              <strong>0</strong>
              <span>Edits reviewed</span>
              <CdxIcon :icon="cdxIconInfo" size="small" />
            </div>
          </div>
        </section>

        <section v-if="!isOptionBVariant" class="personal-dashboard-card" aria-labelledby="baseline-protection-heading">
          <a class="personal-dashboard-card__header" href="#" @click="openOptionAProtectionRequests">
            <h2 id="baseline-protection-heading">Protection requests</h2>
            <CdxIcon :icon="cdxIconNext" />
          </a>

          <template v-if="isOptionAVariant">
            <ul class="personal-dashboard-baseline__protection-metrics" aria-label="Protection request summary">
              <li aria-label="17 open protection requests">
                <CdxIcon :icon="cdxIconSpeechBubble" />
                <strong>17</strong>
                <span>open</span>
              </li>
              <li aria-label="6 unanswered protection requests">
                <CdxIcon :icon="cdxIconClock" />
                <strong>6</strong>
                <span>unanswered</span>
              </li>
              <li aria-label="Oldest open protection request has waited 22 hours">
                <CdxIcon :icon="cdxIconClock" />
                <strong>22h</strong>
                <span>oldest open</span>
              </li>
              <li aria-label="57 new protection requests in the last 24 hours">
                <CdxIcon :icon="cdxIconRecentChanges" />
                <strong>57</strong>
                <span>received today</span>
              </li>
            </ul>

            <CdxButton
              action="progressive"
              weight="primary"
              class="personal-dashboard-baseline__full-button"
              @click="openOptionAProtectionRequests"
            >
              View requests
            </CdxButton>
          </template>

          <template v-else>
            <div class="personal-dashboard-baseline__protection-request">
              <CdxIcon :icon="cdxIconSpeechBubble" />
              <span>Semi-protection request: Page prot...</span>
            </div>
            <CdxButton action="progressive" weight="primary" class="personal-dashboard-baseline__full-button">
              View more
            </CdxButton>
          </template>
        </section>

        <section
          v-if="isOptionAVariant"
          class="personal-dashboard-card"
          aria-labelledby="option-a-speedy-heading"
        >
          <a class="personal-dashboard-card__header" href="#">
            <h2 id="option-a-speedy-heading">Speedy deletion</h2>
            <CdxIcon :icon="cdxIconNext" />
          </a>

          <ul class="personal-dashboard-baseline__protection-metrics" aria-label="Speedy deletion summary">
            <li aria-label="4 pages tagged for speedy deletion">
              <CdxIcon :icon="cdxIconTrash" />
              <strong>4</strong>
              <span>tagged</span>
            </li>
              <li aria-label="2 speedy deletion nominations awaiting administrator check">
                <CdxIcon :icon="cdxIconClock" />
                <strong>2</strong>
                <span>not checked</span>
              </li>
              <li aria-label="Oldest speedy deletion nomination has waited 3 hours">
                <CdxIcon :icon="cdxIconClock" />
                <strong>3h</strong>
                <span>oldest open</span>
              </li>
              <li aria-label="27 new speedy deletion nominations in the last 24 hours">
                <CdxIcon :icon="cdxIconRecentChanges" />
                <strong>27</strong>
                <span>tagged today</span>
              </li>
          </ul>

          <CdxButton action="progressive" weight="primary" class="personal-dashboard-baseline__full-button">
            View pages
          </CdxButton>
        </section>

        <section class="personal-dashboard-card" aria-labelledby="baseline-policies-heading">
          <a class="personal-dashboard-card__header" href="#">
            <h2 id="baseline-policies-heading">Policies and guidelines</h2>
            <CdxIcon :icon="cdxIconNext" />
          </a>
          <p>Review best practices to create a free and reliable encyclopedia.</p>
        </section>

        <section v-if="isOptionAVariant" class="personal-dashboard-card" aria-labelledby="option-a-impact-heading">
          <h2 id="option-a-impact-heading">Your impact</h2>
          <div class="personal-dashboard-baseline__impact">
            <div>
              <CdxIcon :icon="cdxIconSpeechBubble" />
              <strong>0</strong>
              <span>Thanks sent</span>
            </div>
            <div>
              <CdxIcon :icon="cdxIconCheck" />
              <strong>0</strong>
              <span>Edits reviewed</span>
              <CdxIcon :icon="cdxIconInfo" size="small" />
            </div>
          </div>
        </section>
        </template>
      </template>

      <template v-else>
        <header class="moderator-dashboard__hero">
          <h1>Hello, {{ moderator.name }}</h1>
          <p class="moderator-dashboard__subtitle">
            {{ moderator.rights }} · {{ moderator.wiki }}
          </p>
        </header>

        <section class="moderator-dashboard__summary" aria-label="Moderation overview">
          <article
            v-for="summary in summaryItems"
            :key="summary.label"
            class="moderator-dashboard__summary-item"
            :class="{ 'moderator-dashboard__summary-item--strong': summary.strong }"
          >
            <CdxIcon :icon="summary.icon" />
            <span>
              <strong>{{ summary.value }}</strong>
              {{ summary.label }}
            </span>
          </article>
        </section>

        <section class="moderator-dashboard__section" aria-labelledby="judgment-heading">
          <div class="moderator-dashboard__section-heading">
            <div>
              <h2 id="judgment-heading">{{ activeVariant.heading }}</h2>
              <p>{{ activeVariant.description }}</p>
            </div>
            <CdxButton aria-label="Filter recommendations" weight="quiet">
              <CdxIcon :icon="cdxIconFunnel" />
            </CdxButton>
          </div>

          <div class="moderator-dashboard__task-list">
            <button
              v-for="task in visibleTasks"
              :key="task.id"
              class="moderator-dashboard__task-card"
              :class="{ 'moderator-dashboard__task-card--selected': task.id === selectedTask.id }"
              type="button"
              @click="selectedTaskId = task.id"
            >
              <span class="moderator-dashboard__task-icon">
                <CdxIcon :icon="taskIcon(task)" />
              </span>
              <span class="moderator-dashboard__task-body">
                <span class="moderator-dashboard__task-kicker">
                  {{ task.source }} · {{ task.age }}
                </span>
                <strong>{{ task.title }}</strong>
                <span>{{ task.request }}</span>
              </span>
              <span class="moderator-dashboard__priority">{{ task.priority }}</span>
            </button>
          </div>
        </section>

        <section class="moderator-dashboard__detail" aria-labelledby="task-detail-heading">
          <div class="moderator-dashboard__detail-header">
            <span class="moderator-dashboard__task-icon">
              <CdxIcon :icon="taskIcon(selectedTask)" />
            </span>
            <div>
              <p>{{ selectedTask.source }}</p>
              <h2 id="task-detail-heading">{{ selectedTask.title }}</h2>
            </div>
          </div>

          <div class="moderator-dashboard__question">
            <span>Judgment needed</span>
            <p>{{ selectedTask.judgment }}</p>
          </div>

          <div class="moderator-dashboard__reason">
            <span>Request rationale</span>
            <p>{{ selectedTask.reason }}</p>
          </div>

          <div class="moderator-dashboard__variant-note">
            <span>{{ activeVariant.lensLabel }}</span>
            <p>{{ activeVariant.lensCopy }}</p>
          </div>

          <div class="moderator-dashboard__signals">
            <h3>Why this is shown</h3>
            <ul>
              <li v-for="signal in selectedTask.signals" :key="signal">
                <CdxIcon :icon="cdxIconCheck" size="small" />
                <span>{{ signal }}</span>
              </li>
            </ul>
          </div>

          <div class="moderator-dashboard__actions">
            <CdxButton action="progressive" weight="primary">
              {{ selectedTask.nextSurface }}
              <CdxIcon :icon="cdxIconNext" />
            </CdxButton>
            <CdxButton>
              <CdxIcon :icon="cdxIconHistory" />
              Context
            </CdxButton>
          </div>
        </section>

        <section class="moderator-dashboard__section" aria-labelledby="backlogs-heading">
          <h2 id="backlogs-heading">Backlogs represented</h2>
          <div class="moderator-dashboard__backlogs">
            <article>
              <CdxIcon :icon="cdxIconSpeechBubble" />
              <div>
                <strong>Protection requests</strong>
                <p>Noticeboard queue that needs reply-back after action.</p>
              </div>
            </article>
            <article>
              <CdxIcon :icon="cdxIconArticle" />
              <div>
                <strong>Speedy deletion</strong>
                <p>Template/category queue that clears when deleted or untagged.</p>
              </div>
            </article>
            <article>
              <CdxIcon :icon="cdxIconRecentChanges" />
              <div>
                <strong>Review changes</strong>
                <p>Recent edits that may need patrol or revert judgment.</p>
              </div>
            </article>
          </div>
        </section>
      </template>

      <aside
        class="moderator-dashboard__variant-switcher"
        aria-label="Prototype variant switcher"
      >
        <button
          v-if="variantSwitcherOpen"
          class="moderator-dashboard__variant-scrim"
          type="button"
          aria-label="Close prototype variants"
          @click="variantSwitcherOpen = false"
        />

        <div
          v-if="variantSwitcherOpen"
          class="moderator-dashboard__variant-panel"
          role="dialog"
          aria-label="Prototype variants"
        >
          <div class="moderator-dashboard__variant-panel-header">
            <strong>Variants</strong>
            <CdxButton aria-label="Close prototype variants" weight="quiet" @click="variantSwitcherOpen = false">
              <CdxIcon :icon="cdxIconClose" />
            </CdxButton>
          </div>

          <button
            v-for="variant in variants.filter((v) => v.id !== 'option-a')"
            :key="variant.id"
            class="moderator-dashboard__variant-option"
            :class="{ 'moderator-dashboard__variant-option--active': variant.id === activeVariant.id }"
            type="button"
            @click="selectVariant(variant.id)"
          >
            <span>{{ variantOptionLabel(variant.id) }}</span>
          </button>
        </div>

        <CdxButton
          class="moderator-dashboard__variant-trigger"
          aria-label="Open prototype variants"
          :aria-expanded="variantSwitcherOpen"
          weight="quiet"
          @click="variantSwitcherOpen = !variantSwitcherOpen"
        >
          <CdxIcon :icon="cdxIconSettings" />
        </CdxButton>
      </aside>

    </div>
  </ChromeWrapper>
</template>

<style scoped>
.mock-minerva-header {
  display: flex;
  align-items: center;
  gap: var(--spacing-25, 4px);
  min-height: 3rem;
  padding: 0 var(--spacing-50, 8px);
  border-bottom: 1px solid var(--border-color-subtle);
  background-color: var(--background-color-interactive);
  box-shadow: inset 0 -1px 3px 0 rgba(0, 0, 0, 0.08);
}

.mock-minerva-header__brand {
  flex: 0 1 auto;
  overflow: hidden;
  color: var(--color-base);
  font-size: 1rem;
  font-weight: var(--font-weight-normal);
  line-height: 1;
  text-decoration: none;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.mock-minerva-header__actions {
  display: flex;
  flex-shrink: 0;
  align-items: center;
  margin-inline-start: auto;
}

.mock-minerva-header :deep(.cdx-button) {
  width: 2.25rem;
  min-width: 2.25rem;
  height: 2.25rem;
  min-height: 2.25rem;
  padding: var(--spacing-50, 8px);
}

.mock-minerva-header :deep(.cdx-icon) {
  width: 1.25rem;
  min-width: 1.25rem;
  height: 1.25rem;
}

.mock-minerva-header__actions :deep(.cdx-button) {
  color: var(--color-subtle);
}

.mock-minerva-header__notification-count {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 1.75rem;
  height: 1.75rem;
  border: 0;
  border-radius: 50%;
  background-color: #c0392b;
  color: var(--color-inverted);
  font: inherit;
  font-size: 1rem;
  font-weight: var(--font-weight-bold);
}

.mock-minerva-footer {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-100, 16px);
  margin-top: var(--spacing-200, 32px);
  padding: var(--spacing-150, 24px) var(--spacing-100, 16px);
  border-top: 1px solid var(--border-color-subtle);
  background-color: var(--background-color-interactive);
}

.mock-minerva-footer strong {
  font-size: var(--font-size-large);
}

.mock-minerva-footer nav {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-50, 8px);
  line-height: var(--line-height-medium);
}

.moderator-dashboard {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-125, 20px);
  max-width: 44rem;
  margin: 0 auto;
  padding: var(--spacing-100, 16px);
}

.moderator-dashboard--current-treatment {
  gap: var(--spacing-50, 8px);
}

.personal-dashboard-baseline__hero {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
  padding-block: var(--spacing-125, 20px) 0;
}

.moderator-dashboard--current-treatment .personal-dashboard-baseline__hero h1 {
  margin: 0;
  font-family:
    var(--font-family-system-sans, system-ui, sans-serif), var(--font-family-base, sans-serif);
  font-size: 1.5rem;
  font-weight: 600;
  line-height: 1.25;
  white-space: nowrap;
}

.personal-dashboard-baseline__survey {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: var(--spacing-75, 12px);
  font-size: 1.0625rem;
}

.personal-dashboard-baseline__beta {
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-25, 4px);
  padding: 1px var(--spacing-75, 12px);
  border: 1px solid var(--border-color-base);
  border-radius: 999px;
  background-color: var(--background-color-neutral-subtle);
  color: var(--color-base);
}

.personal-dashboard-card {
  padding: var(--spacing-150, 24px) var(--spacing-100, 16px) var(--spacing-100, 16px);
  border: 1px solid var(--border-color-subtle);
  border-radius: var(--border-radius-base);
  background-color: var(--background-color-base);
}

.personal-dashboard-card--linked {
  display: block;
  color: var(--color-base);
  text-decoration: none;
}

.personal-dashboard-card--linked:hover,
.personal-dashboard-card--linked:focus-visible {
  border-color: var(--border-color-progressive);
  outline: none;
}


.moderator-dashboard--current-treatment .personal-dashboard-card h2 {
  margin: 0;
  color: var(--color-base);
  font-size: 1.125rem;
  font-weight: 600;
  line-height: var(--line-height-medium);
}

.personal-dashboard-card p {
  margin: var(--spacing-100, 16px) 0 0;
  font-size: 1rem;
  line-height: var(--line-height-medium);
}

.personal-dashboard-card__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-100, 16px);
  color: var(--color-base);
  text-decoration: none;
}

.personal-dashboard-card__header :deep(.cdx-icon) {
  flex: none;
}

.personal-dashboard-card__header--static {
  cursor: default;
}

.personal-dashboard-card__caption {
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  letter-spacing: 0.02em;
  text-transform: uppercase;
}

.personal-dashboard-card--quiet {
  background-color: var(--background-color-base);
}

.personal-dashboard-card--footer {
  margin-top: var(--spacing-50, 8px);
  padding-block: var(--spacing-100, 16px);
  border-color: transparent;
  background-color: transparent;
}

.personal-dashboard-card--footer .personal-dashboard-card__header h2 {
  font-size: 1rem;
  font-weight: 500;
  color: var(--color-subtle);
}

.personal-dashboard-impact-stack {
  display: flex;
  flex-direction: column;
  margin: var(--spacing-100, 16px) 0 0;
  padding: 0;
}

.personal-dashboard-impact-stack > div {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: var(--spacing-100, 16px);
  padding: var(--spacing-75, 12px) 0;
  border-top: 1px solid var(--border-color-subtle);
}

.personal-dashboard-impact-stack > div:first-child {
  border-top: 0;
  padding-top: 0;
}

.personal-dashboard-impact-stack > div:last-child {
  padding-bottom: 0;
}

.personal-dashboard-impact-stack dt {
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-impact-stack dd {
  margin: 0;
  color: var(--color-base);
  font-size: 1.5rem;
  font-weight: 600;
  line-height: 1.1;
  font-variant-numeric: tabular-nums;
}

.personal-dashboard-impact-stack--compact > div {
  padding: var(--spacing-50, 8px) 0;
}

.personal-dashboard-impact-stack--compact dd {
  font-size: 1.25rem;
}

.personal-dashboard-card__lede {
  margin: var(--spacing-50, 8px) 0 0;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-queue-row {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
  margin-top: var(--spacing-100, 16px);
  padding-top: var(--spacing-75, 12px);
  border-top: 1px solid var(--border-color-subtle);
}

.personal-dashboard-queue-row__label {
  display: flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  color: var(--color-base);
  font-size: var(--font-size-small);
}

.personal-dashboard-queue-row__label :deep(.cdx-icon) {
  width: 1rem;
  min-width: 1rem;
  height: 1rem;
  color: var(--color-subtle);
}

.personal-dashboard-queue-row__label strong {
  font-weight: 600;
}

.personal-dashboard-queue-row__metrics {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-25, 4px) var(--spacing-100, 16px);
  margin: 0;
  padding: 0 0 0 calc(1rem + var(--spacing-50, 8px));
  list-style: none;
}

.personal-dashboard-queue-row__metrics li {
  display: inline-flex;
  align-items: baseline;
  gap: var(--spacing-25, 4px);
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-queue-row__metrics strong {
  color: var(--color-base);
  font-size: var(--font-size-small);
  font-weight: 600;
}

.personal-dashboard-baseline__review-summary {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr);
  gap: 6px;
  align-items: start;
  margin-top: var(--spacing-125, 20px);
  font-size: 1rem;
  line-height: 1.45;
}

.personal-dashboard-baseline__review-summary :deep(.cdx-icon) {
  width: 1.25rem;
  min-width: 1.25rem;
  height: 1.25rem;
  margin-top: 0.2em;
  color: var(--color-subtle);
}

.personal-dashboard-baseline__review-summary p {
  margin: 0;
  font-size: inherit;
  line-height: inherit;
}

.personal-dashboard-baseline__full-button {
  width: 100%;
  max-width: none;
  margin-top: var(--spacing-50, 8px);
}

.personal-dashboard-baseline__protection-request {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr);
  gap: var(--spacing-50, 8px);
  align-items: center;
  margin-top: var(--spacing-150, 24px);
  padding: var(--spacing-75, 12px);
  border: 1px solid var(--border-color-base);
  border-radius: var(--border-radius-base) var(--border-radius-base) 0 0;
  color: var(--color-subtle);
  font-size: 1rem;
  font-style: italic;
  line-height: var(--line-height-medium);
}

.personal-dashboard-baseline__protection-request span {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.personal-dashboard-baseline__protection-request + .personal-dashboard-baseline__full-button {
  margin-top: -1px;
  border-radius: 0 0 var(--border-radius-base) var(--border-radius-base);
}

.personal-dashboard-baseline__protection-metrics {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: var(--spacing-50, 8px) var(--spacing-75, 12px);
  margin: var(--spacing-75, 12px) 0 0;
  padding: var(--spacing-75, 12px) 0 0;
  border-top: 1px solid var(--border-color-subtle);
  list-style: none;
}

.personal-dashboard-baseline__protection-metrics li {
  display: grid;
  grid-template-columns: auto auto minmax(0, 1fr);
  gap: var(--spacing-25, 4px);
  align-items: center;
  min-width: 0;
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-baseline__protection-metrics :deep(.cdx-icon) {
  width: 1rem;
  min-width: 1rem;
  height: 1rem;
  color: var(--color-subtle);
}

.personal-dashboard-baseline__protection-metrics strong {
  color: var(--color-base);
  font-size: var(--font-size-small);
  font-weight: 600;
}

.personal-dashboard-baseline__protection-metrics span {
  overflow-wrap: anywhere;
}

.personal-dashboard-detail {
  display: flex;
  flex-direction: column;
  gap: 0;
  min-height: 100vh;
  background-color: var(--background-color-interactive);
}

.moderator-dashboard--detail-page {
  max-width: none;
  padding: 0;
}

.personal-dashboard-detail__header {
  display: grid;
  grid-template-columns: 3rem minmax(0, 1fr) 3rem;
  align-items: center;
  min-height: 4.25rem;
  padding: 0 var(--spacing-50, 8px);
  border-bottom: 1px solid var(--border-color-subtle);
  background-color: var(--background-color-base);
}

.personal-dashboard-detail__header :deep(.cdx-button) {
  grid-column: 1;
  justify-self: start;
}

.moderator-dashboard .personal-dashboard-detail__header h1 {
  grid-column: 2;
  margin: 0;
  color: var(--color-base);
  font-size: 1rem;
  font-weight: 600;
  line-height: 1.25;
  text-align: center;
}

.personal-dashboard-detail__body {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
  padding: var(--spacing-100, 16px);
}

.personal-dashboard-detail__toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-50, 8px) var(--spacing-100, 16px);
  margin-bottom: var(--spacing-50, 8px);
  padding-bottom: var(--spacing-75, 12px);
  border-bottom: 1px solid var(--border-color-subtle);
}

.personal-dashboard-detail__lede-line {
  margin: 0;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__lede-line strong {
  color: var(--color-base);
  font-weight: var(--font-weight-bold, 700);
}

.personal-dashboard-detail__sort-row {
  display: flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  color: var(--color-subtle);
  font-size: var(--font-size-small);
}

.personal-dashboard-detail__sort-label {
  flex: none;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  font-weight: 600;
}

.personal-dashboard-detail__sort-select {
  flex: 1;
  min-width: 0;
}

.personal-dashboard-detail__requests {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-100, 16px);
}

.personal-dashboard-detail__request {
  display: flex;
  flex-direction: column;
  gap: 0;
  padding: var(--spacing-100, 16px);
  border: 1px solid var(--border-color-subtle);
  border-radius: var(--border-radius-base);
  background: var(--background-color-base);
  cursor: pointer;
  transition: border-color 0.12s ease;
}

.personal-dashboard-detail__request:hover,
.personal-dashboard-detail__request:focus-within {
  border-color: var(--border-color-progressive);
}

.personal-dashboard-detail__title-line {
  margin: 0;
  color: var(--color-base);
  font-size: 1.0625rem;
  font-weight: var(--font-weight-bold, 700);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__request-title {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: var(--spacing-25, 4px) var(--spacing-50, 8px);
}

.personal-dashboard-detail__request-title h2 {
  margin: 0;
  color: var(--color-base);
  font-size: 1.0625rem;
  font-weight: var(--font-weight-bold, 700);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__flags {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-25, 4px);
  margin: 0;
  padding: 0;
  list-style: none;
}

.personal-dashboard-detail__flags li {
  padding: 1px var(--spacing-50, 8px);
  border: 1px solid var(--border-color-subtle);
  border-radius: 999px;
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__importance {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-25, 4px) var(--spacing-100, 16px);
  margin: var(--spacing-25, 4px) 0 0;
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__importance-item {
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-25, 4px);
}

.personal-dashboard-detail__importance-item :deep(.cdx-icon) {
  color: var(--color-subtle);
}

.personal-dashboard-detail__field {
  display: flex;
  align-items: baseline;
  flex-wrap: wrap;
  gap: var(--spacing-25, 4px) var(--spacing-50, 8px);
  margin: var(--spacing-25, 4px) 0 0;
}

.personal-dashboard-detail__field dt {
  margin: 0;
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: var(--font-weight-bold, 700);
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.personal-dashboard-detail__field dd {
  margin: 0;
  color: var(--color-base);
  font-size: var(--font-size-medium);
  font-weight: var(--font-weight-bold, 700);
}

.personal-dashboard-detail__request-summary {
  margin: var(--spacing-100, 16px) 0 0;
  color: var(--color-base);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__signal-label {
  margin: var(--spacing-100, 16px) 0 0;
  color: var(--color-base);
  font-size: var(--font-size-small);
  font-weight: var(--font-weight-bold, 700);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__signal-line,
.personal-dashboard-detail__status-line {
  margin: 0;
  color: var(--color-base);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__status-line {
  margin-top: var(--spacing-75, 12px);
  padding-top: var(--spacing-50, 8px);
  border-top: 1px solid var(--border-color-subtle);
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: var(--font-weight-bold, 700);
  letter-spacing: 0.04em;
  line-height: var(--line-height-small);
  text-transform: uppercase;
}

.personal-dashboard-detail__request-action {
  display: flex;
  justify-content: flex-start;
  margin: var(--spacing-50, 8px) 0 calc(var(--spacing-50, 8px) * -0.5);
}

.personal-dashboard-detail__request-action :deep(.cdx-button) {
  margin-inline-start: calc(var(--spacing-75, 12px) * -1);
}

.personal-dashboard-detail__criterion {
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: var(--font-weight-bold, 700);
  letter-spacing: 0.04em;
}

.personal-dashboard-detail__tagger {
  margin: 0;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__tagger strong {
  color: var(--color-base);
  font-weight: var(--font-weight-bold, 700);
}

.personal-dashboard-detail__lede {
  margin: var(--spacing-25, 4px) 0 0;
  padding: var(--spacing-50, 8px) var(--spacing-75, 12px);
  border-left: 2px solid var(--border-color-subtle);
  background: var(--background-color-neutral-subtle);
  color: var(--color-base);
  font-size: var(--font-size-small);
  font-style: italic;
  line-height: var(--line-height-medium);
}


.personal-dashboard-baseline__changes {
  display: flex;
  flex-direction: column;
  margin: var(--spacing-100, 16px) 0 0;
  padding: 0;
  list-style: none;
}

.personal-dashboard-baseline__changes li {
  padding: var(--spacing-100, 16px) var(--spacing-50, 8px);
  border-top: 1px solid var(--border-color-subtle);
}

.personal-dashboard-baseline__changes li:last-child {
  border-bottom: 1px solid var(--border-color-subtle);
}

.personal-dashboard-baseline__change-title {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: var(--spacing-25, 4px);
  font-size: 0.9375rem;
  line-height: var(--line-height-medium);
}

.personal-dashboard-baseline__change-title strong,
.personal-dashboard-baseline__change-title span {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.personal-dashboard-baseline__change-title strong {
  color: var(--color-base);
}

.personal-dashboard-baseline__change-title span {
  flex: none;
  color: var(--color-subtle);
}

.personal-dashboard-baseline__changes p {
  display: flex;
  gap: var(--spacing-25, 4px);
  margin: var(--spacing-75, 12px) 0 var(--spacing-50, 8px);
  color: var(--color-subtle);
  font-size: var(--font-size-medium);
  line-height: var(--line-height-medium);
}

.personal-dashboard-baseline__delta {
  color: #14866d;
}

.personal-dashboard-baseline__changes a {
  font-size: var(--font-size-medium);
}

.personal-dashboard-baseline__impact {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-75, 12px);
  margin-top: var(--spacing-100, 16px);
}

.personal-dashboard-baseline__impact div {
  display: grid;
  grid-template-columns: auto auto 1fr auto;
  align-items: center;
  gap: var(--spacing-50, 8px);
  color: var(--color-subtle);
  font-size: 1rem;
}

.personal-dashboard-baseline__impact strong {
  color: var(--color-progressive);
}

.personal-dashboard-baseline__impact div:last-child strong {
  color: var(--color-base);
}

.moderator-dashboard__user {
  display: inline-flex;
  align-items: center;
  gap: var(--spacing-25, 4px);
  color: var(--color-base);
  text-decoration: none;
}

.moderator-dashboard__hero {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
}

.moderator-dashboard__eyebrow,
.moderator-dashboard__subtitle,
.moderator-dashboard__section-heading p,
.moderator-dashboard__detail-header p,
.moderator-dashboard__question span,
.moderator-dashboard__reason span,
.moderator-dashboard__task-kicker,
.moderator-dashboard__backlogs p {
  margin: 0;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-small);
}

.moderator-dashboard h1,
.moderator-dashboard h2,
.moderator-dashboard h3 {
  margin: 0;
  font-family:
    var(--font-family-system-sans, system-ui, sans-serif), var(--font-family-base, sans-serif);
  color: var(--color-base);
}

.moderator-dashboard h1 {
  font-size: var(--font-size-xx-large);
  line-height: var(--line-height-xx-large);
}

.moderator-dashboard h2 {
  font-size: var(--font-size-large);
  line-height: var(--line-height-large);
}

.moderator-dashboard h3 {
  font-size: var(--font-size-medium);
  line-height: var(--line-height-medium);
}

.moderator-dashboard__summary {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  border: 1px solid var(--border-color-subtle);
}

.moderator-dashboard__summary-item {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
  min-width: 0;
  padding: var(--spacing-75, 12px);
  border-right: 1px solid var(--border-color-subtle);
  color: var(--color-subtle);
}

.moderator-dashboard__summary-item:last-child {
  border-right: 0;
}

.moderator-dashboard__summary-item strong {
  display: block;
  color: var(--color-base);
  font-size: var(--font-size-large);
}

.moderator-dashboard__summary-item--strong {
  background-color: var(--background-color-progressive-subtle);
}

.moderator-dashboard__message {
  margin: 0;
}

.moderator-dashboard__section,
.moderator-dashboard__detail {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-75, 12px);
}

.moderator-dashboard__section-heading {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: var(--spacing-75, 12px);
}

.moderator-dashboard__task-list {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
}

.moderator-dashboard__task-card {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr) auto;
  gap: var(--spacing-75, 12px);
  width: 100%;
  min-height: 5.5rem;
  padding: var(--spacing-75, 12px);
  border: 1px solid var(--border-color-subtle);
  background: var(--background-color-base);
  color: var(--color-base);
  font: inherit;
  text-align: left;
}

.moderator-dashboard__task-card--selected {
  border-color: var(--border-color-progressive);
  box-shadow: inset 3px 0 0 var(--border-color-progressive);
}

.moderator-dashboard__task-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 2rem;
  height: 2rem;
  border-radius: var(--border-radius-base);
  background-color: var(--background-color-neutral-subtle);
  color: var(--color-subtle);
}

.moderator-dashboard__task-body {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
  min-width: 0;
}

.moderator-dashboard__task-body strong,
.moderator-dashboard__task-body span {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.moderator-dashboard__priority {
  align-self: start;
  padding: 2px var(--spacing-25, 4px);
  border: 1px solid var(--border-color-subtle);
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
}

.moderator-dashboard__detail {
  padding: var(--spacing-100, 16px);
  border: 1px solid var(--border-color-subtle);
  background-color: var(--background-color-neutral-subtle);
}

.moderator-dashboard__detail-header {
  display: flex;
  gap: var(--spacing-75, 12px);
  align-items: flex-start;
}

.moderator-dashboard__question,
.moderator-dashboard__reason,
.moderator-dashboard__variant-note {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
  padding-top: var(--spacing-75, 12px);
  border-top: 1px solid var(--border-color-subtle);
}

.moderator-dashboard__question p,
.moderator-dashboard__reason p,
.moderator-dashboard__variant-note p {
  margin: 0;
  line-height: var(--line-height-medium);
}

.moderator-dashboard__variant-note {
  border-color: var(--border-color-progressive);
}

.moderator-dashboard__signals {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
}

.moderator-dashboard__signals ul {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
  margin: 0;
  padding: 0;
  list-style: none;
}

.moderator-dashboard__signals li,
.moderator-dashboard__backlogs article {
  display: flex;
  gap: var(--spacing-50, 8px);
  align-items: flex-start;
}

.moderator-dashboard__actions {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--spacing-50, 8px);
}

.moderator-dashboard__actions :deep(.cdx-button) {
  width: 100%;
  max-width: none;
}

.moderator-dashboard__backlogs {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
}

.moderator-dashboard__backlogs article {
  padding: var(--spacing-75, 12px);
  border: 1px solid var(--border-color-subtle);
}

.moderator-dashboard__backlogs p {
  margin-top: var(--spacing-25, 4px);
}

.moderator-dashboard__variant-switcher {
  position: fixed;
  right: var(--spacing-50, 8px);
  bottom: calc(var(--spacing-50, 8px) + env(safe-area-inset-bottom));
  z-index: 20;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: var(--spacing-50, 8px);
}

.moderator-dashboard__variant-scrim {
  position: fixed;
  inset: 0;
  z-index: 25;
  border: 0;
  background: rgba(0, 0, 0, 0.16);
}

.moderator-dashboard__variant-trigger {
  width: 1.75rem;
  min-width: 1.75rem;
  height: 1.75rem;
  min-height: 1.75rem;
  padding: 0;
  border-radius: 50%;
  border-color: var(--border-color-subtle);
  background: var(--background-color-base);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

.moderator-dashboard__variant-trigger :deep(.cdx-icon) {
  width: 0.875rem;
  min-width: 0.875rem;
  height: 0.875rem;
}

.moderator-dashboard__variant-panel {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  z-index: 30;
  width: min(18rem, 84vw);
  padding: var(--spacing-100, 16px);
  border: 0;
  border-left: 1px solid var(--border-color-subtle);
  background: var(--background-color-base);
  box-shadow: -4px 0 16px rgba(0, 0, 0, 0.18);
}

.moderator-dashboard__variant-panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-50, 8px);
  margin-bottom: var(--spacing-100, 16px);
  padding-bottom: var(--spacing-75, 12px);
  border-bottom: 1px solid var(--border-color-subtle);
}

.moderator-dashboard__variant-option {
  display: block;
  width: 100%;
  padding: var(--spacing-75, 12px);
  border: 1px solid transparent;
  background: transparent;
  color: var(--color-base);
  font: inherit;
  text-align: left;
}

.moderator-dashboard__variant-option + .moderator-dashboard__variant-option {
  margin-top: var(--spacing-25, 4px);
}

.moderator-dashboard__variant-option--active {
  border-color: var(--border-color-progressive);
  background: var(--background-color-progressive-subtle);
}

@media (min-width: 48rem) {
  .moderator-dashboard {
    padding-block: var(--spacing-150, 24px);
  }

  .moderator-dashboard__actions {
    grid-template-columns: 1fr 1fr;
  }
}
</style>
