<template>
  <view class="page home-page">
    <ClinicTopBar />
    
    <view class="home-hero">
      <view class="home-hero__greeting">
        <text class="home-hero__name">{{ userStore.displayName || '医师 / 患者' }} 工作台</text>
      </view>
      <text class="home-hero__date">{{ todayStr }}</text>
    </view>

    <!-- 临床健康数据仪表盘 -->
    <view class="dashboard-module card">
      <view class="module-header">
        <text class="module-title">今日指征数据采集</text>
        <text class="module-meta">
          <text v-if="totalRecordedDays > 0" class="text-rose">已连续跟踪 {{ totalRecordedDays }} 天</text>
          <text v-else class="text-muted">暂无跟踪数据</text>
        </text>
      </view>

      <view class="task-sections">
        <!-- 已归档数据 -->
        <view class="task-section">
          <view class="task-section__head">
            <text class="task-section__label">已归档记录</text>
          </view>
          <view class="task-chips">
            <view v-for="item in doneChips" :key="item.key" class="task-chip task-chip--done">
              <text class="task-chip__text">{{ item.label }}</text>
            </view>
            <text v-if="!doneChips.length" class="task-empty">尚未提交今日数据</text>
          </view>
        </view>
        
        <!-- 待采数据 -->
        <view class="task-section">
          <view class="task-section__head">
            <text class="task-section__label text-rose">待采集项</text>
          </view>
          <view class="task-chips">
            <view
              v-for="item in todoChips"
              :key="item.key"
              class="task-chip task-chip--todo"
              :class="{ 'task-chip--primary': item.primary }"
              hover-class="task-chip--pressed"
              :hover-stay-time="120"
              @click="openQuickRecord(item)"
            >
              <text class="task-chip__icon">+</text>
              <text class="task-chip__text">{{ item.label }}</text>
            </view>
            <text v-if="!todoChips.length" class="task-empty">所有关键指征已完成采集</text>
          </view>
        </view>
      </view>
    </view>

    <!-- 快捷功能矩阵 -->
    <view class="home-grid">
      <view class="home-grid__item card" @click="goTab('period')">
        <view class="home-grid__icon">📅</view>
        <text class="home-grid__title">周期追踪</text>
        <text class="home-grid__desc">规律数据分析</text>
      </view>
      <view class="home-grid__item card" @click="goSelfCheck">
        <view class="home-grid__icon">📋</view>
        <text class="home-grid__title">症状评估</text>
        <text class="home-grid__desc">AI辅助分诊</text>
      </view>
      <view class="home-grid__item card" @click="goTab('chat')">
        <view class="home-grid__icon">💬</view>
        <text class="home-grid__title">智能咨询</text>
        <text class="home-grid__desc">全天候健康解答</text>
      </view>
    </view>

    <!-- 挂号/服务卡片 -->
    <view class="action-card card" @click="goPage('booking')">
      <view class="action-card__info">
        <text class="action-card__title">门诊挂号与服务预约</text>
        <text class="action-card__desc">快速预约复诊或线下检查项目</text>
      </view>
      <button class="btn-primary action-card__btn">立刻预约</button>
    </view>

    <view v-if="marketingPosts.length" class="section-title">诊所公告与随访计划</view>
    <view v-for="post in marketingPosts.slice(0, 3)" :key="post.id" class="notice-card card" @click="openMarketing(post)">
      <view class="notice-card__content">
        <text class="notice-card__tag">{{ post.type === "PROMOTION" ? "随访" : post.type === "ACTIVITY" ? "宣教" : "通知" }}</text>
        <text class="notice-card__title">{{ post.title }}</text>
        <text class="notice-card__copy">{{ post.subtitle || post.activityInfo || post.introText || "" }}</text>
      </view>
      <text class="notice-card__arrow">›</text>
    </view>

    <!-- 加载 -->
    <view v-if="loading" class="loading-state">
      <text>同步数据中...</text>
    </view>

    <!-- 快捷记录弹窗 -->
    <view v-if="periodStore.activeSheet && periodStore.activeSheet !== 'stateSwitch' && periodStore.activeSheet !== 'modeSettings' && periodStore.activeSheet !== 'dayDetail' && periodStore.activeSheet !== 'startConfirm' && periodStore.activeSheet !== 'endConfirm'" class="overlay" @click="closeHomeSheet">
      <view class="sheet-dialog" @click.stop>
        <view class="sheet-dialog__header">
          <text class="sheet-dialog__title">{{ homeSheetTitle }}</text>
          <text class="sheet-dialog__close" @click="closeHomeSheet">×</text>
        </view>
        <scroll-view class="sheet-dialog__body" scroll-y>
          <ChoiceSection
            v-for="f in homeSheetFields"
            :key="f.key"
            :title="f.label"
            :options="f.options"
            :field="f.key"
            :value="form[f.key]"
            :multi="f.multi"
            :visual="f.visual || 'icon'"
            @select="selectChoice"
          />
          <template v-if="periodStore.activeSheet === 'weight'">
            <view class="form-section">
              <text class="form-section__title">体重数据 (kg)</text>
              <input v-model="form.weight" type="digit" class="form-input" placeholder="请输入精确数值" />
            </view>
          </template>
          <template v-if="periodStore.activeSheet === 'bodyTemp'">
            <view class="form-section">
              <text class="form-section__title">基础体温 (℃)</text>
              <input v-model="form.bodyTemp" type="digit" class="form-input" placeholder="请输入精确数值" />
            </view>
          </template>
        </scroll-view>
        <view class="sheet-dialog__footer">
          <button class="btn-primary sheet-dialog__btn" :loading="saving" @click="saveHomeSheet">确认保存</button>
        </view>
      </view>
    </view>

    <!-- 公告弹窗 -->
    <view v-if="activeMarketing" class="overlay" @click="activeMarketing = null">
      <view class="sheet-dialog" @click.stop>
        <view class="sheet-dialog__header">
          <text class="sheet-dialog__tag">{{ activeMarketing.type === "PROMOTION" ? "随访" : activeMarketing.type === "ACTIVITY" ? "宣教" : "通知" }}</text>
          <text class="sheet-dialog__close" @click="activeMarketing = null">×</text>
        </view>
        <view class="sheet-dialog__body">
          <text class="sheet-dialog__title">{{ activeMarketing.title || "门诊消息" }}</text>
          <text class="sheet-dialog__summary">{{ activeMarketing.subtitle || activeMarketing.introText || "" }}</text>
          <scroll-view class="sheet-dialog__scroll" scroll-y><text class="sheet-dialog__text">{{ activeMarketing.bodyText || activeMarketing.activityInfo || activeMarketing.introText || activeMarketing.subtitle || "暂无详细内容" }}</text></scroll-view>
        </view>
        <view class="sheet-dialog__footer">
          <button class="btn-primary sheet-dialog__btn" @click="activeMarketing = null">关闭</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted } from "vue";
