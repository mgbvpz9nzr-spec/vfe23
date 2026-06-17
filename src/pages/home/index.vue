<template>
  <view class="page home-page">
    <ClinicTopBar />
    <!-- 用户问候 -->
    <view class="home-hero">
      <view class="home-hero__greeting">
        <text class="home-hero__wave">👋</text>
        <text class="home-hero__name">你好，{{ userStore.displayName || '姐姐' }}</text>
      </view>
      <text class="home-hero__date">{{ todayStr }}</text>
    </view>

    <view class="today-card">
      <view class="today-card__head">
        <text class="today-card__title">今日速览</text>
        <text class="today-card__date">
          {{ todayStr }} ·
          <text v-if="totalRecordedDays > 0" class="today-card__streak">已记录 {{ totalRecordedDays }} 天</text>
          <text v-else class="today-card__streak today-card__streak--empty">还没有记录哦</text>
        </text>
      </view>

      <!-- 已记录 / 待补充 两栏：已记录显示勾选,待补充可点击直接补录 -->
      <view class="today-rows">
        <view class="today-row today-row--done">
          <text class="today-row__label">已记录</text>
          <view class="today-row__chips">
            <view v-for="item in doneChips" :key="item.key" class="today-chip today-chip--done">
              <text>{{ item.icon }} {{ item.label }}</text>
            </view>
            <text v-if="!doneChips.length" class="today-row__empty">今天还没记录任何内容</text>
          </view>
        </view>
        <view class="today-row today-row--todo">
          <text class="today-row__label">点击补充</text>
          <view class="today-row__chips">
            <view
              v-for="item in todoChips"
              :key="item.key"
              class="today-chip today-chip--todo"
              :class="{ 'today-chip--primary': item.primary }"
              hover-class="today-chip--pressed"
              :hover-stay-time="120"
              @click="openQuickRecord(item)"
            >
              <text>{{ item.icon }} {{ item.label }}</text>
            </view>
            <text v-if="!todoChips.length" class="today-row__empty">今日重点都已记录,辛苦啦～</text>
          </view>
        </view>
      </view>
    </view>

    <!-- 快速入口 -->
    <view class="home-grid">
      <view class="home-grid__item" @click="goTab('period')">
        <text class="home-grid__icon">🌸</text>
        <text class="home-grid__title">周期记录</text>
        <text class="home-grid__desc">更懂你的规律</text>
      </view>
      <view class="home-grid__item" @click="goSelfCheck">
        <text class="home-grid__icon">🩺</text>
        <text class="home-grid__title">症状自查</text>
        <text class="home-grid__desc">AI 整理就诊建议</text>
      </view>
      <view class="home-grid__item" @click="goTab('chat')">
        <text class="home-grid__icon">💬</text>
        <text class="home-grid__title">小美顾问</text>
        <text class="home-grid__desc">专属 AI 健康建议</text>
      </view>
    </view>

    <!-- 预约卡片 -->
    <view class="home-booking-card" @click="goPage('booking')">
      <view class="home-booking-card__info">
        <text class="home-booking-card__title">需要医生帮忙吗？</text>
        <text class="home-booking-card__desc">预约基础咨询或门店服务</text>
      </view>
      <view class="home-booking-card__action">
        <text>现在预约</text>
      </view>
    </view>

    <view v-if="marketingPosts.length" class="home-section-title">门诊关怀与消息</view>
    <view v-for="post in marketingPosts.slice(0, 3)" :key="post.id" class="marketing-card card" @click="openMarketing(post)">
      <view><text class="marketing-card__tag">{{ post.type === "PROMOTION" ? "关怀" : post.type === "ACTIVITY" ? "活动" : "消息" }}</text><text class="marketing-card__title">{{ post.title }}</text><text class="marketing-card__copy">{{ post.subtitle || post.activityInfo || post.introText || "" }}</text></view><text>查看</text>
    </view>

    <!-- 加载 -->
    <view v-if="loading" class="home-loading">
      <text>加载中...</text>
    </view>

    <!-- 快捷记录弹窗(复用周期页 ChoiceSection + 同款选项) -->
    <view v-if="periodStore.activeSheet && periodStore.activeSheet !== 'stateSwitch' && periodStore.activeSheet !== 'modeSettings' && periodStore.activeSheet !== 'dayDetail' && periodStore.activeSheet !== 'startConfirm' && periodStore.activeSheet !== 'endConfirm'" class="marketing-backdrop" @click="closeHomeSheet">
      <view class="quick-sheet" @click.stop>
        <view class="quick-sheet__handle" />
        <text class="quick-sheet__title">{{ homeSheetTitle }}</text>
        <scroll-view class="quick-sheet__body" scroll-y>
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
              <text class="form-section__title">体重 (kg)</text>
              <input v-model="form.weight" type="digit" class="form-input" placeholder="输入体重" />
            </view>
          </template>
          <template v-if="periodStore.activeSheet === 'bodyTemp'">
            <view class="form-section">
              <text class="form-section__title">基础体温 (℃)</text>
              <input v-model="form.bodyTemp" type="digit" class="form-input" placeholder="输入体温" />
            </view>
          </template>
        </scroll-view>
        <button class="btn-primary quick-sheet__btn" :loading="saving" @click="saveHomeSheet">保存</button>
      </view>
    </view>

    <view v-if="activeMarketing" class="marketing-backdrop" @click="activeMarketing = null">
      <view class="marketing-sheet" @click.stop>
        <view class="marketing-sheet__handle" />
        <text class="marketing-sheet__tag">{{ activeMarketing.type === "PROMOTION" ? "关怀" : activeMarketing.type === "ACTIVITY" ? "活动" : "消息" }}</text>
        <text class="marketing-sheet__title">{{ activeMarketing.title || "门诊消息" }}</text>
        <text class="marketing-sheet__summary">{{ activeMarketing.subtitle || activeMarketing.introText || "" }}</text>
        <scroll-view class="marketing-sheet__copy" scroll-y><text>{{ activeMarketing.bodyText || activeMarketing.activityInfo || activeMarketing.introText || activeMarketing.subtitle || "暂无详细内容" }}</text></scroll-view>
        <button class="marketing-sheet__button" @click="activeMarketing = null">我知道了</button>
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
// 用户总计记录过的天数(records + symptomLogs 日期去重,不要求连续)
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

