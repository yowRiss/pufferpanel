<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useI18n } from 'vue-i18n'
import Chart, { _adapters, Tooltip } from 'chart.js/auto'
import 'chartjs-adapter-date-fns'
import Query from './Query.vue'

const fromCss = (el, prop) => {
  return getComputedStyle(el).getPropertyValue(prop).trim()
}

const defaultFamily = "'Helvetica Neue', 'Helvetica', 'Arial', sans-serif"

const props = defineProps({
  server: { type: Object, required: true }
})

const { t, locale } = useI18n()

const numFormat = new Intl.NumberFormat(
  [locale.value.replace('_', '-'), 'en'],
  { maximumFractionDigits: 2 }
)

const formatCpu = (value) => {
  return numFormat.format(value) + ' %'
}

const formatMemory = (value) => {
  if (!value) return numFormat.format(0) + ' B'
  if (value < Math.pow(2, 10)) return numFormat.format(value) + ' B'
  if (value < Math.pow(2, 20)) return numFormat.format(value / Math.pow(2, 10)) + ' KiB'
  if (value < Math.pow(2, 30)) return numFormat.format(value / Math.pow(2, 20)) + ' MiB'
  if (value < Math.pow(2, 40)) return numFormat.format(value / Math.pow(2, 30)) + ' GiB'
  return numFormat.format(value / Math.pow(2, 40)) + ' TiB'
}

const formatNetworkRate = (value) => {
  if (value === undefined || value === null || isNaN(value) || value <= 0) return numFormat.format(0) + ' B/s'
  if (value < Math.pow(2, 10)) return numFormat.format(value) + ' B/s'
  if (value < Math.pow(2, 20)) return numFormat.format(value / Math.pow(2, 10)) + ' KiB/s'
  if (value < Math.pow(2, 30)) return numFormat.format(value / Math.pow(2, 20)) + ' MiB/s'
  if (value < Math.pow(2, 40)) return numFormat.format(value / Math.pow(2, 30)) + ' GiB/s'
  return numFormat.format(value / Math.pow(2, 40)) + ' TiB/s'
}

const formatBytes = (value) => {
  if (value === undefined || value === null || isNaN(value) || value <= 0) return numFormat.format(0) + ' B'
  if (value < Math.pow(2, 10)) return numFormat.format(value) + ' B'
  if (value < Math.pow(2, 20)) return numFormat.format(value / Math.pow(2, 10)) + ' KiB'
  if (value < Math.pow(2, 30)) return numFormat.format(value / Math.pow(2, 20)) + ' MiB'
  if (value < Math.pow(2, 40)) return numFormat.format(value / Math.pow(2, 30)) + ' GiB'
  return numFormat.format(value / Math.pow(2, 40)) + ' TiB'
}

const intl = new Intl.DateTimeFormat(
  [locale.value.replace('_', '-'), 'en'],
  { hour: 'numeric', minute: 'numeric', second: 'numeric' }
)
_adapters._date.prototype.format = (time) => {
  return intl.format(time)
}

Tooltip.positioners.cursor = (_, eventPosition) => {
  return { x: eventPosition.x, y: eventPosition.y }
}

const cpuChartEl = ref(null)
const memoryChartEl = ref(null)
const networkChartEl = ref(null)

let cpuChart = null
let memoryChart = null
let networkChart = null

const cpu = []
const memory = []
const jvmHeapUsed = []
const jvmHeapAlloc = []
const jvmMetaUsed = []
const jvmMetaAlloc = []

const networkRx = []
const networkTx = []

// Telemetry & Analytics States
const activeTimeframe = ref('live') // 'live' | '1h' | '24h' | '7d' | '30d'
const netMode = ref('server') // 'server' or 'host'
const netStats = ref(null)
const analyticsData = ref(null)
let netInterval = null
let analyticsInterval = null

const timeframeLabels = {
  'live': 'Real-Time (60s)',
  '1h': 'Past 1 Hour',
  '24h': 'Past 24 Hours',
  '7d': 'Past 7 Days',
  '30d': 'Past 30 Days'
}

const currentNetStats = computed(() => {
  if (!netStats.value) return null
  return netMode.value === 'host' ? netStats.value.host : netStats.value.server
})

const combinedTotalFormatted = computed(() => {
  if (!currentNetStats.value) return '0 B'
  const total = (currentNetStats.value.rx_bytes || 0) + (currentNetStats.value.tx_bytes || 0)
  return formatBytes(total)
})

const combinedRateFormatted = computed(() => {
  if (!currentNetStats.value) return '0 B/s'
  const totalRate = (currentNetStats.value.rx_rate || 0) + (currentNetStats.value.tx_rate || 0)
  return formatNetworkRate(totalRate)
})

