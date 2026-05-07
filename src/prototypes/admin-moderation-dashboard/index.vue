<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import {
  CdxButton,
  CdxCheckbox,
  CdxDialog,
  CdxIcon,
  CdxInfoChip,
  CdxRadio,
  CdxSearchInput,
  CdxSelect,
} from '@wikimedia/codex'
import {
  cdxIconAlert,
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
  cdxIconSpeechBubbles,
  cdxIconTrash,
  cdxIconTray,
  cdxIconUserAvatar,
  cdxIconUserAvatarOutline,
  cdxIconUserGroup,
  type Icon,
} from '@wikimedia/codex-icons'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Admin moderation dashboard',
    description: 'Mobile logged-in dashboard prototype for admin moderation triage.',
  },
})

const router = useRouter()
const route = useRoute()

type TaskType = 'protection' | 'deletion'
type TaskStatus = 'needs-review' | 'in-discussion' | 'already-handled'
type DashboardVariantId = 'base' | 'option-a' | 'option-b' | 'current' | 'prototype-v2'
type CardTreatmentId = 'cards-current' | 'cards-compact' | 'cards-evidence' | 'cards-attention'
type ProtectionCardState = 'calm' | 'stale'
type SpeedyCardState = 'calm' | 'contested' | 'stale'
type SpeedyCriterionNamespace = 'general' | 'article' | 'category' | 'file' | 'redirect' | 'template' | 'user'

const defaultVariantId: DashboardVariantId = 'prototype-v2'
const defaultCardTreatmentId: CardTreatmentId = 'cards-attention'
const defaultProtectionState: ProtectionCardState = 'stale'
const defaultSpeedyState: SpeedyCardState = 'stale'

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

interface CardTreatment {
  id: CardTreatmentId
  label: string
}

interface CardStateOption<T extends string> {
  id: T
  label: string
}

interface AttentionCardSummary {
  leadValue: string
  leadLabel: string
  secondary: string
  actionLabel?: string
  needsAttention: boolean
}

interface ProtectionRequestCard {
  id: string
  title: string
  visibility: string
  requestSummary: string
  requesterQuote: string
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
  criterionCode: string
  criterion: string
  visibility: string
  requestSummary: string
  taggerReason: string
  signalLabel: string
  signalLine: string
  status: string
  ageMinutes: number
  contested: boolean
  decisionScore: number
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
    label: 'Prototype v1',
    heading: 'Current Personal Dashboard',
    description: 'Latest prototype version with admin backlog cards and clarified queue metrics.',
    lensLabel: 'Prototype v1',
    lensCopy: 'Latest iteration: current actionability is separated from incoming volume.',
  },
  {
    id: 'prototype-v2',
    label: 'Prototype v2 (latest)',
    heading: 'Current Personal Dashboard',
    description: 'Next prototype version cloned from v1 for the next feedback iteration.',
    lensLabel: 'Prototype v2',
    lensCopy: 'Next iteration canvas: starts from v1 so new changes can diverge without disturbing it.',
  },
]

const cardTreatments: CardTreatment[] = [
  {
    id: 'cards-current',
    label: 'Current',
  },
  {
    id: 'cards-compact',
    label: 'Cards: compact',
  },
  {
    id: 'cards-evidence',
    label: 'Cards: evidence-first',
  },
  {
    id: 'cards-attention',
    label: 'Attention',
  },
]

const visibleCardTreatments = cardTreatments.filter(
  ({ id }) => id === 'cards-current' || id === 'cards-attention',
)

const protectionStateOptions: CardStateOption<ProtectionCardState>[] = [
  {
    id: 'calm',
    label: 'Calm',
  },
  {
    id: 'stale',
    label: 'Stale',
  },
]

const speedyStateOptions: CardStateOption<SpeedyCardState>[] = [
  {
    id: 'calm',
    label: 'Calm',
  },
  {
    id: 'contested',
    label: 'Contested',
  },
  {
    id: 'stale',
    label: 'Stale',
  },
]

interface SpeedyCriterion {
  code: string
  name: string
  namespace: SpeedyCriterionNamespace
}

const speedyCriteria: SpeedyCriterion[] = [
  { code: 'G1', name: 'Patent nonsense', namespace: 'general' },
  { code: 'G2', name: 'Test pages', namespace: 'general' },
  { code: 'G3', name: 'Pure vandalism / blatant hoaxes', namespace: 'general' },
  { code: 'G4', name: 'Recreated previously deleted page', namespace: 'general' },
  { code: 'G5', name: 'Created by banned/blocked user', namespace: 'general' },
  { code: 'G6', name: 'Housekeeping', namespace: 'general' },
  { code: 'G7', name: 'Author request', namespace: 'general' },
  { code: 'G8', name: 'Dependent on non-existent / broken redirect', namespace: 'general' },
  { code: 'G10', name: 'Attack pages', namespace: 'general' },
  { code: 'G11', name: 'Unambiguous advertising', namespace: 'general' },
  { code: 'G12', name: 'Unambiguous copyright violation', namespace: 'general' },
  { code: 'G13', name: 'Abandoned drafts / AfC submissions', namespace: 'general' },
  { code: 'G14', name: 'Unnecessary disambiguation pages', namespace: 'general' },
  { code: 'G15', name: 'Unreviewed LLM content', namespace: 'general' },
  { code: 'A1', name: 'No context', namespace: 'article' },
  { code: 'A2', name: 'Exists on foreign Wikimedia project', namespace: 'article' },
  { code: 'A3', name: 'No content', namespace: 'article' },
  { code: 'A7', name: 'Significance not indicated', namespace: 'article' },
  { code: 'A9', name: 'No significance (musical recordings)', namespace: 'article' },
  { code: 'A10', name: 'Recently created duplicate', namespace: 'article' },
  { code: 'A11', name: 'Obviously invented', namespace: 'article' },
  { code: 'C1', name: 'Empty categories', namespace: 'category' },
  { code: 'C4', name: 'Unused maintenance categories', namespace: 'category' },
  { code: 'F1', name: 'Redundant files', namespace: 'file' },
  { code: 'F2', name: 'Missing or corrupt files', namespace: 'file' },
  { code: 'F3', name: 'Unacceptably licensed files', namespace: 'file' },
  { code: 'F5', name: 'Orphaned non-free use files', namespace: 'file' },
  { code: 'F7', name: 'Clearly invalid fair-use files', namespace: 'file' },
  { code: 'F9', name: 'Copyright-infringing files', namespace: 'file' },
  { code: 'R2', name: 'Inappropriate cross-namespace redirects', namespace: 'redirect' },
  { code: 'R3', name: 'Implausible redirects / typos', namespace: 'redirect' },
  { code: 'R4', name: 'Redirects shadowing Commons pages', namespace: 'redirect' },
  { code: 'T2', name: 'Misrepresentation of policy', namespace: 'template' },
  { code: 'T3', name: 'Duplicated/hardcoded templates', namespace: 'template' },
  { code: 'T5', name: 'Unused template subpage', namespace: 'template' },
  { code: 'U1', name: 'User request to delete own page', namespace: 'user' },
  { code: 'U2', name: 'Userpage of nonexistent user', namespace: 'user' },
  { code: 'U3', name: 'Non-free galleries in user space', namespace: 'user' },
  { code: 'U5', name: 'Misuse as a web host', namespace: 'user' },
]

const criterionNamespaces: SpeedyCriterionNamespace[] = [
  'general',
  'article',
  'category',
  'file',
  'redirect',
  'template',
  'user',
]
const criterionNamespaceLabels = {
  general: 'GENERAL',
  article: 'ARTICLE',
  category: 'CATEGORY',
  file: 'FILE',
  redirect: 'REDIRECT',
  template: 'TEMPLATE',
  user: 'USER',
} satisfies Record<SpeedyCriterionNamespace, string>
const pinnedCriterionCodes = ['G7', 'G15', 'G11', 'G6'] as const
const pinnedSpeedyCriteria = pinnedCriterionCodes
  .map((code) => speedyCriteria.find((criterion) => criterion.code === code))
  .filter((criterion): criterion is SpeedyCriterion => Boolean(criterion))

const protectionAttentionSummaries = {
  calm: {
    leadValue: '17',
    leadLabel: 'open',
    secondary: 'oldest 4h (within 12h target) · 6 no admin reply',
    needsAttention: false,
  },
  stale: {
    leadValue: '22h',
    leadLabel: 'oldest unanswered',
    secondary: 'over 12h target · 17 open, 6 no admin reply',
    actionLabel: 'Open queue →',
    needsAttention: true,
  },
} satisfies Record<ProtectionCardState, AttentionCardSummary>