import { useUserStore } from "@/stores/useUserStore";
import { usePeriodStore } from "@/stores/usePeriodStore";
import { useAppStore } from "@/stores/useAppStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";
import ChoiceSection from "@/components/ChoiceSection.vue";

type Mode = "PERIOD" | "PREPARE" | "PREGNANT" | "MOM";
type TodoKey = "sleep" | "mood" | "habits" | "symptom" | "period" | "weight" | "bodyTemp" | "leucorrhea" | "coc" | "intercourse" | "diet" | "ovuTest" | "pregTest" | "babyFeed";

const userStore = useUserStore();
const periodStore = usePeriodStore();
const appStore = useAppStore();

const loading = ref(false);
const activeMarketing = ref<any | null>(null);
const form = reactive<Record<string, any>>({});
const marketingPosts = computed<any[]>(() => appStore.dashboard?.marketingPosts ?? []);

const totalRecordedDays = computed(() => {
  const dateSet = new Set<string>();
  for (const r of periodStore.records) {
    const d = String((r as any).recordDate || r.startDate).slice(0, 10);
    if (d && d !== "Invalid Date") dateSet.add(d);
  }
  for (const log of periodStore.symptomLogs) {
    const d = String(log.occurredAt).slice(0, 10);
    if (d) dateSet.add(d);
  }
  return dateSet.size;
});