const currentAnalyticsSummary = computed(() => {
  if (!analyticsData.value || activeTimeframe.value === 'live') return null
  return analyticsData.value.ranges?.[activeTimeframe.value]?.summary || null
})

function setNetMode(mode) {
  if (netMode.value === mode) return
  netMode.value = mode
  networkRx.length = 0
  networkTx.length = 0
  if (activeTimeframe.value === 'live' && networkChart) {
    networkChart.update('none')
  }
  fetchNetworkStats()
}

async function setTimeframe(tf) {
  if (activeTimeframe.value === tf) return
  activeTimeframe.value = tf

  if (tf === 'live') {
    applyLiveDatasets()
  } else {
    if (!analyticsData.value) {
      await fetchAnalyticsData()
    }
    applyHistoricalDatasets(tf)
  }
}

function applyLiveDatasets() {
  if (!cpuChart || !memoryChart || !networkChart) return
  const now = new Date().getTime()

  // CPU
  cpuChart.data.datasets[0].data = cpu
  cpuChart.options.scales.x.min = now - 60000
  delete cpuChart.options.scales.x.max
  cpuChart.update()

  // Memory
  memoryChart.data.datasets.forEach(ds => {
    if (ds.label === t('servers.Memory')) {
      ds.data = memory
      ds.hidden = false
    } else if (ds.label === t('servers.JvmHeapUsed')) {
      ds.data = jvmHeapUsed
      ds.hidden = jvmHeapUsed.length === 0
    } else if (ds.label === t('servers.JvmHeapAlloc')) {
      ds.data = jvmHeapAlloc
      ds.hidden = jvmHeapAlloc.length === 0
    } else if (ds.label === t('servers.JvmMetaUsed')) {
      ds.data = jvmMetaUsed
      ds.hidden = jvmMetaUsed.length === 0
    } else if (ds.label === t('servers.JvmMetaAlloc')) {
      ds.data = jvmMetaAlloc
      ds.hidden = jvmMetaAlloc.length === 0
    }
  })
  memoryChart.options.scales.x.min = now - 60000
  delete memoryChart.options.scales.x.max
  memoryChart.update()

  // Network
  networkChart.data.datasets[0].data = networkRx
  networkChart.data.datasets[1].data = networkTx
  networkChart.options.scales.x.min = now - 60000
  delete networkChart.options.scales.x.max
  networkChart.update()
}

function applyHistoricalDatasets(tf) {
  if (!analyticsData.value || !analyticsData.value.ranges?.[tf]) return
  const range = analyticsData.value.ranges[tf]
  const series = range.series || []
  if (series.length === 0) return

  const startT = series[0].t
  const endT = series[series.length - 1].t

  if (cpuChart) {
    cpuChart.data.datasets[0].data = series.map(p => ({ x: p.t, y: p.cpu }))
    cpuChart.options.scales.x.min = startT
    cpuChart.options.scales.x.max = endT
    cpuChart.update()
  }

  if (memoryChart) {
    memoryChart.data.datasets.forEach(ds => {
      if (ds.label === t('servers.Memory')) {
        ds.data = series.map(p => ({ x: p.t, y: p.mem }))
        ds.hidden = false
      } else {
        ds.hidden = true
      }
    })
    memoryChart.options.scales.x.min = startT
    memoryChart.options.scales.x.max = endT
    memoryChart.update()
  }

  if (networkChart) {
    networkChart.data.datasets[0].data = series.map(p => ({ x: p.t, y: p.rx }))
    networkChart.data.datasets[1].data = series.map(p => ({ x: p.t, y: p.tx }))
    networkChart.options.scales.x.min = startT
    networkChart.options.scales.x.max = endT
    networkChart.update()
  }
}

async function fetchNetworkStats() {
  try {
    const res = await fetch(`/js/network-stats.json?t=${Date.now()}`)
    if (!res.ok) return
    const data = await res.json()
    netStats.value = data

    const x = data.timestamp || Date.now()
    const target = netMode.value === 'host' ? data.host : data.server
    if (target) {
      networkRx.push({ x, y: target.rx_rate || 0 })
      networkTx.push({ x, y: target.tx_rate || 0 })

      while (networkRx.length > 60) networkRx.shift()
      while (networkTx.length > 60) networkTx.shift()

      if (activeTimeframe.value === 'live' && networkChart) {
        networkChart.options.scales.x.min = x - (60 * 1000)
        networkChart.update('none')
      }
    }
  } catch (e) {
    // Ignore transient network errors
  }
}