// 枚举值→中文 label;DB 存的是英文,首页展示时统一转中文
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
  return `${d.getFullYear()}年${d.getMonth() + 1}月${d.getDate()}日`;
});

// 当前用户模式
const currentMode = computed<Mode>(() => (periodStore.menstrualState?.mode || "PERIOD") as Mode);

// 今日是否有进行中的经期(startConfirm 或经期体感，不限开始日期)
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

// 是否在排卵 / 易孕窗口
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

// 已记录 chip：按当前模式下当日 record 中已填写的字段
const doneChips = computed(() => {
  const r: any = todayRecord.value ?? {};
  const out: { key: string; icon: string; label: string }[] = [];
  if (r.sleepQuality || r.sleepStart) out.push({ key: "sleep", icon: "😴", label: "睡眠" });
  if (r.mood) out.push({ key: "mood", icon: "😊", label: `心情·${labelOf(moodLabels, r.mood)}` });
  if (Array.isArray(r.habits) && r.habits.length) out.push({ key: "habits", icon: "🏃", label: `习惯·${r.habits.length}项` });
  if (currentMode.value === "MOM" && r.babyFeed) out.push({ key: "babyFeed", icon: "🍼", label: `宝宝·${r.babyFeed}` });
  if (r.weight) out.push({ key: "weight", icon: "⚖️", label: `体重·${r.weight}kg` });
  if (r.bodyTemp) out.push({ key: "bodyTemp", icon: "🌡️", label: `体温·${r.bodyTemp}℃` });
  if (r.ovuTestResult) out.push({ key: "ovuTest", icon: "🔬", label: `排卵·${labelOf(ovuLabels, r.ovuTestResult)}` });
  if (r.pregTestResult) out.push({ key: "pregTest", icon: "🤰", label: `早孕·${labelOf(pregLabels, r.pregTestResult)}` });
  if (todaySymptomCount.value) out.push({ key: "symptom", icon: "📝", label: `症状·${todaySymptomCount.value}条` });
  return out;
});