const moodLabels: Record<string, string> = {
  happy: "开心", calm: "平静", tired: "疲倦",
  anxious: "焦虑", blue: "低落", angry: "烦躁",
};
const ovuLabels: Record<string, string> = {
  negative: "阴性", weak_positive: "弱阳", positive: "阳性", strong_positive: "强阳",
};
const pregLabels: Record<string, string> = {
  negative: "阴性", weak_positive: "弱阳", positive: "阳性",
};

function labelOf(map: Record<string, string>, value: unknown) {
  return map[String(value)] ?? String(value ?? "");
}

const todayStr = computed(() => {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
});

const currentMode = computed<Mode>(() => (periodStore.menstrualState?.mode || "PERIOD") as Mode);

const ongoingPeriod = computed(() => periodStore.records.find((record: any) => {
  if (record.endDate) return false;
  const d = String(record.recordDate || record.startDate).slice(0, 10);
  if (d > isoToday()) return false;
  const dailyFields = [record.mood, record.sleepQuality, record.sleepHours, record.weight,
    record.bodyTemp, record.ovuTestResult, record.pregTestResult, record.leuColor,
    record.notes, record.babyFeed, record.babyCry, record.intercourse, record.exercise];
  const isStartRecord = (record.painLevel ?? "NONE") === "NONE"
    && (record.flowLevel ?? "MEDIUM") === "MEDIUM"
    && (record.discomfort ?? "NONE") === "NONE"
    && dailyFields.every((v: any) => v == null);
  const hasSymptoms = (record.flowLevel && record.flowLevel !== "MEDIUM")
    || (record.painLevel && record.painLevel !== "NONE")
    || (record.discomfort && record.discomfort !== "NONE")
    || !!record.bloodColor;
  return isStartRecord || hasSymptoms;
}));

const inFertile = computed(() => {
  const pred = periodStore.cyclePrediction;
  if (!pred?.fertileWindow) return false;
  const today = isoToday();
  return today >= pred.fertileWindow.start && today <= pred.fertileWindow.end;
});

const todayRecord = computed<any>(() => {
  const today = isoToday();
  return periodStore.findRecordForDate(today, today);
});

const todaySymptomCount = computed(() => periodStore.symptomLogs.filter((log: any) => String(log.occurredAt).slice(0, 10) === isoToday()).length);

const doneChips = computed(() => {
  const r: any = todayRecord.value ?? {};
  const out: { key: string; icon: string; label: string }[] = [];
  if (r.sleepQuality || r.sleepStart) out.push({ key: "sleep", icon: "😴", label: "睡眠数据" });
  if (r.mood) out.push({ key: "mood", icon: "😊", label: `心情: ${labelOf(moodLabels, r.mood)}` });
  if (Array.isArray(r.habits) && r.habits.length) out.push({ key: "habits", icon: "🏃", label: `生活打卡: ${r.habits.length}项` });
  if (currentMode.value === "MOM" && r.babyFeed) out.push({ key: "babyFeed", icon: "🍼", label: `育儿数据` });
  if (r.weight) out.push({ key: "weight", icon: "⚖️", label: `体重: ${r.weight}kg` });
  if (r.bodyTemp) out.push({ key: "bodyTemp", icon: "🌡️", label: `体温: ${r.bodyTemp}℃` });
  if (r.ovuTestResult) out.push({ key: "ovuTest", icon: "🔬", label: `排卵: ${labelOf(ovuLabels, r.ovuTestResult)}` });
  if (r.pregTestResult) out.push({ key: "pregTest", icon: "🤰", label: `早孕: ${labelOf(pregLabels, r.pregTestResult)}` });
  if (todaySymptomCount.value) out.push({ key: "symptom", icon: "📝", label: `症状表现: ${todaySymptomCount.value}项` });
  return out;
});

