<script setup lang="ts">
import { computed, ref } from 'vue'
import MetricCard from '../components/MetricCard.vue'
import PerformanceTrendChart from '../components/PerformanceTrendChart.vue'
import metrics from '../data/metrics.json'

type RegionName = 'North America' | 'Europe' | 'Asia Pacific' | 'Latin America'
type Severity = 'Critical' | 'High' | 'Medium' | 'Low'
type RegionStatus = 'Healthy' | 'Watch' | 'At Risk'
type ExceptionSortKey = 'id' | 'region' | 'reference' | 'type' | 'severity' | 'age' | 'description'

interface DailyRecord {
  daysAgo: number
  region: RegionName
  shipments: number
  onTimeShipments: number
  freightCost: number
}

interface ExceptionRecord {
  id: string
  region: RegionName
  reference: string
  type: string
  severity: Severity
  daysAgo: number
  description: string
}

interface RegionBaseline {
  name: RegionName
  baseShipments: number
  targetOnTime: number
  averageCost: number
}

const regions = metrics.regions as RegionBaseline[]
const exceptions = metrics.exceptions as ExceptionRecord[]

const dateOptions = [
  { label: 'Last 7 Days', value: '7' },
  { label: 'Last 30 Days', value: '30' },
  { label: 'Last 90 Days', value: '90' },
]

const today = new Date()
today.setHours(0, 0, 0, 0)

const dailyRecords: DailyRecord[] = Array.from({ length: 180 }, (_, daysAgo) =>
  regions.flatMap((region, regionIndex) => {
    const dayIndex = 179 - daysAgo
    const volumeWave =
      Math.sin((dayIndex + regionIndex * 8) / 5.2) * 15 +
      Math.cos((dayIndex + regionIndex * 3) / 11) * 9
    const shipments = Math.round(region.baseShipments + volumeWave)
    const onTimeRate =
      region.targetOnTime +
      Math.sin((dayIndex + regionIndex * 6) / 8) * 1.15 +
      Math.cos((dayIndex + regionIndex * 4) / 13) * 0.55
    const freightRate =
      region.averageCost * (1 + Math.sin((dayIndex + regionIndex * 5) / 9) * 0.035)

    return {
      daysAgo,
      region: region.name,
      shipments,
      onTimeShipments: Math.round(shipments * (onTimeRate / 100)),
      freightCost: Math.round(shipments * freightRate),
    }
  }),
).flat()

const selectedRange = ref('30')
const selectedRegion = ref('all')
const selectedDays = computed(() => Number(selectedRange.value))
const regionFilter = computed(() =>
  selectedRegion.value === 'all' ? undefined : (selectedRegion.value as RegionName),
)

const selectedRecords = computed(() =>
  dailyRecords.filter(
    (record) =>
      record.daysAgo < selectedDays.value &&
      (!regionFilter.value || record.region === regionFilter.value),
  ),
)

const previousRecords = computed(() =>
  dailyRecords.filter(
    (record) =>
      record.daysAgo >= selectedDays.value &&
      record.daysAgo < selectedDays.value * 2 &&
      (!regionFilter.value || record.region === regionFilter.value),
  ),
)

function summarize(records: DailyRecord[]) {
  const shipments = records.reduce((sum, record) => sum + record.shipments, 0)
  const onTimeShipments = records.reduce((sum, record) => sum + record.onTimeShipments, 0)
  const freightCost = records.reduce((sum, record) => sum + record.freightCost, 0)
  return {
    shipments,
    onTimeShipments,
    onTimeRate: shipments === 0 ? 0 : (onTimeShipments / shipments) * 100,
    freightCost,
  }
}

const currentSummary = computed(() => summarize(selectedRecords.value))
const previousSummary = computed(() => summarize(previousRecords.value))

const selectedExceptions = computed(() =>
  exceptions
    .filter(
      (exception) =>
        exception.daysAgo < selectedDays.value &&
        (!regionFilter.value || exception.region === regionFilter.value),
    )
    .sort((a, b) => severityRank[a.severity] - severityRank[b.severity] || b.daysAgo - a.daysAgo),
)

const severityRank: Record<Severity, number> = {
  Critical: 0,
  High: 1,
  Medium: 2,
  Low: 3,
}

const severityFilter = ref<Severity | ''>('')
const typeFilter = ref('')
const exceptionSearch = ref('')
const exceptionSortKey = ref<ExceptionSortKey>('severity')
const exceptionSortDirection = ref<'asc' | 'desc'>('asc')
const exceptionTypes = [...new Set(exceptions.map((exception) => exception.type))].sort((a, b) =>
  a.localeCompare(b),
)

