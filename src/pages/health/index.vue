<template>
  <view class="page health-page">
    <ClinicTopBar />

    <view class="action-grid">
      <view class="action-card card" @click="goHistory">
        <text class="action-card__icon">📊</text>
        <view class="action-card__content">
          <text class="action-card__title">临床周期档案</text>
          <text class="action-card__desc">回顾详细体征数据</text>
        </view>
      </view>
      <view class="action-card card" @click="goArchive">
        <text class="action-card__icon">📁</text>
        <view class="action-card__content">
          <text class="action-card__title">病历与健康档案</text>
          <text class="action-card__desc">共 {{ notes.length }} 份归档资料</text>
        </view>
      </view>
    </view>

    <!-- 睡眠监测分析 -->
    <view class="chart-section card">
      <view class="section-header">
        <view>
          <text class="section-title">睡眠监测分析</text>
          <text class="section-sub">{{ trendSummary }}</text>
        </view>
      </view>
      
      <view v-if="trendPoints.length" class="trend-chart">
        <view class="trend-chart__y-axis">
          <text v-for="tick in yAxisTicks" :key="tick.value" class="y-tick" :style="{ bottom: `${(tick.value / trendMaxY) * 100}%` }">
            {{ tick.label }}
          </text>
        </view>
        <view class="trend-chart__plot">
          <view class="grid-line" v-for="tick in yAxisTicks" :key="`g-${tick.value}`" :style="{ bottom: `${(tick.value / trendMaxY) * 100}%` }" />
          <view class="goal-line" :style="{ bottom: `${(RECOMMEND_SLEEP_HOURS / trendMaxY) * 100}%` }">
            <text class="goal-label">临床参考线: 8h</text>
          </view>
          <view
            v-for="(p, idx) in trendPoints"
            :key="`bar-${p.date}`"
            class="bar"
            :style="{ left: `${(xAt(idx) / lineViewW) * 100}%`, height: `${(p.sleep / trendMaxY) * 100}%` }"
          />
          <view
            v-for="(p, idx) in trendPoints"
            :key="`val-${p.date}`"
            class="bar-value"
            :style="{ left: `${(xAt(idx) / lineViewW) * 100}%`, bottom: `calc(${(p.sleep / trendMaxY) * 100}% + 8rpx)` }"
          >
            {{ p.sleep ? `${p.sleep}` : '' }}
          </view>
        </view>
        <view class="trend-chart__x-axis">
          <text v-for="p in trendPoints" :key="`xl-${p.date}`" class="x-tick">{{ p.label }}</text>
        </view>
      </view>
      <view v-else class="empty-state">
        <text>数据样本量不足，请持续录入以生成趋势分析。</text>
      </view>
    </view>

    <!-- 症状体征记录 -->
    <view class="list-section card">
      <view class="section-header">
        <view>
          <text class="section-title">近期体征异常监控</text>
          <text class="section-sub">近 30 天内记录 {{ symptomTotal }} 次异常</text>
        </view>
      </view>
      <view v-if="symptoms.length" class="symptom-list">
        <view v-for="log in symptoms.slice(0, 8)" :key="log.id" class="symptom-item">
          <view class="symptom-item__level">
            <text class="level-badge" :class="`level-${log.severity}`">Lv.{{ log.severity || 1 }}</text>
          </view>
          <view class="symptom-item__content">
            <text class="symptom-tags">{{ joined(log.tags) }}</text>
            <text v-if="log.notes" class="symptom-notes">{{ log.notes }}</text>
          </view>
          <text class="symptom-date">{{ formatShortDate(log.occurredAt || log.createdAt) }}</text>
        </view>
      </view>
      <view v-else class="empty-state">
        <text>未监测到异常体征记录。</text>
      </view>
    </view>

    <!-- 科普/医学文献 -->
    <view class="list-section card">
      <view class="section-header">
        <view>
          <text class="section-title">临床宣教资料</text>
          <text class="section-sub">机构下发的健康指导与科普资料</text>
        </view>
      </view>
      <view v-if="articles.length" class="article-list">
        <view v-for="article in articles.slice(0, 4)" :key="article.id" class="article-item" @click="openArticle(article)">
          <view class="article-item__content">
            <text class="article-category">{{ article.category || "医学宣教" }}</text>
            <text class="article-title">{{ article.title }}</text>
            <text class="article-summary">{{ article.summary || "点击查看详情" }}</text>
          </view>
          <image v-if="article.imageUrl" class="article-thumb" :src="article.imageUrl" mode="aspectFill" />
        </view>
      </view>
      <view v-else class="empty-state">
        <text>暂无下发的宣教资料。</text>
      </view>
    </view>

    <view v-if="activeArticle" class="overlay" @click="activeArticle = null">
      <view class="article-dialog" @click.stop>
        <view class="article-dialog__header">
          <text class="article-dialog__category">{{ activeArticle.category || "医学宣教" }}</text>
          <text class="article-dialog__close" @click="activeArticle = null">×</text>
        </view>
        <scroll-view class="article-dialog__body" scroll-y>
          <image v-if="activeArticle.imageUrl" class="article-hero-img" :src="activeArticle.imageUrl" mode="aspectFill" />
          <text class="article-dialog__title">{{ activeArticle.title }}</text>
          <text class="article-dialog__content">{{ activeArticle.content || activeArticle.summary || "正文内容为空。" }}</text>
        </scroll-view>
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

