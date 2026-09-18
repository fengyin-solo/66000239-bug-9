<template>
  <div class="ecg-waveform-container">
    <div class="flex items-center justify-between mb-2">
      <h3 class="text-lg font-semibold text-emerald-400">
        心电图 - 导联 {{ leadName }}
      </h3>
      <div class="flex items-center gap-2">
        <span class="text-xs text-gray-400">{{ samplingRate }} Hz</span>
        <span class="text-xs text-gray-400">{{ duration }}s</span>
        <span class="text-xs text-gray-500">
          量程 {{ formatMv(yRange.min) }} ~ {{ formatMv(yRange.max) }} mV
        </span>
      </div>
    </div>
    <v-chart
      ref="chartRef"
      class="ecg-chart"
      :option="chartOption"
      autoresize
      :update-options="{ notMerge: true }"
      @datazoom="onDataZoom"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue';
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

// 当前 dataZoom 窗口（百分比）。由用户滚轮/拖动更新；option 重算时必须带上，
// 否则 notMerge 的 setOption 会把浏览窗口重置回整段。
const zoomWindow = ref<{ start: number; end: number }>({ start: 0, end: 100 });

// 纵向振幅量程，随当前可见窗口内的信号动态更新
const yRange = ref<{ min: number; max: number }>({ min: 0, max: 1 });

// 全分辨率点集：x 使用真实数值时间（秒）。不再抽稀——30s × 500Hz 仅 15000 点，
// 数值时间轴可以保证悬停读数、R 峰标注、放大后的坐标三者严格一致，
// 任何一个 R 峰采样点也不会被丢掉。
const waveformData = computed<[number, number][]>(() => {
  const sr = props.samplingRate;
  const data = new Array<[number, number]>(props.samples.length);
  for (let i = 0; i < props.samples.length; i++) {
    data[i] = [i / sr, props.samples[i]];
  }
  return data;
});

const fullTimeMax = computed(() =>
  props.samples.length > 0 ? (props.samples.length - 1) / props.samplingRate : 0
);

/** 1-2-5 级数取整，保证纵向参考线等间距、刻度稀疏易读 */
function niceStep(rawSpan: number): number {
  if (!isFinite(rawSpan) || rawSpan <= 0) return 1;
  const mag = Math.pow(10, Math.floor(Math.log10(rawSpan)));
  const norm = rawSpan / mag;
  let nice: number;
  if (norm <= 1) nice = 0.1;
  else if (norm <= 2) nice = 0.2;
  else if (norm <= 5) nice = 0.5;
  else nice = 1;
  return nice * mag;
}

const round6 = (v: number) => Math.round(v * 1e6) / 1e6;

function formatMv(v: number): string {
  return v.toFixed(3).replace(/0+$/, '').replace(/\.$/, '');
}

/** 依据数据在给定时间窗口内的真实最大/最小振幅，计算纵向量程 */
function rangeForWindow(timeMin: number, timeMax: number) {
  const sr = props.samplingRate;
  const iMin = Math.max(0, Math.floor(timeMin * sr));
  const iMax = Math.min(
    props.samples.length - 1,
    Math.ceil(timeMax * sr)
  );
  let vMin = Infinity;
  let vMax = -Infinity;
  for (let i = iMin; i <= iMax; i++) {
    const v = props.samples[i];
    if (v < vMin) vMin = v;
    if (v > vMax) vMax = v;
  }
  if (vMin === Infinity) {
    vMin = -1;
    vMax = 1;
  }
  // 留出标注空间，避免峰值/谷值贴边
  const span = Math.max(vMax - vMin, 0.2);
  const pad = span * 0.15;
  const step = niceStep((span + pad * 2) / 5);
  const min = Math.floor((vMin - pad) / step) * step;
  const max = Math.ceil((vMax + pad) / step) * step;
  return { min: round6(min), max: round6(max), step };
}

/** 重新采集/换导联后：浏览窗口与纵向量程都恢复为整段数据 */
watch(
  () => [props.samples, props.samplingRate, props.duration] as const,
  () => {
    zoomWindow.value = { start: 0, end: 100 };
    const r = rangeForWindow(0, fullTimeMax.value || props.duration);
    yRange.value = { min: r.min, max: r.max };
  },
  { immediate: true }
);

function readZoomFromChart(): { start: number; end: number } | null {
  const opt = chartRef.value?.getOption?.() as any;
  const dz = opt?.dataZoom?.find((z: any) => xAxisZoom(z));
  if (dz && isFinite(dz.start) && isFinite(dz.end)) {
    return { start: dz.start, end: dz.end };
  }
  return null;
}

function xAxisZoom(z: any): boolean {
  if (!z) return false;
  if (z.xAxisIndex != null) {
    const idx = Array.isArray(z.xAxisIndex) ? z.xAxisIndex : [z.xAxisIndex];
    return idx.includes(0);
  }
  return z.xAxisId == null;
}

