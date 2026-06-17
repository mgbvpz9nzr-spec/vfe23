<template>
  <view class="page history-page">
    <ClinicTopBar show-back />
    <view class="history-header">
      <text class="history-title">每月周期，都在这里</text>
      <text class="history-sub">回顾每一次记录的细节，更了解自己的身体规律。</text>
    </view>

    <view v-if="periodStore.isLoading && !records.length" class="history-empty card">
      <text class="history-empty__icon">🗂️</text>
      <text class="history-empty__title">正在整理周期档案</text>
      <text class="history-empty__desc">稍等一下，记录马上就好。</text>
    </view>

    <view v-else-if="!records.length" class="history-empty card">
      <text class="history-empty__icon">📅</text>
      <text class="history-empty__title">暂无经期记录</text>
      <text class="history-empty__desc">去周期页开始记录第一次经期吧。</text>
      <view class="history-empty__action" @click="goPeriod">开始记录</view>
    </view>

    <template v-else>
      <view class="history-nav">
        <view class="history-nav__btn" :class="{ disabled: historyIndex === 0 }" @click="move(-1)">‹</view>
        <scroll-view class="history-nav__dots" scroll-x :show-scrollbar="false">
          <view class="history-nav__dots-inner">
            <text
              v-for="(_, index) in records"
              :key="index"
              class="history-nav__dot"
              :class="{ active: historyIndex === index }"
              @click="historyIndex = index"
            />
          </view>
        </scroll-view>
        <view class="history-nav__btn" :class="{ disabled: historyIndex === records.length - 1 }" @click="move(1)">›</view>
      </view>

      <swiper class="history-swiper" :style="{ height: swiperHeight }" :current="historyIndex" @change="onSwiperChange">
        <swiper-item v-for="record in records" :key="record.id">
          <scroll-view class="history-slide-scroll" scroll-y>
            <view class="history-card card">
              <view class="history-hero">
                <view class="history-hero__head">
                  <text class="history-hero__date">📅 {{ dateOnly(record.startDate) }}</text>
                  <text class="history-hero__duration" :class="{ live: !record.endDate }">
                    {{ record.endDate ? `持续 ${durationDays(record)} 天` : '进行中' }}
                  </text>
                </view>
                <text class="history-hero__range">{{ dateRange(record) }}</text>
              </view>

              <view class="history-metrics">
                <view v-for="metric in metricsOf(record)" :key="metric.label" class="history-metric">
                  <text class="history-metric__label">{{ metric.label }}</text>
                  <text class="history-metric__value">{{ metric.value }}</text>
                </view>
              </view>

              <view v-if="tagGroups(record).length" class="history-tags">
                <view v-for="group in tagGroups(record)" :key="group.label" class="history-tags__row">
                  <text class="history-tags__label">{{ group.label }}</text>
                  <view class="history-tags__items">
                    <text v-for="tag in group.items" :key="tag" class="history-tag">{{ tag }}</text>
                  </view>
                </view>
              </view>

              <view v-if="record.notes" class="history-notes">📝 {{ record.notes }}</view>

              <view v-if="!analysisOf(record) && !riskOf(record)" class="history-ai-empty">
                <text class="history-ai-empty__icon">🤖</text>
                <text>这一周期还没有 AI 分析报告</text>
                <text class="history-ai-empty__sub">去找 AI 小美点击「分析」生成专属报告。</text>
              </view>

              <view v-if="riskOf(record)" class="report report--risk" :class="`report--${riskClass(riskOf(record)?.riskLevel)}`">
                <view class="report__title">
                  <text>🩺 AI 风险评估</text>
                  <text class="report__badge">{{ riskLabel(riskOf(record)?.riskLevel) }}</text>
                </view>
                <view class="risk-grid">
                  <view class="risk-cell">
                    <text>是否可等到明天</text>
                    <strong>{{ yesNo(riskOf(record)?.canWaitUntilNextDay) }}</strong>
                  </view>
                  <view class="risk-cell">
                    <text>建议科室</text>
                    <strong>{{ riskOf(record)?.recommendedDepartment || '—' }}</strong>
                  </view>
                  <view class="risk-cell risk-cell--full">
                    <text>风险信号</text>
                    <strong>{{ joined(riskOf(record)?.riskSignals) }}</strong>
                  </view>
                  <view class="risk-cell risk-cell--full">
                    <text>建议检查</text>
                    <strong>{{ joined(riskOf(record)?.recommendedChecks) }}</strong>
                  </view>
                  <view v-if="riskOf(record)?.medicalAdvice" class="risk-cell risk-cell--full">
                    <text>医疗建议</text>
                    <text class="risk-cell__copy">{{ riskOf(record)?.medicalAdvice }}</text>
                  </view>
                  <view v-if="riskOf(record)?.urgentAction" class="risk-cell risk-cell--full risk-cell--urgent">
                    <text>紧急处置</text>
                    <text class="risk-cell__copy">{{ riskOf(record)?.urgentAction }}</text>
                  </view>
                </view>
              </view>

              <view v-if="analysisOf(record)" class="report report--analysis">
                <view class="report__title">
                  <text>🤖 AI 综合分析</text>
                  <text v-if="analysisOf(record)?.overallScore != null" class="report__badge">
                    综合评分 {{ analysisOf(record)?.overallScore }}/10
                  </text>
                </view>
                <text v-if="analysisOf(record)?.greeting" class="report__greeting">{{ analysisOf(record)?.greeting }}</text>
                <view v-for="section in reportSections(analysisOf(record))" :key="section.label" class="report-section">
                  <text class="report-section__label">{{ section.label }}</text>
                  <text v-if="section.title" class="report-section__title">{{ section.title }}</text>
                  <text v-if="section.content" class="report-section__copy">{{ section.content }}</text>
                  <text v-if="section.tip" class="report-section__tip">{{ section.tip }}</text>
                  <text v-for="item in section.items" :key="item" class="report-section__item">• {{ item }}</text>
                </view>
              </view>
            </view>
          </scroll-view>
        </swiper-item>
      </swiper>

      <text class="history-counter">{{ historyIndex + 1 }} / {{ records.length }} 条记录 · 可左右滑动</text>
    </template>
  </view>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import type { MenstrualRecord } from "@/api/menstrual";