const speedyAttentionSummaries = {
  calm: {
    leadValue: '4',
    leadLabel: 'in queue',
    secondary: 'oldest 3h (within 6h target) · 0 contested',
    needsAttention: false,
  },
  contested: {
    leadValue: '2',
    leadLabel: 'contested — needs your judgment',
    secondary: '4 in queue · oldest 3h (within target)',
    actionLabel: 'Open contested →',
    needsAttention: true,
  },
  stale: {
    leadValue: '8h',
    leadLabel: 'oldest tagged',
    secondary: 'over 6h target · 7 in queue · 0 contested',
    actionLabel: 'Open queue →',
    needsAttention: true,
  },
} satisfies Record<SpeedyCardState, AttentionCardSummary>

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
  'prototype-v2': [
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
const activeCardTreatmentId = ref<CardTreatmentId>(getInitialCardTreatment())
const activeProtectionState = ref<ProtectionCardState>(getInitialProtectionState())
const activeSpeedyState = ref<SpeedyCardState>(getInitialSpeedyState())
const activeCriteria = ref<string[]>(getInitialCriteria())
const criterionSheetOpen = ref(false)
const modalSearchQuery = ref('')
const optionAProtectionViewOpen = ref(getInitialVariant() === 'option-a' && getInitialModule() === 'protection')
const optionBProtectionViewOpen = ref(isAdminPrototypeVariantId(getInitialVariant()) && getInitialModule() === 'protection')
const optionBSpeedyViewOpen = ref(isAdminPrototypeVariantId(getInitialVariant()) && getInitialModule() === 'speedy')
const selectedTask = computed(() => tasks.value.find((task) => task.id === selectedTaskId.value) ?? tasks.value[0])
const activeVariant = computed(() => variants.find((variant) => variant.id === activeVariantId.value) ?? variants[0])
const displayedVariants = computed(() => variants.slice().reverse())
const activeCardTreatment = computed(() => cardTreatments.find((treatment) => treatment.id === activeCardTreatmentId.value) ?? cardTreatments[0])
const selectedCardTreatmentId = computed<CardTreatmentId>({
  get: () => activeCardTreatmentId.value,
  set: (value) => selectCardTreatment(value),
})
const selectedProtectionState = computed<ProtectionCardState>({
  get: () => activeProtectionState.value,
  set: (value) => selectProtectionState(value),
})
const selectedSpeedyState = computed<SpeedyCardState>({
  get: () => activeSpeedyState.value,
  set: (value) => selectSpeedyState(value),
})
const usesCurrentDashboardTreatment = computed(() => activeVariant.value.id === 'base' || activeVariant.value.id === 'option-a' || isAdminPrototypeVariantId(activeVariant.value.id))
const isOptionAVariant = computed(() => activeVariant.value.id === 'option-a')
const isOptionBVariant = computed(() => isAdminPrototypeVariantId(activeVariant.value.id))
const isCardDirectionVariant = computed(() => isCardDirectionVariantId(activeVariant.value.id))
const isEvidenceCardTreatment = computed(() => isCardDirectionVariant.value && activeCardTreatment.value.id === 'cards-evidence')
const isAttentionCardTreatment = computed(() => isCardDirectionVariant.value && isStatefulCardTreatment(activeCardTreatment.value.id))
const usesPrototypeV2EvidenceFooter = computed(() => activeVariant.value.id === 'prototype-v2' && isAttentionCardTreatment.value)
const usesPrototypeV2ProtectionCardVisuals = computed(() => activeVariant.value.id === 'prototype-v2' && isAttentionCardTreatment.value)
const usesPrototypeV2UrgencyCalibration = computed(() => activeVariant.value.id === 'prototype-v2' && isAttentionCardTreatment.value)
const activeProtectionAttentionSummary = computed(() => protectionAttentionSummaries[activeProtectionState.value])
const activeSpeedyAttentionSummary = computed(() => speedyAttentionSummaries[activeSpeedyState.value])
const isProtectionHomeAlarm = computed(() =>
  activeProtectionAttentionSummary.value.needsAttention &&
  !usesPrototypeV2UrgencyCalibration.value
)
const isSpeedyHomeAlarm = computed(() =>
  activeSpeedyAttentionSummary.value.needsAttention &&
  (
    usesPrototypeV2UrgencyCalibration.value
      ? activeSpeedyState.value === 'contested'
      : true
  )
)
const isProtectionHomeNeutralTimeSignal = computed(() =>
  usesPrototypeV2UrgencyCalibration.value &&
  activeProtectionAttentionSummary.value.needsAttention
)
const isSpeedyHomeNeutralTimeSignal = computed(() =>
  usesPrototypeV2UrgencyCalibration.value &&
  activeSpeedyAttentionSummary.value.needsAttention &&
  activeSpeedyState.value !== 'contested'
)
const cardTreatmentClass = computed(() => isCardDirectionVariant.value ? `moderator-dashboard--${activeCardTreatment.value.id}` : '')
const isOptionAProtectionDetail = computed(() => isOptionAVariant.value && optionAProtectionViewOpen.value)
const isOptionBProtectionDetail = computed(() => isOptionBVariant.value && optionBProtectionViewOpen.value)
const isOptionBSpeedyDetail = computed(() => isOptionBVariant.value && optionBSpeedyViewOpen.value)
const isAnyDetailOpen = computed(() => isOptionAProtectionDetail.value || isOptionBProtectionDetail.value || isOptionBSpeedyDetail.value)
const summaryItems = computed(() => summaryByVariant[activeVariant.value.id])

const visibleTasks = computed(() => tasks.value.filter((task) => task.status !== 'already-handled'))

const protectionSortItems = [
  { value: 'most-active', label: 'Most active' },
  { value: 'oldest', label: 'Waiting longest' },
  { value: 'newest', label: 'Recently requested' },
]
const protectionSortValue = ref(
  isStatefulCardTreatment(activeCardTreatmentId.value) ? getDefaultProtectionSort(activeProtectionState.value) : 'most-active'
)

const pinnedProtectionCategories = [
  'Vandalism',
  'Edit warring',
  'BLP',
  'Sockpuppetry',
  'Disruptive editing',
  'Arbitration',
  'Paid · Recreation',
  'High-risk template',
] as const
const activeProtectionCategories = ref<string[]>([])

const optionBProtectionRequests: ProtectionRequestCard[] = [
  {
    id: 'eurovision',
    title: 'Eurovision Song Contest Asia',
    visibility: '12k views/day · ↑ 4× this week',
    requestSummary: 'Semi-protection requested for 1 week',
    requesterQuote: 'Repeated disruptive edits by multiple new and anonymous editors.',
    signalLabel: 'Edit warring',
    signalLine: '24 reverts in 6h · last revert 12m ago',
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
    requesterQuote: 'Off-topic suicide-encouragement re-posts after we cleared the page two weeks ago.',
    signalLabel: 'Disruptive editing',
    signalLine: '5 off-topic posts in 24h · last 18h ago',
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
    requesterQuote: 'Repeated diagnosis-claims about named living people. Diffs supplied. BLP risk.',
    signalLabel: 'BLP',
    signalLine: '4 named-person edits · 9 reverts in 24h',
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
    requesterQuote: 'Persistent IP socking adds unsourced material after each warning. Temp account already blocked but disruption continues.',
    signalLabel: 'Sockpuppetry',
    signalLine: '3 IPs · 1 temp account blocked · 8 reverts in 48h',
    status: 'No admin reply',
    activeScore: 45,
    actorCount: 4,
    ageHours: 22,
  },
  {
    id: 'groundhog-day',
    title: 'Groundhog Day (film)',
    visibility: '5.1k views/day · stable',
    requestSummary: 'Temporary full protection requested',
    requesterQuote: 'Talk-page dispute over plot wording. Multiple editors won\'t accept the consensus version.',
    signalLabel: 'Edit warring',
    signalLine: '14 reverts in 12h · 22-comment talk thread',
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
    requesterQuote: 'Vandalism resumed immediately after the previous protection expired.',
    signalLabel: 'Vandalism',
    signalLine: 'Protection expired 2h ago · 7 reverts since',
    status: 'No admin reply',
    activeScore: 85,
    actorCount: 3,
    ageHours: 2,
  },
]

const overTargetProtectionCount = computed(() => optionBProtectionRequests.filter((request) => request.ageHours > 12).length)
const showProtectionOverTargetHeader = computed(() =>
  isAttentionCardTreatment.value &&
  activeProtectionState.value === 'stale' &&
  protectionSortValue.value === 'oldest'
)

const protectionCategoryCounts = computed(() => {
  const counts = new Map<string, number>()
  for (const cat of pinnedProtectionCategories) {
    counts.set(cat, 0)
  }
  for (const r of optionBProtectionRequests) {
    counts.set(r.signalLabel, (counts.get(r.signalLabel) ?? 0) + 1)
  }
  return counts
})
const filteredOptionBProtectionRequests = computed(() => {
  if (activeProtectionCategories.value.length === 0) {
    return optionBProtectionRequests as readonly ProtectionRequestCard[]
  }
  const set = new Set(activeProtectionCategories.value)
  return optionBProtectionRequests.filter((r) => set.has(r.signalLabel))
})
const sortedOptionBProtectionRequests = computed(() => {
  const requests = [...filteredOptionBProtectionRequests.value]
  const byMostActive = (a: ProtectionRequestCard, b: ProtectionRequestCard) =>
    b.activeScore - a.activeScore ||
    b.actorCount - a.actorCount ||
    b.ageHours - a.ageHours

  switch (protectionSortValue.value) {
    case 'oldest':
      return requests.sort((a, b) => b.ageHours - a.ageHours || byMostActive(a, b))
    case 'newest':
      return requests.sort((a, b) => a.ageHours - b.ageHours || byMostActive(a, b))
    case 'most-active':
    default:
      return requests.sort(byMostActive)
  }
})

function getProtectionCategoryCount(cat: string) {
  return protectionCategoryCounts.value.get(cat) ?? 0
}

function toggleProtectionCategory(cat: string) {
  const idx = activeProtectionCategories.value.indexOf(cat)
  if (idx === -1) {
    activeProtectionCategories.value = [...activeProtectionCategories.value, cat]
  } else {
    activeProtectionCategories.value = activeProtectionCategories.value.filter((c) => c !== cat)
  }
}

function clearProtectionCategoryFilters() {
  activeProtectionCategories.value = []
}

const speedySortItems = [
  { value: 'contested', label: 'Contested first' },
  { value: 'newest', label: 'Recently tagged' },
  { value: 'oldest', label: 'Waiting longest' },
]
const speedySortValue = ref(
  isStatefulCardTreatment(activeCardTreatmentId.value) ? getDefaultSpeedySort(activeSpeedyState.value) : 'contested'
)

const optionBSpeedyDeletionRequests: SpeedyDeletionCard[] = [
  {
    id: 'smith-industries',
    title: 'Smith Industries Holdings Ltd',
    criterionCode: 'G11',
    criterion: 'G11 · Advertising',
    visibility: '42 views in last hour',
    requestSummary: 'Tagged 14m ago · company-style article',
    taggerReason: 'Promotional tone throughout; reads like marketing copy with no neutral coverage.',
    signalLabel: 'Admin question',
    signalLine: 'Is this only promotion, or is a neutral rewrite plausible?',
    status: 'Needs decision',
    ageMinutes: 14,
    contested: true,
    decisionScore: 90,
  },
  {
    id: 'regional-innovation',
    title: 'Regional innovation timeline',
    criterionCode: 'G15',
    criterion: 'G15 · Unreviewed LLM content',
    visibility: '90 views · contested',
    requestSummary: 'Tagged 3h ago · disputed on talk page',
    taggerReason: 'Generated text patterns; references hallucinated; submitted via AfC by author.',
    signalLabel: 'Admin question',
    signalLine: 'Does the speedy criterion still apply, or should this move to discussion?',
    status: 'Contested tag',
    ageMinutes: 180,
    contested: true,
    decisionScore: 80,
  },
  {
    id: 'henrik-olsen',
    title: 'Henrik Olsen (footballer)',
    criterionCode: 'A1',
    criterion: 'A1 · No context',
    visibility: '4 views · new page',
    requestSummary: 'Tagged 2m after page creation',
    taggerReason: 'Sub-stub: no context, no claims of notability, no sources.',
    signalLabel: 'Admin caution',
    signalLine: 'Creator may still be editing; deletion could be premature.',
    status: 'Too new to judge',
    ageMinutes: 2,
    contested: false,
    decisionScore: 30,
  },
  {
    id: 'orphan-template',
    title: 'Talk:Discontinued template (orphan)',
    criterionCode: 'G6',
    criterion: 'G6 · Housekeeping',
    visibility: 'Housekeeping page · low traffic',
    requestSummary: 'Tagged 1d ago · target template deleted',
    taggerReason: 'Talk page for deleted template; orphaned 7 days.',
    signalLabel: 'Routine check',
    signalLine: 'Confirm no useful talk history before deleting.',
    status: 'Housekeeping cleanup',
    ageMinutes: 1440,
    contested: false,
    decisionScore: 10,
  },
]

const showPrototypeV2SpeedyCriterionFilter = computed(() =>
  activeVariant.value.id === 'prototype-v2' &&
  isAttentionCardTreatment.value &&
  optionBSpeedyViewOpen.value
)
const hasActiveCriteriaFilter = computed(() => activeCriteria.value.length > 0)
const hasAnyActiveFilter = computed(() => hasActiveCriteriaFilter.value)
const isPrototypeV2SpeedyCriterionFiltered = computed(() =>
  showPrototypeV2SpeedyCriterionFilter.value && hasAnyActiveFilter.value
)
const visibleChipCodes = computed(() => {
  const codes = [...pinnedCriterionCodes] as string[]
  for (const code of activeCriteria.value) {
    if (!codes.includes(code)) {
      codes.push(code)
    }
  }
  return codes
})
const criteriaByNamespace = computed(() =>
  criterionNamespaces.map((namespace) => ({
    namespace,
    label: criterionNamespaceLabels[namespace],
    criteria: speedyCriteria.filter((criterion) => criterion.namespace === namespace),
  }))
)
const filteredCriteriaByNamespace = computed(() => {
  const query = modalSearchQuery.value.trim().toLowerCase()
  if (!query) {
    return criteriaByNamespace.value
  }
  return criteriaByNamespace.value
    .map((group) => ({
      ...group,
      criteria: group.criteria.filter(
        (c) => c.code.toLowerCase().includes(query) || c.name.toLowerCase().includes(query)
      ),
    }))
    .filter((group) => group.criteria.length > 0)
})
const criterionCounts = computed(() => {
  const counts = new Map<string, number>()
  speedyCriteria.forEach((criterion) => counts.set(criterion.code, 0))
  optionBSpeedyDeletionRequests.forEach((request) => {
    counts.set(request.criterionCode, (counts.get(request.criterionCode) ?? 0) + 1)
  })
  return counts
})
const allCriterionCount = computed(() => optionBSpeedyDeletionRequests.length)
const filteredOptionBSpeedyDeletionRequests = computed(() => {
  if (!hasActiveCriteriaFilter.value) {
    return optionBSpeedyDeletionRequests
  }
  const set = new Set(activeCriteria.value)
  return optionBSpeedyDeletionRequests.filter((r) => set.has(r.criterionCode))
})
const showSpeedyCriterionEmptyState = computed(() =>
  showPrototypeV2SpeedyCriterionFilter.value &&
  hasAnyActiveFilter.value &&
  sortedOptionBSpeedyDeletionRequests.value.length === 0
)

const sortedOptionBSpeedyDeletionRequests = computed(() => {
  const requests = [...filteredOptionBSpeedyDeletionRequests.value]

  switch (speedySortValue.value) {
    case 'contested':
      return requests.sort((a, b) =>
        Number(b.contested) - Number(a.contested) ||
        b.ageMinutes - a.ageMinutes
      )
    case 'oldest':
      return requests.sort((a, b) => b.ageMinutes - a.ageMinutes)
    case 'newest':
    default:
      return requests.sort((a, b) => a.ageMinutes - b.ageMinutes)
  }
})

function taskIcon(task: Task) {
  return task.type === 'protection' ? cdxIconLock : cdxIconTrash
}

function isDashboardVariant(value: string | null): value is DashboardVariantId {
  return (
    value === 'base' ||
    value === 'option-a' ||
    value === 'option-b' ||
    value === 'current' ||
    value === 'prototype-v2'
  )
}

function isCardTreatment(value: string | null): value is CardTreatmentId {
  return (
    value === 'cards-current' ||
    value === 'cards-compact' ||
    value === 'cards-evidence' ||
    value === 'cards-attention'
  )
}

function isStatefulCardTreatment(value: CardTreatmentId) {
  return value === 'cards-attention'
}

function isCardDirectionVariantId(value: DashboardVariantId) {
  return value === 'current' || value === 'prototype-v2'
}

function isProtectionCardState(value: string | null): value is ProtectionCardState {
  return value === 'calm' || value === 'stale'
}

function isSpeedyCardState(value: string | null): value is SpeedyCardState {
  return value === 'calm' || value === 'contested' || value === 'stale'
}

function getCriterionCount(code: string) {
  return criterionCounts.value.get(code) ?? 0
}

function normalizeCriteriaSelection(value: string | null): string[] {
  if (!value) {
    return []
  }

  const codes = value
    .split(',')
    .map((part) => part.trim().toUpperCase())
    .filter((code) => code && speedyCriteria.some((c) => c.code === code))

  return Array.from(new Set(codes))
}

function isAdminPrototypeVariantId(value: DashboardVariantId) {
  return value === 'option-b' || value === 'current' || value === 'prototype-v2'
}

function getDefaultProtectionSort(_state: ProtectionCardState) {
  return 'most-active'
}

function getDefaultSpeedySort(state: SpeedyCardState) {
  switch (state) {
    case 'calm':
      return 'newest'
    case 'stale':
      return 'oldest'
    case 'contested':
    default:
      return 'contested'
  }
}

function isProtectionRequestUrgent(request: ProtectionRequestCard) {
  if (usesPrototypeV2UrgencyCalibration.value) {
    return false
  }

  return activeProtectionState.value === 'stale' && request.ageHours > 12
}

function getProtectionSignalRecency(request: ProtectionRequestCard) {
  const recencyMatch = request.signalLine.match(/\b(?:Last revert|Last activity|Tagged) (\d+[mh] ago)/)
  return recencyMatch?.[1] ?? `${request.ageHours}h ago`
}

function getProtectionActorIcon(request: ProtectionRequestCard) {
  return request.actorCount === 1 ? cdxIconUserAvatarOutline : cdxIconUserGroup
}

function getProtectionSignalStats(request: ProtectionRequestCard) {
  const stats = request.signalLine
    .split(' · ')
    .map((part) => part.trim())
    .filter((part) => part && !/^(?:Last revert|Last activity|Tagged) \d+[mh] ago$/.test(part))

  return stats.length > 0 ? stats : [request.signalLine]
}

function getSpeedyUrgencyKind(request: SpeedyDeletionCard) {
  if (activeSpeedyState.value === 'calm') {
    return null
  }

  if (
    request.contested &&
    (
      !usesPrototypeV2UrgencyCalibration.value ||
      activeSpeedyState.value === 'contested'
    )
  ) {
    return 'contested'
  }

  if (!usesPrototypeV2UrgencyCalibration.value && activeSpeedyState.value === 'stale' && request.ageMinutes > 360) {
    return 'stale'
  }

  return null
}

function isSpeedyDeletionUrgent(request: SpeedyDeletionCard) {
  return getSpeedyUrgencyKind(request) !== null
}

function getSpeedyUrgencyChipClass(request: SpeedyDeletionCard) {
  const kind = getSpeedyUrgencyKind(request)
  return kind === 'contested' ? 'personal-dashboard-detail__urgency-chip--error' : ''
}

function getSpeedyUrgencyStatus(request: SpeedyDeletionCard) {
  return getSpeedyUrgencyKind(request) === 'contested' ? 'error' : 'warning'
}

function getSpeedyUrgencyIcon(request: SpeedyDeletionCard) {
  return getSpeedyUrgencyKind(request) === 'contested' ? cdxIconSpeechBubbles : cdxIconAlert
}

function getSpeedyUrgencyLabel(request: SpeedyDeletionCard) {
  if (getSpeedyUrgencyKind(request) === 'contested') {
    return 'Contested'
  }

  return `${Math.round(request.ageMinutes / 60)}h · over target`
}

function openProtectForm(request: ProtectionRequestCard) {
  return router.push({
    path: '/admin-moderation-dashboard/protect',
    query: { title: request.title },
    state: { dashboardReturnTo: route.fullPath },
  })
}

function openDeleteForm(request: SpeedyDeletionCard) {
  return router.push({
    path: '/admin-moderation-dashboard/delete',
    query: { title: request.title },
    state: { dashboardReturnTo: route.fullPath },
  })
}

function openHistoryPage(title: string, module: 'protection' | 'speedy') {
  return router.push({
    path: '/admin-moderation-dashboard/history',
    query: { title, module },
    state: { dashboardReturnTo: route.fullPath },
  })
}

function openRequestPage(title: string, context: 'protection' | 'speedy') {
  return router.push({
    path: '/admin-moderation-dashboard/request',
    query: { title, context },
    state: { dashboardReturnTo: route.fullPath },
  })
}

function openDeclinePage(title: string, context: 'protection' | 'speedy') {
  return router.push({
    path: '/admin-moderation-dashboard/decline',
    query: { title, context },
    state: { dashboardReturnTo: route.fullPath },
  })
}

function openArticlePage(title: string, module: 'protection' | 'speedy') {
  return router.push({
    path: '/admin-moderation-dashboard/article',
    query: { title, module },
    state: { dashboardReturnTo: route.fullPath },
  })
}

function getInitialVariant(): DashboardVariantId {
  if (typeof window === 'undefined') {
    return defaultVariantId
  }

  const variant = new URLSearchParams(window.location.search).get('variant')
  return isDashboardVariant(variant) ? variant : defaultVariantId
}

function getInitialModule() {
  if (typeof window === 'undefined') {
    return ''
  }

  return new URLSearchParams(window.location.search).get('module') ?? ''
}

function getInitialCardTreatment(): CardTreatmentId {
  if (typeof window === 'undefined') {
    return defaultCardTreatmentId
  }

  const treatment = new URLSearchParams(window.location.search).get('direction')
  return isCardTreatment(treatment) ? treatment : defaultCardTreatmentId
}

function getInitialProtectionState(): ProtectionCardState {
  if (typeof window === 'undefined') {
    return defaultProtectionState
  }

  const state = new URLSearchParams(window.location.search).get('protection')
  return isProtectionCardState(state) ? state : defaultProtectionState
}

function getInitialSpeedyState(): SpeedyCardState {
  if (typeof window === 'undefined') {
    return defaultSpeedyState
  }

  const state = new URLSearchParams(window.location.search).get('speedy')
  return isSpeedyCardState(state) ? state : defaultSpeedyState
}

function getInitialCriteria(): string[] {
  if (typeof window === 'undefined') {
    return []
  }

  return normalizeCriteriaSelection(new URLSearchParams(window.location.search).get('criterion'))
}

function shouldPreserveCriterionParam(variantId: DashboardVariantId, treatmentId: CardTreatmentId) {
  return variantId === 'prototype-v2' && isStatefulCardTreatment(treatmentId)
}

function syncCriterionSearchParam(url: URL, variantId: DashboardVariantId, treatmentId: CardTreatmentId) {
  if (shouldPreserveCriterionParam(variantId, treatmentId) && activeCriteria.value.length > 0) {
    url.searchParams.set('criterion', activeCriteria.value.join(','))
  } else {
    url.searchParams.delete('criterion')
  }
  url.searchParams.delete('contested')
}

function syncInitialPrototypeSettings() {
  if (typeof window === 'undefined') {
    return
  }

  const url = new URL(window.location.href)
  let changed = false

  if (!isDashboardVariant(url.searchParams.get('variant'))) {
    url.searchParams.set('variant', activeVariantId.value)
    changed = true
  }

  if (isCardDirectionVariantId(activeVariantId.value)) {
    if (!isCardTreatment(url.searchParams.get('direction'))) {
      url.searchParams.set('direction', activeCardTreatmentId.value)
      changed = true
    }

    if (isStatefulCardTreatment(activeCardTreatmentId.value)) {
      if (!isProtectionCardState(url.searchParams.get('protection'))) {
        url.searchParams.set('protection', activeProtectionState.value)
        changed = true
      }

      if (!isSpeedyCardState(url.searchParams.get('speedy'))) {
        url.searchParams.set('speedy', activeSpeedyState.value)
        changed = true
      }
    } else {
      if (url.searchParams.has('protection') || url.searchParams.has('speedy')) {
        url.searchParams.delete('protection')
        url.searchParams.delete('speedy')
        changed = true
      }
    }
  } else if (
    url.searchParams.has('direction') ||
    url.searchParams.has('protection') ||
    url.searchParams.has('speedy')
  ) {
    url.searchParams.delete('direction')
    url.searchParams.delete('protection')
    url.searchParams.delete('speedy')
    changed = true
  }

  const searchBeforeCriterionSync = url.search
  syncCriterionSearchParam(url, activeVariantId.value, activeCardTreatmentId.value)

  if (url.search !== searchBeforeCriterionSync) {
    changed = true
  }

  if (changed) {
    window.history.replaceState({}, '', url)
  }
}

onMounted(syncInitialPrototypeSettings)

function selectVariant(variantId: DashboardVariantId) {
  activeVariantId.value = variantId
  optionAProtectionViewOpen.value = false
  optionBProtectionViewOpen.value = false
  optionBSpeedyViewOpen.value = false

  if (typeof window === 'undefined') {
    return
  }

  const url = new URL(window.location.href)
  url.searchParams.set('variant', variantId)
  url.searchParams.delete('module')
  if (isCardDirectionVariantId(variantId)) {
    url.searchParams.set('direction', activeCardTreatmentId.value)
    if (isStatefulCardTreatment(activeCardTreatmentId.value)) {
      url.searchParams.set('protection', activeProtectionState.value)
      url.searchParams.set('speedy', activeSpeedyState.value)
    } else {
      url.searchParams.delete('protection')
      url.searchParams.delete('speedy')
    }
  } else {
    url.searchParams.delete('direction')
    url.searchParams.delete('protection')
    url.searchParams.delete('speedy')
  }
  syncCriterionSearchParam(url, variantId, activeCardTreatmentId.value)
  window.history.replaceState({}, '', url)
}

function selectCardTreatment(treatmentId: CardTreatmentId) {
  if (!isCardDirectionVariantId(activeVariantId.value)) {
    activeVariantId.value = 'current'
  }
  activeCardTreatmentId.value = treatmentId

  if (typeof window === 'undefined') {
    return
  }

  const url = new URL(window.location.href)
  url.searchParams.set('variant', activeVariantId.value)
  url.searchParams.set('direction', treatmentId)
  if (isStatefulCardTreatment(treatmentId)) {
    url.searchParams.set('protection', activeProtectionState.value)
    url.searchParams.set('speedy', activeSpeedyState.value)
  } else {
    url.searchParams.delete('protection')
    url.searchParams.delete('speedy')
  }
  url.searchParams.delete('module')
  syncCriterionSearchParam(url, activeVariantId.value, treatmentId)
  window.history.replaceState({}, '', url)
}

function selectProtectionState(state: ProtectionCardState) {
  if (!isCardDirectionVariantId(activeVariantId.value)) {
    activeVariantId.value = 'current'
  }
  if (!isStatefulCardTreatment(activeCardTreatmentId.value)) {
    activeCardTreatmentId.value = 'cards-attention'
  }
  activeProtectionState.value = state
  protectionSortValue.value = getDefaultProtectionSort(state)
  syncCardStateParam('protection', state)
}

function selectSpeedyState(state: SpeedyCardState) {
  if (!isCardDirectionVariantId(activeVariantId.value)) {
    activeVariantId.value = 'current'
  }
  if (!isStatefulCardTreatment(activeCardTreatmentId.value)) {
    activeCardTreatmentId.value = 'cards-attention'
  }
  activeSpeedyState.value = state
  speedySortValue.value = getDefaultSpeedySort(state)
  syncCardStateParam('speedy', state)
}

function syncCardStateParam(param: 'protection' | 'speedy', value: ProtectionCardState | SpeedyCardState) {
  if (typeof window === 'undefined') {
    return
  }

  const url = new URL(window.location.href)
  const variantId = isCardDirectionVariantId(activeVariantId.value) ? activeVariantId.value : 'current'
  const treatmentId = isStatefulCardTreatment(activeCardTreatmentId.value) ? activeCardTreatmentId.value : 'cards-attention'
  url.searchParams.set(
    'variant',
    variantId,
  )
  url.searchParams.set(
    'direction',
    treatmentId,
  )
  url.searchParams.set(param, value)
  url.searchParams.delete('module')
  syncCriterionSearchParam(url, variantId, treatmentId)
  window.history.replaceState({}, '', url)
}

function toggleCriterion(code: string) {
  const idx = activeCriteria.value.indexOf(code)
  if (idx === -1) {
    activeCriteria.value = [...activeCriteria.value, code]
  } else {
    activeCriteria.value = activeCriteria.value.filter((c) => c !== code)
  }
  syncCriterionParam()
}

function clearAllFilters() {
  activeCriteria.value = []
  syncCriterionParam()
}

function openCriterionSheet() {
  modalSearchQuery.value = ''
  criterionSheetOpen.value = true
}

function syncCriterionParam() {
  if (typeof window === 'undefined') {
    return
  }

  const url = new URL(window.location.href)
  url.searchParams.set('variant', 'prototype-v2')
  url.searchParams.set('direction', 'cards-attention')
  syncCriterionSearchParam(url, 'prototype-v2', 'cards-attention')
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
  protectionSortValue.value = isAttentionCardTreatment.value ? getDefaultProtectionSort(activeProtectionState.value) : 'most-active'
  optionBProtectionViewOpen.value = true
  syncModuleParam('protection')
}

function openOptionBSpeedy(event: MouseEvent) {
  if (!isOptionBVariant.value) {
    return
  }

  event.preventDefault()
  speedySortValue.value = isAttentionCardTreatment.value ? getDefaultSpeedySort(activeSpeedyState.value) : 'contested'
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
  syncCriterionSearchParam(url, activeVariantId.value, activeCardTreatmentId.value)
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
      return 'Prototype v1'
    case 'prototype-v2':
      return 'Prototype v2 (latest)'
  }
}
</script>