// 待补充 chip：通用 + 模式特色,与周期数据关联(易孕/经期/排卵)
// 怀孕模式不展示经期相关项
const todoChips = computed(() => {
  const r: any = todayRecord.value ?? {};
  const out: { key: TodoKey; icon: string; label: string; primary?: boolean }[] = [];
  const pushIf = (key: TodoKey, icon: string, label: string, cond: boolean, primary = false) => {
    if (cond) out.push({ key, icon, label, primary });
  };

  // 通用：所有模式都建议记录心情和症状
  pushIf("mood", "😊", "记心情", !r.mood, true);
  pushIf("symptom", "📝", "记症状", todaySymptomCount.value === 0);

  if (currentMode.value !== "PREGNANT") {
    // 经期 / 备孕 / 宝妈：通用
    pushIf("sleep", "😴", "记睡眠", !r.sleepQuality && !r.sleepStart);
    pushIf("habits", "🏃", "打卡习惯", !Array.isArray(r.habits) || r.habits.length === 0);
  }

  if (currentMode.value === "MOM") {
    // 宝妈：宝宝成长日记
    pushIf("babyFeed", "🍼", "记宝宝", !r.babyFeed && !r.babyCry, true);
  } else if (currentMode.value === "PREGNANT") {
    // 怀孕：体重 + 症状
    pushIf("weight", "⚖️", "记体重", !r.weight);
  } else if (currentMode.value === "PREPARE") {
    // 备孕：易孕窗口提示基础体温 / 排卵试纸;经期则提示经期体感
    if (inFertile.value) {
      pushIf("bodyTemp", "🌡️", "基础体温", !r.bodyTemp, true);
      pushIf("ovuTest", "🔬", "排卵试纸", !r.ovuTestResult);
    } else {
      pushIf("intercourse", "💕", "甜蜜时刻", !r.intercourse);
    }
  } else {
    // PERIOD 模式
    if (ongoingPeriod.value) {
      pushIf("period", "🩹", "经期体感", !(r.painLevel || r.flowLevel), true);
    } else {
      pushIf("leucorrhea", "💧", "白带观察", !(r.leuColor || r.whiteLeucorrhea));
    }
  }

  return out;
});

function isoToday() {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
}

function goTab(tab: string) {
  uni.switchTab({ url: `/pages/${tab}/index` });
}

function goPage(page: string) {
  uni.navigateTo({ url: `/pages/${page}/index` });
}

function goSelfCheck() {
  uni.navigateTo({ url: "/pages/services/index?type=selfcheck" });
}

function openMarketing(post: any) {
  activeMarketing.value = post;
}

// 快捷记录(复用 periodStore.openSheet + ChoiceSection,字段与周期页一致)
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
const sheetBabyFeedOptions: Option[] = opt(["香甜母乳", "营养配方奶", "混合喂养", "尝了辅食"]);
const sheetBabyCryOptions: Option[] = opt(["乖乖没哭", "哼唧了一小下", "大哭了一场", "闹腾好久"]);
const sheetDietOptions: Option[] = opt(["叶酸打卡", "复合维他命", "吃钙片", "补铁啦", "高蛋白", "绿叶菜", "鲜水果", "全谷物", "小坚果", "戒了冷饮", "今天没喝咖啡", "远离酒精", "远离二手烟"]);

interface SheetField { key: string; label: string; options: Option[]; multi?: boolean; visual?: string }
const sheetFieldsMap: Record<string, SheetField[]> = {
  sleep: [{ key: "sleepQuality", label: "睡眠质量", options: sheetSleepOptions }],
  notes: [{ key: "mood", label: "今天心情", options: sheetMoodOptions }],
  discomfort: [
    { key: "painLevel", label: "疼痛程度", options: sheetPainOptions },
    { key: "flowLevel", label: "经血流量", options: sheetFlowOptions },
    { key: "discomfort", label: "经期不适", options: sheetDiscomfortOptions },
  ],
  habits: [{ key: "habitsArr", label: "好习惯打卡", options: sheetHabitsOptions, multi: true }],
  symptoms: [{ key: "symptomsArr", label: "身体症状", options: sheetSymptomsOptions, multi: true }],
  intercourse: [{ key: "intercourse", label: "今天同房", options: sheetIntercourseOptions }],
  leucorrhea: [
    { key: "leuColor", label: "白带颜色", options: sheetLeuColorOptions },
    { key: "leuState", label: "状态", options: sheetLeuStateOptions },
    { key: "leuOther", label: "其他感受", options: sheetLeuOtherOptions },
  ],
  coc: [{ key: "coc", label: "避孕方式", options: sheetCocOptions }],
  ovuTest: [{ key: "ovuTestResult", label: "排卵试纸", options: sheetOvuOptions }],
  pregTest: [{ key: "pregTestResult", label: "早孕试纸", options: sheetPregOptions }],
  babyRecord: [
    { key: "babyFeed", label: "今天吃了什么", options: sheetBabyFeedOptions },
    { key: "babyCry", label: "今天哭闹", options: sheetBabyCryOptions },
  ],
  prepareDiet: [{ key: "dietTags", label: "饮食打卡", options: sheetDietOptions, multi: true }],
};
const sheetTitles: Record<string, string> = {
  sleep: "睡眠", notes: "心情", discomfort: "经期体感", habits: "好习惯", symptoms: "身体症状",
  intercourse: "同房", leucorrhea: "白带观察", coc: "避孕方式", ovuTest: "排卵试纸",
  pregTest: "早孕试纸", babyRecord: "宝宝记录", prepareDiet: "备孕饮食",
};