const articles = computed(() => dashboard.value?.articles ?? []);
const symptomTotal = computed(() => metrics.value?.month?.summary?.totalSymptoms ?? metrics.value?.week?.summary?.totalSymptoms ?? symptoms.value.length);
const trendSummary = computed(() => {
  const avg = metrics.value?.week?.summary?.avgSleepHours;
  return avg != null ? `周期内日均睡眠 ${avg}h` : "样本量不足";
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
  return Math.max(RECOMMEND_SLEEP_HOURS + 2, Math.ceil(Math.max(observed, 4) / Y_AXIS_STEP) * Y_AXIS_STEP);
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

function formatDate(iso?: string) { return iso?.slice(0, 10) || "—"; }
function formatShortDate(iso?: string) {
  const date = formatDate(iso);
  return date === "—" ? date : date.slice(5);
}
function joined(raw: any) {
  if (Array.isArray(raw)) return raw.join(" | ") || "未指明症状";
  return String(raw || "未指明症状");
}

function goHistory() { uni.navigateTo({ url: "/pages/history/index" }); }
function goArchive() { uni.navigateTo({ url: "/pages/archive/index" }); }
function openArticle(article: any) { activeArticle.value = article; }

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

.action-grid { display: grid; grid-template-columns: 1fr 1fr; gap: $gap-md; margin-bottom: $gap-md; }
.action-card { display: flex; align-items: center; gap: $gap-sm; padding: $gap-md; margin: 0; }
.action-card__icon { font-size: 40rpx; }
.action-card__content { display: flex; flex-direction: column; }
.action-card__title { font-size: $font-sm; font-weight: 700; color: $ink; }
.action-card__desc { font-size: 20rpx; color: $muted; margin-top: 4rpx;}

.section-header { margin-bottom: $gap-md; }
.section-title { display: block; font-size: $font-md; font-weight: 700; color: $ink; }
.section-sub { display: block; font-size: 22rpx; color: $muted; margin-top: 4rpx; }

/* 趋势图 - 专业柱状图样式 */
.trend-chart { position: relative; padding-left: 48rpx; padding-bottom: 40rpx; height: 360rpx; margin-top: $gap-lg;}
.trend-chart__y-axis { position: absolute; left: 0; top: 0; bottom: 40rpx; width: 40rpx; border-right: 1px solid $line;}
.y-tick { position: absolute; right: 8rpx; transform: translateY(50%); font-size: 20rpx; color: $muted; }
.trend-chart__plot { position: absolute; left: 48rpx; right: 0; top: 0; bottom: 40rpx; }
.grid-line { position: absolute; left: 0; right: 0; height: 1px; background: $line; border-top: 1px dashed rgba(226, 232, 240, 0.5); }
.goal-line { position: absolute; left: 0; right: 0; border-top: 1px dashed $rose-dark; z-index: 1;}
.goal-label { position: absolute; right: 0; top: -28rpx; font-size: 18rpx; color: $rose-dark; background: $surface; padding: 0 8rpx; border-radius: 4rpx;}
.bar { position: absolute; bottom: 0; width: 24rpx; background: $rose; border-radius: 4rpx 4rpx 0 0; transform: translateX(-50%); z-index: 2; transition: height 0.3s ease;}
.bar-value { position: absolute; font-size: 18rpx; font-weight: bold; color: $ink; transform: translateX(-50%); z-index: 2;}
.trend-chart__x-axis { position: absolute; left: 48rpx; right: 0; bottom: 0; height: 40rpx; display: flex; align-items: center;}
.x-tick { flex: 1; text-align: center; font-size: 20rpx; color: $muted; }

.empty-state { padding: $gap-lg; text-align: center; color: $muted; font-size: $font-sm; background: $surface; border-radius: $radius-sm; border: 1px dashed $line;}

/* 列表样式 */
.symptom-list { display: flex; flex-direction: column; gap: $gap-sm; }
.symptom-item { display: flex; align-items: flex-start; gap: $gap-sm; padding-bottom: $gap-sm; border-bottom: 1px solid $line; }
.symptom-item:last-child { border-bottom: none; padding-bottom: 0; }
.level-badge { font-size: 20rpx; font-weight: bold; padding: 2rpx 8rpx; border-radius: 4rpx; background: $surface; color: $muted; }
.level-3 { background: #fef08a; color: #854d0e; }
.level-4 { background: #fed7aa; color: #c2410c; }
.level-5 { background: #fecaca; color: #b91c1c; }
.symptom-item__content { flex: 1; display: flex; flex-direction: column; }
.symptom-tags { font-size: $font-sm; font-weight: 600; color: $ink; }
.symptom-notes { font-size: 22rpx; color: $muted; margin-top: 4rpx; }
.symptom-date { font-size: 20rpx; color: $muted; }

.article-list { display: flex; flex-direction: column; gap: $gap-md; }
.article-item { display: flex; gap: $gap-md; padding-bottom: $gap-md; border-bottom: 1px solid $line; }
.article-item:last-child { border-bottom: none; padding-bottom: 0; }
.article-item__content { flex: 1; display: flex; flex-direction: column; justify-content: center; }
.article-category { font-size: 20rpx; color: $rose-dark; font-weight: 600; margin-bottom: 4rpx; }
.article-title { font-size: $font-sm; font-weight: 700; color: $ink; display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 2; overflow: hidden;}
.article-summary { font-size: 22rpx; color: $muted; margin-top: 8rpx; display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 2; overflow: hidden;}
.article-thumb { width: 140rpx; height: 140rpx; border-radius: $radius-sm; background: $surface; flex: 0 0 auto;}

/* 文章弹窗 */
.overlay { position: fixed; inset: 0; z-index: 100; background: rgba(15, 23, 42, 0.5); display: flex; align-items: flex-end; }
.article-dialog { width: 100%; height: 90vh; background: $paper; border-radius: $radius-lg $radius-lg 0 0; display: flex; flex-direction: column; }
.article-dialog__header { display: flex; justify-content: space-between; align-items: center; padding: $gap-md $gap-lg; border-bottom: 1px solid $line; background: $surface; border-radius: $radius-lg $radius-lg 0 0; }
.article-dialog__category { font-size: $font-sm; font-weight: 700; color: $rose-dark; }
.article-dialog__close { font-size: 44rpx; color: $muted; padding: 0 10rpx;}
.article-dialog__body { padding: $gap-lg; flex: 1; overflow-y: auto; }
.article-hero-img { width: 100%; height: 320rpx; border-radius: $radius-md; margin-bottom: $gap-md; background: $surface;}
.article-dialog__title { display: block; font-size: $font-xl; font-weight: 700; color: $ink; margin-bottom: $gap-lg; line-height: 1.4;}
.article-dialog__content { display: block; font-size: $font-md; color: $ink; line-height: 1.8; white-space: pre-wrap; }
</style>