/** 滚轮缩放/拖动浏览时，读取当前窗口并同步纵向振幅刻度 */
function onDataZoom(params: any) {
  let win = readZoomFromChart();
  if (
    !win &&
    params &&
    isFinite(params.start) &&
    isFinite(params.end)
  ) {
    win = { start: params.start, end: params.end };
  }
  if (!win) return;

  zoomWindow.value = win;

  const tMax = fullTimeMax.value || props.duration;
  const tMin = (win.start / 100) * tMax;
  const tMaxVis = (win.end / 100) * tMax;
  const r = rangeForWindow(tMin, tMaxVis);
  if (
    Math.abs(r.min - yRange.value.min) > 1e-6 ||
    Math.abs(r.max - yRange.value.max) > 1e-6
  ) {
    yRange.value = { min: r.min, max: r.max };
  }
}

const chartOption = computed(() => {
  // R 峰标注：坐标使用真实数值 (time, amplitude)，与曲线同一数值坐标系，
  // 无论取点多密、如何放大，标注都精确钉在对应采样点（曲线最高处）。
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
      // 负向 R 波（如 aVR）把文字放到点下方，避免压住曲线
      position: (rp.amplitude >= 0 ? 'top' : 'bottom') as 'top' | 'bottom',
    },
  }));

  // 背景等间距纵向参考线：每 0.2s 一条，贯穿整个纵向量程。
  // 挂在 series.markLine 上才会真正被画出来（xAxis 为数值轴，放大后自动跟随）。
  const verticalRefLines: any[] = [];
  const stepX = 0.2;
  const yMin = yRange.value.min;
  const yMax = yRange.value.max;
  for (let x = 0; x <= props.duration + 1e-9; x += stepX) {
    verticalRefLines.push([
      { xAxis: round6(x), yAxis: yMin },
      { xAxis: round6(x), yAxis: yMax },
    ]);
  }

  const yStep = niceStep((yRange.value.max - yRange.value.min) / 5);

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
      axisPointer: {
        type: 'line',
        // 数值轴上吸附到最近的真实采样点，读出的时间就是采样时间，
        // 与 R 峰标注、放大后坐标完全同步
        snap: true,
        lineStyle: { color: 'rgba(16, 185, 129, 0.5)', type: 'dashed' },
        label: {
          backgroundColor: '#10b981',
          formatter: (val: any) => `${Number(val.axisValue).toFixed(3)}s`,
        },
      },
      backgroundColor: 'rgba(0, 0, 0, 0.8)',
      borderColor: '#10b981',
      textStyle: { color: '#fff', fontSize: 12 },
      formatter: (params: any) => {
        const p = Array.isArray(params) ? params[0] : params;
        const pt = p?.value;
        if (!Array.isArray(pt)) return '';
        return `时间: ${Number(pt[0]).toFixed(3)}s<br/>振幅: ${Number(pt[1]).toFixed(3)} mV`;
      },
    },
    xAxis: {
      type: 'value',
      name: '时间 (s)',
      nameTextStyle: { color: '#9ca3af', fontSize: 11 },
      min: 0,
      max: fullTimeMax.value || props.duration,
      axisLine: { lineStyle: { color: '#374151' } },
      axisLabel: {
        color: '#9ca3af',
        fontSize: 10,
        formatter: (v: number) => Number(v).toFixed(1),
      },
      splitLine: { show: false },
    },
    yAxis: {
      type: 'value',
      name: 'mV',
      nameTextStyle: { color: '#9ca3af', fontSize: 11 },
      // 纵向振幅刻度随放大窗口内的实际起伏更新；min/max 对齐到等间距 step，
      // splitLine 即水平参考线，真正渲染且与刻度一一对应
      min: yRange.value.min,
      max: yRange.value.max,
      interval: yStep,
      axisLine: { lineStyle: { color: '#374151' } },
      axisLabel: {
        color: '#9ca3af',
        fontSize: 10,
        formatter: (v: number) => Number(v).toFixed(2),
      },
      splitNumber: 5,
      splitLine: {
        show: true,
        lineStyle: { color: 'rgba(16, 185, 129, 0.12)', type: 'dashed' },
      },
    },
    dataZoom: [
      {
        type: 'inside',
        xAxisIndex: 0,
        start: zoomWindow.value.start,
        end: zoomWindow.value.end,
        zoomOnMouseWheel: true,
        moveOnMouseMove: true,
        moveOnMouseWheel: false,
      },
      {
        type: 'slider',
        xAxisIndex: 0,
        height: 20,
        bottom: 5,
        start: zoomWindow.value.start,
        end: zoomWindow.value.end,
        borderColor: '#374151',
        fillerColor: 'rgba(16, 185, 129, 0.15)',
        handleStyle: { color: '#10b981' },
        textStyle: { color: '#9ca3af' },
        labelFormatter: (v: number) => `${Number(v).toFixed(1)}s`,
      },
    ],
    series: [
      {
        name: 'ECG',
        type: 'line',
        data: waveformData.value,
        showSymbol: false,
        lineStyle: {
          color: '#10b981',
          width: 1.5,
        },
        markLine: {
          symbol: 'none',
          silent: true,
          animation: false,
          data: verticalRefLines,
          lineStyle: {
            color: 'rgba(16, 185, 129, 0.12)',
            type: 'dashed',
            width: 1,
          },
          label: { show: false },
        },
        markPoint: {
          data: rPeakMarkers,
          animation: false,
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