<template>
  <ChromeWrapper
    skin="mobile"
    lang="en"
    dir="ltr"
  >
    <template #header>
      <header class="mock-minerva-header">
        <CdxButton
          weight="quiet"
          aria-label="Main menu"
        >
          <CdxIcon :icon="cdxIconMenu" />
        </CdxButton>
        <a
          class="mock-minerva-header__brand"
          href="#"
        >mwdd-en</a>
        <div class="mock-minerva-header__actions">
          <CdxButton
            weight="quiet"
            aria-label="Search"
          >
            <CdxIcon :icon="cdxIconSearch" />
          </CdxButton>
          <button
            class="mock-minerva-header__notification-count"
            type="button"
            aria-label="Notifications"
          >
            1
          </button>
          <CdxButton
            weight="quiet"
            aria-label="User menu"
          >
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
        cardTreatmentClass,
        {
          'moderator-dashboard--current-treatment': usesCurrentDashboardTreatment,
          'moderator-dashboard--detail-page': isAnyDetailOpen,
        },
      ]"
    >
      <template v-if="usesCurrentDashboardTreatment">
        <template v-if="isOptionAProtectionDetail">
          <section
            class="personal-dashboard-detail"
            aria-labelledby="option-a-protection-heading"
          >
            <header class="personal-dashboard-detail__header">
              <CdxButton
                aria-label="Back to dashboard"
                weight="quiet"
                @click="closeOptionAProtectionRequests"
              >
                <CdxIcon :icon="cdxIconPrevious" />
              </CdxButton>
              <h1 id="option-a-protection-heading">
                Protection requests
              </h1>
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
                  <h2 class="personal-dashboard-detail__title-line">
                    Eurovision Song Contest Asia
                  </h2>
                  <p class="personal-dashboard-detail__importance">
                    <span class="personal-dashboard-detail__importance-item">
                      <CdxIcon
                        :icon="cdxIconEye"
                        size="small"
                      />
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
                  <h2 class="personal-dashboard-detail__title-line">
                    Talk:Blue Whale Challenge
                  </h2>
                  <p class="personal-dashboard-detail__importance">
                    <span class="personal-dashboard-detail__importance-item">
                      <CdxIcon
                        :icon="cdxIconEye"
                        size="small"
                      />
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
          <section
            class="personal-dashboard-detail"
            aria-labelledby="option-b-protection-detail-heading"
          >
            <header class="personal-dashboard-detail__header">
              <CdxButton
                aria-label="Back to dashboard"
                weight="quiet"
                @click="closeOptionBModule"
              >
                <CdxIcon :icon="cdxIconPrevious" />
              </CdxButton>
              <h1 id="option-b-protection-detail-heading">
                Pages for protection
              </h1>
            </header>

            <div class="personal-dashboard-detail__body">
              <div
                v-if="usesPrototypeV2ProtectionCardVisuals"
                class="personal-dashboard-detail__list-toolbar"
              >
                <div class="personal-dashboard-detail__chip-strip-wrapper">
                  <div
                    class="personal-dashboard-detail__chip-strip"
                    :class="{ 'personal-dashboard-detail__chip-strip--has-clear': activeProtectionCategories.length > 0 }"
                    role="group"
                    aria-label="Filter by protection request category"
                  >
                    <CdxButton
                      class="personal-dashboard-detail__chip"
                      :class="{ 'personal-dashboard-detail__chip--active': activeProtectionCategories.length === 0 }"
                      :aria-pressed="activeProtectionCategories.length === 0"
                      weight="normal"
                      @click="clearProtectionCategoryFilters"
                    >
                      <span class="personal-dashboard-detail__chip-label">All</span>
                      <span class="personal-dashboard-detail__chip-count">{{ optionBProtectionRequests.length }}</span>
                    </CdxButton>
                    <CdxButton
                      v-for="cat in pinnedProtectionCategories"
                      :key="cat"
                      class="personal-dashboard-detail__chip"
                      :class="{
                        'personal-dashboard-detail__chip--active': activeProtectionCategories.includes(cat),
                        'personal-dashboard-detail__chip--empty': getProtectionCategoryCount(cat) === 0,
                      }"
                      :aria-pressed="activeProtectionCategories.includes(cat)"
                      weight="normal"
                      @click="toggleProtectionCategory(cat)"
                    >
                      <span class="personal-dashboard-detail__chip-label">{{ cat }}</span>
                      <span
                        v-if="getProtectionCategoryCount(cat) > 0"
                        class="personal-dashboard-detail__chip-count"
                      >{{ getProtectionCategoryCount(cat) }}</span>
                    </CdxButton>
                  </div>

                  <div
                    v-if="activeProtectionCategories.length > 0"
                    class="personal-dashboard-detail__chip-clear-fade"
                    aria-hidden="true"
                  />
                  <CdxButton
                    v-if="activeProtectionCategories.length > 0"
                    class="personal-dashboard-detail__chip-clear"
                    weight="quiet"
                    aria-label="Clear all filters"
                    @click="clearProtectionCategoryFilters"
                  >
                    <CdxIcon
                      :icon="cdxIconClose"
                      size="x-small"
                    />
                  </CdxButton>
                </div>

                <label class="personal-dashboard-detail__sort-menu-row">
                  <span class="personal-dashboard-detail__sort-menu-label">Sort by</span>
                  <CdxSelect
                    class="personal-dashboard-detail__sort-menu"
                    :menu-items="protectionSortItems"
                    :selected="protectionSortValue"
                    aria-label="Sort by"
                    @update:selected="protectionSortValue = $event"
                  />
                </label>
              </div>

              <div
                v-else
                class="personal-dashboard-detail__toolbar"
              >
                <p
                  v-if="isAttentionCardTreatment && activeProtectionState === 'stale'"
                  class="personal-dashboard-detail__lede-line"
                >
                  <strong
                    :class="{
                      'personal-dashboard-detail__lede-alert': !usesPrototypeV2UrgencyCalibration,
                      'personal-dashboard-detail__lede-neutral': usesPrototypeV2UrgencyCalibration,
                    }"
                  >
                    {{ overTargetProtectionCount }} over 12h target
                  </strong>
                  · <strong>17</strong> total open · <strong>6</strong> no admin reply
                </p>
                <p
                  v-else
                  class="personal-dashboard-detail__lede-line"
                >
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
                <p
                  v-if="showProtectionOverTargetHeader"
                  class="personal-dashboard-detail__group-heading"
                  :class="{ 'personal-dashboard-detail__group-heading--neutral': usesPrototypeV2UrgencyCalibration }"
                >
                  Over 12h target ({{ overTargetProtectionCount }})
                </p>

                <article
                  v-for="request in sortedOptionBProtectionRequests"
                  :key="request.id"
                  class="personal-dashboard-detail__request"
                  :class="{ 'personal-dashboard-detail__request--v2': isAttentionCardTreatment }"
                >
                  <template v-if="isAttentionCardTreatment">
                    <CdxInfoChip
                      v-if="isProtectionRequestUrgent(request)"
                      class="personal-dashboard-detail__urgency-chip"
                      status="warning"
                      :icon="cdxIconAlert"
                    >
                      {{ request.ageHours }}h · over target
                    </CdxInfoChip>
                    <div
                      v-if="usesPrototypeV2ProtectionCardVisuals"
                      class="personal-dashboard-detail__category-row"
                    >
                      <span
                        class="personal-dashboard-detail__category-eyebrow"
                      >
                        {{ request.signalLabel }} · {{ getProtectionSignalRecency(request) }}
                      </span>
                      <span
                        class="personal-dashboard-detail__actor-pill"
                        :aria-label="`${request.actorCount} ${request.actorCount === 1 ? 'editor' : 'editors'} involved`"
                      >
                        <CdxIcon
                          class="personal-dashboard-detail__actor-icon"
                          :icon="getProtectionActorIcon(request)"
                          size="small"
                          aria-hidden="true"
                        />
                        {{ request.actorCount }}
                      </span>
                    </div>
                    <span
                      v-else
                      class="personal-dashboard-detail__category-eyebrow"
                    >{{ request.signalLabel }}</span>
                    <h2 class="personal-dashboard-detail__title-line">
                      {{ request.title }}
                    </h2>
                    <p class="personal-dashboard-detail__importance">
                      <span class="personal-dashboard-detail__importance-item">
                        <CdxIcon
                          :icon="cdxIconEye"
                          size="small"
                        />
                        {{ request.visibility }}
                      </span>
                    </p>
                    <p class="personal-dashboard-detail__requester-quote">
                      “{{ request.requesterQuote }}”
                    </p>
                    <p
                      v-if="usesPrototypeV2ProtectionCardVisuals"
                      class="personal-dashboard-detail__protection-stats"
                    >
                      <template
                        v-for="(stat, index) in getProtectionSignalStats(request)"
                        :key="stat"
                      >
                        <span>{{ stat }}</span>
                        <span
                          v-if="index < getProtectionSignalStats(request).length - 1"
                          class="personal-dashboard-detail__protection-stats-separator"
                          aria-hidden="true"
                        >·</span>
                      </template>
                    </p>
                    <p
                      v-else
                      class="personal-dashboard-detail__system-signal"
                      :class="{ 'personal-dashboard-detail__system-signal--urgent': isProtectionRequestUrgent(request) }"
                    >
                      {{ request.signalLine }}
                    </p>
                    <footer
                      class="personal-dashboard-detail__action-footer"
                      :class="{ 'personal-dashboard-detail__action-footer--evidence': usesPrototypeV2EvidenceFooter }"
                    >
                      <template v-if="usesPrototypeV2EvidenceFooter">
                        <div class="personal-dashboard-detail__primary-action-row">
                          <CdxButton
                            class="personal-dashboard-detail__evidence-button"
                            action="progressive"
                            weight="primary"
                            size="small"
                            @click.stop="openHistoryPage(request.title, 'protection')"
                          >
                            View history
                          </CdxButton>
                        </div>
                        <nav
                          class="personal-dashboard-detail__action-links"
                          aria-label="Protection request destinations"
                        >
                          <a
                            href="#"
                            @click.prevent="openRequestPage(request.title, 'protection')"
                          >View request</a>
                        </nav>
                      </template>
                      <template v-else>
                        <CdxButton
                          class="personal-dashboard-detail__protect-button"
                          action="progressive"
                          weight="primary"
                          size="small"
                          @click.stop="openProtectForm(request)"
                        >
                          Protect
                        </CdxButton>
                        <nav
                          class="personal-dashboard-detail__action-links"
                          aria-label="Protection request destinations"
                        >
                          <a
                            href="#"
                            @click.prevent="openHistoryPage(request.title, 'protection')"
                          >History</a>
                          <a
                            href="#"
                            @click.prevent="openRequestPage(request.title, 'protection')"
                          >View request</a>
                          <a
                            class="personal-dashboard-detail__action-link--workflow"
                            href="#"
                            @click.prevent="openDeclinePage(request.title, 'protection')"
                          >
                            Decline
                          </a>
                        </nav>
                      </template>
                    </footer>
                  </template>

                  <template v-else>
                    <h2 class="personal-dashboard-detail__title-line">
                      {{ request.title }}
                    </h2>
                    <p class="personal-dashboard-detail__importance">
                      <span class="personal-dashboard-detail__importance-item">
                        <CdxIcon
                          :icon="cdxIconEye"
                          size="small"
                        />
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
                  </template>
                </article>
              </div>
            </div>
          </section>
        </template>

        <template v-else-if="isOptionBSpeedyDetail">
          <section
            class="personal-dashboard-detail"
            aria-labelledby="option-b-speedy-detail-heading"
          >
            <header class="personal-dashboard-detail__header">
              <CdxButton
                aria-label="Back to dashboard"
                weight="quiet"
                @click="closeOptionBModule"
              >
                <CdxIcon :icon="cdxIconPrevious" />
              </CdxButton>
              <h1 id="option-b-speedy-detail-heading">
                Pages for speedy deletion
              </h1>
            </header>

            <div class="personal-dashboard-detail__body">
              <div
                v-if="showPrototypeV2SpeedyCriterionFilter"
                class="personal-dashboard-detail__list-toolbar"
              >
                <div class="personal-dashboard-detail__chip-strip-wrapper">
                  <div
                    class="personal-dashboard-detail__chip-strip"
                    :class="{ 'personal-dashboard-detail__chip-strip--has-clear': hasAnyActiveFilter }"
                    role="group"
                    aria-label="Filter by speedy deletion criterion"
                  >
                    <CdxButton
                      class="personal-dashboard-detail__chip"
                      :class="{ 'personal-dashboard-detail__chip--active': !hasAnyActiveFilter }"
                      :aria-pressed="!hasAnyActiveFilter"
                      weight="normal"
                      @click="clearAllFilters"
                    >
                      <span class="personal-dashboard-detail__chip-label">All</span>
                      <span class="personal-dashboard-detail__chip-count">{{ allCriterionCount }}</span>
                    </CdxButton>
                    <CdxButton
                      v-for="code in visibleChipCodes"
                      :key="code"
                      class="personal-dashboard-detail__chip"
                      :class="{
                        'personal-dashboard-detail__chip--active': activeCriteria.includes(code),
                        'personal-dashboard-detail__chip--empty': getCriterionCount(code) === 0,
                      }"
                      :aria-pressed="activeCriteria.includes(code)"
                      weight="normal"
                      @click="toggleCriterion(code)"
                    >
                      <span class="personal-dashboard-detail__chip-label">{{ code }}</span>
                      <span
                        v-if="getCriterionCount(code) > 0"
                        class="personal-dashboard-detail__chip-count"
                      >{{ getCriterionCount(code) }}</span>
                    </CdxButton>
                    <CdxButton
                      class="personal-dashboard-detail__chip personal-dashboard-detail__chip--more"
                      weight="normal"
                      @click="openCriterionSheet"
                    >
                      <CdxIcon
                        :icon="cdxIconFunnel"
                        size="x-small"
                      />
                      <span class="personal-dashboard-detail__chip-label">Filters</span>
                    </CdxButton>
                  </div>

                  <div
                    v-if="hasAnyActiveFilter"
                    class="personal-dashboard-detail__chip-clear-fade"
                    aria-hidden="true"
                  />
                  <CdxButton
                    v-if="hasAnyActiveFilter"
                    class="personal-dashboard-detail__chip-clear"
                    weight="quiet"
                    aria-label="Clear all filters"
                    @click="clearAllFilters"
                  >
                    <CdxIcon
                      :icon="cdxIconClose"
                      size="x-small"
                    />
                  </CdxButton>
                </div>

                <label class="personal-dashboard-detail__sort-menu-row">
                  <span class="personal-dashboard-detail__sort-menu-label">Sort by</span>
                  <CdxSelect
                    class="personal-dashboard-detail__sort-menu"
                    :menu-items="speedySortItems"
                    :selected="speedySortValue"
                    aria-label="Sort by"
                    @update:selected="speedySortValue = $event"
                  />
                </label>
              </div>

              <div class="personal-dashboard-detail__requests">
                <div
                  v-if="showSpeedyCriterionEmptyState"
                  class="personal-dashboard-detail__empty-state"
                >
                  <p>No pages match your current filters.</p>
                  <CdxButton
                    size="small"
                    @click="clearAllFilters"
                  >
                    Clear all filters
                  </CdxButton>
                </div>

                <template v-else>
                  <p
                    v-if="showPrototypeV2SpeedyCriterionFilter"
                    class="personal-dashboard-detail__results-count"
                  >
                    Showing {{ sortedOptionBSpeedyDeletionRequests.length }}
                    {{ sortedOptionBSpeedyDeletionRequests.length === 1 ? 'page' : 'pages' }}
                  </p>
                  <article
                    v-for="request in sortedOptionBSpeedyDeletionRequests"
                    :key="request.id"
                    class="personal-dashboard-detail__request"
                    :class="{ 'personal-dashboard-detail__request--v2': isAttentionCardTreatment }"
                  >
                    <template v-if="isAttentionCardTreatment">
                      <CdxInfoChip
                        v-if="getSpeedyUrgencyKind(request)"
                        class="personal-dashboard-detail__urgency-chip"
                        :class="getSpeedyUrgencyChipClass(request)"
                        :status="getSpeedyUrgencyStatus(request)"
                        :icon="getSpeedyUrgencyIcon(request)"
                      >
                        {{ getSpeedyUrgencyLabel(request) }}
                      </CdxInfoChip>
                      <span class="personal-dashboard-detail__criterion-eyebrow">{{ request.criterion }}</span>
                      <h2 class="personal-dashboard-detail__title-line">
                        {{ request.title }}
                      </h2>
                      <p class="personal-dashboard-detail__importance">
                        <span class="personal-dashboard-detail__importance-item">
                          <CdxIcon
                            :icon="cdxIconEye"
                            size="small"
                          />
                          {{ request.visibility }}
                        </span>
                      </p>
                      <p class="personal-dashboard-detail__tagger-reason">
                        “{{ request.taggerReason }}”
                      </p>
                      <p
                        class="personal-dashboard-detail__system-signal"
                        :class="{ 'personal-dashboard-detail__system-signal--error': isSpeedyDeletionUrgent(request) }"
                      >
                        <strong>{{ request.signalLabel }}</strong>
                        · {{ request.signalLine }}
                      </p>
                      <footer
                        class="personal-dashboard-detail__action-footer"
                        :class="{ 'personal-dashboard-detail__action-footer--evidence': usesPrototypeV2EvidenceFooter }"
                      >
                        <template v-if="usesPrototypeV2EvidenceFooter">
                          <div class="personal-dashboard-detail__primary-action-row">
                            <CdxButton
                              class="personal-dashboard-detail__evidence-button"
                              action="progressive"
                              weight="primary"
                              size="small"
                              @click.stop="openArticlePage(request.title, 'speedy')"
                            >
                              View page
                            </CdxButton>
                          </div>
                          <nav
                            class="personal-dashboard-detail__action-links"
                            aria-label="Speedy deletion destinations and actions"
                          >
                            <a
                              href="#"
                              @click.prevent="openHistoryPage(request.title, 'speedy')"
                            >History</a>
                            <a
                              href="#"
                              @click.prevent="openRequestPage(request.title, 'speedy')"
                            >Talk</a>
                          </nav>
                        </template>
                        <template v-else>
                          <CdxButton
                            class="personal-dashboard-detail__delete-button"
                            action="destructive"
                            weight="primary"
                            size="small"
                            @click.stop="openDeleteForm(request)"
                          >
                            Delete
                          </CdxButton>
                          <nav
                            class="personal-dashboard-detail__action-links"
                            aria-label="Speedy deletion destinations and actions"
                          >
                            <a
                              href="#"
                              @click.prevent="openHistoryPage(request.title, 'speedy')"
                            >History</a>
                            <a
                              href="#"
                              @click.prevent="openRequestPage(request.title, 'speedy')"
                            >Talk</a>
                            <a
                              class="personal-dashboard-detail__action-link--workflow"
                              href="#"
                              @click.prevent="openDeclinePage(request.title, 'speedy')"
                            >
                              Decline
                            </a>
                          </nav>
                        </template>
                      </footer>
                    </template>

                    <template v-else>
                      <span class="personal-dashboard-detail__criterion">{{ request.criterion }}</span>
                      <h2 class="personal-dashboard-detail__title-line">
                        {{ request.title }}
                      </h2>
                      <p class="personal-dashboard-detail__importance">
                        <span class="personal-dashboard-detail__importance-item">
                          <CdxIcon
                            :icon="cdxIconEye"
                            size="small"
                          />
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
                    </template>
                  </article>
                </template>
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
                <CdxIcon
                  :icon="cdxIconLabFlask"
                  size="small"
                />
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
              <ul
                v-if="!isAttentionCardTreatment"
                class="personal-dashboard-baseline__protection-metrics"
                aria-label="Pages for protection details"
              >
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
              <ul
                v-if="isAttentionCardTreatment && !activeProtectionAttentionSummary.needsAttention"
                class="personal-dashboard-baseline__protection-metrics"
                aria-label="Pages for protection calm details"
              >
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
                <li aria-label="Longest unresolved protection request has waited 4 hours">
                  <CdxIcon :icon="cdxIconClock" />
                  <strong>4h</strong>
                  <span>longest wait</span>
                </li>
              </ul>
              <div
                v-if="isAttentionCardTreatment && activeProtectionAttentionSummary.needsAttention"
                class="personal-dashboard-card__attention"
                aria-label="Pages for protection attention summary"
              >
                <p
                  class="personal-dashboard-card__attention-lead"
                  :class="{
                    'personal-dashboard-card__attention-lead--warning': isProtectionHomeAlarm,
                    'personal-dashboard-card__attention-lead--neutral': isProtectionHomeNeutralTimeSignal,
                  }"
                >
                  <CdxIcon
                    v-if="isProtectionHomeAlarm"
                    :icon="cdxIconAlert"
                    size="small"
                  />
                  <span>
                    <strong>{{ activeProtectionAttentionSummary.leadValue }}</strong>
                    {{ activeProtectionAttentionSummary.leadLabel }}
                  </span>
                </p>
                <p class="personal-dashboard-card__attention-secondary">
                  {{ activeProtectionAttentionSummary.secondary }}
                </p>
                <span
                  v-if="activeProtectionAttentionSummary.actionLabel"
                  class="personal-dashboard-card__attention-action"
                >
                  {{ activeProtectionAttentionSummary.actionLabel }}
                </span>
              </div>
              <p
                v-if="isEvidenceCardTreatment"
                class="personal-dashboard-card__evidence"
              >
                <strong>Why it matters</strong>
                <span>6 requests have no admin reply; the oldest has waited 22h.</span>
              </p>
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
              <ul
                v-if="!isAttentionCardTreatment"
                class="personal-dashboard-baseline__protection-metrics"
                aria-label="Pages for speedy deletion details"
              >
                <li aria-label="4 speedy deletion pages are in the queue">
                  <CdxIcon :icon="cdxIconTrash" />
                  <strong>4</strong>
                  <span>in queue</span>
                </li>
                <li aria-label="2 speedy deletion pages are contested">
                  <CdxIcon :icon="cdxIconSpeechBubbles" />
                  <strong>2</strong>
                  <span>contested</span>
                </li>
                <li aria-label="Longest speedy deletion page has waited 3 hours">
                  <CdxIcon :icon="cdxIconClock" />
                  <strong>3h</strong>
                  <span>longest wait</span>
                </li>
              </ul>
              <ul
                v-if="isAttentionCardTreatment && !activeSpeedyAttentionSummary.needsAttention"
                class="personal-dashboard-baseline__protection-metrics"
                aria-label="Pages for speedy deletion calm details"
              >
                <li aria-label="4 speedy deletion pages are in the queue">
                  <CdxIcon :icon="cdxIconTrash" />
                  <strong>4</strong>
                  <span>in queue</span>
                </li>
                <li aria-label="0 speedy deletion pages are contested">
                  <CdxIcon :icon="cdxIconSpeechBubbles" />
                  <strong>0</strong>
                  <span>contested</span>
                </li>
                <li aria-label="Longest speedy deletion page has waited 3 hours">
                  <CdxIcon :icon="cdxIconClock" />
                  <strong>3h</strong>
                  <span>longest wait</span>
                </li>
              </ul>
              <div
                v-if="isAttentionCardTreatment && activeSpeedyAttentionSummary.needsAttention"
                class="personal-dashboard-card__attention"
                aria-label="Pages for speedy deletion attention summary"
              >
                <p
                  class="personal-dashboard-card__attention-lead"
                  :class="{
                    'personal-dashboard-card__attention-lead--warning': isSpeedyHomeAlarm,
                    'personal-dashboard-card__attention-lead--neutral': isSpeedyHomeNeutralTimeSignal,
                  }"
                >
                  <CdxIcon
                    v-if="isSpeedyHomeAlarm"
                    :icon="cdxIconAlert"
                    size="small"
                  />
                  <span>
                    <strong>{{ activeSpeedyAttentionSummary.leadValue }}</strong>
                    {{ activeSpeedyAttentionSummary.leadLabel }}
                  </span>
                </p>
                <p class="personal-dashboard-card__attention-secondary">
                  {{ activeSpeedyAttentionSummary.secondary }}
                </p>
                <span
                  v-if="activeSpeedyAttentionSummary.actionLabel"
                  class="personal-dashboard-card__attention-action"
                >
                  {{ activeSpeedyAttentionSummary.actionLabel }}
                </span>
              </div>
              <p
                v-if="isEvidenceCardTreatment"
                class="personal-dashboard-card__evidence"
              >
                <strong>Why it matters</strong>
                <span>2 pages need an admin decision before the queue can clear.</span>
              </p>
            </a>
          </template>

          <section
            v-if="!isOptionBVariant"
            class="personal-dashboard-card"
            aria-labelledby="baseline-review-heading"
          >
            <a
              class="personal-dashboard-card__header"
              href="#"
            >
              <h2 id="baseline-review-heading">Review changes</h2>
              <CdxIcon :icon="cdxIconNext" />
            </a>
            <div class="personal-dashboard-baseline__review-summary">
              <CdxIcon :icon="cdxIconEdit" />
              <p>NewContributor edited the Page protection demo May 5 article</p>
            </div>
            <CdxButton
              action="progressive"
              weight="primary"
              class="personal-dashboard-baseline__full-button"
            >
              View more edits
            </CdxButton>
          </section>

          <section
            v-if="!isOptionBVariant && !isOptionAVariant"
            class="personal-dashboard-card"
            aria-labelledby="baseline-impact-heading"
          >
            <h2 id="baseline-impact-heading">
              Your impact
            </h2>
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
                <CdxIcon
                  :icon="cdxIconInfo"
                  size="small"
                />
              </div>
            </div>
          </section>

          <section
            v-if="!isOptionBVariant"
            class="personal-dashboard-card"
            aria-labelledby="baseline-protection-heading"
          >
            <a
              class="personal-dashboard-card__header"
              href="#"
              @click="openOptionAProtectionRequests"
            >
              <h2 id="baseline-protection-heading">Protection requests</h2>
              <CdxIcon :icon="cdxIconNext" />
            </a>

            <template v-if="isOptionAVariant">
              <ul
                class="personal-dashboard-baseline__protection-metrics"
                aria-label="Protection request summary"
              >
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
              <CdxButton
                action="progressive"
                weight="primary"
                class="personal-dashboard-baseline__full-button"
              >
                View more
              </CdxButton>
            </template>
          </section>

          <section
            v-if="isOptionAVariant"
            class="personal-dashboard-card"
            aria-labelledby="option-a-speedy-heading"
          >
            <a
              class="personal-dashboard-card__header"
              href="#"
            >
              <h2 id="option-a-speedy-heading">Speedy deletion</h2>
              <CdxIcon :icon="cdxIconNext" />
            </a>

            <ul
              class="personal-dashboard-baseline__protection-metrics"
              aria-label="Speedy deletion summary"
            >
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

            <CdxButton
              action="progressive"
              weight="primary"
              class="personal-dashboard-baseline__full-button"
            >
              View pages
            </CdxButton>
          </section>

          <section
            class="personal-dashboard-card"
            aria-labelledby="baseline-policies-heading"
          >
            <a
              class="personal-dashboard-card__header"
              href="#"
            >
              <h2 id="baseline-policies-heading">Policies and guidelines</h2>
              <CdxIcon :icon="cdxIconNext" />
            </a>
            <p>Review best practices to create a free and reliable encyclopedia.</p>
          </section>

          <section
            v-if="isOptionAVariant"
            class="personal-dashboard-card"
            aria-labelledby="option-a-impact-heading"
          >
            <h2 id="option-a-impact-heading">
              Your impact
            </h2>
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
                <CdxIcon
                  :icon="cdxIconInfo"
                  size="small"
                />
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

        <section
          class="moderator-dashboard__summary"
          aria-label="Moderation overview"
        >
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

        <section
          class="moderator-dashboard__section"
          aria-labelledby="judgment-heading"
        >
          <div class="moderator-dashboard__section-heading">
            <div>
              <h2 id="judgment-heading">
                {{ activeVariant.heading }}
              </h2>
              <p>{{ activeVariant.description }}</p>
            </div>
            <CdxButton
              aria-label="Filter recommendations"
              weight="quiet"
            >
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

        <section
          class="moderator-dashboard__detail"
          aria-labelledby="task-detail-heading"
        >
          <div class="moderator-dashboard__detail-header">
            <span class="moderator-dashboard__task-icon">
              <CdxIcon :icon="taskIcon(selectedTask)" />
            </span>
            <div>
              <p>{{ selectedTask.source }}</p>
              <h2 id="task-detail-heading">
                {{ selectedTask.title }}
              </h2>
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
              <li
                v-for="signal in selectedTask.signals"
                :key="signal"
              >
                <CdxIcon
                  :icon="cdxIconCheck"
                  size="small"
                />
                <span>{{ signal }}</span>
              </li>
            </ul>
          </div>

          <div class="moderator-dashboard__actions">
            <CdxButton
              action="progressive"
              weight="primary"
            >
              {{ selectedTask.nextSurface }}
              <CdxIcon :icon="cdxIconNext" />
            </CdxButton>
            <CdxButton>
              <CdxIcon :icon="cdxIconHistory" />
              Context
            </CdxButton>
          </div>
        </section>

        <section
          class="moderator-dashboard__section"
          aria-labelledby="backlogs-heading"
        >
          <h2 id="backlogs-heading">
            Backlogs represented
          </h2>
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

      <div class="personal-dashboard-detail__criterion-dialog-host">
        <CdxDialog
          v-model:open="criterionSheetOpen"
          class="personal-dashboard-detail__criterion-dialog"
          title="Filters"
          use-close-button
          :fixed-height="false"
          render-in-place
        >
          <div class="personal-dashboard-detail__filter-search">
            <CdxSearchInput
              v-model="modalSearchQuery"
              aria-label="Search criteria"
              placeholder="Search criteria (e.g. G11, spam)"
            />
          </div>

          <section
            v-for="group in filteredCriteriaByNamespace"
            :key="group.namespace"
            class="personal-dashboard-detail__filter-section"
          >
            <h3 class="personal-dashboard-detail__filter-section-heading">
              {{ group.label }}
            </h3>
            <ul
              class="personal-dashboard-detail__filter-list"
              role="list"
            >
              <li
                v-for="criterion in group.criteria"
                :key="criterion.code"
                class="personal-dashboard-detail__filter-row"
              >
                <CdxCheckbox
                  :model-value="activeCriteria.includes(criterion.code)"
                  class="personal-dashboard-detail__filter-checkbox"
                  @update:model-value="toggleCriterion(criterion.code)"
                >
                  <span class="personal-dashboard-detail__filter-checkbox-label">
                    <strong>{{ criterion.code }}</strong>
                    · {{ criterion.name }}
                  </span>
                </CdxCheckbox>
                <span
                  v-if="getCriterionCount(criterion.code) > 0"
                  class="personal-dashboard-detail__filter-row-count"
                >{{ getCriterionCount(criterion.code) }}</span>
              </li>
            </ul>
          </section>

          <p
            v-if="modalSearchQuery && filteredCriteriaByNamespace.length === 0"
            class="personal-dashboard-detail__filter-empty"
          >
            No criteria matching "{{ modalSearchQuery }}".
          </p>

          <template #footer>
            <div class="personal-dashboard-detail__filter-footer">
              <CdxButton
                weight="quiet"
                :disabled="!hasAnyActiveFilter"
                @click="clearAllFilters"
              >
                Clear all
              </CdxButton>
              <CdxButton
                action="progressive"
                weight="primary"
                @click="criterionSheetOpen = false"
              >
                Show {{ sortedOptionBSpeedyDeletionRequests.length }}
                {{ sortedOptionBSpeedyDeletionRequests.length === 1 ? 'result' : 'results' }}
              </CdxButton>
            </div>
          </template>
        </CdxDialog>
      </div>

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
            <strong>Prototype settings</strong>
            <CdxButton
              aria-label="Close prototype variants"
              weight="quiet"
              @click="variantSwitcherOpen = false"
            >
              <CdxIcon :icon="cdxIconClose" />
            </CdxButton>
          </div>

          <section
            class="moderator-dashboard__variant-section"
            aria-labelledby="moderator-dashboard-variants-heading"
          >
            <h2
              id="moderator-dashboard-variants-heading"
              class="moderator-dashboard__variant-section-label"
            >
              Version
            </h2>
            <button
              v-for="variant in displayedVariants"
              :key="variant.id"
              class="moderator-dashboard__variant-option"
              :class="{ 'moderator-dashboard__variant-option--active': variant.id === activeVariant.id }"
              type="button"
              @click="selectVariant(variant.id)"
            >
              <span>{{ variantOptionLabel(variant.id) }}</span>
            </button>
          </section>

          <fieldset
            v-if="isCardDirectionVariant"
            class="moderator-dashboard__variant-section moderator-dashboard__variant-fieldset"
          >
            <legend class="moderator-dashboard__variant-section-label">
              Card style
            </legend>
            <div
              class="moderator-dashboard__variant-radio-group"
              role="radiogroup"
              aria-label="Card style"
            >
              <CdxRadio
                v-for="treatment in visibleCardTreatments"
                :key="treatment.id"
                v-model="selectedCardTreatmentId"
                name="moderator-dashboard-card-style"
                :input-value="treatment.id"
              >
                {{ treatment.label }}
              </CdxRadio>
            </div>
          </fieldset>

          <fieldset
            v-if="isAttentionCardTreatment"
            class="moderator-dashboard__variant-section moderator-dashboard__variant-fieldset"
          >
            <legend class="moderator-dashboard__variant-section-label">
              Attention scenario
            </legend>

            <div class="moderator-dashboard__variant-subgroup">
              <span class="moderator-dashboard__variant-subgroup-label">Protection queue</span>
              <div
                class="moderator-dashboard__variant-radio-group"
                role="radiogroup"
                aria-label="Protection queue state"
              >
                <CdxRadio
                  v-for="state in protectionStateOptions"
                  :key="state.id"
                  v-model="selectedProtectionState"
                  name="moderator-dashboard-protection-state"
                  :input-value="state.id"
                >
                  {{ state.label }}
                </CdxRadio>
              </div>
            </div>

            <div class="moderator-dashboard__variant-subgroup">
              <span class="moderator-dashboard__variant-subgroup-label">Speedy deletion queue</span>
              <div
                class="moderator-dashboard__variant-radio-group"
                role="radiogroup"
                aria-label="Speedy deletion queue state"
              >
                <CdxRadio
                  v-for="state in speedyStateOptions"
                  :key="state.id"
                  v-model="selectedSpeedyState"
                  name="moderator-dashboard-speedy-state"
                  :input-value="state.id"
                >
                  {{ state.label }}
                </CdxRadio>
              </div>
            </div>
          </fieldset>

          <div
            v-if="!isCardDirectionVariant"
            class="moderator-dashboard__variant-help"
          >
            Card style controls are available on Prototype v1 and Prototype v2.
          </div>
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