const todoChips = computed(() => {
  const r: any = todayRecord.value ?? {};
  const out: { key: TodoKey; icon: string; label: string; primary?: boolean }[] = [];
  const pushIf = (key: TodoKey, icon: string, label: string, cond: boolean, primary = false) => {
    if (cond) out.push({ key, icon, label, primary });
  };

  pushIf("mood", "😊", "记录心情状态", !r.mood, true);
  pushIf("symptom", "📝", "记录今日症状", todaySymptomCount.value === 0);

  if (currentMode.value !== "PREGNANT") {
    pushIf("sleep", "😴", "记录睡眠情况", !r.sleepQuality && !r.sleepStart);
    pushIf("habits", "🏃", "日常行为打卡", !Array.isArray(r.habits) || r.habits.length === 0);
  }

  if (currentMode.value === "MOM") {
    pushIf("babyFeed", "🍼", "婴儿成长数据", !r.babyFeed && !r.babyCry, true);
  } else if (currentMode.value === "PREGNANT") {
    pushIf("weight", "⚖️", "测量体重", !r.weight);
  } else if (currentMode.value === "PREPARE") {
    if (inFertile.value) {
      pushIf("bodyTemp", "🌡️", "录入基础体温", !r.bodyTemp, true);
      pushIf("ovuTest", "🔬", "排卵试纸结果", !r.ovuTestResult);
    } else {
      pushIf("intercourse", "💕", "性生活记录", !r.intercourse);
    }
  } else {
    if (ongoingPeriod.value) {
      pushIf("period", "🩹", "经期指征评估", !(r.painLevel || r.flowLevel), true);
    } else {
      pushIf("leucorrhea", "💧", "白带性状观察", !(r.leuColor || r.whiteLeucorrhea));
    }
  }

  return out;
});

function isoToday() {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
}

function goTab(tab: string) { uni.switchTab({ url: `/pages/${tab}/index` }); }
function goPage(page: string) { uni.navigateTo({ url: `/pages/${page}/index` }); }
function goSelfCheck() { uni.navigateTo({ url: "/pages/services/index?type=selfcheck" }); }
function openMarketing(post: any) { activeMarketing.value = post; }

type Option = { value: string | number; label: string; icon?: string };
function opt(list: string[]): Option[] { return list.map((v) => ({ value: v, label: v })); }

const sheetMoodOptions: Option[] = [
  { value: "happy", label: "开心" }, { value: "calm", label: "平静" }, { value: "tired", label: "疲倦" },
  { value: "anxious", label: "焦虑" }, { value: "blue", label: "低落" }, { value: "angry", label: "烦躁" },
];
const sheetSleepOptions: Option[] = [
  { value: "POOR", label: "差" }, { value: "NORMAL", label: "一般" }, { value: "GOOD", label: "好" }, { value: "EXCELLENT", label: "很好" },
];
const sheetPainOptions: Option[] = [
  { value: "NONE", label: "无痛" }, { value: "MILD", label: "轻微" }, { value: "MODERATE", label: "中等" }, { value: "SEVERE", label: "较重" },
];
const sheetFlowOptions: Option[] = [
  { value: "LIGHT", label: "量少" }, { value: "MEDIUM", label: "适中" }, { value: "HEAVY", label: "量多" },
];
const sheetDiscomfortOptions: Option[] = [
  { value: "NONE", label: "无不适" }, { value: "BLOATING", label: "腹胀" }, { value: "CRAMPS", label: "痉挛" },
  { value: "HEADACHE", label: "头痛" }, { value: "FATIGUE", label: "乏力" }, { value: "MOOD", label: "情绪波动" },
];
const sheetOvuOptions: Option[] = [
  { value: "negative", label: "阴性" }, { value: "weak_positive", label: "弱阳" }, { value: "positive", label: "阳性" }, { value: "strong_positive", label: "强阳" },
];
const sheetPregOptions: Option[] = [
  { value: "negative", label: "阴性" }, { value: "weak_positive", label: "弱阳" }, { value: "positive", label: "阳性" },
];
const sheetLeuColorOptions: Option[] = [
  { value: "clear", label: "透明无色" }, { value: "white", label: "乳白色" }, { value: "yellow", label: "偏黄色" }, { value: "brown", label: "褐/咖啡色" },
];
const sheetLeuStateOptions: Option[] = [
  { value: "normal", label: "正常状态" }, { value: "egg_white", label: "像蛋清" }, { value: "thick", label: "有点粘稠" }, { value: "watery", label: "像水一样" },
];
const sheetLeuOtherOptions: Option[] = [
  { value: "none", label: "完全没有" }, { value: "smell", label: "有异味" }, { value: "itchy", label: "发痒" }, { value: "burning", label: "灼热感" },
];
const sheetIntercourseOptions: Option[] = [{ value: "yes", label: "有" }, { value: "no", label: "没有" }];
const sheetCocOptions: Option[] = [{ value: "condom", label: "避孕套" }, { value: "emergency", label: "紧急避孕药" }, { value: "short_term", label: "短效避孕药" }, { value: "none", label: "没有" }];
const sheetHabitsOptions: Option[] = opt(["多喝水", "吃水果", "清淡饮食", "补充蛋白", "规律运动", "散步", "早睡", "午休", "热敷", "泡脚", "放松冥想", "不喝冰饮"]);
const sheetSymptomsOptions: Option[] = opt(["腹胀", "腹痛", "腰酸", "头痛", "头晕", "乏力", "乳房胀痛", "恶心", "腹泻", "便秘", "长痘", "浮肿"]);
const sheetBabyFeedOptions: Option[] = opt(["母乳", "配方奶", "混合喂养", "辅食"]);
const sheetBabyCryOptions: Option[] = opt(["正常状态", "轻微哭闹", "剧烈哭闹", "持续烦躁"]);
const sheetDietOptions: Option[] = opt(["叶酸", "复合维他命", "钙片", "铁剂", "高蛋白", "绿叶菜", "鲜水果", "全谷物", "坚果", "忌冷饮", "无咖啡", "无酒精", "无二手烟"]);

