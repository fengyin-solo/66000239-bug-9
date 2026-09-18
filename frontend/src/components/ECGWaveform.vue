<template>
  <div class="ecg-waveform-container">
    <div class="flex items-center justify-between mb-2">
      <h3 class="text-lg font-semibold text-emerald-400">
        心电图 - 导联 {{ leadName }}
      </h3>
      <div class="flex items-center gap-2">
        <span class="text-xs text-gray-400">{{ samplingRate }} Hz</span>
        <span class="text-xs text-gray-400">{{ duration }}s</span>
      </div>
    </div>
    <v-chart
      ref="chartRef"
      class="ecg-chart"
      :option="chartOption"
      autoresize
      :update-options="{ notMerge: true }"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';
import VChart from 'vue-echarts';
import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { LineChart, ScatterChart } from 'echarts/charts';
import {
  TitleComponent,
  TooltipComponent,
  GridComponent,
  MarkPointComponent,
  MarkLineComponent,
  DataZoomComponent,
  LegendComponent,
} from 'echarts/components';
import type { RPeak } from '../types';

use([
  CanvasRenderer,
  LineChart,
  ScatterChart,
  TitleComponent,
  TooltipComponent,
  GridComponent,
  MarkPointComponent,
  MarkLineComponent,
  DataZoomComponent,
  LegendComponent,
]);

const props = withDefaults(
  defineProps<{
    samples: number[];
    rPeaks: RPeak[];
    leadName: string;
    samplingRate: number;
    duration: number;
    scrollOffset?: number;
  }>(),
  {
    scrollOffset: 0,
  }
);

const chartRef = ref<InstanceType<typeof VChart> | null>(null);

const chartOption = computed(() => {
  const totalSamples = props.samples.length;

  // Downsample for performance: show every 2nd point if too many samples
  const step = totalSamples > 5000 ? 2 : 1;
  // Numeric [time, amplitude] pairs so markers, tooltip and zoom share one coordinate system
  const seriesData: [number, number][] = [];
  for (let i = 0; i < totalSamples; i += step) {
    seriesData.push([i / props.samplingRate, props.samples[i]]);
  }

  // Amplitude range of the actual signal -> dynamic vertical scale
  let minSample = Infinity;
  let maxSample = -Infinity;
  for (let i = 0; i < totalSamples; i++) {
    const v = props.samples[i];
    if (v < minSample) minSample = v;
    if (v > maxSample) maxSample = v;
  }
  if (!isFinite(minSample) || !isFinite(maxSample)) {
    minSample = -1;
    maxSample = 1;
  }
  const pad = Math.max(0.2, (maxSample - minSample) * 0.1);
  const yMin = Math.floor((minSample - pad) * 10) / 10;
  const yMax = Math.ceil((maxSample + pad) * 10) / 10;

  // R-peak markers pinned to the exact peak coordinates
  const rPeakMarkers = props.rPeaks.map((rp) => ({
    name: 'R',
    coord: [rp.time, rp.amplitude],
    value: `${rp.amplitude.toFixed(2)} mV`,
    symbol: 'triangle',
    symbolSize: 10,
    itemStyle: { color: '#ef4444' },
    label: {
      show: true,
      formatter: 'R',
      color: '#ef4444',
      fontSize: 10,
      position: 'top' as const,
    },
  }));

  // Grid lines for medical ECG appearance (0.2s / 0.5mV spacing)
  const gridLines: any[] = [];
  for (let k = 0; k * 0.2 <= props.duration + 1e-9; k++) {
    gridLines.push({ xAxis: Math.round(k * 0.2 * 1000) / 1000 });
  }
  for (let y = Math.ceil(yMin / 0.5) * 0.5; y <= yMax + 1e-9; y += 0.5) {
    gridLines.push({ yAxis: Math.round(y * 1000) / 1000 });
  }

  return {
    backgroundColor: '#0a0a0a',
    animation: false,
    grid: {
      left: 60,
      right: 30,
      top: 30,
      bottom: 50,
    },
    tooltip: {
      trigger: 'axis',
      backgroundColor: 'rgba(0, 0, 0, 0.8)',
      borderColor: '#10b981',
      textStyle: { color: '#fff', fontSize: 12 },
      formatter: (params: any) => {
        const p = Array.isArray(params) ? params[0] : params;
        const [t, v] = (p.value ?? [0, 0]) as [number, number];
        return `时间: ${t.toFixed(3)}s<br/>振幅: ${v.toFixed(3)} mV`;
      },
    },
    xAxis: {
      type: 'value',
      min: 0,
      max: props.duration,
      name: '时间 (s)',
      nameTextStyle: { color: '#9ca3af', fontSize: 11 },
      axisLine: { lineStyle: { color: '#374151' } },
      axisLabel: {
        color: '#9ca3af',
        fontSize: 10,
        formatter: (val: number) => Number(val.toFixed(2)).toString(),
      },
      splitLine: {
        show: true,
        lineStyle: { color: 'rgba(16, 185, 129, 0.08)', type: 'dashed' },
      },
    },
    yAxis: {
      type: 'value',
      name: 'mV',
      nameTextStyle: { color: '#9ca3af', fontSize: 11 },
      min: yMin,
      max: yMax,
      axisLine: { lineStyle: { color: '#374151' } },
      axisLabel: { color: '#9ca3af', fontSize: 10 },
      splitLine: {
        show: true,
        lineStyle: { color: 'rgba(16, 185, 129, 0.08)', type: 'dashed' },
      },
    },
    dataZoom: [
      {
        type: 'inside',
        xAxisIndex: 0,
        start: 0,
        end: 100,
      },
      {
        type: 'slider',
        xAxisIndex: 0,
        height: 20,
        bottom: 5,
        borderColor: '#374151',
        fillerColor: 'rgba(16, 185, 129, 0.15)',
        handleStyle: { color: '#10b981' },
        textStyle: { color: '#9ca3af' },
      },
    ],
    series: [
      {
        name: 'ECG',
        type: 'line',
        data: seriesData,
        showSymbol: false,
        lineStyle: {
          color: '#10b981',
          width: 1.5,
        },
        markPoint: {
          data: rPeakMarkers,
          animation: false,
        },
        markLine: {
          silent: true,
          symbol: 'none',
          animation: false,
          label: { show: false },
          lineStyle: { color: 'rgba(16, 185, 129, 0.12)', width: 1 },
          data: gridLines,
        },
        z: 10,
      },
    ],
  };
});
</script>

<style scoped>
.ecg-waveform-container {
  width: 100%;
  background: #0a0a0a;
  border: 1px solid rgba(16, 185, 129, 0.2);
  border-radius: 8px;
  padding: 16px;
}

.ecg-chart {
  width: 100%;
  height: 320px;
}
</style>