.personal-dashboard-card__evidence {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
  margin: var(--spacing-75, 12px) 0 0;
  padding: var(--spacing-75, 12px) 0 0;
  border-top: 1px solid var(--border-color-subtle);
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-card__evidence strong {
  color: var(--color-base);
  font-size: var(--font-size-small);
  font-weight: 600;
}

.moderator-dashboard--cards-attention .personal-dashboard-card--linked {
  padding-block: var(--spacing-100, 16px);
}

.personal-dashboard-card__attention {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
  margin-top: var(--spacing-75, 12px);
  padding-top: var(--spacing-75, 12px);
  border-top: 1px solid var(--border-color-subtle);
}

.personal-dashboard-card__attention .personal-dashboard-card__attention-lead {
  display: flex;
  align-items: center;
  gap: var(--spacing-50, 8px);
  margin: 0;
  color: var(--color-base);
  font-size: 0.9375rem;
  line-height: var(--line-height-medium);
}

.personal-dashboard-card__attention .personal-dashboard-card__attention-lead--warning {
  color: var(--color-error);
}

.personal-dashboard-card__attention .personal-dashboard-card__attention-lead--neutral {
  color: var(--color-subtle);
}

.personal-dashboard-card__attention-lead :deep(.cdx-icon) {
  flex: none;
  width: 1rem;
  min-width: 1rem;
  height: 1rem;
  color: var(--color-error);
}

.personal-dashboard-card__attention-lead strong {
  font-weight: var(--font-weight-bold, 700);
  font-variant-numeric: tabular-nums;
}

.personal-dashboard-card__attention .personal-dashboard-card__attention-secondary {
  margin: 0;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-small);
  font-variant-numeric: tabular-nums;
}