interface SheetField { key: string; label: string; options: Option[]; multi?: boolean; visual?: string }
const sheetFieldsMap: Record<string, SheetField[]> = {
  sleep: [{ key: "sleepQuality", label: "睡眠质量评估", options: sheetSleepOptions }],
  notes: [{ key: "mood", label: "心理状态评估", options: sheetMoodOptions }],
  discomfort: [
    { key: "painLevel", label: "疼痛等级", options: sheetPainOptions },
    { key: "flowLevel", label: "经血流量评估", options: sheetFlowOptions },
    { key: "discomfort", label: "伴随症状", options: sheetDiscomfortOptions },
  ],
  habits: [{ key: "habitsArr", label: "生活方式干预执行", options: sheetHabitsOptions, multi: true }],
  symptoms: [{ key: "symptomsArr", label: "体征监测", options: sheetSymptomsOptions, multi: true }],
  intercourse: [{ key: "intercourse", label: "性生活记录", options: sheetIntercourseOptions }],
  leucorrhea: [
    { key: "leuColor", label: "分泌物颜色", options: sheetLeuColorOptions },
    { key: "leuState", label: "分泌物性状", options: sheetLeuStateOptions },
    { key: "leuOther", label: "其他伴随症状", options: sheetLeuOtherOptions },
  ],
  coc: [{ key: "coc", label: "避孕措施", options: sheetCocOptions }],
  ovuTest: [{ key: "ovuTestResult", label: "排卵监测结果", options: sheetOvuOptions }],
  pregTest: [{ key: "pregTestResult", label: "HCG检测结果", options: sheetPregOptions }],
  babyRecord: [
    { key: "babyFeed", label: "喂养情况", options: sheetBabyFeedOptions },
    { key: "babyCry", label: "情绪及啼哭评估", options: sheetBabyCryOptions },
  ],
  prepareDiet: [{ key: "dietTags", label: "营养补充与饮食管理", options: sheetDietOptions, multi: true }],
};
const sheetTitles: Record<string, string> = {
  sleep: "睡眠评估", notes: "心理评估", discomfort: "经期指征", habits: "生活干预", symptoms: "体征监测",
  intercourse: "性生活", leucorrhea: "分泌物监测", coc: "避孕档案", ovuTest: "排卵监测",
  pregTest: "HCG检测", babyRecord: "育儿档案", prepareDiet: "备孕营养",
};

const homeSheetFields = computed<SheetField[]>(() => sheetFieldsMap[periodStore.activeSheet ?? ""] ?? []);
const homeSheetTitle = computed(() => sheetTitles[periodStore.activeSheet ?? ""] || "数据录入");

const chipToSheet: Record<string, string> = {
  mood: "notes", sleep: "sleep", habits: "habits", symptom: "symptoms",
  period: "discomfort", weight: "weight", bodyTemp: "bodyTemp",
  leucorrhea: "leucorrhea", coc: "coc", intercourse: "intercourse",
  ovuTest: "ovuTest", pregTest: "pregTest", babyFeed: "babyRecord",
  diet: "prepareDiet",
};

const saving = ref(false);