const tableExceptions = computed(() => {
  const search = exceptionSearch.value.trim().toLowerCase()
  return selectedExceptions.value
    .filter(
      (exception) =>
        (!severityFilter.value || exception.severity === severityFilter.value) &&
        (!typeFilter.value || exception.type === typeFilter.value) &&
        (!search ||
          [exception.id, exception.reference, exception.description].some((value) =>
            value.toLowerCase().includes(search),
          )),
    )
    .sort((a, b) => {
      let comparison = 0
      switch (exceptionSortKey.value) {
        case 'severity':
          comparison = severityRank[a.severity] - severityRank[b.severity]
          break
        case 'age':
          comparison = a.daysAgo - b.daysAgo
          break
        case 'id':
        case 'region':
        case 'reference':
        case 'type':
        case 'description':
          comparison = a[exceptionSortKey.value].localeCompare(b[exceptionSortKey.value])
          break
      }

      if (comparison !== 0) {
        return comparison * (exceptionSortDirection.value === 'asc' ? 1 : -1)
      }
      if (exceptionSortKey.value === 'severity' && a.daysAgo !== b.daysAgo) {
        return b.daysAgo - a.daysAgo
      }
      return a.id.localeCompare(b.id)
    })
})

const criticalExceptions = computed(
  () => selectedExceptions.value.filter((exception) => exception.severity === 'Critical').length,
)

const tableCriticalExceptions = computed(
  () => tableExceptions.value.filter((exception) => exception.severity === 'Critical').length,
)

const regionalPerformance = computed(() =>
  regions
    .filter((region) => !regionFilter.value || region.name === regionFilter.value)
    .map((region) => {
      const records = selectedRecords.value.filter((record) => record.region === region.name)
      const summary = summarize(records)
      const openExceptions = selectedExceptions.value.filter(
        (exception) => exception.region === region.name,
      ).length
      const status: RegionStatus =
        summary.onTimeRate >= 95 ? 'Healthy' : summary.onTimeRate >= 93 ? 'Watch' : 'At Risk'
      return { ...region, ...summary, openExceptions, status }
    }),
)

interface AttentionInsight {
  title: string
  detail: string
  tone: 'critical' | 'warning' | 'watch' | 'healthy'
}

const attentionInsight = computed<AttentionInsight>(() => {
  const criticalByRegion = selectedExceptions.value
    .filter((exception) => exception.severity === 'Critical')
    .reduce((counts, exception) => {
      counts.set(exception.region, (counts.get(exception.region) ?? 0) + 1)
      return counts
    }, new Map<RegionName, number>())

  if (criticalExceptions.value > 0) {
    const affectedRegions = [...criticalByRegion.entries()]
      .sort(([regionA], [regionB]) => regionA.localeCompare(regionB))
      .map(([region]) => region)
    return {
      title: `${criticalExceptions.value} critical ${criticalExceptions.value === 1 ? 'exception requires' : 'exceptions require'} leadership attention.`,
      detail: `Affected ${affectedRegions.length === 1 ? 'region' : 'regions'}: ${affectedRegions.join(', ')}.`,
      tone: 'critical',
    }
  }

  const lowestPerformingRegion = regionalPerformance.value
    .filter((region) => region.onTimeRate < 95)
    .sort((regionA, regionB) => regionA.onTimeRate - regionB.onTimeRate)[0]

  if (lowestPerformingRegion) {
    return {
      title: `${lowestPerformingRegion.name} is below the 95% on-time delivery target.`,
      detail: `Current performance is ${lowestPerformingRegion.onTimeRate.toFixed(1)}%, ${(95 - lowestPerformingRegion.onTimeRate).toFixed(1)} percentage points below target.`,
      tone: 'warning',
    }
  }

  const highSeverityCounts = selectedExceptions.value
    .filter((exception) => exception.severity === 'High' || exception.severity === 'Critical')
    .reduce((counts, exception) => {
      counts.set(exception.type, (counts.get(exception.type) ?? 0) + 1)
      return counts
    }, new Map<string, number>())
  const recurringHighSeverityType = [...highSeverityCounts.entries()]
    .filter(([, count]) => count > 1)
    .sort(([typeA, countA], [typeB, countB]) => countB - countA || typeA.localeCompare(typeB))[0]

  if (recurringHighSeverityType) {
    const [type, count] = recurringHighSeverityType
    return {
      title: `${type} is the most common high-severity exception type in this view.`,
      detail: `${count} open ${type.toLowerCase()} exceptions are currently recorded.`,
      tone: 'watch',
    }
  }

  return {
    title: 'No major issues detected in this view.',
    detail: 'Regional delivery is meeting target and no critical or recurring high-severity exceptions were found.',
    tone: 'healthy',
  }
})