.personal-dashboard-card__attention-action {
  margin-top: var(--spacing-25, 4px);
  color: var(--color-progressive);
  font-size: var(--font-size-small);
  font-weight: 600;
  line-height: var(--line-height-small);
}

.moderator-dashboard--cards-compact {
  gap: var(--spacing-25, 4px);
}

.moderator-dashboard--cards-compact .personal-dashboard-card--linked {
  padding: var(--spacing-75, 12px) var(--spacing-100, 16px);
}

.moderator-dashboard--cards-compact .personal-dashboard-baseline__protection-metrics {
  grid-template-columns: 1fr;
  gap: var(--spacing-25, 4px);
  margin-top: var(--spacing-50, 8px);
  padding-top: var(--spacing-50, 8px);
}

.moderator-dashboard--cards-compact .personal-dashboard-baseline__protection-metrics li {
  grid-template-columns: auto 2.5rem minmax(0, 1fr);
}

.moderator-dashboard--cards-evidence .personal-dashboard-card--linked {
  border-color: var(--border-color-base);
}

.moderator-dashboard--cards-evidence .personal-dashboard-baseline__protection-metrics {
  margin-top: var(--spacing-100, 16px);
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
  gap: var(--spacing-75, 12px);
  padding: var(--spacing-100, 16px);
}

