<template>
  <view class="page history-page">
    <ClinicTopBar show-back />
    <view class="history-header">
      <text class="history-title">临床周期档案库</text>
      <text class="history-sub">记录历史体征数据，辅助长期健康分析。</text>
    </view>

    <view v-if="periodStore.isLoading && !records.length" class="history-empty card">
      <text class="history-empty__title">正在检索档案数据...</text>
      <text class="history-empty__desc">请稍候，系统正在从数据库中提取。</text>
    </view>

    <view v-else-if="!records.length" class="history-empty card">
      <text class="history-empty__title">暂无周期归档</text>
      <text class="history-empty__desc">您还未在系统中产生历史周期记录。</text>
      <button class="btn-primary history-empty__action" @click="goPeriod">新建记录</button>
    </view>

    <template v-else>
      <view class="history-nav">
        <button class="btn-secondary nav-btn" :disabled="historyIndex === 0" @click="move(-1)">上一周期</button>
        <text class="nav-indicator">{{ historyIndex + 1 }} / {{ records.length }}</text>
        <button class="btn-secondary nav-btn" :disabled="historyIndex === records.length - 1" @click="move(1)">下一周期</button>
      </view>

      <swiper class="history-swiper" :style="{ height: swiperHeight }" :current="historyIndex" @change="onSwiperChange">
        <swiper-item v-for="record in records" :key="record.id">
          <scroll-view class="history-slide-scroll" scroll-y>
            <view class="history-card card">
              <view class="history-hero">
                <view class="history-hero__head">
                  <text class="history-hero__date">归档编号: {{ dateOnly(record.startDate).replace(/-/g, '') }}</text>
                  <text class="history-hero__duration" :class="{ live: !record.endDate }">
                    {{ record.endDate ? `持续 ${durationDays(record)} 天` : '监测进行中' }}
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

              <view v-if="record.notes" class="history-notes">
                <text class="history-notes__label">临床备注：</text>
                <text>{{ record.notes }}</text>
              </view>

              <view v-if="!analysisOf(record) && !riskOf(record)" class="history-ai-empty">
                <text class="history-ai-empty__title">暂无 AI 分析评估报告</text>
                <text class="history-ai-empty__sub">该周期尚未生成智能分析报告，可通过咨询助手获取。</text>
              </view>

              <view v-if="riskOf(record)" class="report report--risk" :class="`report--${riskClass(riskOf(record)?.riskLevel)}`">
                <view class="report__title">
                  <text>临床风险评估报告</text>
                  <text class="report__badge">{{ riskLabel(riskOf(record)?.riskLevel) }}</text>
                </view>
                <view class="risk-grid">
                  <view class="risk-cell">
                    <text class="risk-cell__label">紧急程度</text>
                    <strong>{{ yesNo(riskOf(record)?.canWaitUntilNextDay) }}</strong>
                  </view>
                  <view class="risk-cell">
                    <text class="risk-cell__label">建议就诊科室</text>
                    <strong>{{ riskOf(record)?.recommendedDepartment || '—' }}</strong>
                  </view>
                  <view class="risk-cell risk-cell--full">
                    <text class="risk-cell__label">识别到的风险信号</text>
                    <strong>{{ joined(riskOf(record)?.riskSignals) }}</strong>
                  </view>
                  <view class="risk-cell risk-cell--full">
                    <text class="risk-cell__label">建议辅助检查</text>
                    <strong>{{ joined(riskOf(record)?.recommendedChecks) }}</strong>
                  </view>
                  <view v-if="riskOf(record)?.medicalAdvice" class="risk-cell risk-cell--full">
                    <text class="risk-cell__label">医疗建议</text>
                    <text class="risk-cell__copy">{{ riskOf(record)?.medicalAdvice }}</text>
                  </view>
                  <view v-if="riskOf(record)?.urgentAction" class="risk-cell risk-cell--full risk-cell--urgent">
                    <text class="risk-cell__label">紧急干预措施</text>
                    <text class="risk-cell__copy">{{ riskOf(record)?.urgentAction }}</text>
                  </view>
                </view>
              </view>

              <view v-if="analysisOf(record)" class="report report--analysis">
                <view class="report__title">
                  <text>综合体征分析报告</text>
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
const moodMap: Record<string, string> = { happy: "开心", anxious: "焦虑", blue: "忧伤", angry: "火大", other: "其他" };
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
  return record.endDate ? `${dateOnly(record.startDate)} ~ ${dateOnly(record.endDate)}` : `${dateOnly(record.startDate)} 起监测中`;
}

