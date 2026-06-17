<template>
  <view class="page health-page">
    <ClinicTopBar />

    <view class="health-action-row">
      <view class="health-action card" @click="goHistory">
        <text class="health-action__icon">📅</text>
        <view>
          <text class="health-action__title">经期记录</text>
          <text class="health-action__sub">回顾周期档案</text>
        </view>
      </view>
      <view class="health-action card" @click="goArchive">
        <text class="health-action__icon">📋</text>
        <view>
          <text class="health-action__title">健康档案</text>
          <text class="health-action__sub">共 {{ notes.length }} 条 · 与医生共享</text>
        </view>
      </view>
    </view>

    <view class="health-section-head">
      <view>
        <text class="health-section-title">本周睡眠时间</text>
        <text class="health-section-sub">{{ trendSummary }}</text>
      </view>
      <view class="trend-legend">
        <text><i class="sleep-dot" />睡眠</text>
        <text><i class="goal-dot" />建议 8h</text>
      </view>
    </view>
    <view v-if="trendPoints.length" class="trend-card card">
      <view class="trend-chart">
        <view class="trend-chart__y-axis">
          <text v-for="tick in yAxisTicks" :key="tick.value" class="trend-chart__y-tick" :style="{ bottom: `${(tick.value / trendMaxY) * 100}%` }">
            {{ tick.label }}
          </text>
        </view>
        <view class="trend-chart__plot">
          <view class="trend-chart__grid" v-for="tick in yAxisTicks" :key="`g-${tick.value}`" :style="{ bottom: `${(tick.value / trendMaxY) * 100}%` }" />
          <view class="trend-chart__goal" :style="{ bottom: `${(RECOMMEND_SLEEP_HOURS / trendMaxY) * 100}%` }">
            <text class="trend-chart__goal-label">建议 8h</text>
          </view>
          <view
            v-for="(p, idx) in trendPoints"
            :key="`bar-${p.date}`"
            class="trend-chart__bar"
            :style="{ left: `${(xAt(idx) / lineViewW) * 100}%`, height: `${(p.sleep / trendMaxY) * 100}%` }"
          />
          <view
            v-for="(p, idx) in trendPoints"
            :key="`pt-${p.date}`"
            class="trend-chart__dot"
            :style="{ left: `${(xAt(idx) / lineViewW) * 100}%`, bottom: `${(p.sleep / trendMaxY) * 100}%` }"
          />
          <view
            v-for="(p, idx) in trendPoints"
            :key="`lp-${p.date}`"
            class="trend-chart__value"
            :style="{ left: `${(xAt(idx) / lineViewW) * 100}%`, bottom: `calc(${(p.sleep / trendMaxY) * 100}% + 30rpx)` }"
          >
            {{ p.sleep ? `${p.sleep}h` : '—' }}
          </view>
        </view>
      </view>
      <view class="trend-chart__x-axis">
        <text v-for="p in trendPoints" :key="`xl-${p.date}`" class="trend-chart__x-tick">{{ p.label }}</text>
      </view>
    </view>
    <view v-else class="section-empty card">
      <text class="section-empty__icon">📈</text>
      <text>睡眠数据正在积累</text>
      <text class="section-empty__sub">在周期页记录睡眠时间后，这里会展示本周变化。</text>
    </view>

    <view class="health-section-head">
      <view>
        <text class="health-section-title">症状记录</text>
        <text class="health-section-sub">近 30 天共 {{ symptomTotal }} 条</text>
      </view>
    </view>
    <view v-if="symptoms.length" class="symptom-list card">
      <view v-for="log in symptoms.slice(0, 8)" :key="log.id" class="symptom-row">
        <view class="symptom-row__severity">
          <i v-for="n in 5" :key="n" :class="{ active: n <= Number(log.severity || 1) }" />
        </view>
        <view class="symptom-row__body">
          <text class="symptom-row__tags">{{ joined(log.tags) }}</text>
          <text v-if="log.notes" class="symptom-row__notes">{{ log.notes }}</text>
        </view>
        <text class="symptom-row__time">{{ formatShortDate(log.occurredAt || log.createdAt) }}</text>
      </view>
    </view>
    <view v-else class="section-empty card">
      <text class="section-empty__icon">🌿</text>
      <text>还没有症状记录</text>
      <text class="section-empty__sub">没有不适也很好，持续关注身体变化即可。</text>
    </view>

    <view class="health-section-head">
      <view>
        <text class="health-section-title">健康科普小课堂</text>
        <text class="health-section-sub">用简单易懂的方式了解身体</text>
      </view>
    </view>
    <view v-if="articles.length" class="article-grid">
      <view v-for="article in articles.slice(0, 6)" :key="article.id" class="article-card card" @click="openArticle(article)">
        <image v-if="article.imageUrl" class="article-card__cover" :src="article.imageUrl" mode="aspectFill" />
        <view v-else class="article-card__cover article-card__cover--empty">📰</view>
        <text class="article-card__category">{{ article.category || "健康科普" }}</text>
        <text class="article-card__title">{{ article.title }}</text>
        <text class="article-card__summary">{{ article.summary || "用简单易懂的方式，了解自己的身体。" }}</text>
      </view>
    </view>
    <view v-else class="section-empty card">
      <text class="section-empty__icon">📚</text>
      <text>新的科普内容正在准备中</text>
      <text class="section-empty__sub">稍后回来看看，诊所会持续更新健康知识。</text>
    </view>

    <view v-if="activeArticle" class="article-backdrop" @click="activeArticle = null">
      <view class="article-sheet" @click.stop>
        <view class="article-sheet__handle" />
        <image v-if="activeArticle.imageUrl" class="article-sheet__cover" :src="activeArticle.imageUrl" mode="aspectFill" />
        <text class="article-sheet__category">{{ activeArticle.category || "健康科普" }}</text>
        <text class="article-sheet__title">{{ activeArticle.title }}</text>
        <text class="article-sheet__summary">{{ activeArticle.summary || "用简单易懂的方式，了解自己的身体。" }}</text>
        <scroll-view class="article-sheet__body" scroll-y>
          <text>{{ activeArticle.content || activeArticle.summary || "内容正在持续完善中。" }}</text>
        </scroll-view>
        <button class="article-sheet__close" @click="activeArticle = null">我知道了</button>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import * as menstrualApi from "@/api/menstrual";