import { usePeriodStore } from "@/stores/usePeriodStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

type ReportSection = { label: string; title: string; content: string; tip: string; items: string[] };

const periodStore = usePeriodStore();
const historyIndex = ref(0);
const records = computed(() => periodStore.records);
const currentRecord = computed(() => records.value[historyIndex.value]);
const swiperHeight = computed(() => {
  const record = currentRecord.value;
  if (!record) return "900rpx";
  const hasReport = Boolean(analysisOf(record) || riskOf(record));
  const sections = reportSections(analysisOf(record)).length;
  return `${hasReport ? 2450 + sections * 250 : 1200}rpx`;
});

const flowLevelMap: Record<string, string> = { HEAVY: "偏多", MEDIUM: "适中", LIGHT: "偏少" };
const flowDetailMap: Record<string, string> = { very_light: "很少", light: "较少", normal: "正常", heavy: "较多", very_heavy: "很多" };
const painMap: Record<string, string> = { NONE: "无痛", MILD: "轻微", MODERATE: "中度", SEVERE: "严重" };
const discomfortMap: Record<string, string> = { NONE: "无不适", BLOATING: "腹胀", CRAMPS: "痉挛", HEADACHE: "头痛", FATIGUE: "乏力", MOOD: "情绪波动" };
const sleepMap: Record<string, string> = { POOR: "差", NORMAL: "一般", GOOD: "好", EXCELLENT: "很好" };
const moodMap: Record<string, string> = { happy: "😊 开心", anxious: "😰 焦虑", blue: "😢 忧伤", angry: "😤 火大", other: "🤔 其他" };
const bloodColorMap: Record<string, string> = { light_pink: "淡粉", red: "鲜红", dark_red: "暗红", brown: "褐色", black: "黑色" };
const bloodStateMap: Record<string, string> = { nothing: "无异常", bad_smell: "异味", block: "血块", zhazhuang: "渣状" };

