<script setup lang="ts">
import { computed, ref } from 'vue'
import MetricCard from '../components/MetricCard.vue'
import metrics from '../data/metrics.json'

type RegionName = 'North America' | 'Europe' | 'Asia Pacific' | 'Latin America'
type Severity = 'Critical' | 'High' | 'Medium' | 'Low'
type RegionStatus = 'Healthy' | 'Watch' | 'At Risk'

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

const criticalExceptions = computed(
  () => selectedExceptions.value.filter((exception) => exception.severity === 'Critical').length,
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

interface TrendPoint {
  label: string
  shipments: number
  onTimeRate: number
  x: number
  barY: number
  barHeight: number
  lineY: number
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

  const maxShipments = Math.max(...buckets.map((bucket) => bucket.shipments), 1)
  const plot = { left: 58, top: 28, width: 650, height: 172 }
  const slot = plot.width / bucketCount

  return buckets.map((bucket, index) => ({
    ...bucket,
    x: plot.left + slot * (index + 0.5),
    barY: plot.top + plot.height * (1 - bucket.shipments / (maxShipments * 1.15)),
    barHeight: plot.height * (bucket.shipments / (maxShipments * 1.15)),
    lineY: plot.top + plot.height * (1 - (bucket.onTimeRate - 88) / 12),
  }))
})

const trendPath = computed(() =>
  chartPoints.value.map((point) => `${point.x},${point.lineY}`).join(' '),
)

const chartScale = computed(() => {
  const max = Math.max(...chartPoints.value.map((point) => point.shipments), 1)
  return [0, 0.33, 0.66, 1].map((ratio) => ({
    label: new Intl.NumberFormat('en-US', { notation: 'compact', maximumFractionDigits: 1 }).format(
      max * ratio,
    ),
    y: 28 + 172 * (1 - ratio),
  }))
})

const targetY = 28 + 172 * (1 - (95 - 88) / 12)

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

function exceptionAge(daysAgo: number) {
  return daysAgo === 0 ? 'Today' : `${daysAgo}d`
}
</script>