async function fetchAnalyticsData() {
  try {
    const res = await fetch(`/js/analytics-history.json?t=${Date.now()}`)
    if (!res.ok) return
    const data = await res.json()
    analyticsData.value = data
    if (activeTimeframe.value !== 'live') {
      applyHistoricalDatasets(activeTimeframe.value)
    }
  } catch (e) {
    // Ignore transient network errors
  }
}

function exportAnalyticsData() {
  if (!analyticsData.value) return
  const currentRange = activeTimeframe.value === 'live' ? '24h' : activeTimeframe.value
  const dataToExport = {
    exported_at: new Date().toISOString(),
    server_id: props.server.id || '35ca4939',
    timeframe: currentRange,
    summary: analyticsData.value.ranges?.[currentRange]?.summary,
    series: analyticsData.value.ranges?.[currentRange]?.series
  }
  const blob = new Blob([JSON.stringify(dataToExport, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `server-analytics-${props.server.id || 'server'}-${currentRange}-${Date.now()}.json`
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

function addData(d) {
  const x = new Date().getTime()

  cpu.push({ x, y: d.cpu })
  memory.push({ x, y: d.memory })

  if (d.jvm) {
    jvmHeapUsed.push({ x, y: d.jvm.heapUsed })
    jvmHeapAlloc.push({ x, y: d.jvm.heapTotal - d.jvm.heapUsed })
    jvmMetaUsed.push({ x, y: d.jvm.metaspaceUsed })
    jvmMetaAlloc.push({ x, y: d.jvm.metaspaceTotal - d.jvm.metaspaceUsed })
  }

  for (let graph of [cpu, memory, jvmHeapUsed, jvmHeapAlloc, jvmMetaUsed, jvmMetaAlloc]) {
    while (graph.length > 60) {
      graph.shift()
    }
  }

  if (activeTimeframe.value === 'live') {
    for (let chart of [cpuChart, memoryChart]) {
      if (chart) {
        chart.options.scales.x.min = x - (60 * 1000)
        chart.update('none')
      }
    }
  }
}

const chartOptions = (mode) => {
  const options = {
    responsive: true,
    aspectRatio: (ctx) => {
      if (ctx.chart.canvas) {
        return parseFloat(fromCss(ctx.chart.canvas.parentElement, 'aspect-ratio')) || 2
      } else return 2
    },
    parsing: false,
    locale: locale.value.split('_')[0] || 'en',
    interaction: {
      mode: 'nearest',
      axis: 'x',
      intersect: false
    },
    animations: {
      y: {
        duration: 0
      }
    },
    plugins: {
      tooltip: {
        position: 'cursor',
        usePointStyle: true,
        callbacks: {
          label: (ctx) => {
            return ' ' + ctx.dataset.label + ': ' + ctx.chart.scales[ctx.dataset.yAxisID].options.ticks.callback(ctx.parsed.y)
          },
          labelPointStyle: () => 'circle'
        },
        itemSort: (a, b) => {
          return a.datasetIndex < b.datasetIndex
        },
        multiKeyBackground: (ctx) => fromCss(ctx.chart.canvas, 'background-color') || '#fff',
        padding: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-padding') || 6,
        backgroundColor: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-background-color') || 'rgba(0, 0, 0, 0.8)',
        cornerRadius: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-corner-radius') || 6,
        borderColor: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-border-color') || 'rgba(0, 0, 0, 0)',
        borderWidth: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-border-width') || 0,
        titleAlign: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-title-align') || 'left',
        titleColor: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-font-color') || '#fff',
        bodyColor: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-font-color') || '#fff',
        bodySpacing: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-tooltip-body-spacing')) || 2,
        titleFont: {
          family: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-title-font-family') || fromCss(ctx.chart.canvas, '--chartjs-font-family') || defaultFamily,
          size: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-tooltip-title-font-size') || fromCss(ctx.chart.canvas, '--chartjs-font-size')) || 12,
          weight: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-title-font-weight') || 'bold'
        },
        bodyFont: {
          family: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-body-font-family') || fromCss(ctx.chart.canvas, '--chartjs-font-family') || defaultFamily,
          size: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-tooltip-body-font-size') || fromCss(ctx.chart.canvas, '--chartjs-font-size')) || 12,
          weight: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-tooltip-body-font-weight') || fromCss(ctx.chart.canvas, '--chartjs-font-weight')
        }
      },
      legend: {
        display: false
      },
      title: {
        display: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-title-display') == 'true',
        align: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-title-align') || 'center',
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-title-font-color') || fromCss(ctx.chart.canvas, '--chartjs-color') || fromCss(ctx.chart.canvas, 'color'),
        position: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-title-position') || 'top',
        font: {
          family: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-title-font-family') || fromCss(ctx.chart.canvas, '--chartjs-font-family') || defaultFamily,
          size: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-title-font-size') || fromCss(ctx.chart.canvas, '--chartjs-font-size')) || 12,
          weight: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-title-font-weight') || 'bold'
        }
      }
    },
    scales: {
      x: {
        type: 'timeseries',
        min: new Date().getTime() - (60 * 1000),
        grid: {
          color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-grid-color')
        },
        ticks: {
          min: 12,
          source: 'data',
          color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-color') || fromCss(ctx.chart.canvas, '--chartjs-color') || fromCss(ctx.chart.canvas, 'color'),
          font: {
            family: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-family') || fromCss(ctx.chart.canvas, '--chartjs-font-family') || defaultFamily,
            size: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-axis-font-size') || fromCss(ctx.chart.canvas, '--chartjs-font-size')) || 12,
            weight: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-weight') || fromCss(ctx.chart.canvas, '--chartjs-font-weight')
          }
        }
      }
    },
    elements: {
      line: {
        tension: 0.3
      },
      point: {
        pointStyle: false,
        hoverRadius: 20
      }
    }
  }

  if (mode === 'memory') {
    options.plugins.title.text = t('servers.Memory')
    options.scales.memory = {
      type: 'linear',
      position: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-y-position') || 'left',
      min: 0,
      suggestedMax: 1024 * 1024,
      grid: {
        display: true,
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-grid-color')
      },
      ticks: {
        callback: formatMemory,
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-color') || fromCss(ctx.chart.canvas, '--chartjs-color') || fromCss(ctx.chart.canvas, 'color'),
        font: {
          family: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-family') || fromCss(ctx.chart.canvas, '--chartjs-font-family') || defaultFamily,
          size: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-axis-font-size') || fromCss(ctx.chart.canvas, '--chartjs-font-size')) || 12,
          weight: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-weight') || fromCss(ctx.chart.canvas, '--chartjs-font-weight')
        }
      }
    }
  }

  if (mode === 'cpu') {
    options.plugins.title.text = t('servers.CPU')
    options.scales.cpu = {
      type: 'linear',
      position: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-y-position') || 'left',
      min: 0,
      suggestedMax: 100,
      grid: {
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-grid-color')
      },
      ticks: {
        callback: formatCpu,
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-color') || fromCss(ctx.chart.canvas, '--chartjs-color') || fromCss(ctx.chart.canvas, 'color'),
        font: {
          family: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-family') || fromCss(ctx.chart.canvas, '--chartjs-font-family') || defaultFamily,
          size: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-axis-font-size') || fromCss(ctx.chart.canvas, '--chartjs-font-size')) || 12,
          weight: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-weight') || fromCss(ctx.chart.canvas, '--chartjs-font-weight')
        }
      }
    }
  }

  if (mode === 'network') {
    options.plugins.title.text = t('servers.Network') || 'Network Traffic'
    options.plugins.legend = {
      display: true,
      position: 'top',
      align: 'end',
      labels: {
        usePointStyle: true,
        boxWidth: 8,
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-color') || fromCss(ctx.chart.canvas, '--chartjs-color') || fromCss(ctx.chart.canvas, 'color'),
        font: {
          family: defaultFamily,
          size: 11
        }
      }
    }
    options.scales.network = {
      type: 'linear',
      position: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-y-position') || 'left',
      min: 0,
      suggestedMax: 1024 * 5,
      grid: {
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-grid-color')
      },
      ticks: {
        callback: formatNetworkRate,
        color: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-color') || fromCss(ctx.chart.canvas, '--chartjs-color') || fromCss(ctx.chart.canvas, 'color'),
        font: {
          family: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-family') || fromCss(ctx.chart.canvas, '--chartjs-font-family') || defaultFamily,
          size: (ctx) => parseInt(fromCss(ctx.chart.canvas, '--chartjs-axis-font-size') || fromCss(ctx.chart.canvas, '--chartjs-font-size')) || 12,
          weight: (ctx) => fromCss(ctx.chart.canvas, '--chartjs-axis-font-weight') || fromCss(ctx.chart.canvas, '--chartjs-font-weight')
        }
      }
    }
  }

  return options
}

let task = null
let stopListener = null

onMounted(() => {
  cpuChart = new Chart(cpuChartEl.value, {
    type: 'line',
    options: chartOptions('cpu'),
    data: {
      datasets: [
        {
          label: t('servers.CPU'),
          yAxisID: 'cpu',
          borderColor: fromCss(cpuChartEl.value, '--chartjs-series-line-cpu'),
          backgroundColor: fromCss(cpuChartEl.value, '--chartjs-series-fill-cpu'),
          data: cpu
        }
      ]
    }
  })

  memoryChart = new Chart(memoryChartEl.value, {
    type: 'line',
    options: chartOptions('memory'),
    data: {
      datasets: [
        {
          label: t('servers.JvmMetaUsed'),
          yAxisID: 'memory',
          fill: 'origin',
          borderColor: fromCss(memoryChartEl.value, '--chartjs-series-line-jvm-metaspace-used'),
          backgroundColor: fromCss(memoryChartEl.value, '--chartjs-series-fill-jvm-metaspace-used'),
          stack: 'jvmMemory',
          hidden: true,
          data: jvmMetaUsed
        },
        {
          label: t('servers.JvmMetaAlloc'),
          yAxisID: 'memory',
          fill: '-1',
          borderColor: fromCss(memoryChartEl.value, '--chartjs-series-line-jvm-metaspace-allocated'),
          backgroundColor: fromCss(memoryChartEl.value, '--chartjs-series-fill-jvm-metaspace-allocated'),
          stack: 'jvmMemory',
          hidden: true,
          data: jvmMetaAlloc
        },
        {
          label: t('servers.JvmHeapUsed'),
          yAxisID: 'memory',
          fill: '-1',
          borderColor: fromCss(memoryChartEl.value, '--chartjs-series-line-jvm-heapspace-used'),
          backgroundColor: fromCss(memoryChartEl.value, '--chartjs-series-fill-jvm-heapspace-used'),
          stack: 'jvmMemory',
          hidden: true,
          data: jvmHeapUsed
        },
        {
          label: t('servers.JvmHeapAlloc'),
          yAxisID: 'memory',
          fill: '-1',
          borderColor: fromCss(memoryChartEl.value, '--chartjs-series-line-jvm-heapspace-allocated'),
          backgroundColor: fromCss(memoryChartEl.value, '--chartjs-series-fill-jvm-heapspace-allocated'),
          stack: 'jvmMemory',
          hidden: true,
          data: jvmHeapAlloc
        },
        {
          label: t('servers.Memory'),
          yAxisID: 'memory',
          borderColor: fromCss(memoryChartEl.value, '--chartjs-series-line-memory'),
          backgroundColor: fromCss(memoryChartEl.value, '--chartjs-series-fill-memory'),
          data: memory
        }
      ]
    }
  })

  networkChart = new Chart(networkChartEl.value, {
    type: 'line',
    options: chartOptions('network'),
    data: {
      datasets: [
        {
          label: 'Inbound (Rx)',
          yAxisID: 'network',
          fill: 'origin',
          borderColor: fromCss(networkChartEl.value, '--chartjs-series-line-network-rx') || '#10b981',
          backgroundColor: fromCss(networkChartEl.value, '--chartjs-series-fill-network-rx') || 'rgba(16, 185, 129, 0.15)',
          data: networkRx
        },
        {
          label: 'Outbound (Tx)',
          yAxisID: 'network',
          fill: 'origin',
          borderColor: fromCss(networkChartEl.value, '--chartjs-series-line-network-tx') || '#3b82f6',
          backgroundColor: fromCss(networkChartEl.value, '--chartjs-series-fill-network-tx') || 'rgba(59, 130, 246, 0.15)',
          data: networkTx
        }
      ]
    }
  })

  fetchNetworkStats()
  fetchAnalyticsData()
  netInterval = setInterval(fetchNetworkStats, 1000)
  analyticsInterval = setInterval(fetchAnalyticsData, 15000)

  stopListener = props.server.on('stat', addData)

  task = props.server.startTask(async () => {
    if (props.server.needsPolling() && props.server.hasScope('server.stats')) {
      addData(await props.server.getStats())
    }
  }, 5000)
})

onUnmounted(() => {
  if (cpuChart) cpuChart.destroy()
  if (memoryChart) memoryChart.destroy()
  if (networkChart) networkChart.destroy()
  if (task) props.server.stopTask(task)
  if (stopListener) stopListener()
  if (netInterval) clearInterval(netInterval)
  if (analyticsInterval) clearInterval(analyticsInterval)
})
</script>

<template>
  <Query :server="server" />

  <div class="analytics-container">
    <!-- Top Bar: Timeframe Selector & Mode Switches -->
    <div class="analytics-toolbar">
      <div class="toolbar-left">
        <div class="timeframe-group">
          <button
            type="button"
            class="timeframe-btn"
            :class="{ active: activeTimeframe === 'live' }"
            @click="setTimeframe('live')"
          >
            <span class="live-dot" :class="{ pulsing: activeTimeframe === 'live' }"></span>
            Real-Time
          </button>
          <button
            type="button"
            class="timeframe-btn"
            :class="{ active: activeTimeframe === '1h' }"
            @click="setTimeframe('1h')"
          >
            1 Hour
          </button>
          <button
            type="button"
            class="timeframe-btn"
            :class="{ active: activeTimeframe === '24h' }"
            @click="setTimeframe('24h')"
          >
            24 Hours
          </button>
          <button
            type="button"
            class="timeframe-btn"
            :class="{ active: activeTimeframe === '7d' }"
            @click="setTimeframe('7d')"
          >
            7 Days
          </button>
          <button
            type="button"
            class="timeframe-btn"
            :class="{ active: activeTimeframe === '30d' }"
            @click="setTimeframe('30d')"
          >
            30 Days
          </button>
        </div>
      </div>

      <div class="toolbar-right">
        <!-- Live-only host/server toggle -->
        <div v-if="activeTimeframe === 'live'" class="net-mode-group">
          <button
            type="button"
            class="net-mode-btn"
            :class="{ selected: netMode === 'server' }"
            @click="setNetMode('server')"
          >
            Game Port 25565
          </button>
          <button
            type="button"
            class="net-mode-btn"
            :class="{ selected: netMode === 'host' }"
            @click="setNetMode('host')"
          >
            Host Interface (eno1)
          </button>
        </div>

        <!-- Export button -->
        <button
          type="button"
          class="export-btn"
          title="Export Analytics Data (JSON)"
          @click="exportAnalyticsData"
        >
          <span class="export-icon">⤓</span> Export
        </button>
      </div>
    </div>

    <!-- Mode Banner / Heading -->
    <div class="analytics-subheading">
      <div class="subheading-info">
        <span class="section-title">
          {{ activeTimeframe === 'live' ? 'Live Telemetry Stream' : `Historical Analytics (${timeframeLabels[activeTimeframe]})` }}
        </span>
        <span class="subheading-desc">
          {{ activeTimeframe === 'live' 
             ? 'Real-time telemetry updated every 1s' 
             : `Aggregated historical metrics from persistent SQLite storage` }}
        </span>
      </div>
      <div v-if="activeTimeframe !== 'live' && analyticsData" class="data-badge">
        {{ currentAnalyticsSummary?.samples || 0 }} samples recorded
      </div>
    </div>

    <!-- Dynamic Metric KPI Cards -->
    <!-- Case 1: Real-Time Cards -->
    <div v-if="activeTimeframe === 'live'" class="kpi-cards-grid">
      <!-- Inbound Card -->
      <div class="kpi-card rx-card">
        <div class="kpi-top">
          <span class="kpi-icon rx-icon">↓</span>
          <span class="kpi-label">Inbound (Rx)</span>
        </div>
        <div class="kpi-main-val">
          {{ currentNetStats?.rx_rate_formatted || '0 B/s' }}
        </div>
        <div class="kpi-footer">
          <span class="footer-label">Total Received</span>
          <span class="footer-val">{{ currentNetStats?.rx_total_formatted || '0 B' }}</span>
        </div>
      </div>

      <!-- Outbound Card -->
      <div class="kpi-card tx-card">
        <div class="kpi-top">
          <span class="kpi-icon tx-icon">↑</span>
          <span class="kpi-label">Outbound (Tx)</span>
        </div>
        <div class="kpi-main-val">
          {{ currentNetStats?.tx_rate_formatted || '0 B/s' }}
        </div>
        <div class="kpi-footer">
          <span class="footer-label">Total Sent</span>
          <span class="footer-val">{{ currentNetStats?.tx_total_formatted || '0 B' }}</span>
        </div>
      </div>

      <!-- Connections / Flow Card -->
      <div class="kpi-card conn-card">
        <div class="kpi-top">
          <span class="kpi-icon conn-icon">⇄</span>
          <span class="kpi-label">{{ netMode === 'server' ? 'Active Connections' : 'Combined Speed' }}</span>
        </div>
        <div class="kpi-main-val">
          <template v-if="netMode === 'server'">
            {{ currentNetStats?.connections ?? 0 }} <span class="kpi-unit">players</span>
          </template>
          <template v-else>
            {{ combinedRateFormatted }}
          </template>
        </div>
        <div class="kpi-footer">
          <span class="footer-label">{{ netMode === 'server' ? 'Protocol' : 'Duplex' }}</span>
          <span class="footer-val">{{ netMode === 'server' ? 'TCP / Minecraft' : 'Full Duplex' }}</span>
        </div>
      </div>

      <!-- Combined Total Card -->
      <div class="kpi-card total-card">
        <div class="kpi-top">
          <span class="kpi-icon total-icon">📊</span>
          <span class="kpi-label">Total Bandwidth</span>
        </div>
        <div class="kpi-main-val">
          {{ combinedTotalFormatted }}
        </div>
        <div class="kpi-footer">
          <span class="footer-label">Combined</span>
          <span class="footer-val">Rx + Tx</span>
        </div>
      </div>
    </div>

    <!-- Case 2: Historical Analytics Cards -->
    <div v-else class="kpi-cards-grid analytics-mode-grid">
      <!-- Total Bandwidth in Period -->
      <div class="kpi-card total-card">
        <div class="kpi-top">
          <span class="kpi-icon total-icon">🌐</span>
          <span class="kpi-label">Period Bandwidth</span>
        </div>
        <div class="kpi-main-val">
          {{ formatBytes(currentAnalyticsSummary?.total_bytes) }}
        </div>
        <div class="kpi-footer">
          <span class="footer-label">In: {{ formatBytes(currentAnalyticsSummary?.total_rx_bytes) }}</span>
          <span class="footer-val">Out: {{ formatBytes(currentAnalyticsSummary?.total_tx_bytes) }}</span>
        </div>
      </div>

      <!-- Transfer Speeds (Avg & Peak) -->
      <div class="kpi-card tx-card">
        <div class="kpi-top">
          <span class="kpi-icon tx-icon">⚡</span>
          <span class="kpi-label">Network Rates</span>
        </div>
        <div class="kpi-main-val">
          {{ formatNetworkRate((currentAnalyticsSummary?.avg_rx_rate || 0) + (currentAnalyticsSummary?.avg_tx_rate || 0)) }}
        </div>
        <div class="kpi-footer">
          <span class="footer-label">Avg Combined</span>
          <span class="footer-val">Peak: {{ formatNetworkRate(currentAnalyticsSummary?.peak_tx_rate) }}</span>
        </div>
      </div>

      <!-- CPU Utilization (Avg & Peak) -->
      <div class="kpi-card cpu-card">
        <div class="kpi-top">
          <span class="kpi-icon cpu-icon">⚙</span>
          <span class="kpi-label">CPU Utilization</span>
        </div>
        <div class="kpi-main-val">
          {{ currentAnalyticsSummary?.avg_cpu ?? 0 }} %
        </div>
        <div class="kpi-footer">
          <span class="footer-label">Avg Load</span>
          <span class="footer-val">Peak: {{ currentAnalyticsSummary?.peak_cpu ?? 0 }} %</span>
        </div>
      </div>

      <!-- Memory Allocation (Avg & Peak) -->
      <div class="kpi-card mem-card">
        <div class="kpi-top">
          <span class="kpi-icon mem-icon">💾</span>
          <span class="kpi-label">Memory Footprint</span>
        </div>
        <div class="kpi-main-val">
          {{ formatBytes(currentAnalyticsSummary?.avg_memory) }}
        </div>
        <div class="kpi-footer">
          <span class="footer-label">Avg RAM</span>
          <span class="footer-val">Peak: {{ formatBytes(currentAnalyticsSummary?.peak_memory) }}</span>
        </div>
      </div>

      <!-- Player Engagement / Peak -->
      <div class="kpi-card conn-card">
        <div class="kpi-top">
          <span class="kpi-icon conn-icon">👥</span>
          <span class="kpi-label">Player Activity</span>
        </div>
        <div class="kpi-main-val">
          {{ currentAnalyticsSummary?.peak_connections ?? 0 }} <span class="kpi-unit">peak</span>
        </div>
        <div class="kpi-footer">
          <span class="footer-label">Max Players</span>
          <span class="footer-val">Avg: {{ currentAnalyticsSummary?.avg_connections ?? 0 }}</span>
        </div>
      </div>
    </div>
  </div>

  <div class="chart network">
    <canvas ref="networkChartEl"/>
  </div>
  <div class="chart memory">
    <canvas ref="memoryChartEl"/>
  </div>
  <div class="chart cpu">
    <canvas ref="cpuChartEl"/>
  </div>
</template>

<style scoped lang="scss">
.analytics-container {
  margin: 1.25rem 0 1rem 0;
}

.analytics-toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 12px;
  margin-bottom: 12px;
}

.toolbar-left, .toolbar-right {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
}

.timeframe-group {
  display: flex;
  background-color: var(--backdrop);
  padding: 4px;
  border-radius: 12px;
  border: 1px solid rgba(128, 128, 128, 0.2);
  gap: 4px;
}

.timeframe-btn {
  border: none;
  background: transparent;
  color: var(--text-disabled);
  padding: 6px 14px;
  font-size: 0.85rem;
  font-weight: 500;
  border-radius: 8px;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: all 0.15s ease-in-out;

  &:hover {
    color: var(--text);
  }

  &.active {
    background-color: var(--background);
    color: var(--text);
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.18);
    font-weight: 600;
  }
}

.live-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: #6b7280;
  display: inline-block;

  &.pulsing {
    background-color: #10b981;
    box-shadow: 0 0 8px #10b981aa;
    animation: live-pulse 1.8s infinite;
  }
}

@keyframes live-pulse {
  0% { transform: scale(0.95); box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.7); }
  70% { transform: scale(1.15); box-shadow: 0 0 0 6px rgba(16, 185, 129, 0); }
  100% { transform: scale(0.95); box-shadow: 0 0 0 0 rgba(16, 185, 129, 0); }
}