function parseJson(raw: any) {
  if (!raw) return null;
  if (typeof raw !== "string") return raw;
  try { return JSON.parse(raw); } catch { return null; }
}

function asArray(raw: any): string[] {
  if (Array.isArray(raw)) return raw.map(String);
  if (typeof raw !== "string" || !raw.trim()) return [];
  try {
    const parsed = JSON.parse(raw);
    if (Array.isArray(parsed)) return parsed.map(String);
  } catch {}
  return raw.split(/[,，、]/).map((item) => item.trim()).filter(Boolean);
}

const analysisOf = (record?: MenstrualRecord) => parseJson(record?.aiAnalysis);
const riskOf = (record?: MenstrualRecord) => parseJson(record?.riskAssessment);
const dateOnly = (iso?: string) => iso?.slice(0, 10) || "—";

function durationDays(record: MenstrualRecord) {
  if (!record.endDate) return 0;
  return Math.max(1, Math.round((new Date(record.endDate).getTime() - new Date(record.startDate).getTime()) / 86400000) + 1);
}

function dateRange(record: MenstrualRecord) {
  return record.endDate ? `${dateOnly(record.startDate).slice(5)} → ${dateOnly(record.endDate).slice(5)}` : `${dateOnly(record.startDate)} 进行中`;
}

function timeOnly(value?: string) {
  if (!value) return "";
  const date = new Date(value);
  return Number.isNaN(date.getTime()) ? "" : `${String(date.getHours()).padStart(2, "0")}:${String(date.getMinutes()).padStart(2, "0")}`;
}

function metricsOf(record: MenstrualRecord) {
  const flow = [flowLevelMap[record.flowLevel || ""] || "—", flowDetailMap[record.flowDetail || ""]].filter(Boolean).join(" · ");
  const sleepTimes = record.sleepStart && record.sleepEnd ? `${timeOnly(record.sleepStart)} → ${timeOnly(record.sleepEnd)}` : "";
  const sleep = [sleepMap[record.sleepQuality || ""] || "—", sleepTimes, record.sleepHours != null ? `${record.sleepHours}h` : ""].filter(Boolean).join(" · ");
  return [
    { label: "流量", value: flow },
    { label: "疼痛", value: painMap[record.painLevel || ""] || "无痛" },
    { label: "不适", value: discomfortMap[record.discomfort || "NONE"] || "—" },
    { label: "睡眠", value: sleep },
    { label: "血色", value: bloodColorMap[record.bloodColor || ""] || "—" },
    { label: "体重", value: record.weight != null ? `${record.weight} kg` : "—" },
    { label: "基础体温", value: record.bodyTemp != null ? `${record.bodyTemp}°C` : "—" },
    { label: "心情", value: moodMap[record.mood || ""] || "—" },
  ];
}

function tagGroups(record: MenstrualRecord) {
  const groups = [
    { label: "症状", items: asArray(record.symptoms) },
    { label: "经血", items: asArray(record.bloodState).map((item) => bloodStateMap[item] || item) },
    { label: "习惯", items: asArray(record.habits) },
  ];
  return groups.filter((group) => group.items.length);
}

function reportSections(analysis: any): ReportSection[] {
  if (!analysis) return [];
  const fromObject = (label: string, obj: any): ReportSection | null => {
    if (!obj || typeof obj !== "object") return null;
    const title = String(obj.title || "");
    const content = String(obj.content || "");
    if (!title && !content) return null;
    return {
      label,
      title,
      content,
      tip: obj.suggestion ? `💡 ${obj.suggestion}` : obj.encouragement ? `🌷 ${obj.encouragement}` : obj.medicalAdvice ? `⚠️ ${obj.medicalAdvice}` : "",
      items: asArray(obj.suggestions),
    };
  };
  const result = [
    fromObject("周期报告", analysis.cycleReport),
    fromObject("睡眠", analysis.sleepReport),
    fromObject("体重", analysis.weightReport),
    fromObject("症状", analysis.symptomReport),
    fromObject("习惯", analysis.habitReport),
  ].filter(Boolean) as ReportSection[];
  if (analysis.nextCycle?.predictedStart || analysis.nextCycle?.expectations) {
    result.push({ label: "下月预测", title: String(analysis.nextCycle.predictedStart || ""), content: String(analysis.nextCycle.expectations || ""), tip: "", items: [] });
  }
  if (asArray(analysis.monthlyTips).length) result.push({ label: "本月小贴士", title: "", content: "", tip: "", items: asArray(analysis.monthlyTips) });
  if (asArray(analysis.riskWarnings).length) result.push({ label: "风险提示", title: "", content: "", tip: "", items: asArray(analysis.riskWarnings) });
  if (analysis.encouragement) result.push({ label: "医生寄语", title: "", content: String(analysis.encouragement), tip: "", items: [] });
  return result;
}