const homeSheetFields = computed<SheetField[]>(() => sheetFieldsMap[periodStore.activeSheet ?? ""] ?? []);
const homeSheetTitle = computed(() => sheetTitles[periodStore.activeSheet ?? ""] || "快捷记录");

// chip key → period sheet type(与周期页字段对齐)
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
  // 无对应 sheet 的类型:直接默认值保存
  if (!sheet || !sheetFieldsMap[sheet]) {
    periodStore.saveDailyRecord(date, { recordDate: date }, date).then(() => {
      appStore.showToast("已记录", "success");
    }).catch((e: any) => {
      appStore.showToast(e?.message || "记录失败", "error");
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
    appStore.showToast("已保存", "success");
    closeHomeSheet();
  } catch (e: any) {
    appStore.showToast(e?.message || "保存失败", "error");
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
.home-page {
  padding: $gap-md $gap-md $gap-xl;
  min-height: 100vh;
}

/* ====== 今日速览：手绘风卡片 ====== */
.today-card {
  position: relative;
  margin-bottom: $gap-md;
  padding: 24rpx 26rpx 22rpx;
  background: #fffaf3;
  border: 2rpx dashed #f9a8d4;
  border-radius: 22rpx 18rpx 24rpx 16rpx;
  box-shadow: 0 4rpx 14rpx rgba(244, 114, 182, 0.08);
  transform: rotate(-0.4deg);

  &::before,
  &::after {
    content: "";
    position: absolute;
    background: #fbcfe8;
    opacity: .5;
  }
  &::before {
    top: -2rpx; left: 18%;
    width: 60%; height: 4rpx;
    border-radius: 4rpx;
    transform: rotate(-1.2deg);
  }
  &::after {
    bottom: -2rpx; right: 14%;
    width: 50%; height: 4rpx;
    border-radius: 4rpx;
    transform: rotate(1deg);
  }

  &__head {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    margin-bottom: $gap-md;
  }
  &__title {
    font-size: $font-xl;
    font-weight: 900;
    color: $ink;
    letter-spacing: 1rpx;
  }
  &__date {
    color: $muted;
    font-size: $font-xs;
    font-weight: 600;
  }
  &__streak {
    color: $rose-dark;
    font-weight: 800;
    &--empty { color: $muted; font-weight: 600; }
  }
}

.today-rows {
  display: flex;
  flex-direction: column;
  gap: $gap-sm;
}

.today-row {
  display: flex;
  flex-direction: column;
  gap: 10rpx;
  padding: 14rpx 16rpx;
  border-radius: 16rpx 20rpx 18rpx 14rpx;
  background: #fff;
  border: 2rpx dashed rgba(244, 114, 182, 0.32);

  &--done { border-color: rgba(34, 197, 94, 0.32); }
  &--todo { border-color: rgba(244, 114, 182, 0.32); }

  &__label {
    font-size: 22rpx;
    font-weight: 800;
    color: $muted;
    letter-spacing: 1rpx;
  }
  &--done &__label { color: #15803d; }
  &--todo &__label { color: $rose-dark; }

  &__chips {
    display: flex;
    flex-wrap: wrap;
    gap: 10rpx;
  }
  &__empty {
    color: $muted;
    font-size: 22rpx;
    font-weight: 600;
  }
}

.today-chip {
  display: inline-flex;
  align-items: center;
  gap: 6rpx;
  padding: 10rpx 18rpx;
  border-radius: 999rpx;
  font-size: 22rpx;
  font-weight: 700;
  line-height: 1.2;
  white-space: nowrap;
  border: 2rpx solid transparent;
  transition: transform .12s ease, background .12s ease;

  &--done {
    background: #f0fdf4;
    color: #166534;
    border-color: #bbf7d0;
  }

  &--todo {
    background: #fff1f2;
    color: $rose-dark;
    border-color: #fbcfe8;
    border-style: dashed;
  }
  &--primary {
    background: linear-gradient(135deg, $rose, $rose-dark);
    color: #fff;
    border-color: transparent;
    box-shadow: 0 4rpx 12rpx rgba(236, 72, 153, .22);
  }
  &--pressed { transform: scale(0.96); }
}

/* ====== 旧样式保留 ====== */
.home-section-title { font-size:$font-md;font-weight:800;margin:$gap-lg 0 $gap-sm; }.marketing-card { display:flex;justify-content:space-between;align-items:center;gap:$gap-md;margin-bottom:$gap-sm; }.marketing-card__tag,.marketing-card__title,.marketing-card__copy { display:block; }.marketing-card__tag { color:$rose-dark;font-size:$font-xs;font-weight:700; }.marketing-card__title { font-size:$font-sm;font-weight:800; }.marketing-card__copy { color:$muted;font-size:$font-xs; }

.home-hero {
  padding: $gap-xl $gap-sm $gap-lg;
  &__greeting {
    display: flex;
    align-items: center;
    gap: $gap-sm;
  }
  &__wave { font-size: 48rpx; }
  &__name { font-size: $font-xl; font-weight: 800; color: $ink; }
  &__date { font-size: $font-sm; color: $muted; margin-top: $gap-xs; }
}

.home-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: $gap-sm;
  margin-bottom: $gap-md;
}

.home-grid__item {
  background: $paper;
  border-radius: $radius-md;
  padding: $gap-lg $gap-md;
  text-align: center;
  box-shadow: $shadow-sm;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: $gap-xs;
}

.home-grid__icon { font-size: 48rpx; }
.home-grid__title { font-size: $font-md; font-weight: 700; color: $ink; }
.home-grid__desc { font-size: $font-xs; color: $muted; }

.home-booking-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: $paper;
  border-radius: $radius-lg;
  padding: $gap-lg;
  box-shadow: $shadow-sm;

  &__title { font-size: $font-md; font-weight: 700; color: $ink; display: block; }
  &__desc { font-size: $font-sm; color: $muted; margin-top: 4rpx; }

  &__action {
    background: $blush;
    color: $rose-dark;
    padding: 12rpx 32rpx;
    border-radius: $radius-full;
    font-size: $font-sm;
    font-weight: 700;
    white-space: nowrap;
  }
}

.home-loading {
  text-align: center;
  padding: $gap-xl;
  color: $muted;
  font-size: $font-sm;
}
.marketing-backdrop { position: fixed; inset: 0; z-index: 30; display: flex; align-items: flex-end; background: rgba(39, 39, 42, .46); backdrop-filter: blur(8rpx); }
.quick-sheet { display: flex; width: 100%; box-sizing: border-box; flex-direction: column; padding: $gap-md $gap-lg calc(36rpx + env(safe-area-inset-bottom)); border-radius: 40rpx 40rpx 0 0; background: $paper; max-height: 70vh; }
.quick-sheet__handle { width: 80rpx; height: 8rpx; margin: 0 auto $gap-md; border-radius: $radius-full; background: #e4e4e7; }
.quick-sheet__title { display: block; margin-bottom: $gap-md; color: $ink; font-size: $font-lg; font-weight: 800; text-align: center; }
.quick-sheet__body { max-height: 56vh; overflow-y: auto; }
.quick-sheet__btn { width: 100%; margin-top: $gap-md; }
.form-section { margin-bottom: 18rpx; padding: 20rpx; border-radius: $radius-lg; background: $surface; }
.form-section__title { display: block; margin-bottom: 14rpx; font-size: $font-sm; font-weight: 800; }
.form-input { width: 100%; box-sizing: border-box; padding: 16rpx 20rpx; border: 2rpx solid $line; border-radius: $radius-md; background: $paper; font-size: $font-sm; }
.marketing-sheet { display: flex; width: 100%; max-height: 82vh; box-sizing: border-box; flex-direction: column; padding: $gap-md $gap-lg calc(36rpx + env(safe-area-inset-bottom)); border-radius: 40rpx 40rpx 0 0; background: $paper; }
.marketing-sheet__handle { width: 80rpx; height: 8rpx; margin: 0 auto $gap-lg; border-radius: $radius-full; background: #e4e4e7; }
.marketing-sheet__tag { color: $rose-dark; font-size: $font-xs; font-weight: 800; }
.marketing-sheet__title { display: block; margin-top: 4rpx; color: $ink; font-size: $font-xl; font-weight: 800; line-height: 1.35; }
.marketing-sheet__summary { display: block; margin-top: $gap-sm; color: $muted; font-size: $font-sm; line-height: 1.6; }
.marketing-sheet__copy { max-height: 38vh; margin: $gap-lg 0; color: $ink; font-size: $font-sm; line-height: 1.9; white-space: pre-wrap; }
.marketing-sheet__button { width: 100%; flex: 0 0 auto; border-radius: $radius-full; background: $ink; color: #fff; font-size: $font-sm; font-weight: 700; }
</style>