import { usePeriodStore } from "@/stores/usePeriodStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

const periodStore = usePeriodStore();
const notes = ref<any[]>([]);
const symptoms = ref<any[]>([]);
const metrics = ref<any>(null);
const dashboard = ref<any>(null);
const activeArticle = ref<any | null>(null);

const RECOMMEND_SLEEP_HOURS = 8;
const Y_AXIS_STEP = 2;
const lineViewW = 100;
const lineViewH = 60;

const articles = computed(() => dashboard.value?.articles ?? []);
const symptomTotal = computed(() => metrics.value?.month?.summary?.totalSymptoms ?? metrics.value?.week?.summary?.totalSymptoms ?? symptoms.value.length);
const trendSummary = computed(() => {
  const avg = metrics.value?.week?.summary?.avgSleepHours;
  return avg != null ? `平均睡眠 ${avg}h · 建议 ${RECOMMEND_SLEEP_HOURS}h` : "等待数据累积";
});
type SleepPoint = { date: string; label: string; sleep: number };
const trendPoints = computed<SleepPoint[]>(() => {
  const week = metrics.value?.week;
  const series = week?.sleepSeries ?? [];
  if (!series.length) return [];
  return series.map((item: any) => ({
    date: item.date,
    label: item.date.slice(5),
    sleep: Number(item.hours || 0),
  }));
});

const trendMaxY = computed(() => {
  const values = trendPoints.value.map((p) => p.sleep);
  const observed = values.length ? Math.max(...values) : 0;
  return Math.max(RECOMMEND_SLEEP_HOURS, Math.ceil(Math.max(observed, 4) / Y_AXIS_STEP) * Y_AXIS_STEP);
});

const yAxisTicks = computed(() => {
  const max = trendMaxY.value;
  const ticks: { value: number; label: string }[] = [];
  for (let v = 0; v <= max; v += Y_AXIS_STEP) ticks.push({ value: v, label: `${v}h` });
  return ticks;
});

function xAt(index: number) {
  const n = trendPoints.value.length;
  if (n <= 1) return lineViewW / 2;
  return ((index + 0.5) / n) * lineViewW;
}

function yAt(hours: number) {
  const ratio = Math.max(0, Math.min(1, hours / trendMaxY.value));
  return lineViewH - ratio * lineViewH;
}

function formatDate(iso?: string) {
  return iso?.slice(0, 10) || "—";
}

function formatShortDate(iso?: string) {
  const date = formatDate(iso);
  return date === "—" ? date : date.slice(5);
}

