<script setup lang="ts">
import { computed } from 'vue'
import {
  BarController,
  BarElement,
  CategoryScale,
  Chart as ChartJS,
  Legend,
  LinearScale,
  LineController,
  LineElement,
  PointElement,
  Tooltip,
  type ChartData,
  type ChartOptions,
} from 'chart.js'
import { Chart } from 'vue-chartjs'

interface TrendPoint {
  label: string
  shipments: number
  onTimeRate: number
}

const props = defineProps<{
  points: TrendPoint[]
}>()

ChartJS.register(
  BarController,
  BarElement,
  CategoryScale,
  Legend,
  LinearScale,
  LineController,
  LineElement,
  PointElement,
  Tooltip,
)

const chartData = computed<ChartData<'bar' | 'line', number[], string>>(() => ({
  labels: props.points.map((point) => point.label),
  datasets: [
    // Chart.js draws datasets in reverse order, so lower values overlay them.
    {
      type: 'bar',
      label: 'Shipments',
      data: props.points.map((point) => point.shipments),
      yAxisID: 'yShipments',
      backgroundColor: 'rgba(202, 217, 232, 0.48)',
      hoverBackgroundColor: 'rgba(167, 191, 213, 0.42)',
      borderRadius: 4,
      categoryPercentage: 0.72,
      barPercentage: 0.82,
      order: 2,
    },
    {
      type: 'line',
      label: 'On-time delivery',
      data: props.points.map((point) => point.onTimeRate),
      yAxisID: 'yOnTime',
      borderColor: '#199b8d',
      backgroundColor: '#199b8d',
      borderWidth: 3,
      pointRadius: 4,
      pointHoverRadius: 5,
      pointBackgroundColor: '#fff',
      pointBorderColor: '#199b8d',
      pointBorderWidth: 2,
      pointHoverBackgroundColor: '#fff',
      pointHoverBorderColor: '#11796f',
      pointHoverBorderWidth: 2.5,
      pointStyle: 'circle',
      tension: 0.28,
      order: 0,
    },
    {
      type: 'line',
      label: '95% target',
      data: props.points.map(() => 95),
      yAxisID: 'yOnTime',
      borderColor: '#d6a168',
      borderDash: [5, 5],
      borderWidth: 1,
      pointStyle: 'line',
      pointRadius: 0,
      pointHoverRadius: 0,
      order: 1,
    },
  ],
}))

const chartOptions: ChartOptions<'bar' | 'line'> = {
  responsive: true,
  maintainAspectRatio: false,
  interaction: {
    intersect: false,
    mode: 'index',
  },
  plugins: {
    legend: {
      align: 'start',
      labels: {
        boxWidth: 14,
        boxHeight: 8,
        color: '#667487',
        padding: 18,
        usePointStyle: true,
        font: {
          family: 'Inter, -apple-system, BlinkMacSystemFont, Segoe UI, sans-serif',
          size: 10,
        },
      },
    },
    tooltip: {
      backgroundColor: '#17283e',
      padding: 10,
      titleFont: { size: 11 },
      bodyFont: { size: 10 },
      callbacks: {
        label(context) {
          if (context.dataset.label === '95% target') return ' Target: 95%'
          if (context.dataset.yAxisID === 'yShipments') {
            return ` Shipments: ${Number(context.parsed.y).toLocaleString('en-US')}`
          }
          return ` On-time delivery: ${Number(context.parsed.y).toFixed(1)}%`
        },
      },
    },
  },
  scales: {
    x: {
      grid: {
        display: false,
      },
      ticks: {
        color: '#8793a2',
        maxRotation: 0,
        autoSkip: true,
        maxTicksLimit: 12,
        font: {
          family: 'Inter, -apple-system, BlinkMacSystemFont, Segoe UI, sans-serif',
          size: 9,
        },
      },
    },
    yShipments: {
      type: 'linear',
      position: 'left',
      beginAtZero: true,
      title: {
        display: true,
        text: 'Shipments',
        color: '#647286',
        font: { size: 10, weight: 600 },
      },
      grid: {
        color: '#edf0f4',
      },
      ticks: {
        color: '#8793a2',
        precision: 0,
        font: { size: 9 },
        callback(value) {
          return new Intl.NumberFormat('en-US', {
            notation: 'compact',
            maximumFractionDigits: 1,
          }).format(Number(value))
        },
      },
    },
    yOnTime: {
      type: 'linear',
      position: 'right',
      min: 88,
      max: 100,
      title: {
        display: true,
        text: 'On-time delivery (%)',
        color: '#477a78',
        font: { size: 10, weight: 600 },
      },
      grid: {
        drawOnChartArea: false,
      },
      ticks: {
        stepSize: 2,
        color: '#718b88',
        font: { size: 9 },
        callback(value) {
          return `${value}%`
        },
      },
    },
  },
}
</script>

<template>
  <div class="trend-chart-container">
    <Chart
      type="bar"
      :data="chartData"
      :options="chartOptions"
      aria-label="Shipment volume and on-time delivery over time, including the 95% target"
      role="img"
    />
  </div>
</template>