function openQuickRecord(item: { key: TodoKey; primary?: boolean }) {
  const sheet = chipToSheet[item.key];
  const date = isoToday();
  if (sheet === "weight" || sheet === "bodyTemp") {
    periodStore.openSheet(sheet as any, date, {});
    return;
  }
  if (!sheet || !sheetFieldsMap[sheet]) {
    periodStore.saveDailyRecord(date, { recordDate: date }, date).then(() => {
      appStore.showToast("已归档", "success");
    }).catch((e: any) => {
      appStore.showToast(e?.message || "操作失败", "error");
    });
    return;
  }
  periodStore.openSheet(sheet as any, date, {});
}

function selectChoice({ field, value, multi }: { field: string; value: string | number; multi: boolean }) {
  if (multi) {
    const arr = (form[field] ?? []) as (string | number)[];
    const idx = arr.indexOf(value);
    if (idx >= 0) arr.splice(idx, 1); else arr.push(value);
    form[field] = [...arr];
  } else {
    form[field] = value;
  }
}

async function saveHomeSheet() {
  if (saving.value) return;
  saving.value = true;
  const date = isoToday();
  const sheet = periodStore.activeSheet;
  try {
    if (sheet === "weight") {
      await periodStore.saveDailyRecord(date, { weight: Number(form.weight || 0), recordDate: date }, date);
    } else if (sheet === "bodyTemp") {
      await periodStore.saveDailyRecord(date, { bodyTemp: Number(form.bodyTemp || 0), recordDate: date }, date);
    } else if (sheet === "symptoms") {
      for (const tag of (form.symptomsArr ?? [])) {
        await periodStore.createSymptomLog({ occurredAt: `${date}T12:00:00`, tags: [tag], severity: 3, category: "PHYSICAL", notes: "" });
      }
    } else {
      const body: Record<string, any> = { recordDate: date };
      const fields = sheetFieldsMap[sheet ?? ""] ?? [];
      for (const f of fields) {
        if (form[f.key] !== undefined && form[f.key] !== "") {
          body[f.key] = form[f.key];
        }
      }
      if (sheet === "intercourse") body.intercourse = form.intercourse === "yes";
      if (sheet === "habits") body.habits = form.habitsArr ?? [];
      if (sheet === "leucorrhea") body.whiteLeucorrhea = [form.leuColor, form.leuState, form.leuOther].filter(Boolean);
      if (sheet === "discomfort") body.bloodState = form.bloodStateTags ?? [];
      if (sheet === "coc") body.cocType = form.coc;
      if (sheet === "prepareDiet") { body.dietTags = form.dietTags ?? []; }
      await periodStore.saveDailyRecord(date, body, date);
    }
    appStore.showToast("数据已保存", "success");
    closeHomeSheet();
  } catch (e: any) {
    appStore.showToast(e?.message || "数据保存失败", "error");
  } finally {
    saving.value = false;
  }
}

function closeHomeSheet() {
  Object.keys(form).forEach((k) => delete form[k]);
  periodStore.activeSheet = null as any;
}

onMounted(async () => {
  loading.value = true;
  try {
    await Promise.all([
      appStore.loadDashboard(true),
      periodStore.loadAll(),
    ]);
  } catch {
    // 静默处理
  } finally {
    loading.value = false;
  }
});
</script>

<style lang="scss" scoped>
.home-page { padding: $gap-md $gap-md $gap-xl; min-height: 100vh; }

.home-hero {
  padding: $gap-lg $gap-sm;
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
}
.home-hero__greeting { display: flex; align-items: center; }
.home-hero__name { font-size: $font-xl; font-weight: 700; color: $ink; }
.home-hero__date { font-size: $font-sm; color: $muted; font-weight: 500; }

.dashboard-module {
  margin-bottom: $gap-md;
}
.module-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid $line;
  padding-bottom: $gap-sm;
  margin-bottom: $gap-sm;
}
.module-title { font-size: $font-lg; font-weight: 700; color: $ink; }
.module-meta { font-size: $font-xs; font-weight: 600; }

.task-sections { display: flex; flex-direction: column; gap: $gap-md; }
.task-section__head { margin-bottom: $gap-xs; }
.task-section__label { font-size: 24rpx; font-weight: 700; color: $muted; }