function joined(raw: any) {
  if (Array.isArray(raw)) return raw.join(" · ") || "未填写症状";
  return String(raw || "未填写症状");
}

function goHistory() {
  uni.navigateTo({ url: "/pages/history/index" });
}

function goArchive() {
  uni.navigateTo({ url: "/pages/archive/index" });
}

function openArticle(article: any) {
  activeArticle.value = article;
}

onMounted(async () => {
  const [recordsResult, symptomsResult, metricsResult, dashboardResult] = await Promise.allSettled([
    menstrualApi.fetchRecords(),
    menstrualApi.fetchSymptomLogs(30),
    menstrualApi.fetchMetrics(),
    menstrualApi.fetchDashboard(),
  ]);
  if (recordsResult.status === "fulfilled") notes.value = recordsResult.value.records ?? [];
  if (symptomsResult.status === "fulfilled") symptoms.value = symptomsResult.value.logs ?? [];
  if (metricsResult.status === "fulfilled") metrics.value = metricsResult.value;
  if (dashboardResult.status === "fulfilled") dashboard.value = dashboardResult.value;
  if (!periodStore.records.length) void periodStore.loadAll();
});
</script>

<style lang="scss" scoped>
.health-page { min-height: 100vh; padding: $gap-md $gap-md 80rpx; }
.health-action-row { display: grid; grid-template-columns: 1fr 1fr; gap: $gap-sm; margin-bottom: $gap-md; padding-top: calc(env(safe-area-inset-top) + 10rpx); }
.health-action { display: flex; align-items: center; gap: $gap-sm; padding: $gap-md; background: $paper; border-radius: $radius-lg; box-shadow: $shadow-sm; }
.health-action__icon { display: grid; width: 80rpx; height: 80rpx; flex: 0 0 auto; place-items: center; border-radius: 50%; background: $blush; font-size: 40rpx; }
.health-action view { min-width: 0; flex: 1; }
.health-action__title { display: block; color: $ink; font-size: $font-sm; font-weight: 800; }
.health-action__sub { display: block; margin-top: 4rpx; color: $muted; font-size: 20rpx; line-height: 1.5; }
.health-section-head { display: flex; align-items: flex-end; justify-content: space-between; gap: $gap-md; margin: 44rpx $gap-xs $gap-md; }
.health-section-title { display: block; color: $ink; font-size: $font-lg; font-weight: 800; }
.health-section-sub { display: block; margin-top: 2rpx; color: $muted; font-size: $font-xs; }
.trend-legend { display: flex; gap: $gap-sm; color: $muted; font-size: 20rpx; }
.trend-legend text { display: flex; align-items: center; gap: 6rpx; }
.trend-legend i { width: 12rpx; height: 12rpx; border-radius: 50%; }
.sleep-dot { background: $rose; }
.goal-dot { background: transparent; border: 1px dashed $rose-dark; }
.trend-card { padding: $gap-lg $gap-md $gap-md; }
.trend-chart { display: flex; gap: 8rpx; }
.trend-chart__y-axis { position: relative; width: 40rpx; flex: 0 0 auto; height: 320rpx; }
.trend-chart__y-tick { position: absolute; right: 0; transform: translateY(50%); color: $muted; font-size: 18rpx; line-height: 1; }
.trend-chart__plot { position: relative; flex: 1; min-width: 0; height: 320rpx; }
.trend-chart__grid { position: absolute; left: 0; right: 0; height: 1px; background: $line; }
.trend-chart__goal { position: absolute; left: 0; right: 0; z-index: 2; border-top: 1px dashed $rose-dark; }
.trend-chart__goal-label { position: absolute; right: 4rpx; top: -22rpx; padding: 2rpx 10rpx; border-radius: 999rpx; background: $blush; color: $rose-dark; font-size: 18rpx; line-height: 1.4; }
.trend-chart__bar { position: absolute; bottom: 0; z-index: 1; width: 7%; border-radius: 10rpx 10rpx 0 0; background: linear-gradient(180deg, $rose, $rose-dark); transform: translateX(-50%); box-shadow: $shadow-sm; }
.trend-chart__dot { position: absolute; z-index: 3; width: 18rpx; height: 18rpx; border-radius: 50%; background: $rose-dark; border: 3rpx solid $paper; box-shadow: $shadow-sm; transform: translate(-50%, 50%); }
.trend-chart__value { position: absolute; z-index: 3; transform: translate(-50%, 0); color: $rose-dark; font-size: 18rpx; font-weight: 700; line-height: 1; white-space: nowrap; }
.trend-chart__value { position: absolute; z-index: 3; transform: translateX(-50%); color: $rose-dark; font-size: 18rpx; font-weight: 700; line-height: 1; white-space: nowrap; }
.trend-chart__x-axis { display: flex; gap: 8rpx; margin-top: 6rpx; padding-left: 48rpx; }
.trend-chart__x-tick { flex: 1; min-width: 0; color: $muted; font-size: 18rpx; text-align: center; }
.symptom-list { padding: 0 $gap-md; }
.symptom-row { display: flex; align-items: center; gap: $gap-sm; padding: $gap-md 0; border-bottom: 1px solid $line; }
.symptom-row:last-child { border-bottom: none; }
.symptom-row__severity { display: flex; flex: 0 0 auto; gap: 4rpx; }
.symptom-row__severity i { width: 8rpx; height: 8rpx; border-radius: 50%; background: #e4e4e7; }
.symptom-row__severity i.active { background: $rose-dark; }
.symptom-row__body { display: flex; min-width: 0; flex: 1; flex-direction: column; }
.symptom-row__tags { overflow: hidden; color: $ink; font-size: $font-sm; font-weight: 700; text-overflow: ellipsis; white-space: nowrap; }
.symptom-row__notes { overflow: hidden; color: $muted; font-size: $font-xs; text-overflow: ellipsis; white-space: nowrap; }
.symptom-row__time { flex: 0 0 auto; color: $muted; font-size: 20rpx; }
.article-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: $gap-sm; }
.article-card { position: relative; overflow: hidden; padding: 0 0 $gap-md; }
.article-card__cover { display: flex; width: 100%; height: 180rpx; align-items: center; justify-content: center; background: $blush; font-size: 48rpx; }
.article-card__category { display: block; margin: $gap-sm $gap-md 0; color: $rose-dark; font-size: 20rpx; font-weight: 800; }
.article-card__title { display: -webkit-box; overflow: hidden; margin: 4rpx $gap-md 0; color: $ink; font-size: $font-sm; font-weight: 800; line-height: 1.45; -webkit-box-orient: vertical; -webkit-line-clamp: 2; }
.article-card__summary { display: -webkit-box; overflow: hidden; margin: 6rpx $gap-md 0; color: $muted; font-size: 20rpx; line-height: 1.5; -webkit-box-orient: vertical; -webkit-line-clamp: 2; }
.section-empty { display: flex; flex-direction: column; align-items: center; padding: 44rpx $gap-lg; color: $ink; font-size: $font-sm; text-align: center; }
.section-empty__icon { margin-bottom: $gap-sm; font-size: 48rpx; }
.section-empty__sub { margin-top: $gap-xs; color: $muted; font-size: $font-xs; }
.article-backdrop { position: fixed; inset: 0; z-index: 30; display: flex; align-items: flex-end; background: rgba(39, 39, 42, .46); backdrop-filter: blur(8rpx); }
.article-sheet { display: flex; width: 100%; max-height: 88vh; flex-direction: column; overflow: hidden; padding: $gap-md $gap-lg calc(36rpx + env(safe-area-inset-bottom)); border-radius: 40rpx 40rpx 0 0; background: $paper; }
.article-sheet__handle { width: 80rpx; height: 8rpx; flex: 0 0 auto; margin: 0 auto $gap-lg; border-radius: $radius-full; background: #e4e4e7; }
.article-sheet__cover { width: 100%; height: 280rpx; flex: 0 0 auto; border-radius: $radius-lg; background: $blush; }
.article-sheet__category { display: block; margin-top: $gap-lg; color: $rose-dark; font-size: $font-xs; font-weight: 800; }
.article-sheet__title { display: block; margin-top: 4rpx; color: $ink; font-size: $font-xl; font-weight: 800; line-height: 1.35; }
.article-sheet__summary { display: block; margin-top: $gap-sm; color: $muted; font-size: $font-sm; line-height: 1.6; }
.article-sheet__body { min-height: 160rpx; margin-top: $gap-md; color: $ink; font-size: $font-sm; line-height: 1.9; white-space: pre-wrap; }
.article-sheet__close { width: 100%; flex: 0 0 auto; margin-top: $gap-md; border-radius: $radius-full; background: $ink; color: #fff; font-size: $font-sm; font-weight: 700; }
</style>