.personal-dashboard-detail__list-toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: var(--spacing-50, 8px);
  margin: 0 calc(var(--spacing-100, 16px) * -1);
  padding: 0 var(--spacing-100, 16px) var(--spacing-75, 12px);
  border-bottom: 1px solid var(--border-color-subtle);
}

.personal-dashboard-detail__chip-strip-wrapper {
  position: relative;
  flex: 1 1 100%;
  min-width: 0;
}

@media (min-width: 40rem) {
  .personal-dashboard-detail__chip-strip-wrapper {
    flex: 1 1 0;
  }
}

.personal-dashboard-detail__chip-strip {
  display: flex;
  align-items: center;
  gap: var(--spacing-25, 4px);
  overflow-x: auto;
  scrollbar-width: none;
}

.personal-dashboard-detail__chip-strip::-webkit-scrollbar {
  display: none;
}

.personal-dashboard-detail__chip-strip--has-clear {
  padding-right: 3rem;
}

.personal-dashboard-detail__chip-clear-fade {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  width: 3.5rem;
  background: linear-gradient(
    to left,
    var(--background-color-base, #fff) 30%,
    rgba(255, 255, 255, 0) 100%
  );
  pointer-events: none;
}

.cdx-button.personal-dashboard-detail__chip {
  flex: 0 0 auto;
  gap: var(--spacing-25, 4px);
  font-weight: var(--font-weight-normal, 400);
}

.cdx-button.personal-dashboard-detail__chip--active:enabled {
  background-color: var(--background-color-progressive-subtle, #eaf3ff);
  border-color: var(--border-color-progressive, #36c);
  color: var(--color-progressive, #36c);
  font-weight: 500;
}

.cdx-button.personal-dashboard-detail__chip--active:enabled:hover {
  background-color: var(--background-color-progressive-subtle--hover, #d9e2ff);
  border-color: var(--border-color-progressive--hover, #36c);
  color: var(--color-progressive--hover, #36c);
}

.cdx-button.personal-dashboard-detail__chip--empty {
  color: var(--color-subtle);
}

.cdx-button.personal-dashboard-detail__chip--more:enabled {
  color: var(--color-progressive, #36c);
}

.cdx-button.personal-dashboard-detail__chip-clear {
  position: absolute;
  top: 50%;
  right: 0;
  width: 1.75rem;
  min-width: 1.75rem;
  height: 1.75rem;
  padding: 0;
  transform: translateY(-50%);
}

.personal-dashboard-detail__chip-label {
  font-variant-numeric: tabular-nums;
}

.personal-dashboard-detail__chip-count {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 1.125rem;
  height: 1.125rem;
  padding: 0 var(--spacing-25, 4px);
  border-radius: var(--border-radius-pill, 9999px);
  background-color: var(--background-color-neutral-subtle, #eaecf0);
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-x-small);
  font-weight: 500;
  font-variant-numeric: tabular-nums;
  line-height: 1;
}

.personal-dashboard-detail__chip--active .personal-dashboard-detail__chip-count {
  background-color: var(--background-color-progressive, #36c);
  color: var(--color-inverted, #fff);
}

.personal-dashboard-detail__sort-menu-row {
  display: flex;
  flex: 1 0 100%;
  align-items: center;
  gap: var(--spacing-50, 8px);
  min-width: 0;
  margin-left: auto;
  color: var(--color-subtle, #54595d);
  font-size: var(--font-size-small);
}

.personal-dashboard-detail__sort-menu-label {
  flex: 0 0 auto;
}

.personal-dashboard-detail__sort-menu {
  flex: 1 1 auto;
  min-width: 0;
}

@media (min-width: 40rem) {
  .personal-dashboard-detail__sort-menu-row {
    flex: 0 0 auto;
  }

  .personal-dashboard-detail__sort-menu {
    width: 11rem;
  }
}

.personal-dashboard-detail__sort-menu :deep(.cdx-select-vue__handle) {
  min-width: 0;
  font-size: var(--font-size-small);
}

.personal-dashboard-detail__sort-menu :deep(.cdx-select-vue__handle .cdx-select-vue__handle__current-selection) {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
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

/* Applied only to activity-count phrases, for example "2 contested"; never to time-count phrases. */
.personal-dashboard-detail__lede-line .personal-dashboard-detail__lede-alert {
  color: var(--color-error);
  font-weight: var(--font-weight-bold, 700);
}

.personal-dashboard-detail__lede-line .personal-dashboard-detail__lede-neutral {
  color: var(--color-subtle);
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

.personal-dashboard-detail__empty-state {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: var(--spacing-75, 12px);
  padding: var(--spacing-100, 16px);
  border: 1px solid var(--border-color-subtle);
  border-radius: var(--border-radius-base);
  background-color: var(--background-color-base);
}

.personal-dashboard-detail__empty-state p {
  margin: 0;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__group-heading {
  margin: var(--spacing-25, 4px) 0 calc(var(--spacing-50, 8px) * -1);
  color: var(--color-error);
  font-size: var(--font-size-x-small);
  font-weight: var(--font-weight-bold, 700);
  letter-spacing: 0.04em;
  line-height: var(--line-height-medium);
  text-transform: uppercase;
}

.personal-dashboard-detail__group-heading--neutral {
  color: var(--color-subtle);
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

.personal-dashboard-detail__request--v2 {
  cursor: default;
}

.personal-dashboard-detail__request:hover,
.personal-dashboard-detail__request:focus-within {
  border-color: var(--border-color-progressive);
}

.personal-dashboard-detail__request--v2:hover,
.personal-dashboard-detail__request--v2:focus-within {
  border-color: var(--border-color-subtle);
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

.personal-dashboard-detail__urgency-chip {
  align-self: flex-start;
  margin: 0 0 var(--spacing-50, 8px);
  border-color: var(--border-color-warning);
  background-color: var(--background-color-warning-subtle);
  font-variant-numeric: tabular-nums;
}

.personal-dashboard-detail__urgency-chip--error {
  border-color: var(--border-color-error);
  background-color: var(--background-color-error-subtle);
}

.personal-dashboard-detail__requester-quote,
.personal-dashboard-detail__tagger-reason {
  margin: var(--spacing-75, 12px) 0 0;
  padding-inline-start: var(--spacing-50, 8px);
  border-inline-start: 2px solid var(--border-color-subtle);
  color: var(--color-base);
  font-size: var(--font-size-small);
  font-style: italic;
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__system-signal {
  margin: var(--spacing-75, 12px) 0 0;
  color: var(--color-base);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__system-signal--urgent {
  color: var(--color-warning);
}

.personal-dashboard-detail__system-signal--error {
  color: var(--color-error);
}

.personal-dashboard-detail__system-signal strong {
  font-weight: var(--font-weight-bold, 700);
}

.personal-dashboard-detail__action-footer {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
  margin-top: var(--spacing-100, 16px);
  padding-top: var(--spacing-75, 12px);
  border-top: 1px solid var(--border-color-subtle);
}

.personal-dashboard-detail__action-footer .personal-dashboard-detail__protect-button,
.personal-dashboard-detail__action-footer .personal-dashboard-detail__delete-button {
  display: block;
  width: 100%;
  font-weight: var(--font-weight-normal, 400);
}

.personal-dashboard-detail__primary-action-row {
  display: flex;
  justify-content: flex-end;
}

.personal-dashboard-detail__action-footer--evidence .personal-dashboard-detail__primary-action-row {
  display: block;
}

.personal-dashboard-detail__action-footer .personal-dashboard-detail__evidence-button {
  display: block;
  width: 100%;
  font-weight: var(--font-weight-normal, 400);
}

.personal-dashboard-detail__action-links {
  display: flex;
  flex-wrap: nowrap;
  align-items: center;
  gap: var(--spacing-75, 12px);
  min-width: 0;
  font-size: var(--font-size-x-small);
  line-height: var(--line-height-medium);
  white-space: nowrap;
}

.personal-dashboard-detail__action-links a {
  color: var(--color-progressive);
  text-decoration: none;
}

.personal-dashboard-detail__criterion-dialog-host :deep(.cdx-dialog-backdrop) {
  background-color: var(--background-color-backdrop-light, rgba(0, 0, 0, 0.32));
}

.personal-dashboard-detail__criterion-dialog-host :deep(.personal-dashboard-detail__criterion-dialog.cdx-dialog) {
  width: min(100%, 28rem);
  max-height: 80vh;
}

@media (max-width: 39.99rem) {
  .personal-dashboard-detail__criterion-dialog-host :deep(.cdx-dialog-backdrop) {
    align-items: flex-end;
  }

  .personal-dashboard-detail__criterion-dialog-host :deep(.personal-dashboard-detail__criterion-dialog.cdx-dialog) {
    border-bottom: 0;
    border-radius: var(--border-radius-base) var(--border-radius-base) 0 0;
    box-shadow: var(--box-shadow-drop-large, 0 -2px 12px rgba(0, 0, 0, 0.12));
  }
}

.personal-dashboard-detail__criterion-dialog-host :deep(.personal-dashboard-detail__criterion-dialog .cdx-dialog__header) {
  padding-block: var(--spacing-75, 12px) var(--spacing-50, 8px);
}

.personal-dashboard-detail__filter-search {
  position: sticky;
  top: 0;
  z-index: 1;
  margin: 0 calc(var(--spacing-50, 8px) * -1);
  padding: 0 var(--spacing-50, 8px) var(--spacing-50, 8px);
  background-color: var(--background-color-base, #fff);
}

.personal-dashboard-detail__filter-section {
  margin-top: var(--spacing-100, 16px);
}

.personal-dashboard-detail__filter-section + .personal-dashboard-detail__filter-section {
  padding-top: var(--spacing-100, 16px);
  border-top: 1px solid var(--border-color-subtle);
}

.personal-dashboard-detail__filter-section-heading {
  margin: 0 0 var(--spacing-50, 8px);
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: var(--font-weight-bold, 700);
  letter-spacing: 0.04em;
  line-height: var(--line-height-medium);
  text-transform: uppercase;
}

.personal-dashboard-detail__filter-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: var(--spacing-25, 4px);
}

.personal-dashboard-detail__filter-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-50, 8px);
  padding: var(--spacing-25, 4px) 0;
}

.personal-dashboard-detail__filter-checkbox {
  flex: 1 1 auto;
  min-width: 0;
  margin: 0;
}

.personal-dashboard-detail__filter-checkbox-label {
  font-size: var(--font-size-small);
}

.personal-dashboard-detail__filter-checkbox-label strong {
  font-weight: var(--font-weight-bold, 700);
}

.personal-dashboard-detail__filter-row-count {
  flex: 0 0 auto;
  min-width: 1.5rem;
  padding: var(--spacing-12, 2px) var(--spacing-25, 4px);
  border-radius: var(--border-radius-pill, 9999px);
  background-color: rgba(0, 0, 0, 0.06);
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: var(--font-weight-bold, 700);
  font-variant-numeric: tabular-nums;
  text-align: center;
}

.personal-dashboard-detail__filter-empty {
  margin: var(--spacing-100, 16px) 0;
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  text-align: center;
}

.personal-dashboard-detail__filter-footer {
  display: flex;
  justify-content: space-between;
  gap: var(--spacing-50, 8px);
  width: 100%;
}

.personal-dashboard-detail__results-count {
  margin: 0 0 var(--spacing-25, 4px);
  color: var(--color-subtle);
  font-size: var(--font-size-small);
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

.personal-dashboard-detail__criterion-eyebrow,
.personal-dashboard-detail__category-eyebrow {
  display: block;
  margin: 0 0 var(--spacing-25, 4px);
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: var(--font-weight-bold, 700);
  letter-spacing: 0.04em;
  line-height: var(--line-height-medium);
  text-transform: uppercase;
}

.personal-dashboard-detail__category-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--spacing-75, 12px);
  margin: 0 0 var(--spacing-25, 4px);
}

.personal-dashboard-detail__category-row .personal-dashboard-detail__category-eyebrow {
  min-width: 0;
  margin: 0;
  letter-spacing: 0.06em;
}

.personal-dashboard-detail__actor-pill {
  display: inline-flex;
  flex: none;
  align-items: center;
  gap: var(--spacing-25, 4px);
  padding: var(--spacing-12, 2px) var(--spacing-50, 8px) var(--spacing-12, 2px) var(--spacing-25, 4px);
  border-radius: var(--border-radius-pill, 9999px);
  background-color: var(--background-color-neutral-subtle, rgba(0, 0, 0, 0.05));
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: 500;
  font-variant-numeric: tabular-nums;
  line-height: 1;
}

.personal-dashboard-detail__actor-icon {
  flex: none;
  color: var(--color-subtle);
}

.personal-dashboard-detail__protection-stats {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0 var(--spacing-25, 4px);
  margin: var(--spacing-75, 12px) 0 0;
  color: var(--color-base);
  font-size: var(--font-size-small);
  font-weight: var(--font-weight-normal, 400);
  line-height: var(--line-height-medium);
}

.personal-dashboard-detail__protection-stats-separator {
  color: var(--color-subtle);
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
  box-sizing: border-box;
  width: min(18rem, 84vw);
  padding: var(--spacing-100, 16px);
  overflow-y: auto;
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

.moderator-dashboard__variant-section + .moderator-dashboard__variant-section,
.moderator-dashboard__variant-section + .moderator-dashboard__variant-help {
  margin-top: var(--spacing-100, 16px);
  padding-top: var(--spacing-100, 16px);
  border-top: 1px solid var(--border-color-subtle);
}

.moderator-dashboard__variant-fieldset {
  margin-inline: 0;
  padding-inline: 0;
  padding-bottom: 0;
  border-inline: 0;
  border-bottom: 0;
}

.moderator-dashboard__variant-section-label {
  display: block;
  margin-bottom: var(--spacing-50, 8px);
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  font-weight: 600;
}

.moderator-dashboard__variant-section h2 {
  margin-top: 0;
  line-height: var(--line-height-small, 1.4285714);
}

.moderator-dashboard__variant-radio-group {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-50, 8px);
}

.moderator-dashboard__variant-subgroup {
  margin-top: var(--spacing-75, 12px);
}

.moderator-dashboard__variant-subgroup + .moderator-dashboard__variant-subgroup {
  margin-top: var(--spacing-100, 16px);
}

.moderator-dashboard__variant-subgroup-label {
  display: block;
  margin: 0 0 var(--spacing-25, 4px) var(--spacing-25, 4px);
  color: var(--color-subtle);
  font-size: var(--font-size-x-small);
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.moderator-dashboard__variant-help {
  color: var(--color-subtle);
  font-size: var(--font-size-small);
  line-height: var(--line-height-medium, 1.6);
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