.net-mode-group {
  display: flex;
  gap: 4px;
  background-color: var(--backdrop);
  padding: 3px;
  border-radius: 10px;
  border: 1px solid rgba(128, 128, 128, 0.18);
}

.net-mode-btn {
  border: none;
  background: transparent;
  color: var(--text-disabled);
  padding: 6px 12px;
  font-size: 0.82rem;
  font-weight: 500;
  border-radius: 7px;
  cursor: pointer;
  transition: all 0.15s ease-in-out;

  &:hover {
    color: var(--text);
  }

  &.selected {
    background-color: var(--background);
    color: var(--text);
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.15);
    font-weight: 600;
  }
}

.export-btn {
  border: 1px solid rgba(128, 128, 128, 0.25);
  background-color: var(--background);
  color: var(--text);
  padding: 6px 14px;
  font-size: 0.82rem;
  font-weight: 600;
  border-radius: 8px;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 5px;
  transition: all 0.15s ease;

  &:hover {
    background-color: var(--backdrop);
    border-color: rgba(128, 128, 128, 0.4);
  }
}

.export-icon {
  font-size: 0.95rem;
}

.analytics-subheading {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 12px;
}

.subheading-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.section-title {
  font-size: 1.15rem;
  font-weight: 700;
  color: var(--text);
}

