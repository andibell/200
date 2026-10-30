<script setup lang="ts">
defineProps<{
  label: string
  value: string
  detail: string
  change?: string
  changeDirection?: 'positive' | 'negative' | 'neutral'
  icon: 'shipments' | 'ontime' | 'exceptions' | 'cost'
}>()
</script>

<template>
  <article class="metric-card">
    <div class="metric-card__top">
      <p class="metric-card__label">{{ label }}</p>
      <span class="metric-card__icon" aria-hidden="true">
        <svg v-if="icon === 'shipments'" viewBox="0 0 24 24" fill="none">
          <path d="M3 7.5 12 3l9 4.5v9L12 21l-9-4.5v-9Z" />
          <path d="m3.5 7.7 8.5 4.4 8.5-4.4M12 12.2V21M7.5 5.3l9 4.6" />
        </svg>
        <svg v-else-if="icon === 'ontime'" viewBox="0 0 24 24" fill="none">
          <circle cx="12" cy="12" r="9" />
          <path d="M12 7v5l3.5 2" />
        </svg>
        <svg v-else-if="icon === 'exceptions'" viewBox="0 0 24 24" fill="none">
          <path d="m12 3 9 16H3l9-16Z" />
          <path d="M12 9v4m0 3h.01" />
        </svg>
        <svg v-else viewBox="0 0 24 24" fill="none">
          <path d="M4 7h16M6 3v4m12-4v4M5 11h14v9H5z" />
          <path d="M8 15h3m2 0h3" />
        </svg>
      </span>
    </div>
    <div class="metric-card__value">{{ value }}</div>
    <div class="metric-card__footer">
      <span
        v-if="change"
        class="metric-card__change"
        :class="[`is-${changeDirection ?? 'neutral'}`, { 'is-down': change.startsWith('-') }]"
      >
        <svg viewBox="0 0 16 16" aria-hidden="true">
          <path d="m3 10 5-5 5 5" />
        </svg>
        {{ change }}
      </span>
      <span class="metric-card__detail">{{ detail }}</span>
    </div>
  </article>
</template>