const joined = (raw: any) => asArray(raw).join(" · ") || "—";
const yesNo = (value: any) => value === true ? "是" : value === false ? "否" : "—";
const riskClass = (level?: string) => ({ URGENT: "urgent", CARE: "care", OBSERVE: "observe", NORMAL: "normal" }[level || ""] || "normal");
const riskLabel = (level?: string) => ({ URGENT: "紧急", CARE: "需关注", OBSERVE: "观察", NORMAL: "正常" }[level || ""] || level || "已评估");

function move(delta: number) {
  historyIndex.value = Math.min(Math.max(historyIndex.value + delta, 0), records.value.length - 1);
}

function onSwiperChange(event: any) {
  historyIndex.value = Number(event.detail.current) || 0;
}

function goPeriod() {
  uni.switchTab({ url: "/pages/period/index" });
}

onMounted(() => {
  if (!periodStore.records.length) void periodStore.loadAll();
});
</script>

<style lang="scss" scoped>
.history-page { min-height: 100vh; padding: $gap-md $gap-md 80rpx; overflow: hidden; }
.history-header { padding: $gap-lg $gap-xs; }
.history-title { display: block; color: $ink; font-size: $font-xl; font-weight: 800; letter-spacing: -1rpx; }
.history-sub { display: block; margin-top: 4rpx; color: $muted; font-size: $font-sm; }
.history-empty { display: flex; flex-direction: column; align-items: center; margin-top: 120rpx; padding: 64rpx $gap-xl; text-align: center; }
.history-empty__icon { font-size:64rpx; }
.history-empty__title { margin-top: $gap-md; color: $ink; font-size: $font-lg; font-weight: 800; }
.history-empty__desc { margin-top: $gap-xs; color: $muted; font-size: $font-sm; }
.history-empty__action { margin-top: $gap-lg; padding: 16rpx 40rpx; border-radius: $radius-full; background: $rose-dark; color: #fff; font-size: $font-sm; font-weight: 700; }
.history-nav { display: flex; align-items: center; gap: $gap-sm; margin-bottom: $gap-md; padding: 0 $gap-xs; }
.history-nav__btn { display: flex; width: 72rpx; height: 72rpx; flex: 0 0 auto; align-items: center; justify-content: center; border-radius: 50%; background: $paper; color: $rose-dark; font-size: 44rpx; box-shadow: $shadow-sm; }
.history-nav__btn.disabled { opacity: .3; pointer-events: none; }
.history-nav__dots { flex: 1; min-width: 0; white-space: nowrap; }
.history-nav__dots-inner { display: flex; min-width: 100%; align-items: center; justify-content: center; gap: 12rpx; }
.history-nav__dot { display: inline-block; width: 14rpx; height: 14rpx; border-radius: 999rpx; background: #e4e4e7; transition: width .2s; }
.history-nav__dot.active { width: 40rpx; background: $rose-dark; }
.history-swiper { width: 100%; transition: height .25s ease; }
.history-slide-scroll { height: 100%; }
.history-card { margin: 4rpx; padding: $gap-lg; }
.history-hero { display: flex; flex-direction: column; gap: $gap-sm; }
.history-hero__head { display: flex; align-items: center; justify-content: space-between; gap: $gap-sm; }
.history-hero__date { color: $rose-dark; font-size: $font-sm; font-weight: 700; }
.history-hero__duration { color: $muted; font-size: $font-xs; font-weight: 600; }
.history-hero__duration.live { color: $rose-dark; }
.history-hero__range { color: $ink; font-size: $font-xl; font-weight: 800; }
.history-metrics { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: $gap-sm; margin-top: $gap-lg; }
.history-metric { display: flex; min-width: 0; flex-direction: column; gap: 4rpx; padding: $gap-md; border-radius: $radius-md; background: $surface; }
.history-metric__label { color: $muted; font-size: $font-xs; }
.history-metric__value { color: $ink; font-size: $font-sm; font-weight: 700; word-break: break-all; }
.history-tags { display: flex; flex-direction: column; gap: $gap-md; margin-top: $gap-lg; padding-top: $gap-lg; border-top: 1px solid $line; }
.history-tags__row { display: flex; align-items: flex-start; gap: $gap-sm; }
.history-tags__label { width: 64rpx; flex: 0 0 auto; padding-top: 8rpx; color: $muted; font-size: $font-xs; font-weight: 700; }
.history-tags__items { display: flex; flex: 1; flex-wrap: wrap; gap: $gap-xs; }
.history-tag { padding: 6rpx 16rpx; border-radius: $radius-full; background: $blush; color: $rose-dark; font-size: $font-xs; font-weight: 600; }
.history-notes { margin-top: $gap-lg; padding: $gap-md; border-radius: $radius-md; background: $surface; color: $muted; font-size: $font-sm; line-height: 1.7; }
.history-ai-empty { display: flex; flex-direction: column; align-items: center; margin-top: $gap-lg; padding: 44rpx $gap-lg; border-radius: $radius-lg; background: $blush; color: $ink; font-size: $font-sm; text-align: center; }
.history-ai-empty__icon { margin-bottom:$gap-sm;font-size:48rpx; }
.history-ai-empty__sub { margin-top: $gap-xs; color: $muted; font-size: $font-xs; }
.report { margin-top: $gap-lg; padding: $gap-lg; border-radius: $radius-lg; background: $blush; }
.report--risk { border-left: 6rpx solid $rose-dark; }
.report--normal { border-left-color: #10b981; background: #ecfdf5; }
.report--observe { border-left-color: #f59e0b; background: #fffbeb; }
.report--care { border-left-color: #f97316; background: #fff7ed; }
.report--urgent { border-left-color: #ef4444; background: #fef2f2; }
.report__title { display: flex; align-items: center; justify-content: space-between; gap: $gap-sm; color: $ink; font-size: $font-md; font-weight: 800; }
.report__badge { padding: 6rpx 14rpx; border-radius: $radius-full; background: $rose-dark; color: #fff; font-size: 20rpx; font-weight: 800; }
.report__greeting { display: block; margin-top: $gap-md; color: $ink; font-size: $font-sm; line-height: 1.7; }
.risk-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: $gap-sm; margin-top: $gap-md; }
.risk-cell { display: flex; flex-direction: column; gap: 4rpx; padding: $gap-sm; border-radius: $radius-sm; background: rgba(255,255,255,.72); }
.risk-cell > text:first-child { color: $muted; font-size: 20rpx; font-weight: 700; }
.risk-cell strong { color: $ink; font-size: $font-xs; }
.risk-cell--full { grid-column: 1 / -1; }
.risk-cell--urgent { background: rgba(239,68,68,.1); }
.risk-cell__copy { color: $ink; font-size: $font-xs; line-height: 1.7; }
.report-section { display: flex; flex-direction: column; gap: 6rpx; margin-top: $gap-md; padding: $gap-md; border-radius: $radius-md; background: rgba(255,255,255,.72); }
.report-section__label { color: $rose-dark; font-size: 20rpx; font-weight: 800; }
.report-section__title { color: $ink; font-size: $font-sm; font-weight: 800; }
.report-section__copy, .report-section__item { color: $ink; font-size: $font-xs; line-height: 1.7; }
.report-section__tip { color: $rose-dark; font-size: $font-xs; line-height: 1.7; }
.history-counter { display: block; padding: $gap-md 0; color: $muted; font-size: $font-xs; text-align: center; }
</style>
