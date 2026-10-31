# FastForward Logistics Operations Command Center

## Overview

FastForward Logistics is a fictional freight and supply chain company. This dashboard gives its operations leadership one place to review network health, compare regions, and identify delivery issues that may need attention.

It is designed primarily for a VP of Operations preparing for leadership discussions, with useful detail for operations managers and regional leaders investigating performance.

## Features

- **Executive overview:** shipment volume, on-time delivery, open exceptions, and freight cost, with period comparisons where applicable.
- **Performance trend:** shipment volume and on-time delivery over time, including a 95% delivery target reference.
- **Global filters:** view the last 7, 30, or 90 days and select a region. These filters update the dashboard consistently and can be reset.
- **Regional Performance:** compare shipment volume, on-time delivery, open exceptions, freight cost, and operational status. Sort the table by its performance columns and expand a region for target comparison, period changes, exception severity counts, and common exception types.
- **Open Exceptions:** review exception ID, region, shipment/lane reference, type, severity, age, description, and status. Sort by table columns; filter by severity or type; and search IDs, references, or descriptions. Table controls only affect the exception rows and preserve a helpful empty state.
- **Needs Attention:** a concise insight highlights critical exceptions, below-target regional delivery, or recurring high-severity issue types.

## Data

All data is fictional and synthetic. No real client, company, customer, employee, shipper, lane, cost, or operational data is used. Regional baselines and sample exceptions are stored in `src/data/metrics.json`; the dashboard generates synthetic daily shipment, delivery, and cost records from those baselines. There is no live operational data source.

## Technology

- Vue 3 with TypeScript and the Composition API
- Vite
- Chart.js with vue-chartjs
- Vue Router

## Run locally

Use Node.js `^22.18.0` or `>=24.12.0` and npm.

```sh
npm install
npm run dev
```

Vite prints a local URL when the development server is ready.

## Production build

```sh
npm run build
```

This type-checks the Vue/TypeScript project and creates the production build in `dist/`. To serve that build locally, run `npm run preview`.

## Project structure

```text
src/
  components/
    MetricCard.vue                 Reusable executive metric card
    PerformanceTrendChart.vue      Dual-axis shipment and on-time chart
  data/
    metrics.json                   Synthetic regional baselines and exceptions
  router/
    index.ts                       Application routes
  views/
    HomeView.vue                   Dashboard calculations, filters, and tables
  assets/
    base.css                       Base styles
    main.css                       Dashboard layout and component styles
```

## UX approach

The page prioritizes executive information hierarchy: key metrics and a needs-attention insight come before trends and regional detail. Global date and region filters provide a consistent dashboard-wide view. Regional Performance uses progressive disclosure: sorting supports comparison, while expandable rows reveal concise explanations without nesting another dashboard. Open Exceptions supports focused triage with local sorting, severity and type filters, and search, without changing the executive metrics or other dashboard sections.
