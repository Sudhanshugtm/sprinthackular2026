<script setup lang="ts">
import { computed, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { CdxIcon, CdxMessage } from '@wikimedia/codex'
import {
  cdxIconArticleRedirect,
  cdxIconEdit,
  cdxIconEllipsis,
  cdxIconHistory,
  cdxIconInfo,
  cdxIconLanguage,
  cdxIconLink,
  cdxIconLock,
  cdxIconMove,
  cdxIconTrash,
  cdxIconUnStar,
} from '@wikimedia/codex-icons'

import ChromeWrapper from '@/components/ChromeWrapper.vue'

definePage({
  meta: {
    title: 'Article',
    description:
      "Mobile prototype mirroring today's Minerva article view, surfaced from the admin moderation dashboard.",
  },
})

const route = useRoute()
const router = useRouter()

const pageTitle = computed(() => {
  const t = route.query.title
  if (typeof t === 'string' && t.trim()) return t.trim()
  return 'Apple1'
})

const isTalk = computed(() => pageTitle.value.startsWith('Talk:'))

type Module = 'protection' | 'speedy'
const moduleParam = computed<Module>(() => {
  const m = route.query.module
  if (m === 'speedy') return 'speedy'
  return 'protection'
})

interface SpeedyBanner {
  criterion: string
  label: string
}

interface Section {
  heading: string
  body: string
}

interface ArticleFixture {
  banner?: SpeedyBanner
  lead: string
  sections: Section[]
  references: string[]
  categories: string[]
}

interface TalkPost {
  author: string
  timestamp: string
  body: string
}

interface TalkThread {
  heading: string
  posts: TalkPost[]
}

interface TalkFixture {
  banner?: SpeedyBanner
  threads: TalkThread[]
}

// Banner copy mirrors the {{db-…}} templates we set up in local mwdd: a red-bordered
// notice linking to the criterion. Faithful to today's Minerva — no v2 contextual
// "Delete this page" widgets layered in.
function bannerNotice(banner: SpeedyBanner) {
  return `<strong>This page may meet the criteria for speedy deletion as ${banner.label}.</strong> If this page does not meet the criteria, or you intend to fix it, please remove this notice, but do not remove this notice from pages that you have created yourself.`
}

const articleFixtures: Record<string, ArticleFixture> = {
  Apple1: {
    lead: "The <strong>orange</strong>, also called <strong>sweet orange</strong> to distinguish it from the bitter orange (<em>Citrus × aurantium</em>), is the fruit of a tree in the family Rutaceae. Botanically, this is the hybrid <em>Citrus</em> × <em>sinensis</em>, between the pomelo (<em>Citrus maxima</em>) and the mandarin orange (<em>Citrus reticulata</em>).",
    sections: [
      {
        heading: 'Cultivation',
        body: 'The orange tree is widely cultivated in tropical and subtropical climates for its sweet fruit, which is peeled or cut and eaten whole, or processed into juice, marmalade, and essential oil.',
      },
    ],
    references: [
      'Smith, J. (2018). <em>Citrus: A History</em>. Cambridge University Press.',
    ],
    categories: ['Citrus', 'Fruits'],
  },
  'Eurovision Song Contest Asia': {
    lead: 'The <strong>Eurovision Song Contest Asia</strong> is an annual international song competition organised since 2024 by the Asia-Pacific Broadcasting Union (ABU), modelled on the original European Eurovision Song Contest. The first edition was held in Singapore in November 2024.',
    sections: [
      {
        heading: 'History',
        body: 'The contest was announced in 2022 as a regional counterpart to the European competition, with the inaugural edition held in November 2024. Twelve broadcasters participated in the first contest. By 2026 participation expanded to seventeen broadcasters across South, Southeast, and East Asia.',
      },
      {
        heading: 'Format',
        body: "Each participating broadcaster nominates one entry, performed live during a televised final. Voting is split between national juries and audience televoting from each participating country, with no televoting permitted in a country's own entry.",
      },
      {
        heading: '2026 contest',
        body: 'The 2026 contest is scheduled for late May 2026 and will be held in Bangkok, following Thailand\'s win at the 2025 edition. Seventeen broadcasters have confirmed participation; rehearsal coverage began in early May 2026.',
      },
    ],
    references: [
      "Tan, R. (2024). \"Asia's first Eurovision draws 80 million viewers across the region\". <em>The Straits Times</em>.",
      'Asia-Pacific Broadcasting Union (2026). <em>Eurovision Song Contest Asia 2026: Participating Broadcasters</em>.',
    ],
    categories: ['Music competitions', 'Asian television', 'Annual events'],
  },
  'Narcissistic personality disorder': {
    lead: '<strong>Narcissistic personality disorder</strong> (<strong>NPD</strong>) is a personality disorder characterised by a long-term pattern of grandiose self-importance, an excessive need for admiration, and a lack of empathy. It is one of the ten personality disorders recognised in the <em>Diagnostic and Statistical Manual of Mental Disorders</em>, fifth edition.',
    sections: [
      {
        heading: 'Signs and symptoms',
        body: 'Individuals with NPD often exaggerate accomplishments, expect to be recognised as superior, and have a sense of entitlement. Interpersonal difficulties are common, and the disorder frequently co-occurs with mood and substance-use disorders.',
      },
      {
        heading: 'Diagnosis',
        body: 'Diagnosis follows DSM-5-TR criteria and requires a pervasive pattern beginning by early adulthood, present across contexts. Differential diagnosis distinguishes NPD from other Cluster B personality disorders.',
      },
      {
        heading: 'Treatment',
        body: 'Long-term psychotherapy, particularly schema therapy and transference-focused psychotherapy, is the principal treatment. Medication is used only to address co-occurring conditions.',
      },
    ],
    references: [
      'American Psychiatric Association (2022). <em>Diagnostic and Statistical Manual of Mental Disorders</em> (5th ed., text rev.).',
      'Caligor, E.; Levy, K. N.; Yeomans, F. E. (2015). "Narcissistic personality disorder: diagnostic and clinical challenges". <em>American Journal of Psychiatry</em>. 172 (5).',
    ],
    categories: ['Personality disorders', 'Psychiatric diagnoses'],
  },
  'Cardig Air': {
    lead: '<strong>Cardig Air</strong> is an Indonesian cargo airline founded in 2000 and headquartered in Jakarta. The airline operates scheduled and charter cargo services across Southeast Asia.',
    sections: [
      {
        heading: 'History',
        body: 'Cardig Air was established in 2000 as a regional cargo carrier. Its first scheduled service began the same year between Jakarta and Singapore. The fleet has expanded gradually since then to include narrow- and wide-body freighters.',
      },
      {
        heading: 'Fleet',
        body: 'As of 2025, Cardig Air operates a small fleet of Boeing 737 freighters and Boeing 777 freighters, primarily on regional routes within Southeast Asia.',
      },
    ],
    references: [
      "\"Cardig Air Profile\". <em>CH-Aviation</em> (2025).",
    ],
    categories: ['Airlines of Indonesia', 'Cargo airlines'],
  },
  'Groundhog Day (film)': {
    lead: '<strong>Groundhog Day</strong> is a 1993 American fantasy comedy film directed by Harold Ramis and starring Bill Murray, Andie MacDowell, and Chris Elliott. Murray plays Phil Connors, a cynical television weatherman covering the annual Groundhog Day event in Punxsutawney, Pennsylvania, who finds himself in a time loop, repeating the same day over and over.',
    sections: [
      {
        heading: 'Plot',
        body: 'A self-centred TV weatherman is sent to cover the Groundhog Day festival and finds himself reliving the same day in an endless loop. Over many iterations he progresses through despair, hedonism, and eventually self-improvement, which breaks the cycle.',
      },
      {
        heading: 'Cast',
        body: 'Bill Murray as Phil Connors. Andie MacDowell as Rita Hanson. Chris Elliott as Larry. Stephen Tobolowsky as Ned Ryerson.',
      },
      {
        heading: 'Reception',
        body: "<em>Groundhog Day</em> received broadly positive reviews on release and has since been recognised as one of the most influential American comedies. The film's premise has become a cultural shorthand for repeating cycles of behaviour.",
      },
    ],
    references: [
      'Ebert, R. (1993). "Groundhog Day". <em>Chicago Sun-Times</em>.',
      'Library of Congress (2006). <em>National Film Registry: 2006 Selections</em>.',
    ],
    categories: ['1993 films', 'American fantasy films', 'Time loop films'],
  },
  '2025-26 Persian Gulf Pro League': {
    lead: 'The <strong>2025-26 Persian Gulf Pro League</strong> is the 25th season of the Persian Gulf Pro League, the top professional football league in Iran. The season began in August 2025 and is scheduled to conclude in May 2026.',
    sections: [
      {
        heading: 'Teams',
        body: 'Sixteen clubs are competing in the 2025-26 season, including Persepolis, Esteghlal, Sepahan, and Tractor S.C.',
      },
      {
        heading: 'Standings',
        body: 'Persepolis lead the table at the start of the second half of the season, ahead of Sepahan and Esteghlal.',
      },
    ],
    references: [
      "\"Persian Gulf Pro League 2025-26 fixtures\". <em>Iran Football Association</em>.",
    ],
    categories: ['Persian Gulf Pro League seasons', '2025-26 in Asian football'],
  },
  'Smith Industries Holdings Ltd': {
    banner: { criterion: 'G11', label: 'unambiguous advertising or promotion' },
    lead: '<strong>Smith Industries Holdings Ltd</strong> is a leading global provider of innovative solutions across multiple sectors. Founded with a vision to deliver world-class value, the company has consistently set industry benchmarks for excellence and customer satisfaction.',
    sections: [
      {
        heading: 'About us',
        body: "Smith Industries Holdings Ltd is committed to driving meaningful impact through best-in-class products and unparalleled service. Our team of dedicated professionals leverages cutting-edge methodologies to deliver transformative outcomes for our valued clients.",
      },
      {
        heading: 'Our values',
        body: 'Excellence, integrity, and innovation guide every decision we make. We empower our people to think boldly, act decisively, and exceed expectations at every turn.',
      },
      {
        heading: 'Leadership',
        body: 'Under visionary leadership, Smith Industries Holdings Ltd continues to redefine industry standards and unlock new opportunities for sustainable growth and stakeholder value.',
      },
    ],
    references: [],
    categories: ['Companies'],
  },
  'Regional innovation timeline': {
    banner: { criterion: 'G15', label: 'unreviewed LLM-generated content' },
    lead: 'The <strong>Regional innovation timeline</strong> presents a comprehensive overview of key milestones in regional innovation. From early experiments in the late 1990s to the rapid acceleration of the past decade, this timeline highlights pivotal moments that shaped the regional ecosystem.',
    sections: [
      {
        heading: '1990s',
        body: 'In 1998, several pioneering initiatives were launched across the region, laying the groundwork for future development. Key stakeholders began to recognise the importance of fostering an environment conducive to innovation, leading to the establishment of foundational frameworks.',
      },
      {
        heading: '2000s',
        body: "By 2003, the ecosystem had expanded considerably, with new actors entering the space and existing organisations broadening their remit. The decade saw a notable shift toward collaborative models, with cross-sector partnerships emerging as a defining feature of the period.",
      },
      {
        heading: '2010s',
        body: 'The 2010s saw rapid acceleration as digital infrastructure matured and new funding mechanisms became available. By the end of the decade, the region was widely cited as a model for emerging innovation hubs.',
      },
      {
        heading: '2020s',
        body: 'By 2024 the region was recognised as a global leader, with multiple awards and citations from international bodies. The current period is characterised by deeper integration with global value chains and continued investment in talent development.',
      },
    ],
    references: [
      "Innovation Quarterly (2018). \"Regional dynamics in emerging ecosystems\".",
      'Global Innovation Index (2024). <em>Annual report</em>.',
    ],
    categories: ['Regional development', 'Timelines'],
  },
  'Henrik Olsen (footballer)': {
    banner: { criterion: 'A1', label: 'no context' },
    lead: '<strong>Henrik Olsen</strong> is a footballer.',
    sections: [],
    references: [],
    categories: [],
  },
}

const talkFixtures: Record<string, TalkFixture> = {
  'Talk:Blue Whale Challenge': {
    threads: [
      {
        heading: 'Removing off-topic re-posts again',
        posts: [
          {
            author: 'AdminGamma',
            timestamp: '02:15, 21 April 2026 (UTC)',
            body: 'Cleared three off-topic posts encouraging viewers to participate. Going to log this and request talk-page semi-protection if it recurs.',
          },
          {
            author: 'Reporter2',
            timestamp: '18:14, 5 May 2026 (UTC)',
            body: 'Same thread is back from a fresh account. Filed at WP:RFPP. Diffs available on request.',
          },
        ],
      },
      {
        heading: 'Suggested protection levels',
        posts: [
          {
            author: 'NewAccountBWC',
            timestamp: '06:30, 5 May 2026 (UTC)',
            body: 'I would like to discuss why this article should not be locked. There are perspectives that need to be aired.',
          },
        ],
      },
    ],
  },
  'Talk:Discontinued template (orphan)': {
    banner: { criterion: 'G6', label: 'housekeeping' },
    threads: [
      {
        heading: 'Archive of original template discussion',
        posts: [
          {
            author: 'OldHand',
            timestamp: '04:55, 5 May 2025 (UTC)',
            body: 'Original discussion archive for {{tl|Discontinued}}, retired following the consolidation discussion at WikiProject Templates Archive 12. Leaving the talk page in place for context.',
          },
        ],
      },
    ],
  },
}

const articleFixture = computed<ArticleFixture | null>(() => {
  if (isTalk.value) return null
  return (
    articleFixtures[pageTitle.value] ?? {
      lead: `<strong>${pageTitle.value}</strong> is a placeholder article rendered for prototype QA.`,
      sections: [],
      references: [],
      categories: [],
    }
  )
})

const talkFixture = computed<TalkFixture | null>(() => {
  if (!isTalk.value) return null
  return (
    talkFixtures[pageTitle.value] ?? {
      threads: [
        {
          heading: 'Discussion',
          posts: [
            {
              author: 'AnonymousUser',
              timestamp: '12:00, 7 May 2026 (UTC)',
              body: `Placeholder talk-page thread rendered for ${pageTitle.value}.`,
            },
          ],
        },
      ],
    }
  )
})

const banner = computed<SpeedyBanner | undefined>(() => {
  return articleFixture.value?.banner ?? talkFixture.value?.banner
})

const overflowOpen = ref(false)
function toggleOverflow() {
  overflowOpen.value = !overflowOpen.value
}

// Just-protected confirmation banner — surfaced when protect.vue forwards an
// admin to the article view after a successful submit. The lock icon is the
// in-context confirmation; this banner adds the missing "what next" affordances.
const justProtected = computed(() => route.query['just-protected'] === 'true')

const protectedDuration = computed(() => {
  const v = route.query.duration
  return typeof v === 'string' ? v : ''
})

const protectedRequestPath = computed(() => {
  const v = route.query.request
  return typeof v === 'string' ? v : ''
})

const justProtectedDismissed = ref(false)
const showJustProtectedBanner = computed(
  () => justProtected.value && !justProtectedDismissed.value,
)

function dismissJustProtectedBanner() {
  justProtectedDismissed.value = true
  // Clean the query so a refresh doesn't resurrect the banner.
  const cleaned = { ...route.query }
  delete cleaned['just-protected']
  delete cleaned.duration
  delete cleaned.request
  router.replace({ path: route.path, query: cleaned })
}

function goToProtectionRequest(event?: Event) {
  if (event) event.preventDefault()
  if (!protectedRequestPath.value) return
  router.push(protectedRequestPath.value)
}

function openDeleteForm(event?: Event) {
  if (event) event.preventDefault()
  overflowOpen.value = false
  const query: Record<string, string> = { title: pageTitle.value }
  if (banner.value?.criterion) query.criterion = banner.value.criterion
  router.push({ path: '/admin-moderation-dashboard/delete', query })
}

function openProtectForm(event?: Event) {
  if (event) event.preventDefault()
  overflowOpen.value = false
  router.push({
    path: '/admin-moderation-dashboard/protect',
    query: { title: pageTitle.value, module: moduleParam.value },
  })
}

// Canonical "return to dashboard" URL — module mirrors the article context.
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

function goBackToDashboard(event?: Event) {
  if (event) event.preventDefault()
  router.push(fallbackDashboardRoute.value)
}
</script>

<template>
  <ChromeWrapper skin="mobile">
    <article class="article-page">
      <h1 class="article-page__title">{{ pageTitle }}</h1>

      <p class="article-page__back">
        &larr; <a href="#" @click="goBackToDashboard">Dashboard</a>
      </p>

      <nav class="article-page__tabs" aria-label="Page tabs">
        <a
          class="article-page__tab"
          :class="{ 'article-page__tab--active': !isTalk }"
          href="#"
          @click.prevent
          >Page</a
        >
        <a
          class="article-page__tab"
          :class="{ 'article-page__tab--active': isTalk }"
          href="#"
          @click.prevent
          >Discussion</a
        >
      </nav>

      <!--
        Just-protected confirmation — single-line Codex success message with an
        inline call-to-action link. Quiet enough that the article body stays the
        focus, loud enough that the admin sees what just happened.
      -->
      <CdxMessage
        v-if="showJustProtectedBanner"
        class="just-protected"
        type="success"
        fade-in
        allow-user-dismiss
        dismiss-button-label="Dismiss confirmation"
        @user-dismissed="dismissJustProtectedBanner"
      >
        <span class="just-protected__line">
          <strong>Protected</strong><template v-if="protectedDuration"> · {{ protectedDuration }}</template>.
          <a
            v-if="protectedRequestPath"
            class="just-protected__link"
            href="#"
            @click="goToProtectionRequest"
          >Reply on RFPP &rarr;</a>
        </span>
      </CdxMessage>

      <!-- Page-actions toolbar — faithful to today's Minerva: Translate · Watch · History · Edit · ⋮ -->
      <div class="article-page__page-actions">
        <button
          class="article-page__action-btn"
          type="button"
          aria-label="Change language"
        >
          <CdxIcon :icon="cdxIconLanguage" />
        </button>
        <button class="article-page__action-btn" type="button" aria-label="Watch this page">
          <CdxIcon :icon="cdxIconUnStar" />
        </button>
        <button class="article-page__action-btn" type="button" aria-label="View history">
          <CdxIcon :icon="cdxIconHistory" />
        </button>
        <button class="article-page__action-btn" type="button" aria-label="Edit">
          <CdxIcon :icon="cdxIconEdit" />
        </button>
        <button
          class="article-page__action-btn"
          type="button"
          aria-label="More page actions"
          :aria-expanded="overflowOpen"
          @click="toggleOverflow"
        >
          <CdxIcon :icon="cdxIconEllipsis" />
        </button>

        <ul v-if="overflowOpen" class="article-page__overflow">
          <li>
            <a href="#" @click.prevent>
              <CdxIcon :icon="cdxIconArticleRedirect" />What links here
            </a>
          </li>
          <li>
            <a href="#" @click.prevent>
              <CdxIcon :icon="cdxIconLink" />Permanent link
            </a>
          </li>
          <li>
            <a href="#" @click.prevent>
              <CdxIcon :icon="cdxIconInfo" />Page information
            </a>
          </li>
          <li>
            <a href="#" @click.prevent>
              <CdxIcon :icon="cdxIconEdit" />Edit full page
            </a>
          </li>
          <li>
            <a href="#" @click="openDeleteForm">
              <CdxIcon :icon="cdxIconTrash" />Delete
            </a>
          </li>
          <li>
            <a href="#" @click.prevent>
              <CdxIcon :icon="cdxIconMove" />Move
            </a>
          </li>
          <li>
            <a href="#" @click="openProtectForm">
              <CdxIcon :icon="cdxIconLock" />Protect
            </a>
          </li>
        </ul>
      </div>

      <!-- Speedy deletion banner (today's MW behaviour, not the v2 contextual surface) -->
      <div v-if="banner" class="article-page__banner" role="note">
        <p v-html="bannerNotice(banner)" />
      </div>

      <!-- Article body -->
      <template v-if="articleFixture">
        <p class="article-page__lead" v-html="articleFixture.lead" />

        <section
          v-for="section in articleFixture.sections"
          :key="section.heading"
          class="article-page__section"
        >
          <h2 class="article-page__section-heading">{{ section.heading }}</h2>
          <p v-html="section.body" />
        </section>

        <section v-if="articleFixture.references.length" class="article-page__section">
          <h2 class="article-page__section-heading">References</h2>
          <ol class="article-page__references">
            <li v-for="(ref, i) in articleFixture.references" :key="i" v-html="ref" />
          </ol>
        </section>

        <footer
          v-if="articleFixture.categories.length"
          class="article-page__categories"
        >
          <strong>Categories:</strong>
          <span v-for="(cat, i) in articleFixture.categories" :key="cat">
            <a href="#" @click.prevent>{{ cat }}</a><template
              v-if="i < articleFixture.categories.length - 1"
              > · </template
            >
          </span>
        </footer>
      </template>

      <!-- Talk body -->
      <template v-if="talkFixture">
        <section
          v-for="thread in talkFixture.threads"
          :key="thread.heading"
          class="article-page__section"
        >
          <h2 class="article-page__section-heading">{{ thread.heading }}</h2>
          <div
            v-for="(post, i) in thread.posts"
            :key="i"
            class="article-page__talk-post"
          >
            <p v-html="post.body" />
            <p class="article-page__talk-sig">
              &mdash; <a href="#" @click.prevent>{{ post.author }}</a>
              {{ post.timestamp }}
            </p>
          </div>
        </section>
      </template>
    </article>
  </ChromeWrapper>
</template>

<style scoped>
.article-page {
  box-sizing: border-box;
  padding: var(--spacing-100, 16px);
  background-color: var(--background-color-base, #fff);
  color: var(--color-base, #202122);
}

.article-page__title {
  margin: 0 0 var(--spacing-50, 8px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.875rem;
  font-weight: 400;
  line-height: 1.2;
  border: none;
  word-break: break-word;
}

.article-page__back {
  margin: 0 0 var(--spacing-100, 16px);
  font-size: var(--font-size-medium, 1rem);
}

.article-page__back a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.article-page__back a:hover {
  text-decoration: underline;
}

.article-page__tabs {
  display: flex;
  gap: var(--spacing-150, 24px);
  margin: 0 0 var(--spacing-50, 8px);
  font-size: var(--font-size-medium, 1rem);
  font-weight: var(--font-weight-bold, 700);
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.article-page__tab {
  padding-bottom: var(--spacing-25, 4px);
  color: var(--color-destructive, #bf3c2c);
  text-decoration: none;
  border-bottom: 2px solid transparent;
}

.article-page__tab--active {
  color: var(--color-base, #202122);
  border-bottom-color: var(--color-base, #202122);
}

.article-page__page-actions {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--spacing-150, 24px);
  margin: 0 0 var(--spacing-100, 16px);
  padding: var(--spacing-50, 8px) 0;
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.article-page__action-btn {
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

.article-page__action-btn:last-of-type {
  margin-left: auto;
}

.article-page__action-btn:hover {
  background-color: var(--background-color-neutral-subtle, #f8f9fa);
}

.article-page__overflow {
  position: absolute;
  top: calc(100% + 4px);
  right: 0;
  z-index: 5;
  list-style: none;
  margin: 0;
  padding: var(--spacing-50, 8px) 0;
  background: var(--background-color-base, #fff);
  border: 1px solid var(--border-color-base, #c8ccd1);
  border-radius: var(--border-radius-base, 2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
  min-width: 220px;
}

.article-page__overflow li {
  margin: 0;
}

.article-page__overflow a {
  display: flex;
  align-items: center;
  gap: var(--spacing-75, 12px);
  padding: var(--spacing-50, 8px) var(--spacing-100, 16px);
  color: var(--color-base, #202122);
  text-decoration: none;
  font-weight: var(--font-weight-bold, 700);
}

.article-page__overflow a :deep(.cdx-icon) {
  flex-shrink: 0;
  color: var(--color-subtle, #54595d);
}

.article-page__overflow a:hover {
  background-color: var(--background-color-neutral-subtle, #f8f9fa);
}

.article-page__banner {
  margin: 0 0 var(--spacing-150, 24px);
  padding: var(--spacing-75, 12px) var(--spacing-100, 16px);
  background-color: #fee7e6;
  border: 2px solid #d33;
  border-radius: var(--border-radius-base, 2px);
  font-size: var(--font-size-small, 0.875rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-base, #202122);
}

.article-page__banner p {
  margin: 0;
}

.article-page__lead {
  margin: 0 0 var(--spacing-150, 24px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.6);
}

.article-page__section {
  margin: 0 0 var(--spacing-150, 24px);
}

.article-page__section-heading {
  margin: 0 0 var(--spacing-75, 12px);
  padding-bottom: var(--spacing-25, 4px);
  font-family: 'Linux Libertine', Georgia, 'Times New Roman', serif;
  font-size: 1.5rem;
  font-weight: 400;
  line-height: 1.3;
  border-bottom: 1px solid var(--border-color-subtle, #eaecf0);
}

.article-page__section p {
  margin: 0 0 var(--spacing-100, 16px);
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.6);
}

.article-page__references {
  margin: 0;
  padding-left: var(--spacing-150, 24px);
  font-size: var(--font-size-small, 0.875rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-subtle, #54595d);
}

.article-page__references li {
  margin-bottom: var(--spacing-50, 8px);
}

.article-page__categories {
  margin: var(--spacing-150, 24px) 0 0;
  padding: var(--spacing-50, 8px) var(--spacing-75, 12px);
  background-color: var(--background-color-neutral-subtle, #f8f9fa);
  border-top: 1px solid var(--border-color-subtle, #eaecf0);
  font-size: var(--font-size-small, 0.875rem);
  color: var(--color-subtle, #54595d);
}

.article-page__categories a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

.article-page__categories a:hover {
  text-decoration: underline;
}

.article-page__talk-post {
  margin: 0 0 var(--spacing-100, 16px);
  padding-left: var(--spacing-100, 16px);
  border-left: 2px solid var(--border-color-subtle, #eaecf0);
}

.article-page__talk-post p {
  margin: 0 0 var(--spacing-50, 8px);
}

.article-page__talk-sig {
  font-size: var(--font-size-small, 0.875rem);
  color: var(--color-subtle, #54595d);
}

.article-page__talk-sig a {
  color: var(--color-progressive, #36c);
  text-decoration: none;
}

/* Just-protected confirmation — wraps a single-line Codex CdxMessage. */
.just-protected {
  margin: 0 0 var(--spacing-100, 16px);
}

.just-protected__line {
  font-size: var(--font-size-medium, 1rem);
  line-height: var(--line-height-medium, 1.5);
  color: var(--color-base, #202122);
}

.just-protected__link {
  margin-left: var(--spacing-25, 4px);
  color: var(--color-progressive, #36c);
  text-decoration: none;
  white-space: nowrap;
}

.just-protected__link:hover {
  text-decoration: underline;
}
</style>