function timeOnly(value?: string) {
  if (!value) return "";
  const date = new Date(value);
  return Number.isNaN(date.getTime()) ? "" : `${String(date.getHours()).padStart(2, "0")}:${String(date.getMinutes()).padStart(2, "0")}`;
}

function metricsOf(record: MenstrualRecord) {
  const flow = [flowLevelMap[record.flowLevel || ""] || "—", flowDetailMap[record.flowDetail || ""]].filter(Boolean).join(" · ");
  const sleepTimes = record.sleepStart && record.sleepEnd ? `${timeOnly(record.sleepStart)} ~ ${timeOnly(record.sleepEnd)}` : "";
  const sleep = [sleepMap[record.sleepQuality || ""] || "—", sleepTimes, record.sleepHours != null ? `${record.sleepHours}h` : ""].filter(Boolean).join(" · ");
  return [
    { label: "出血量评估", value: flow },
    { label: "VAS疼痛评分", value: painMap[record.painLevel || ""] || "无痛" },
    { label: "主要伴随症状", value: discomfortMap[record.discomfort || "NONE"] || "—" },
    { label: "睡眠质量评估", value: sleep },
    { label: "血液颜色表征", value: bloodColorMap[record.bloodColor || ""] || "—" },
    { label: "体重指标", value: record.weight != null ? `${record.weight} kg` : "—" },
    { label: "基础体温(BBT)", value: record.bodyTemp != null ? `${record.bodyTemp}°C` : "—" },
    { label: "心理状态", value: moodMap[record.mood || ""] || "—" },
  ];
}

function tagGroups(record: MenstrualRecord) {
  const groups = [
    { label: "体征", items: asArray(record.symptoms) },
    { label: "性状", items: asArray(record.bloodState).map((item) => bloodStateMap[item] || item) },
    { label: "干预", items: asArray(record.habits) },
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
      tip: obj.suggestion ? `临床建议: ${obj.suggestion}` : obj.medicalAdvice ? `⚠️ ${obj.medicalAdvice}` : "",
      items: asArray(obj.suggestions),
    };
  };
  const result = [
    fromObject("周期回顾", analysis.cycleReport),
    fromObject("睡眠分析", analysis.sleepReport),
    fromObject("体征分析", analysis.symptomReport),
    fromObject("干预反馈", analysis.habitReport),
  ].filter(Boolean) as ReportSection[];
  if (analysis.nextCycle?.predictedStart || analysis.nextCycle?.expectations) {
    result.push({ label: "预测模型", title: String(analysis.nextCycle.predictedStart || ""), content: String(analysis.nextCycle.expectations || ""), tip: "", items: [] });
  }
  if (asArray(analysis.riskWarnings).length) result.push({ label: "风险提示", title: "", content: "", tip: "", items: asArray(analysis.riskWarnings) });
  return result;
}

const joined = (raw: any) => asArray(raw).join("、") || "—";
const yesNo = (value: any) => value === true ? "是" : value === false ? "否" : "—";
const riskClass = (level?: string) => ({ URGENT: "urgent", CARE: "care", OBSERVE: "observe", NORMAL: "normal" }[level || ""] || "normal");
const riskLabel = (level?: string) => ({ URGENT: "紧急预警", CARE: "重点关注", OBSERVE: "临床观察", NORMAL: "指标正常" }[level || ""] || level || "已评估");

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
.history-header { padding: $gap-lg 0 $gap-md; }
.history-title { display: block; color: $ink; font-size: $font-xl; font-weight: 800; }
.history-sub { display: block; margin-top: 4rpx; color: $muted; font-size: $font-sm; }
.history-empty { display: flex; flex-direction: column; align-items: center; margin-top: $gap-xl; padding: 64rpx $gap-xl; text-align: center; }
.history-empty__title { color: $ink; font-size: $font-lg; font-weight: 700; }
.history-empty__desc { margin-top: 12rpx; color: $muted; font-size: $font-sm; }
.history-empty__action { margin-top: $gap-lg; }

.history-nav { display: flex; align-items: center; justify-content: space-between; margin-bottom: $gap-md; }
.nav-btn { margin: 0; padding: 8rpx 24rpx; font-size: 24rpx; }
.nav-btn[disabled] { opacity: 0.5; }
.nav-indicator { font-size: $font-sm; font-weight: 700; color: $ink; }

.history-swiper { width: 100%; transition: height .25s ease; }
.history-slide-scroll { height: 100%; }