interface TrendPoint {
  label: string
  shipments: number
  onTimeRate: number
}

const chartPoints = computed<TrendPoint[]>(() => {
  const days = selectedDays.value
  const bucketCount = days === 7 ? 7 : days === 30 ? 10 : 12
  const bucketSize = days / bucketCount
  const buckets = Array.from({ length: bucketCount }, (_, index) => {
    const lower = days - (index + 1) * bucketSize
    const upper = days - index * bucketSize
    const records = selectedRecords.value.filter(
      (record) => record.daysAgo >= lower && record.daysAgo < upper,
    )
    const summary = summarize(records)
    const date = new Date(today)
    date.setDate(date.getDate() - Math.floor((lower + upper) / 2))
    return {
      label: new Intl.DateTimeFormat('en-US', { month: 'short', day: 'numeric' }).format(date),
      shipments: summary.shipments,
      onTimeRate: summary.onTimeRate,
    }
  })

  return buckets
})

const updatedAt = new Intl.DateTimeFormat('en-US', {
  month: 'short',
  day: 'numeric',
  year: 'numeric',
  hour: 'numeric',
  minute: '2-digit',
}).format(new Date())

const shipmentChange = computed(() =>
  percentageChange(currentSummary.value.shipments, previousSummary.value.shipments),
)
const freightChange = computed(() =>
  percentageChange(currentSummary.value.freightCost, previousSummary.value.freightCost),
)
const onTimeChange = computed(
  () => currentSummary.value.onTimeRate - previousSummary.value.onTimeRate,
)

function percentageChange(current: number, previous: number) {
  if (previous === 0) return 0
  return ((current - previous) / previous) * 100
}

function formatNumber(value: number) {
  return new Intl.NumberFormat('en-US').format(value)
}

function formatCurrency(value: number) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 0,
  }).format(value)
}

function formatCompactCurrency(value: number) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    notation: 'compact',
    maximumFractionDigits: 1,
  }).format(value)
}

function formatPercentChange(value: number) {
  return `${value > 0 ? '+' : ''}${value.toFixed(1)}%`
}

function formatPointChange(value: number) {
  return `${value > 0 ? '+' : ''}${value.toFixed(1)} pp`
}

function resetFilters() {
  selectedRange.value = '30'
  selectedRegion.value = 'all'
}

function sortExceptionsBy(key: ExceptionSortKey) {
  if (exceptionSortKey.value === key) {
    exceptionSortDirection.value = exceptionSortDirection.value === 'asc' ? 'desc' : 'asc'
    return
  }

  exceptionSortKey.value = key
  exceptionSortDirection.value = key === 'age' ? 'desc' : 'asc'
}

function exceptionSortAria(key: ExceptionSortKey) {
  if (exceptionSortKey.value !== key) return 'none'
  return exceptionSortDirection.value === 'asc' ? 'ascending' : 'descending'
}

function exceptionSortLabel(label: string, key: ExceptionSortKey) {
  return `Sort by ${label}${exceptionSortKey.value === key ? `, ${exceptionSortDirection.value === 'asc' ? 'ascending' : 'descending'}` : ''}`
}

function exceptionSortIndicator(key: ExceptionSortKey) {
  if (exceptionSortKey.value !== key) return '↕'
  return exceptionSortDirection.value === 'asc' ? '↑' : '↓'
}

function clearExceptionFilters() {
  severityFilter.value = ''
  typeFilter.value = ''
  exceptionSearch.value = ''
}

function exceptionAge(daysAgo: number) {
  return daysAgo === 0 ? 'Today' : `${daysAgo}d`
}
</script>