.subheading-desc {
  font-size: 0.8rem;
  color: var(--text-disabled);
}

.data-badge {
  font-size: 0.75rem;
  padding: 3px 10px;
  border-radius: 9999px;
  background-color: var(--backdrop);
  color: var(--text-disabled);
  border: 1px solid rgba(128, 128, 128, 0.2);
}

.kpi-cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(210px, 1fr));
  gap: 12px;
  margin-bottom: 1rem;

  &.analytics-mode-grid {
    grid-template-columns: repeat(auto-fit, minmax(190px, 1fr));
  }
}

.kpi-card {
  background-color: var(--background);
  border: 1px solid rgba(128, 128, 128, 0.18);
  border-radius: 14px;
  padding: 14px 16px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  min-height: 106px;
  transition: transform 0.15s ease, box-shadow 0.15s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 14px rgba(0, 0, 0, 0.08);
  }
}

.kpi-top {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.kpi-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 6px;
  font-size: 0.95rem;
  font-weight: bold;

  &.rx-icon {
    background-color: rgba(16, 185, 129, 0.18);
    color: #10b981;
  }

  &.tx-icon {
    background-color: rgba(59, 130, 246, 0.18);
    color: #3b82f6;
  }

  &.conn-icon {
    background-color: rgba(245, 158, 11, 0.18);
    color: #f59e0b;
  }

  &.total-icon {
    background-color: rgba(139, 92, 246, 0.18);
    color: #8b5cf6;
  }

  &.cpu-icon {
    background-color: rgba(7, 167, 227, 0.18);
    color: #07a7e3;
  }

  &.mem-icon {
    background-color: rgba(236, 72, 153, 0.18);
    color: #ec4899;
  }
}

.kpi-label {
  font-size: 0.8rem;
  font-weight: 500;
  color: var(--text-disabled);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.kpi-main-val {
  font-size: 1.55rem;
  font-weight: 700;
  color: var(--text);
  margin-bottom: 8px;
  line-height: 1.2;
  font-feature-settings: 'tnum';
  font-variant-numeric: tabular-nums;
}

.kpi-unit {
  font-size: 0.9rem;
  font-weight: normal;
  color: var(--text-disabled);
}

.kpi-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.8rem;
  padding-top: 6px;
  border-top: 1px solid rgba(128, 128, 128, 0.12);
}

.footer-label {
  color: var(--text-disabled);
}

.footer-val {
  font-weight: 600;
  color: var(--text);
}
</style>