.history-card { margin-bottom: $gap-md; }
.history-hero { display: flex; flex-direction: column; gap: $gap-sm; border-bottom: 1px solid $line; padding-bottom: $gap-md; margin-bottom: $gap-md;}
.history-hero__head { display: flex; align-items: center; justify-content: space-between; }
.history-hero__date { color: $muted; font-size: 22rpx; font-weight: 600; }
.history-hero__duration { color: $muted; font-size: 22rpx; font-weight: 600; padding: 4rpx 12rpx; background: $surface; border-radius: 4rpx;}
.history-hero__duration.live { color: $rose-dark; background: $blush; }
.history-hero__range { color: $ink; font-size: $font-lg; font-weight: 700; }

.history-metrics { display: grid; grid-template-columns: repeat(2, 1fr); gap: 1px; background: $line; border: 1px solid $line; border-radius: $radius-sm; overflow: hidden;}
.history-metric { display: flex; flex-direction: column; padding: 16rpx; background: $paper; }
.history-metric__label { color: $muted; font-size: 22rpx; }
.history-metric__value { color: $ink; font-size: 24rpx; font-weight: 600; margin-top: 4rpx;}

.history-tags { display: flex; flex-direction: column; gap: 16rpx; margin-top: $gap-md; padding-top: $gap-md; border-top: 1px dashed $line; }
.history-tags__row { display: flex; align-items: flex-start; gap: 16rpx; }
.history-tags__label { width: 72rpx; flex: 0 0 auto; color: $muted; font-size: 22rpx; font-weight: 600; line-height: 40rpx;}
.history-tags__items { display: flex; flex: 1; flex-wrap: wrap; gap: 12rpx; }
.history-tag { padding: 4rpx 12rpx; border: 1px solid $line; border-radius: 4rpx; background: $surface; color: $ink; font-size: 22rpx; }

.history-notes { margin-top: $gap-md; padding: 16rpx; border-radius: $radius-sm; background: $surface; color: $ink; font-size: $font-sm; line-height: 1.6; }
.history-notes__label { font-weight: 600; color: $muted; }

.history-ai-empty { margin-top: $gap-md; padding: 32rpx; border: 1px dashed $line; border-radius: $radius-md; text-align: center; }
.history-ai-empty__title { display: block; font-size: $font-sm; font-weight: 600; color: $muted; }
.history-ai-empty__sub { display: block; margin-top: 8rpx; font-size: 22rpx; color: $muted; }

.report { margin-top: $gap-md; padding: $gap-md; border: 1px solid $line; border-radius: $radius-md; background: $paper; }
.report--risk { border-top: 4px solid #ef4444; }
.report--normal { border-top-color: #10b981; }
.report--observe { border-top-color: #f59e0b; }
.report--care { border-top-color: #f97316; }
.report--urgent { border-top-color: #ef4444; background: #fef2f2; }
.report--analysis { border-top: 4px solid $rose-dark; }

.report__title { display: flex; align-items: center; justify-content: space-between; font-size: $font-md; font-weight: 700; color: $ink; padding-bottom: 16rpx; border-bottom: 1px solid $line; margin-bottom: 16rpx;}
.report__badge { padding: 4rpx 12rpx; border-radius: 4rpx; background: $ink; color: #fff; font-size: 20rpx; font-weight: 600; }
.report--risk .report__badge { background: #ef4444; }

.risk-grid { display: flex; flex-direction: column; gap: 16rpx; }
.risk-cell { display: flex; flex-direction: column; padding: 12rpx; background: $surface; border-radius: 4rpx;}
.risk-cell__label { color: $muted; font-size: 22rpx; margin-bottom: 4rpx;}
.risk-cell strong { color: $ink; font-size: 24rpx; }
.risk-cell--urgent { background: rgba(239, 68, 68, 0.1); border: 1px solid rgba(239, 68, 68, 0.2); }
.risk-cell__copy { color: $ink; font-size: 24rpx; line-height: 1.6; }

.report-section { display: flex; flex-direction: column; margin-bottom: $gap-md; }
.report-section:last-child { margin-bottom: 0; }
.report-section__label { color: $rose-dark; font-size: 22rpx; font-weight: 700; margin-bottom: 4rpx; }
.report-section__title { color: $ink; font-size: 26rpx; font-weight: 600; margin-bottom: 8rpx;}
.report-section__copy, .report-section__item { color: $ink; font-size: 24rpx; line-height: 1.6; }
.report-section__tip { margin-top: 8rpx; padding: 8rpx; background: $blush; color: $rose-dark; font-size: 22rpx; border-radius: 4rpx; }
</style>