<template>
  <div class="app-shell">
    <main id="overview" class="main-content">
      <header class="page-header">
        <div class="page-header__title">
          <div class="breadcrumb"><span>Operations</span><span class="breadcrumb__slash">/</span>Overview</div>
          <div class="page-header__heading">
            <div>
              <h1>Operations Command Center</h1>
              <p>Executive Network Performance</p>
            </div>
            <span class="header-tag"><span></span> NETWORK OVERVIEW</span>
          </div>
        </div>
        <div class="page-header__meta">
          <span class="refresh-dot"></span>
          <div>
            <span class="page-header__meta-label">LAST UPDATED</span>
            <time :datetime="new Date().toISOString()">{{ updatedAt }}</time>
          </div>
          <div class="avatar" aria-label="Executive operations view">EO</div>
        </div>
      </header>

      <div class="dashboard-content">
        <section class="filter-bar" aria-label="Dashboard filters">
          <div class="filter-bar__intro">
            <span class="filter-bar__icon" aria-hidden="true">
              <svg viewBox="0 0 24 24" fill="none">
                <path d="M4 6h16M7 12h10m-7 6h4" />
                <circle cx="8" cy="6" r="1.5" />
                <circle cx="15" cy="12" r="1.5" />
                <circle cx="12" cy="18" r="1.5" />
              </svg>
            </span>
            <div><strong>View filters</strong><span>Refine your network view</span></div>
          </div>
          <div class="filter-control">
            <label for="date-range">DATE RANGE</label>
            <select id="date-range" v-model="selectedRange">
              <option v-for="option in dateOptions" :key="option.value" :value="option.value">
                {{ option.label }}
              </option>
            </select>
          </div>
          <div class="filter-control">
            <label for="region-filter">REGION</label>
            <select id="region-filter" v-model="selectedRegion">
              <option value="all">All Regions</option>
              <option v-for="region in regions" :key="region.name" :value="region.name">
                {{ region.name }}
              </option>
            </select>
          </div>
          <button class="reset-button" type="button" @click="resetFilters">
            <svg viewBox="0 0 20 20" fill="none" aria-hidden="true">
              <path d="M4.2 8a6 6 0 1 1-.2 4m.2-8v4h4" />
            </svg>
            Reset filters
          </button>
        </section>

        <section class="metrics-section" aria-labelledby="metrics-title">
          <div class="section-heading">
            <div>
              <span class="eyebrow">NETWORK HEALTH</span>
              <h2 id="metrics-title">Executive overview</h2>
            </div>
            <span class="period-note">Compared with previous {{ selectedDays }} days</span>
          </div>
          <div class="metrics-grid">
            <MetricCard
              label="Shipment volume"
              :value="formatNumber(currentSummary.shipments)"
              :detail="`vs. prior ${selectedDays} days`"
              :change="formatPercentChange(shipmentChange)"
              :change-direction="shipmentChange >= 0 ? 'positive' : 'negative'"
              icon="shipments"
            />
            <MetricCard
              label="On-time delivery"
              :value="`${currentSummary.onTimeRate.toFixed(1)}%`"
              :detail="currentSummary.onTimeRate >= 95
                ? `${(currentSummary.onTimeRate - 95).toFixed(1)} pp above 95% target`
                : `${(95 - currentSummary.onTimeRate).toFixed(1)} pp below 95% target`"
              :change="formatPointChange(onTimeChange)"
              :change-direction="onTimeChange >= 0 ? 'positive' : 'negative'"
              icon="ontime"
            />
            <MetricCard
              label="Open exceptions"
              :value="formatNumber(selectedExceptions.length)"
              :detail="`${criticalExceptions} critical requiring attention`"
              icon="exceptions"
            />
            <MetricCard
              label="Freight cost"
              :value="formatCompactCurrency(currentSummary.freightCost)"
              :detail="`vs. prior ${selectedDays} days`"
              :change="formatPercentChange(freightChange)"
              :change-direction="freightChange <= 0 ? 'positive' : 'negative'"
              icon="cost"
            />
          </div>
        </section>

        <aside
          class="attention-callout"
          :class="`attention-callout--${attentionInsight.tone}`"
          aria-labelledby="attention-title"
          aria-live="polite"
        >
          <span class="attention-callout__icon" aria-hidden="true">
            <svg viewBox="0 0 24 24" fill="none">
              <path d="M12 3.5 21 20H3l9-16.5Z" />
              <path d="M12 9v4m0 3h.01" />
            </svg>
          </span>
          <div class="attention-callout__content">
            <div class="attention-callout__label" id="attention-title">Needs Attention</div>
            <p class="attention-callout__title">{{ attentionInsight.title }}</p>
            <p class="attention-callout__detail">{{ attentionInsight.detail }}</p>
          </div>
          <span class="attention-callout__scope">
            {{ dateOptions.find((option) => option.value === selectedRange)?.label }}
            <template v-if="selectedRegion !== 'all'"> · {{ selectedRegion }}</template>
          </span>
        </aside>

        <section id="performance" class="panel trend-panel" aria-labelledby="trend-title">
          <div class="panel-heading">
            <div>
              <span class="eyebrow">VOLUME &amp; SERVICE</span>
              <h2 id="trend-title">Performance trend</h2>
              <p>Daily shipment volume and on-time delivery rate</p>
            </div>
            <span class="trend-period">{{ dateOptions.find((option) => option.value === selectedRange)?.label }}</span>
          </div>
          <PerformanceTrendChart :points="chartPoints" />
        </section>

        <section id="regions" class="panel regional-panel" aria-labelledby="regions-title">
          <div class="panel-heading panel-heading--row">
            <div>
              <span class="eyebrow">NETWORK BREAKDOWN</span>
              <h2 id="regions-title">Regional Performance</h2>
              <p>Compare service, volume, and cost across the network</p>
            </div>
            <span class="table-count">{{ regionalPerformance.length }} REGIONS</span>
          </div>
          <div class="table-scroll">
            <table class="performance-table">
              <thead>
                <tr>
                  <th scope="col">REGION</th>
                  <th scope="col">SHIPMENTS</th>
                  <th scope="col">ON-TIME</th>
                  <th scope="col">OPEN EXCEPTIONS</th>
                  <th scope="col">FREIGHT COST</th>
                  <th scope="col">STATUS</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="region in regionalPerformance" :key="region.name">
                  <th scope="row">
                    <span class="region-name">
                      <span class="region-indicator" :class="`region-indicator--${region.status.toLowerCase().replace(' ', '-')}`"></span>
                      {{ region.name }}
                    </span>
                  </th>
                  <td class="numeric-cell">{{ formatNumber(region.shipments) }}</td>
                  <td>
                    <span class="on-time-cell">
                      <span class="mini-progress">
                        <span :style="{ width: `${region.onTimeRate}%` }"></span>
                      </span>
                      {{ region.onTimeRate.toFixed(1) }}%
                    </span>
                  </td>
                  <td>
                    <span class="exception-count" :class="{ 'exception-count--high': region.openExceptions > 2 }">
                      {{ region.openExceptions }}
                    </span>
                  </td>
                  <td class="numeric-cell">{{ formatCurrency(region.freightCost) }}</td>
                  <td>
                    <span class="status-badge" :class="`status-badge--${region.status.toLowerCase().replace(' ', '-')}`">
                      <span></span>{{ region.status }}
                    </span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="table-footer">
            <span>Status thresholds: <strong>Healthy</strong> ≥95% <span class="footer-divider">·</span> <strong>Watch</strong> 93–94.9% <span class="footer-divider">·</span> <strong>At Risk</strong> &lt;93%</span>
            <span>Based on selected period</span>
          </div>
        </section>

        <section id="exceptions" class="panel exceptions-panel" aria-labelledby="exceptions-title">
          <div class="panel-heading panel-heading--row">
            <div>
              <span class="eyebrow">LEADERSHIP ATTENTION</span>
              <h2 id="exceptions-title">Open Exceptions</h2>
              <p>Unresolved operational issues in the selected period</p>
            </div>
            <div class="exception-summary">
              <span class="exception-summary__critical">{{ tableCriticalExceptions }} critical</span>
              <span>{{ tableExceptions.length }} shown</span>
            </div>
          </div>
          <div class="exception-controls" role="group" aria-label="Exception table filters">
            <label class="exception-control" for="exception-severity">
              <span>Severity</span>
              <select id="exception-severity" v-model="severityFilter">
                <option value="">All severities</option>
                <option value="Critical">Critical</option>
                <option value="High">High</option>
                <option value="Medium">Medium</option>
                <option value="Low">Low</option>
              </select>
            </label>
            <label class="exception-control" for="exception-type">
              <span>Type</span>
              <select id="exception-type" v-model="typeFilter">
                <option value="">All types</option>
                <option v-for="type in exceptionTypes" :key="type" :value="type">{{ type }}</option>
              </select>
            </label>
            <label class="exception-control exception-control--search" for="exception-search">
              <span>Search</span>
              <input
                id="exception-search"
                v-model="exceptionSearch"
                type="search"
                placeholder="ID, shipment/lane, description"
              />
            </label>
          </div>
          <div class="table-scroll">
            <table class="exceptions-table">
              <thead>
                <tr>
                  <th scope="col" :aria-sort="exceptionSortAria('id')">
                    <button class="sort-button" type="button" :aria-label="exceptionSortLabel('exception ID', 'id')" @click="sortExceptionsBy('id')">
                      EXCEPTION <span class="sort-indicator" aria-hidden="true">{{ exceptionSortIndicator('id') }}</span>
                    </button>
                  </th>
                  <th scope="col" :aria-sort="exceptionSortAria('region')">
                    <button class="sort-button" type="button" :aria-label="exceptionSortLabel('region', 'region')" @click="sortExceptionsBy('region')">
                      REGION <span class="sort-indicator" aria-hidden="true">{{ exceptionSortIndicator('region') }}</span>
                    </button>
                  </th>
                  <th scope="col" :aria-sort="exceptionSortAria('reference')">
                    <button class="sort-button" type="button" :aria-label="exceptionSortLabel('shipment or lane', 'reference')" @click="sortExceptionsBy('reference')">
                      SHIPMENT / LANE <span class="sort-indicator" aria-hidden="true">{{ exceptionSortIndicator('reference') }}</span>
                    </button>
                  </th>
                  <th scope="col" :aria-sort="exceptionSortAria('type')">
                    <button class="sort-button" type="button" :aria-label="exceptionSortLabel('type', 'type')" @click="sortExceptionsBy('type')">
                      TYPE <span class="sort-indicator" aria-hidden="true">{{ exceptionSortIndicator('type') }}</span>
                    </button>
                  </th>
                  <th scope="col" :aria-sort="exceptionSortAria('severity')">
                    <button class="sort-button" type="button" :aria-label="exceptionSortLabel('severity', 'severity')" @click="sortExceptionsBy('severity')">
                      SEVERITY <span class="sort-indicator" aria-hidden="true">{{ exceptionSortIndicator('severity') }}</span>
                    </button>
                  </th>
                  <th scope="col" :aria-sort="exceptionSortAria('age')">
                    <button class="sort-button" type="button" :aria-label="exceptionSortLabel('age', 'age')" @click="sortExceptionsBy('age')">
                      AGE <span class="sort-indicator" aria-hidden="true">{{ exceptionSortIndicator('age') }}</span>
                    </button>
                  </th>
                  <th scope="col" :aria-sort="exceptionSortAria('description')">
                    <button class="sort-button" type="button" :aria-label="exceptionSortLabel('description', 'description')" @click="sortExceptionsBy('description')">
                      DESCRIPTION <span class="sort-indicator" aria-hidden="true">{{ exceptionSortIndicator('description') }}</span>
                    </button>
                  </th>
                  <th scope="col">STATUS</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="exception in tableExceptions" :key="exception.id">
                  <th scope="row" class="exception-id">{{ exception.id }}</th>
                  <td>{{ exception.region }}</td>
                  <td class="reference-cell">{{ exception.reference }}</td>
                  <td>{{ exception.type }}</td>
                  <td>
                    <span class="severity-badge" :class="`severity-badge--${exception.severity.toLowerCase()}`">
                      <span></span>{{ exception.severity }}
                    </span>
                  </td>
                  <td class="age-cell">{{ exceptionAge(exception.daysAgo) }}</td>
                  <td class="description-cell">{{ exception.description }}</td>
                  <td><span class="open-status"><span></span>Open</span></td>
                </tr>
                <tr v-if="tableExceptions.length === 0">
                  <td colspan="8" class="empty-state">
                    <template v-if="selectedExceptions.length === 0">
                      <strong>No open exceptions in this view</strong>
                      <span>Try a longer date range or a different region.</span>
                    </template>
                    <template v-else>
                      <strong>No exceptions match these filters</strong>
                      <span>Adjust your search or clear the table filters.</span>
                      <button class="empty-state__reset" type="button" @click="clearExceptionFilters">
                        Clear table filters
                      </button>
                    </template>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="exceptions-footer">
            <span>Exceptions are shown for visibility and leadership triage.</span>
            <span>All data is fictional and synthetic</span>
          </div>
        </section>
        <footer class="page-footer">
          <span>FASTFORWARD LOGISTICS <span class="footer-divider">·</span> OPERATIONS INTELLIGENCE</span>
          <span>All figures shown are synthetic demo data</span>
        </footer>
      </div>
    </main>
  </div>
</template>