<template>
  <div class="app-shell">
    <aside class="sidebar">
      <a class="brand" href="#overview" aria-label="FastForward Logistics home">
        <span class="brand__mark" aria-hidden="true">
          <svg viewBox="0 0 32 32" fill="none">
            <path d="M5 8h13l-4 5h13M27 24H14l4-5H5" />
          </svg>
        </span>
        <span class="brand__name">fastforward<span>LOGISTICS</span></span>
      </a>

      <p class="sidebar__caption">WORKSPACE</p>
      <nav class="side-nav" aria-label="Dashboard sections">
        <a class="side-nav__link is-active" href="#overview">
          <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
            <rect x="3.5" y="3.5" width="7" height="7" rx="1.5" />
            <rect x="13.5" y="3.5" width="7" height="7" rx="1.5" />
            <rect x="3.5" y="13.5" width="7" height="7" rx="1.5" />
            <rect x="13.5" y="13.5" width="7" height="7" rx="1.5" />
          </svg>
          <span>Overview</span>
        </a>
        <a class="side-nav__link" href="#performance">
          <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
            <path d="M4 19V5m0 14h16" />
            <path d="m7 15 4-4 3 2 5-6" />
          </svg>
          <span>Performance</span>
        </a>
        <a class="side-nav__link" href="#regions">
          <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
            <circle cx="12" cy="12" r="8.5" />
            <path d="M3.8 12h16.4M12 3.5c2.3 2.3 3.4 5.1 3.4 8.5s-1.1 6.2-3.4 8.5C9.7 18.2 8.6 15.4 8.6 12s1.1-6.2 3.4-8.5Z" />
          </svg>
          <span>Regional performance</span>
        </a>
        <a class="side-nav__link" href="#exceptions">
          <svg viewBox="0 0 24 24" fill="none" aria-hidden="true">
            <path d="M12 3.5 21 20H3l9-16.5Z" />
            <path d="M12 9v4m0 3h.01" />
          </svg>
          <span>Open exceptions</span>
          <span class="side-nav__count">{{ selectedExceptions.length }}</span>
        </a>
      </nav>

      <div class="sidebar__bottom">
        <div class="network-status">
          <span class="network-status__dot"></span>
          <div>
            <strong>Network status</strong>
            <span>Live operations view</span>
          </div>
        </div>
        <div class="sidebar__version">FASTFORWARD INTERNAL <span>·</span> v1.0</div>
      </div>
    </aside>

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

        <section id="performance" class="panel trend-panel" aria-labelledby="trend-title">
          <div class="panel-heading">
            <div>
              <span class="eyebrow">VOLUME &amp; SERVICE</span>
              <h2 id="trend-title">Performance trend</h2>
              <p>Daily shipment volume and on-time delivery rate</p>
            </div>
            <span class="trend-period">{{ dateOptions.find((option) => option.value === selectedRange)?.label }}</span>
          </div>
          <div class="chart-legend">
            <span><i class="legend-swatch legend-swatch--bar"></i> Shipments</span>
            <span><i class="legend-swatch legend-swatch--line"></i> On-time delivery</span>
            <span><i class="legend-swatch legend-swatch--target"></i> 95% target</span>
          </div>
          <div class="chart-wrap">
            <svg
              class="trend-chart"
              viewBox="0 0 760 260"
              role="img"
              :aria-label="`Shipment volume and on-time delivery trend over the last ${selectedDays} days`"
            >
              <g v-for="(scale, index) in chartScale" :key="scale.label + index">
                <line x1="58" :y1="scale.y" x2="708" :y2="scale.y" class="chart-grid" />
                <text x="48" :y="scale.y + 4" text-anchor="end" class="chart-axis-label">
                  {{ scale.label }}
                </text>
                <text
                  x="718"
                  :y="scale.y + 4"
                  text-anchor="start"
                  class="chart-axis-label chart-axis-label--right"
                >
                  {{ 88 + index * 4 }}%
                </text>
              </g>
              <line x1="58" :y1="targetY" x2="708" :y2="targetY" class="chart-target-line" />
              <g v-for="(point, index) in chartPoints" :key="point.label + index">
                <rect
                  :x="point.x - Math.min(20, 280 / chartPoints.length)"
                  :y="point.barY"
                  :width="Math.min(40, 560 / chartPoints.length)"
                  :height="point.barHeight"
                  rx="4"
                  class="chart-bar"
                >
                  <title>{{ point.label }}: {{ formatNumber(point.shipments) }} shipments</title>
                </rect>
                <text
                  :x="point.x"
                  y="226"
                  text-anchor="middle"
                  class="chart-date-label"
                >
                  {{ point.label }}
                </text>
              </g>
              <polyline :points="trendPath" class="chart-line" />
              <circle
                v-for="(point, index) in chartPoints"
                :key="`point-${point.label}-${index}`"
                :cx="point.x"
                :cy="point.lineY"
                r="3.5"
                class="chart-marker"
              >
                <title>{{ point.label }}: {{ point.onTimeRate.toFixed(1) }}% on time</title>
              </circle>
            </svg>
          </div>
          <div class="chart-axis-caption">
            <span>SHIPMENTS</span>
            <span>ON-TIME DELIVERY</span>
          </div>
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
              <span class="exception-summary__critical">{{ criticalExceptions }} critical</span>
              <span>{{ selectedExceptions.length }} total</span>
            </div>
          </div>
          <div class="table-scroll">
            <table class="exceptions-table">
              <thead>
                <tr>
                  <th scope="col">EXCEPTION</th>
                  <th scope="col">REGION</th>
                  <th scope="col">SHIPMENT / LANE</th>
                  <th scope="col">TYPE</th>
                  <th scope="col">SEVERITY</th>
                  <th scope="col">AGE</th>
                  <th scope="col">DESCRIPTION</th>
                  <th scope="col">STATUS</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="exception in selectedExceptions" :key="exception.id">
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
                <tr v-if="selectedExceptions.length === 0">
                  <td colspan="8" class="empty-state">
                    <strong>No open exceptions in this view</strong>
                    <span>Try a longer date range or a different region.</span>
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