.task-chips { display: flex; flex-wrap: wrap; gap: 16rpx; }
.task-chip {
  display: inline-flex;
  align-items: center;
  gap: 8rpx;
  padding: 12rpx 20rpx;
  border-radius: $radius-sm;
  font-size: 24rpx;
  font-weight: 600;
  border: 1px solid transparent;
  background: $surface;
  transition: all 0.2s;
}
.task-chip--done {
  background: #f0fdf4;
  color: #166534;
  border-color: #bbf7d0;
}
.task-chip--todo {
  background: $paper;
  color: $ink;
  border-color: $line;
}
.task-chip--primary {
  background: $blush;
  color: $rose-dark;
  border-color: $rose-dark;
}
.task-chip--pressed { opacity: 0.8; transform: scale(0.98); }
.task-empty { font-size: 24rpx; color: $muted; font-style: italic; }

.home-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: $gap-md;
  margin-bottom: $gap-md;
}
.home-grid__item {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: $gap-lg $gap-sm;
}
.home-grid__icon { font-size: 40rpx; margin-bottom: 8rpx; }
.home-grid__title { font-size: $font-md; font-weight: 700; color: $ink; }
.home-grid__desc { font-size: 20rpx; color: $muted; margin-top: 4rpx;}

.action-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: $gap-md;
}
.action-card__info { display: flex; flex-direction: column; }
.action-card__title { font-size: $font-md; font-weight: 700; color: $ink; }
.action-card__desc { font-size: 22rpx; color: $muted; margin-top: 4rpx; }
.action-card__btn { margin: 0; font-size: 24rpx; padding: 16rpx 32rpx; }

.section-title { font-size: $font-md; font-weight: 700; color: $muted; margin: $gap-lg 0 $gap-sm; }
.notice-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: $gap-sm;
  padding: $gap-md $gap-lg;
}
.notice-card__content { display: flex; flex-direction: column; gap: 4rpx; }
.notice-card__tag { display: inline-block; padding: 2rpx 8rpx; background: $blush; color: $rose-dark; border-radius: 4rpx; font-size: 20rpx; font-weight: bold; align-self: flex-start;}
.notice-card__title { font-size: $font-sm; font-weight: 700; color: $ink; }
.notice-card__copy { font-size: 22rpx; color: $muted; }
.notice-card__arrow { font-size: 32rpx; color: $muted; }

.loading-state { text-align: center; padding: $gap-xl; color: $muted; font-size: $font-sm; }

/* 弹窗统一样式 */
.overlay { position: fixed; inset: 0; z-index: 50; display: flex; align-items: center; justify-content: center; background: rgba(15, 23, 42, 0.4); backdrop-filter: blur(4px); padding: $gap-xl; }
.sheet-dialog { width: 100%; max-height: 80vh; background: $paper; border-radius: $radius-lg; display: flex; flex-direction: column; box-shadow: $shadow-lg; overflow: hidden;}
.sheet-dialog__header { display: flex; justify-content: space-between; align-items: center; padding: $gap-md $gap-lg; border-bottom: 1px solid $line; background: $surface; }
.sheet-dialog__title { font-size: $font-md; font-weight: 700; color: $ink; }
.sheet-dialog__close { font-size: 40rpx; color: $muted; padding: 0 10rpx;}
.sheet-dialog__body { padding: $gap-lg; flex: 1; min-height: 0; overflow-y: auto; }
.sheet-dialog__footer { padding: $gap-md $gap-lg; border-top: 1px solid $line; }
.sheet-dialog__btn { width: 100%; margin: 0; }
.sheet-dialog__tag { color: $rose-dark; font-size: $font-xs; font-weight: bold; }
.sheet-dialog__summary { display: block; margin-top: $gap-sm; color: $muted; font-size: $font-sm; }
.sheet-dialog__text { display: block; margin-top: $gap-md; color: $ink; font-size: $font-sm; line-height: 1.8; }

.form-section { margin-bottom: $gap-md; }
.form-section__title { display: block; margin-bottom: 12rpx; font-size: $font-sm; font-weight: 600; color: $muted; }
.form-input { width: 100%; height: 80rpx; background: $paper; border: 1px solid $line; border-radius: $radius-sm; padding: 0 $gap-md; font-size: $font-md; color: $ink; box-sizing: border-box;}
</style>
