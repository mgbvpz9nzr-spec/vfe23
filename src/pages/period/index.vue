<template>
  <view class="page period-page">
    <ClinicTopBar />
    <view class="status-pill" :style="{ background: statusBarColor }">
      <text>{{ statusBarText }}</text>
      <text class="status-pill__date">{{ selectedDateLabel }}</text>
    </view>
    <view class="mode-card card" @click="openSheet('stateSwitch')">
      <view>
        <text class="mode-card__eyebrow">当前模式</text>
        <view class="mode-card__title"><text>{{ currentModeInfo.icon }}</text><text>{{ currentModeInfo.label }}</text></view>
        <text class="mode-card__desc">{{ modeDateText }}</text>
      </view>
      <text class="mode-card__action">切换</text>
    </view>

    <view class="calendar card">
      <view class="calendar__header">
        <view class="calendar__nav" @click="changeMonth(-1)">‹</view>
        <view>
          <text class="calendar__title">{{ calendar.year }} 年 {{ calendar.month + 1 }} 月</text>
          <text class="calendar__subtitle">点击日期后，记录会保存到当天</text>
        </view>
        <view class="calendar__nav" @click="changeMonth(1)">›</view>
      </view>
      <view class="calendar__week">
        <text v-for="name in weekdays" :key="name">{{ name }}</text>
      </view>
      <view class="calendar__grid">
        <view
          v-for="cell in calendar.cells"
          :key="cell.key"
          class="calendar-day"
          :class="[
            `calendar-day--${cell.kind}`,
            { 'calendar-day--out': cell.isOut, 'calendar-day--today': cell.isToday, 'calendar-day--selected': cell.key === selectedDate },
          ]"
          @click="selectDate(cell.key)"
        >
          <text class="calendar-day__number">{{ cell.day }}</text>
          <text v-if="cell.pregnancyWeek != null && !cell.isOut" class="calendar-day__meta calendar-day__meta--pregnant">{{ cell.pregnancyWeek }}周{{ cell.checkupText ? `·${cell.checkupText}` : '' }}</text>
          <text v-else-if="cell.cycleDay && !cell.isOut" class="calendar-day__meta">第{{ cell.cycleDay }}天</text>
          <text v-if="cell.kind === 'predicted' && !cell.isOut" class="calendar-day__pred">🌸</text>
          <view v-if="cell.hasRecord" class="calendar-day__record" />
          <view v-if="cell.probability > 0 && !cell.isOut && currentMode !== 'PREGNANT'" class="calendar-day__prob" :style="{ opacity: Math.max(.18, cell.probability / 100) }" />
        </view>
      </view>
      <view v-if="currentMode === 'PREGNANT'" class="calendar__legend calendar__legend--pregnant">
        <text><i class="legend-dot legend-dot--t1" />孕早期 T1</text>
        <text><i class="legend-dot legend-dot--t2" />孕中期 T2</text>
        <text><i class="legend-dot legend-dot--t3" />孕晚期 T3</text>
        <text><i class="legend-dot legend-dot--due" />预产期</text>
      </view>
      <view v-else class="calendar__legend">
        <text><i class="legend-dot legend-dot--period" />经期</text>
        <text><i class="legend-dot legend-dot--predicted" />预测</text>
        <text><i class="legend-dot legend-dot--ovulation" />易孕/排卵</text>
      </view>
    </view>

    <view class="selected-card card">
      <view>
        <text class="selected-card__eyebrow">当前记录日期</text>
        <text class="selected-card__title">{{ selectedDateLabel }}</text>
        <text class="selected-card__desc">{{ selectedDayDescription }}</text>
      </view>
      <view class="selected-card__today" @click="goToday">回到今天</view>
    </view>

    <view class="period-section-head">
      <view>
        <text class="period-section-title">快捷记录</text>
        <text class="period-section-sub">选择项目记录，或让小美一起看看</text>
      </view>
      <view class="period-ai-button" @click="wakeAiAssistant"><text class="period-ai-button__icon">✨</text><text>问小美</text></view>
    </view>
    <view class="quick-grid">
      <view v-for="item in quickItems" :key="item.type || item.label" class="quick-item" :class="{ 'quick-item--primary': item.primary, 'quick-item--editing': item.value }" @click="openQuickItem(item)">
        <text class="quick-item__icon">{{ item.icon }}</text>
        <text class="quick-item__label">{{ item.label }}</text>
        <text class="quick-item__value">{{ item.value || '点击记录' }}</text>
        <text class="quick-item__action">{{ item.value ? '✓' : '+' }}</text>
      </view>
    </view>

    <view v-if="periodStore.activeSheet" class="sheet-backdrop" @click="closeSheet">
      <view class="sheet-panel" @click.stop>
        <view class="sheet-header">
          <view>
            <text class="sheet-title">{{ sheetTitle }}</text>
            <text class="sheet-date">{{ selectedDateLabel }}</text>
          </view>
          <text class="sheet-close" @click="closeSheet">×</text>
        </view>

        <scroll-view class="sheet-body" scroll-y>
        <view v-if="periodStore.activeSheet === 'startConfirm' || periodStore.activeSheet === 'endConfirm'" class="confirm-box">
          <text class="confirm-box__icon">{{ periodStore.activeSheet === 'startConfirm' ? '🩸' : '✅' }}</text>
          <text class="confirm-box__title">{{ periodStore.activeSheet === 'startConfirm' ? '月经今天来了吗？' : '月经今天走了吗？' }}</text>
          <text class="confirm-box__copy">{{ periodStore.activeSheet === 'startConfirm' ? '今天是经期开始的第一天吗？点击确定后，我们将为您记录这一天。' : '今天月经已经彻底干净了吗？点击确定后，本次经期将被标记为结束。' }}</text>
        </view>

        <template v-else-if="periodStore.activeSheet === 'stateSwitch'">
          <view class="mode-grid">
            <view
              v-for="mode in modes"
              :key="mode.value"
              class="mode-choice"
              :class="{ 'mode-choice--active': currentMode === mode.value }"
              @click="pickMode(mode.value)"
            >
              <text class="mode-choice__icon">{{ mode.icon }}</text>
              <text>{{ mode.label }}</text>
            </view>
          </view>
        </template>

        <template v-else-if="periodStore.activeSheet === 'modeSettings'">
          <view class="form-section">
            <text class="form-section__title">{{ modeSettingLabel }}</text>
            <picker mode="date" :value="form[modeSettingKey]" @change="setPickerValue(modeSettingKey, $event)">
              <view class="picker-field">{{ form[modeSettingKey] || '请选择日期' }}</view>
            </picker>
          </view>
        </template>

        <template v-else-if="periodStore.activeSheet === 'dayDetail'">
          <view class="day-hero" :class="`day-hero--${selectedKind}`">
            <view class="day-hero__icon"><text>{{ dayDetailIcon }}</text></view>
            <view class="day-hero__body">
              <text class="day-hero__eyebrow">{{ dayDetailKindLabel }}</text>
              <text class="day-hero__title">{{ selectedDayStatus }}</text>
              <text class="day-hero__summary">{{ selectedRecord ? '这一天已有健康记录' : selectedDate > today ? '这是未来的日子' : '这一天还没有记录' }}</text>
            </view>
          </view>

          <view class="day-advice card">
            <text class="day-advice__icon">💡</text>
            <text class="day-advice__text">{{ selectedDayAdvice }}</text>
          </view>

          <view v-if="selectedRecord && selectedRecordSummary" class="day-record-card card">
            <view class="day-record-card__head">
              <text class="day-record-card__title">当天记录</text>
              <text class="day-record-card__date">{{ dayRecordShortDate }}</text>
            </view>
            <text class="day-record-card__copy">{{ selectedRecordSummary }}</text>
          </view>

          <view v-if="selectedDate <= today" class="day-actions">
            <view v-if="canRecordPeriod" class="day-action" @click="openTodayPeriodConfirm">
              <view class="day-action__icon"><text>{{ selectedPeriodAction === 'endConfirm' ? '✅' : '🩸' }}</text></view>
              <view class="day-action__body">
                <text class="day-action__title">{{ selectedPeriodAction === 'endConfirm' ? '月经今天走了吗？' : '月经今天来了吗？' }}</text>
                <text class="day-action__sub">{{ selectedPeriodAction === 'endConfirm' ? '点击结束本次经期' : '点击标记经期开始' }}</text>
              </view>
              <text class="day-action__arrow">›</text>
            </view>
            <view class="day-action day-action--primary" @click="openFromDay('recordFull')">
              <view class="day-action__icon"><text>＋</text></view>
              <view class="day-action__body">
                <text class="day-action__title">完善当天记录</text>
                <text class="day-action__sub">症状 / 睡眠 / 心情 / 体征</text>
              </view>
              <text class="day-action__arrow">›</text>
            </view>
          </view>
        </template>

        <template v-else-if="periodStore.activeSheet === 'recordFull'">
          <text class="full-record-intro">更详细的记录能让 AI 和医生更懂您</text>
          <choice-section title="痛经感受" :options="painOptions" field="painLevel" :value="form.painLevel" visual="level" @select="selectChoice" />
          <choice-section title="经血流量" :options="flowOptions" field="flowLevel" :value="form.flowLevel" visual="level" @select="selectChoice" />
          <view class="form-section">
            <text class="form-section__title">经血颜色</text>
            <view class="color-bar">
              <view v-for="option in bloodColorOptions" :key="String(option.value)" class="color-cell" :class="{ 'color-cell--active': form.bloodColor === option.value }" :style="{ background: option.color }" @click="form.bloodColor = option.value"><text>{{ option.label }}</text></view>
            </view>
          </view>
          <choice-section title="经血状态（可多选）" :options="bloodStateOptions" field="bloodStateTags" :value="form.bloodStateTags" multi visual="icon" @select="selectChoice" />
          <choice-section title="主要不适" :options="discomfortOptions" field="discomfort" :value="form.discomfort" visual="icon" @select="selectChoice" />
          <choice-section title="睡眠质量" :options="sleepOptions" field="sleepQuality" :value="form.sleepQuality" visual="icon" @select="selectChoice" />
          <choice-section title="今天心情" :options="moodOptions" field="mood" :value="form.mood" visual="icon" @select="selectChoice" />
          <choice-section title="健康打卡（可多选）" :options="habitsOptions" field="habitsArr" :value="form.habitsArr" multi visual="icon" @select="selectChoice" />
          <view class="form-section">
            <text class="form-section__title">随手记</text>
            <textarea v-model="form.notes" class="text-area" maxlength="500" placeholder="记录下特别的感受吧..." />
          </view>
        </template>

        <template v-else-if="periodStore.activeSheet === 'discomfort'">
          <choice-section title="疼痛程度" :options="painOptions" field="painLevel" :value="form.painLevel" visual="level" @select="selectChoice" />
          <choice-section title="经血流量" :options="flowOptions" field="flowLevel" :value="form.flowLevel" visual="level" @select="selectChoice" />
          <view class="form-section">
            <text class="form-section__title">经血颜色</text>
            <view class="color-bar">
              <view v-for="option in bloodColorOptions" :key="String(option.value)" class="color-cell" :class="{ 'color-cell--active': form.bloodColor === option.value }" :style="{ background: option.color }" @click="form.bloodColor = option.value">
                <text>{{ option.label }}</text>
              </view>
            </view>
          </view>
          <choice-section title="经血状态（可多选）" :options="bloodStateOptions" field="bloodStateTags" :value="form.bloodStateTags" multi visual="icon" @select="selectChoice" />
          <choice-section title="主要不适" :options="discomfortOptions" field="discomfort" :value="form.discomfort" visual="icon" @select="selectChoice" />
        </template>

        <template v-else-if="periodStore.activeSheet === 'sleep'">
          <choice-section title="睡眠质量" :options="sleepOptions" field="sleepQuality" :value="form.sleepQuality" visual="icon" @select="selectChoice" />
          <view class="form-section">
            <text class="form-section__title">入睡时间</text>
            <picker mode="time" :value="form.sleepStartTime" @change="setPickerValue('sleepStartTime', $event)">
              <view class="picker-field">{{ form.sleepStartTime || '请选择' }}</view>
            </picker>
          </view>
          <view class="form-section">
            <text class="form-section__title">起床时间</text>
            <picker mode="time" :value="form.sleepEndTime" @change="setPickerValue('sleepEndTime', $event)">
              <view class="picker-field">{{ form.sleepEndTime || '请选择' }}</view>
            </picker>
          </view>
          <text v-if="sleepHoursText" class="form-hint">预计睡眠 {{ sleepHoursText }}</text>
        </template>

        <template v-else-if="periodStore.activeSheet === 'notes'">
          <choice-section title="今天心情" :options="moodOptions" field="mood" :value="form.mood" visual="icon" @select="selectChoice" />
          <view class="form-section">
            <text class="form-section__title">随手记</text>
            <textarea v-model="form.notes" class="text-area" maxlength="500" placeholder="记录今天的感受..." />
          </view>
        </template>

        <template v-else-if="periodStore.activeSheet === 'weight' || periodStore.activeSheet === 'bodyTemp'">
          <view class="form-section">
            <text class="form-section__title">{{ periodStore.activeSheet === 'weight' ? '体重（kg）' : '基础体温（°C）' }}</text>
            <input
              v-model="numericValue"
              class="input-field"
              type="digit"
              :placeholder="periodStore.activeSheet === 'weight' ? '例如 52.5' : '例如 36.55'"
            />
          </view>
        </template>

        <template v-else-if="periodStore.activeSheet === 'ovuTest'">
          <choice-section title="排卵试纸结果" :options="ovuOptions" field="ovuTestResult" :value="form.ovuTestResult" visual="level" @select="selectChoice" />
        </template>

        <template v-else-if="periodStore.activeSheet === 'pregTest'">
          <choice-section title="早孕试纸结果" :options="pregOptions" field="pregTestResult" :value="form.pregTestResult" visual="level" @select="selectChoice" />
        </template>

        <template v-else-if="periodStore.activeSheet === 'symptomLog'">
          <choice-section title="症状标签（可多选）" :options="symptomLogOptions" field="selectedTags" :value="form.selectedTags" multi visual="icon" @select="selectChoice" />
          <choice-section title="严重程度" :options="severityOptions" field="severity" :value="form.severity" visual="level" @select="selectChoice" />
        </template>

        <template v-else-if="simpleSheet">
          <choice-section
            v-for="field in simpleSheet.fields"
            :key="field.key"
            :title="field.label"
            :options="field.options"
            :field="field.key"
            :value="form[field.key]"
            :multi="field.multi"
            visual="icon"
            @select="selectChoice"
          />
          <view v-if="simpleSheet.noteKey" class="form-section">
            <text class="form-section__title">随手记</text>
            <textarea v-model="form[simpleSheet.noteKey]" class="text-area" maxlength="500" placeholder="补充记录..." />
          </view>
        </template>
        </scroll-view>

        <view v-if="!['stateSwitch', 'dayDetail'].includes(periodStore.activeSheet || '')" class="sheet-actions">
          <button class="btn-secondary" @click="closeSheet">取消</button>
          <button class="btn-primary" :loading="saving" @click="saveSheet">{{ isPeriodConfirm ? '确定' : '保存' }}</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { computed, onMounted, reactive, ref } from "vue";
import { onShow } from "@dcloudio/uni-app";
import { usePeriodStore, type SheetType } from "@/stores/usePeriodStore";
import { useAppStore } from "@/stores/useAppStore";
import type { MenstrualRecord } from "@/api/menstrual";
import ChoiceSection from "@/components/ChoiceSection.vue";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

type Option = { value: string | number; label: string; icon?: string; color?: string };
type QuickItem = { type: SheetType; icon: string; label: string; value?: string; primary?: boolean };
type Mode = "PERIOD" | "PREPARE" | "PREGNANT" | "MOM";
type SimpleSheet = { fields: Array<{ key: string; label: string; options: Option[]; multi?: boolean }>; noteKey?: string };
type CalendarKind = "none" | "period" | "predicted" | "ovulation" | "peak" | "t1" | "t2" | "t3" | "due";
type CalendarCell = { key: string; day: number; kind: CalendarKind; isToday: boolean; isOut: boolean; probability: number; hasRecord: boolean; cycleDay?: number; pregnancyWeek?: number; pregnancyDay?: number; checkupText?: string };

const periodStore = usePeriodStore();
const appStore = useAppStore();
const today = isoDate(new Date());
const selectedDate = ref(today);
const saving = ref(false);
const form = reactive<Record<string, any>>({});
const weekdays = ["日", "一", "二", "三", "四", "五", "六"];
const modes: Array<{ value: Mode; icon: string; label: string }> = [
  { value: "PERIOD", icon: "🌸", label: "记经期" },
  { value: "PREPARE", icon: "🌱", label: "备孕中" },
  { value: "PREGNANT", icon: "🤰", label: "怀孕中" },
  { value: "MOM", icon: "🍼", label: "宝妈" },
];

const flowOptions: Option[] = [
  { value: "LIGHT", label: "偏少", icon: "●" }, { value: "MEDIUM", label: "适中", icon: "●●" }, { value: "HEAVY", label: "偏多", icon: "●●●" },
];
const painOptions: Option[] = [
  { value: "NONE", label: "无", icon: "○" }, { value: "MILD", label: "轻微", icon: "◔" }, { value: "MODERATE", label: "中度", icon: "◑" }, { value: "SEVERE", label: "严重", icon: "●" },
];
const bloodColorOptions: Option[] = [
  { value: "light_pink", label: "淡粉", color: "#f9a8d4" }, { value: "red", label: "鲜红", color: "#ef4444" },
  { value: "dark_red", label: "暗红", color: "#991b1b" }, { value: "brown", label: "褐色", color: "#92400e" }, { value: "black", label: "深色", color: "#3f3f46" },
];
const bloodStateOptions: Option[] = [
  { value: "nothing", label: "无异常", icon: "✓" }, { value: "bad_smell", label: "异味", icon: "〰" },
  { value: "block", label: "血块", icon: "●" }, { value: "zhazhuang", label: "渣状", icon: "✦" },
];
const discomfortOptions: Option[] = [
  { value: "NONE", label: "无不适" }, { value: "BLOATING", label: "腹胀" }, { value: "CRAMPS", label: "痉挛" },
  { value: "HEADACHE", label: "头痛" }, { value: "FATIGUE", label: "乏力" }, { value: "MOOD", label: "情绪波动" },
];
const sleepOptions: Option[] = [
  { value: "POOR", label: "差" }, { value: "NORMAL", label: "一般" }, { value: "GOOD", label: "好" }, { value: "EXCELLENT", label: "很好" },
];
const moodOptions: Option[] = [
  { value: "happy", label: "开心" }, { value: "calm", label: "平静" }, { value: "tired", label: "疲倦" },
  { value: "anxious", label: "焦虑" }, { value: "blue", label: "低落" }, { value: "angry", label: "烦躁" },
];
const ovuOptions: Option[] = [
  { value: "negative", label: "阴性" }, { value: "weak_positive", label: "弱阳" },
  { value: "positive", label: "阳性" }, { value: "strong_positive", label: "强阳" },
];
const pregOptions: Option[] = [
  { value: "negative", label: "阴性" }, { value: "weak_positive", label: "弱阳" }, { value: "positive", label: "阳性" },
];
const habitsOptions = options(["多喝水", "吃水果", "清淡饮食", "补充蛋白", "规律运动", "散步", "早睡", "午休", "热敷", "泡脚", "放松冥想", "不喝冰饮"]);
const symptomsOptions = options(["腹胀", "腹痛", "腰酸", "头痛", "头晕", "乏力", "乳房胀痛", "恶心", "腹泻", "便秘", "长痘", "浮肿"]);
const intercourseOptions: Option[] = [{ value: "yes", label: "有" }, { value: "no", label: "没有" }];
const cocOptions: Option[] = [{ value: "condom", label: "避孕套" }, { value: "emergency", label: "紧急避孕药" }, { value: "short_term", label: "短效避孕药" }, { value: "none", label: "没有" }];
const leuColorOptions: Option[] = [{ value: "clear", label: "透明无色" }, { value: "white", label: "乳白色" }, { value: "yellow", label: "偏黄色" }, { value: "brown", label: "褐/咖啡色" }];
const leuStateOptions: Option[] = [{ value: "normal", label: "正常状态" }, { value: "egg_white", label: "像蛋清" }, { value: "thick", label: "有点粘稠" }, { value: "watery", label: "像水一样" }];
const leuOtherOptions: Option[] = [{ value: "none", label: "完全没有" }, { value: "smell", label: "有异味" }, { value: "itchy", label: "发痒" }, { value: "burning", label: "灼热感" }];
const dietOptions = options(["叶酸打卡", "复合维他命", "吃钙片", "补铁啦", "高蛋白", "绿叶菜", "鲜水果", "全谷物", "小坚果", "戒了冷饮", "今天没喝咖啡", "远离酒精", "远离二手烟"]);
const babyFeedOptions = options(["香甜母乳", "营养配方奶", "混合喂养", "尝了辅食"]);
const babyCryOptions = options(["乖乖没哭", "哼唧了一小下", "大哭了一场", "闹腾好久"]);
const babyDiaperOptions = options(["便便正常", "好像偏少", "拉了好多", "暂时没拉"]);
const babySleepOptions = options(["呼呼大睡", "睡得一般", "睡渣宝宝", "根本不睡"]);
const symptomLogOptions = options(["腹痛", "头痛", "头晕", "乏力", "恶心", "异常出血", "白带异常", "瘙痒", "发热", "情绪低落"]);
const severityOptions: Option[] = [1, 2, 3, 4, 5].map((value) => ({ value, label: `${value} 级` }));

const simpleSheets: Record<string, SimpleSheet> = {
  habits: { fields: [{ key: "habitsArr", label: "今天坚持了哪些好习惯？", options: habitsOptions, multi: true }] },
  symptoms: { fields: [{ key: "symptomsArr", label: "今天有哪些身体症状？", options: symptomsOptions, multi: true }] },
  intercourse: { fields: [{ key: "intercourse", label: "今天有安排同房吗？", options: intercourseOptions }] },
  leucorrhea: { fields: [
    { key: "leuColor", label: "颜色", options: leuColorOptions },
    { key: "leuState", label: "状态", options: leuStateOptions },
    { key: "leuOther", label: "异常感觉", options: leuOtherOptions },
  ] },
  coc: { fields: [{ key: "coc", label: "避孕方式", options: cocOptions }] },
  prepareDiet: { fields: [{ key: "dietTags", label: "今天的备孕营养打卡", options: dietOptions, multi: true }], noteKey: "dietNotes" },
  babyRecord: { fields: [
    { key: "babyFeed", label: "今天吃了什么", options: babyFeedOptions },
    { key: "babyCry", label: "今天哭鼻子了吗", options: babyCryOptions },
    { key: "babyDiaper", label: "排便情况", options: babyDiaperOptions },
    { key: "babySleep", label: "睡眠怎样", options: babySleepOptions },
  ], noteKey: "babyNotes" },
};

const calendar = computed(() => buildCalendar(periodStore.monthOffset));
const selectedRecord = computed(() => findRecordForDate(selectedDate.value));
const currentMode = computed(() => (periodStore.menstrualState.mode || "PERIOD") as Mode);
const currentModeInfo = computed(() => modes.find((mode) => mode.value === currentMode.value) ?? modes[0]);
const canRecordPeriod = computed(() => currentMode.value === "PERIOD" || currentMode.value === "PREPARE");
const ongoingPeriod = computed(() => periodStore.records.find((record) => !record.endDate && isPeriodEpisode(record) && recordStart(record) <= today));
const selectedPeriodAction = computed<"startConfirm" | "endConfirm">(() => {
  const ongoing = ongoingPeriod.value;
  return ongoing && selectedDate.value >= recordStart(ongoing) ? "endConfirm" : "startConfirm";
});
const isPeriodConfirm = computed(() => ["startConfirm", "endConfirm"].includes(periodStore.activeSheet || ""));
const simpleSheet = computed(() => simpleSheets[periodStore.activeSheet ?? ""] ?? null);
const modeSettingKey = computed(() => form.targetMode === "PREPARE" ? "prepareStartDate" : form.targetMode === "PREGNANT" ? "pregnantStartDate" : "babyBirthDate");
const modeSettingLabel = computed(() => form.targetMode === "PREPARE" ? "备孕开始日期" : form.targetMode === "PREGNANT" ? "末次月经日期" : "宝宝出生日期");
const modeDateText = computed(() => {
  const state = periodStore.menstrualState;
  if (currentMode.value === "PREPARE") return state.prepareStartDate ? `从 ${state.prepareStartDate} 开始` : "记录备孕日常";
  if (currentMode.value === "PREGNANT") return state.dueDate ? `预产期 ${state.dueDate}` : "记录孕期日常";
  if (currentMode.value === "MOM") return state.babyBirthDate ? `宝宝出生于 ${state.babyBirthDate}` : "记录育儿日常";
  return "记录周期与身体变化";
});
const selectedDateLabel = computed(() => `${selectedDate.value.slice(0, 4)} 年 ${Number(selectedDate.value.slice(5, 7))} 月 ${Number(selectedDate.value.slice(8, 10))} 日`);
const selectedKind = computed(() => calendarKindForDate(selectedDate.value));
const selectedDayDescription = computed(() => ({
  period: "经期记录日", predicted: "预测经期日", ovulation: "易孕期", peak: "预测排卵日",
  t1: "孕早期 · 第 1-13 周", t2: "孕中期 · 第 14-27 周", t3: "孕晚期 · 第 28-40 周",
  due: "预产期", none: selectedRecord.value ? "已有健康记录" : "普通日期",
}[selectedKind.value]));

const statusBarColor = computed(() => selectedKind.value === "period" ? "#ec4899" : selectedKind.value === "predicted" ? "#f9a8d4" : ["ovulation", "peak"].includes(selectedKind.value) ? "#a78bfa" : selectedKind.value === "t1" ? "#fbcfe8" : selectedKind.value === "t2" ? "#c4b5fd" : selectedKind.value === "t3" ? "#fdba74" : selectedKind.value === "due" ? "#ec4899" : "#a1a1aa");
const statusBarText = computed(() => selectedKind.value === "period" ? "经期中，记得照顾好自己" : selectedKind.value === "predicted" ? "预测经期将至" : ["ovulation", "peak"].includes(selectedKind.value) ? "易孕窗口期" : selectedKind.value === "t1" ? "孕早期 · 小生命开始萌芽" : selectedKind.value === "t2" ? "孕中期 · 最舒适的阶段" : selectedKind.value === "t3" ? "孕晚期 · 做好准备见面啦" : selectedKind.value === "due" ? "预产期到了🌟" : "安心记录每一天");
const selectedDayStatus = computed(() => {
  const ck = selectedCheckup.value;
  if (ck) return `第${ck.week}周 · ${fullCheckupLabel(ck.week)}`;
  return selectedKind.value === "period" ? "经期中" : selectedKind.value === "predicted" ? "预测经期将至" : selectedKind.value === "peak" ? "排卵高峰日" : selectedKind.value === "ovulation" ? "易孕排卵期" : selectedKind.value === "t1" ? "孕早期 T1" : selectedKind.value === "t2" ? "孕中期 T2" : selectedKind.value === "t3" ? "孕晚期 T3" : selectedKind.value === "due" ? "预产期" : "普通日期";
});
const selectedDayAdvice = computed(() => {
  const ck = selectedCheckup.value;
  if (ck) return `本周建议去医院做${fullCheckupLabel(ck.week)}，记得提前预约挂号，带上产检本。`;
  return selectedKind.value === "period"
    ? "身体现在需要多一点呵护，注意保暖，记得记录疼痛和流量变化。"
    : selectedKind.value === "predicted"
      ? "经期可能快要到来，留意情绪和身体变化，提前做好准备。"
      : ["peak", "ovulation"].includes(selectedKind.value)
        ? "当前处于易孕窗口期，请根据自己的备孕计划做好记录和安排。"
        : selectedKind.value === "t1"
          ? "孕早期是关键阶段，注意补充叶酸，避免劳累，定期产检。"
          : selectedKind.value === "t2"
            ? "孕中期是最舒适的阶段，可以适当运动，感受宝宝的第一次胎动。"
            : selectedKind.value === "t3"
              ? "孕晚期宝宝快速长大，准备好待产包，注意胎动和自身变化。"
              : selectedKind.value === "due"
                ? "预产期到了！保持冷静，留意宫缩和见红，有情况随时就医。"
                : selectedDate.value > today ? "等到这一天再来记录身体变化吧。" : "记录心情、睡眠和日常习惯，慢慢看见身体规律。";
});
// 当前选中日期对应的产检项
const selectedCheckup = computed(() => {
  const c = calendar.value?.cells?.find((cell: CalendarCell) => cell.key === selectedDate.value);
  if (!c?.pregnancyWeek || !c.checkupText) return null;
  return prenatalCheckups.find((p) => p.week === c.pregnancyWeek!) ?? null;
});
function fullCheckupLabel(week: number): string {
  const map: Record<number, string> = { 6: "早孕确认检查", 12: "NT检查建档", 16: "唐氏筛查", 20: "大排畸四维", 24: "糖耐量测试", 30: "小排畸检查", 34: "常规胎心监护", 36: "分娩综合评估", 37: "每周常规产检" };
  return map[week] || "常规产检";
}
const dayDetailIcon = computed(() => ({
  period: "🩸", predicted: "🌸", ovulation: "🥚", peak: "✨",
  t1: "🌱", t2: "💛", t3: "🍊", due: "🎀",
  none: "📅",
}[selectedKind.value] || "📅"));
const dayDetailKindLabel = computed(() => ({
  period: "经期", predicted: "预测经期", ovulation: "易孕期", peak: "排卵高峰",
  t1: "孕早期", t2: "孕中期", t3: "孕晚期", due: "预产期",
  none: "普通日",
}[selectedKind.value] || "普通日"));
const dayRecordShortDate = computed(() => {
  const d = selectedDate.value;
  if (!d || d.length < 10) return "";
  return `${Number(d.slice(5, 7))}月${Number(d.slice(8, 10))}日`;
});
const selectedRecordSummary = computed(() => {
  const record: any = selectedRecord.value;
  if (!record) return "";
  return [
    valueLabel(record.painLevel, painOptions) && `疼痛：${valueLabel(record.painLevel, painOptions)}`,
    valueLabel(record.flowLevel, flowOptions) && `流量：${valueLabel(record.flowLevel, flowOptions)}`,
    valueLabel(record.sleepQuality, sleepOptions) && `睡眠：${valueLabel(record.sleepQuality, sleepOptions)}`,
    record.notes,
  ].filter(Boolean).join(" · ");
});

// 经期相关记录是通用功能：除怀孕模式外,其他模式都可以记录经期开始/结束和经期体感;
// 各模式保留自己的特色项,删除与通用项重复的子项(例如"孕期症状"与"身体小症状"功能重叠)。
const periodCommon = (record: any): QuickItem[] => [
  { type: "sleep" as SheetType, icon: "😴", label: "睡眠起止", value: valueLabel(record?.sleepQuality, sleepOptions) },
  { type: "notes" as SheetType, icon: "😊", label: "今天心情", value: valueLabel(record?.mood, moodOptions) || record?.notes },
  { type: "habits" as SheetType, icon: "🏃", label: "好习惯打卡", value: arrayText(record.habits ?? record.habitsArr) },
  { type: "symptomLog" as SheetType, icon: "📝", label: "身体小症状", value: symptomLogSummary.value },
];
// 经期确认 + 经期体感细节(仅在经期内才显示)
const periodBlock = (record: any): QuickItem[] => [
  { type: selectedPeriodAction.value as SheetType, icon: selectedPeriodAction.value === "endConfirm" ? "✅" : "🩸", label: selectedPeriodAction.value === "endConfirm" ? "月经今天走了吗？" : "月经今天来了吗？", primary: true },
  ...(selectedPeriodAction.value === "endConfirm" ? [{ type: "discomfort" as SheetType, icon: "🩹", label: "经期体感细节", value: valueLabel(record.painLevel, painOptions) || valueLabel(record.flowLevel, flowOptions) }] : []),
];

const quickItems = computed<QuickItem[]>(() => {
  const record: any = selectedRecord.value ?? {};
  if (currentMode.value === "PREGNANT") {
    // 怀孕模式:无经期记录,只保留孕期相关 + 通用项
    return [
      { type: "weight" as SheetType, icon: "⚖️", label: "孕期体重", value: record.weight ? `${record.weight} kg` : "" },
      ...periodCommon(record),
    ];
  }
  if (currentMode.value === "MOM") {
    // 宝妈模式:经期相关通用 + 育儿记录 + 通用项
    return [
      ...periodBlock(record),
      { type: "babyRecord" as SheetType, icon: "🍼", label: "宝宝成长日记", value: record.babyFeed || record.babyCry },
      ...periodCommon(record),
    ];
  }
  if (currentMode.value === "PREPARE") {
    // 备孕模式:经期相关通用 + 备孕特色 + 通用项
    return [
      ...periodBlock(record),
      { type: "prepareDiet" as SheetType, icon: "🥦", label: "备孕饮食打卡", value: arrayText(record.dietTags) },
      { type: "ovuTest" as SheetType, icon: "🔬", label: "排卵试纸", value: valueLabel(record.ovuTestResult, ovuOptions) },
      { type: "pregTest" as SheetType, icon: "🤰", label: "早孕试纸", value: valueLabel(record.pregTestResult, pregOptions) },
      { type: "bodyTemp" as SheetType, icon: "🌡️", label: "基础体温", value: record.bodyTemp ? `${record.bodyTemp} °C` : "" },
      { type: "intercourse" as SheetType, icon: "💕", label: "甜蜜时刻", value: boolText(record.intercourse) },
      ...periodCommon(record),
    ];
  }
  // 经期模式(默认):经期相关通用 + 经期特色 + 通用项
  return [
    ...periodBlock(record),
    { type: "leucorrhea" as SheetType, icon: "💧", label: "白带观察", value: leucorrheaText(record) },
    { type: "coc" as SheetType, icon: "💊", label: "避孕方式", value: valueLabel(record.coc ?? record.cocType, cocOptions) },
    { type: "intercourse" as SheetType, icon: "💕", label: "甜蜜时刻", value: boolText(record.intercourse) },
    { type: "weight" as SheetType, icon: "⚖️", label: "体重管理", value: record.weight ? `${record.weight} kg` : "" },
    ...periodCommon(record),
  ];
});
const symptomLogSummary = computed(() => {
  const count = periodStore.symptomLogs.filter((log) => String(log.occurredAt).slice(0, 10) === selectedDate.value).length;
  return count ? `${count} 条` : "";
});

const sheetTitle = computed(() => {
  const titles: Record<string, string> = {
  dayDetail: "当天概况", recordFull: "完善记录细节",
  discomfort: "记录经期体感", sleep: "记录睡眠", notes: "心情笔记", weight: "记录体重", bodyTemp: "记录基础体温",
  ovuTest: "排卵试纸", pregTest: "早孕试纸", startConfirm: "月经来啦", endConfirm: "月经走啦",
  stateSwitch: "切换模式", modeSettings: "设置模式日期", symptomLog: "记录症状日志", habits: "好习惯打卡",
  symptoms: "身体小症状", intercourse: "甜蜜时刻", leucorrhea: "白带观察", coc: "避孕方式",
  prepareDiet: "备孕营养打卡", babyRecord: "宝宝成长日记",
  };
  return titles[periodStore.activeSheet ?? ""] ?? "记录";
});

const numericValue = computed({
  get: () => String(periodStore.activeSheet === "weight" ? form.weight ?? "" : form.bodyTemp ?? ""),
  set: (value: string) => { form[periodStore.activeSheet === "weight" ? "weight" : "bodyTemp"] = value; },
});

const sleepHoursText = computed(() => {
  if (!form.sleepStartTime || !form.sleepEndTime) return "";
  let minutes = timeMinutes(form.sleepEndTime) - timeMinutes(form.sleepStartTime);
  if (minutes <= 0) minutes += 24 * 60;
  return `${(minutes / 60).toFixed(1)} 小时`;
});

function isoDate(date: Date) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  const d = String(date.getDate()).padStart(2, "0");
  return `${y}-${m}-${d}`;
}

function dateAt(key: string) {
  return new Date(`${key}T00:00:00`);
}

function addDays(key: string, amount: number) {
  const date = dateAt(key);
  date.setDate(date.getDate() + amount);
  return isoDate(date);
}

function options(values: string[]): Option[] {
  return values.map((value) => ({ value, label: value }));
}

function recordStart(record: MenstrualRecord) {
  return String((record as any).recordDate || record.startDate).slice(0, 10);
}

/** 纯经期开始记录(startConfirm 创建):所有症状为默认值,且无日常字段 */
function isPeriodStartRecord(record: any): boolean {
  const dailyFields = [record.mood, record.sleepQuality, record.sleepHours, record.weight,
    record.bodyTemp, record.ovuTestResult, record.pregTestResult, record.leuColor,
    record.notes, record.babyFeed, record.babyCry, record.intercourse, record.exercise,
    record.bloodColor, record.cocType, record.dietTags];
  const allDefaults = (record.painLevel ?? "NONE") === "NONE"
    && (record.flowLevel ?? "MEDIUM") === "MEDIUM"
    && (record.discomfort ?? "NONE") === "NONE";
  return allDefaults && dailyFields.every((v) => v == null) && !record.endDate;
}

/** 是否有经期体感特征(非默认值),用于判定经期是否在进行中 */
function hasPeriodSymptoms(record: any): boolean {
  return (record.flowLevel && record.flowLevel !== "MEDIUM")
    || (record.painLevel && record.painLevel !== "NONE")
    || (record.discomfort && record.discomfort !== "NONE")
    || !!record.bloodColor
    || (Array.isArray(record.bloodState) && record.bloodState.includes("PERIOD_CONFIRMED"));
}

/** 判断是否为经期相关记录(startConfirm 创建 或 有经期体感),用于 ongoing 判定 */
function isPeriodEpisode(record: any): boolean {
  return isPeriodStartRecord(record) || hasPeriodSymptoms(record);
}

function findRecordForDate(key: string) {
  return periodStore.findRecordForDate(key, today);
}

function normalizedProbability(value: unknown) {
  const raw = Number(value || 0);
  return raw > 1 ? raw / 100 : raw;
}

/** 预计算 period/predicted/ovulation 三个日期集合，只依赖 records 和 cyclePrediction */
const periodSets = computed(() => {
  const period = new Set<string>();
  const predicted = new Set<string>();
  const ovulation = new Set<string>();
  let peak = String(periodStore.cyclePrediction?.fertileWindow?.peakDay || periodStore.assessment?.predictedOvulationDate || "");

  periodStore.records.forEach((record) => {
    // 仅经期记录标粉色(经期开始或经期体感)
    if (!isPeriodEpisode(record)) return;
    const start = recordStart(record);
    // 进行中的经期(无 endDate):扩展至预期经期长度,让日历上能看到完整经期
    const avgDuration = Number(periodStore.assessment?.averageDurationDays || 5);
    const end = record.endDate ? String(record.endDate).slice(0, 10) : addDays(start, avgDuration - 1);
    for (let key = start; key <= end && key <= addDays(start, 10); key = addDays(key, 1)) period.add(key);
  });

  const probabilities = periodStore.cyclePrediction?.dailyProbabilities ?? [];
  probabilities.forEach((day: any) => {
    const periodProb = normalizedProbability(day.periodProb);
    const fertilityProb = normalizedProbability(day.fertilityProb);
    const ovulationProb = normalizedProbability(day.ovulationProb);
    if (periodProb >= .25 && !period.has(day.date)) predicted.add(day.date);
    if (fertilityProb >= .25 || ovulationProb >= .25) ovulation.add(day.date);
    if (ovulationProb >= .65) peak = day.date;
  });

  const latestStart = [...periodStore.records].map(recordStart).sort().pop();
  const avgCycle = Number(periodStore.cyclePrediction?.avgCycle || periodStore.assessment?.avgCycle || 28);
  // 预测经期长度：优先用评估里的平均持续天数,兜底 5 天
  const avgDuration2 = Number(periodStore.assessment?.averageDurationDays || 5);
  const horizonDays = 120;
  let next = String(periodStore.cyclePrediction?.predictedNextStart || periodStore.assessment?.predictedNextStart || "").slice(0, 10);
  // 兜底：只要有一条历史记录就用 latestStart + avgCycle 推算
  if (!next && latestStart) next = addDays(latestStart, avgCycle);
  // next 推进到 today 之后(覆盖历史预测)
  while (next && next < today) next = addDays(next, avgCycle);
  if (next) {
    // 生成接下来 ~120 天(约 4 个周期)的预测窗口：每个周期开头 avgDuration2 天标 predicted
    for (let offset = 0; offset < horizonDays; offset += 1) {
      const key = addDays(next, offset);
      const inCycleWindow = (offset % avgCycle) < avgDuration2;
      if (inCycleWindow && !period.has(key)) predicted.add(key);
    }
    // 排卵与易孕窗口：与经期预测同周期数,每周期 7 天
    const maxCycles = Math.ceil(horizonDays / avgCycle);
    for (let cycle = 0; cycle < maxCycles; cycle += 1) {
      const cycleStart = addDays(next, cycle * avgCycle);
      const ovu = addDays(cycleStart, -14);
      for (let off = -5; off <= 1; off += 1) ovulation.add(addDays(ovu, off));
      if (!peak) peak = ovu;
    }
  }
  const fertile = periodStore.cyclePrediction?.fertileWindow;
  if (fertile?.start && fertile?.end) for (let key = fertile.start; key <= fertile.end; key = addDays(key, 1)) ovulation.add(key);
  return { period, predicted, ovulation, peak };
});

function calendarKindForDate(key: string): CalendarKind {
  // 怀孕模式:返回孕期阶段标记(t1/t2/t3/due)
  if (currentMode.value === "PREGNANT") {
    const p = pregnancySets.value;
    if (p.due && key === p.due) return "due";
    if (p.t3.has(key)) return "t3";
    if (p.t2.has(key)) return "t2";
    if (p.t1.has(key)) return "t1";
    return "none";
  }
  const sets = periodSets.value;
  if (sets.period.has(key)) return "period";
  if (sets.peak === key) return "peak";
  if (sets.ovulation.has(key)) return "ovulation";
  if (sets.predicted.has(key)) return "predicted";
  return "none";
}

// 孕期产检里程碑(周序,取该周第一天)
const prenatalCheckups: Array<{ week: number; label: string }> = [
  { week: 6, label: "早孕" },
  { week: 12, label: "NT" },
  { week: 16, label: "唐筛" },
  { week: 20, label: "排畸" },
  { week: 24, label: "糖耐" },
  { week: 30, label: "小排" },
  { week: 34, label: "胎心" },
  { week: 36, label: "评估" },
  { week: 37, label: "产检" },
];
function checkupForWeek(w: number) {
  return prenatalCheckups.find((c) => c.week === w)?.label ?? "";
}

/** 孕期日期集合:LMP → 预产期,划分 T1(孕早期 1-13 周)/ T2(孕中期 14-27 周)/ T3(孕晚期 28-40 周) */
const pregnancySets = computed(() => {
  const empty = { t1: new Set<string>(), t2: new Set<string>(), t3: new Set<string>(), due: "" };
  if (currentMode.value !== "PREGNANT") return empty;
  const state: any = periodStore.menstrualState ?? {};
  const lmp = state.pregnantStartDate ? String(state.pregnantStartDate).slice(0, 10) : "";
  if (!lmp) return empty;
  const due = state.dueDate ? String(state.dueDate).slice(0, 10) : addDays(lmp, 280);
  const t1 = new Set<string>();
  const t2 = new Set<string>();
  const t3 = new Set<string>();
  // T1: 1-13 周 → 第 1-90 天; T2: 14-27 周 → 第 91-195 天; T3: 28-40 周 → 第 196-280 天
  for (let d = 0; d <= 90; d += 1) t1.add(addDays(lmp, d));
  for (let d = 91; d <= 195; d += 1) t2.add(addDays(lmp, d));
  for (let d = 196; d <= 280; d += 1) t3.add(addDays(lmp, d));
  return { t1, t2, t3, due };
});

function buildCalendar(offset: number) {
  const base = new Date();
  const target = new Date(base.getFullYear(), base.getMonth() + offset, 1);
  const year = target.getFullYear();
  const month = target.getMonth();
  const first = new Date(year, month, 1);
  const cells: CalendarCell[] = [];
  const probabilities = new Map<string, number>();
  for (const item of periodStore.cyclePrediction?.dailyProbabilities ?? []) {
    const raw = Number(item.periodProb || 0);
    probabilities.set(item.date, raw <= 1 ? Math.round(raw * 100) : Math.round(raw));
  }
  const sortedStarts = [...periodStore.records].filter(isPeriodEpisode).map(recordStart).sort();
  const latestStart = sortedStarts[sortedStarts.length - 1];
  // 所有记录的日期(经期 + 日常),给 hasRecord 黄点用
  const allRecordDates = new Set<string>();
  for (const r of periodStore.records) {
    allRecordDates.add(recordStart(r));
  }
  const symptomDateSet = new Set<string>();
  for (const log of periodStore.symptomLogs) {
    symptomDateSet.add(String(log.occurredAt).slice(0, 10));
  }

  const makeCell = (date: Date, isOut: boolean): CalendarCell => {
    const key = isoDate(date);
    const cycleDay = latestStart && key >= latestStart ? Math.floor((dateAt(key).getTime() - dateAt(latestStart).getTime()) / 86400000) + 1 : undefined;
    // 怀孕模式:基于 LMP 计算当前孕周(整周)与孕天数
    let pregnancyDay: number | undefined;
    let pregnancyWeek: number | undefined;
    if (currentMode.value === "PREGNANT") {
      const lmp = (periodStore.menstrualState as any)?.pregnantStartDate;
      if (lmp) {
        pregnancyDay = Math.floor((dateAt(key).getTime() - dateAt(String(lmp).slice(0, 10)).getTime()) / 86400000) + 1;
        if (pregnancyDay >= 1 && pregnancyDay <= 280) {
          pregnancyWeek = Math.floor((pregnancyDay - 1) / 7) + 1;
        } else {
          pregnancyDay = undefined;
          pregnancyWeek = undefined;
        }
      }
    }
    return {
      key,
      day: date.getDate(),
      kind: calendarKindForDate(key),
      isToday: key === today,
      isOut,
      probability: probabilities.get(key) ?? 0,
      hasRecord: allRecordDates.has(key) || symptomDateSet.has(key),
      cycleDay: cycleDay && cycleDay > 0 && cycleDay <= 45 ? cycleDay : undefined,
      pregnancyWeek,
      pregnancyDay,
      checkupText: currentMode.value === "PREGNANT" && pregnancyWeek ? checkupForWeek(pregnancyWeek) : "",
    };
  };
  for (let i = first.getDay(); i > 0; i -= 1) {
    const date = new Date(year, month, 1 - i);
    cells.push(makeCell(date, true));
  }
  const count = new Date(year, month + 1, 0).getDate();
  for (let day = 1; day <= count; day += 1) {
    cells.push(makeCell(new Date(year, month, day), false));
  }
  while (cells.length % 7) {
    const date = new Date(year, month + 1, cells.length - first.getDay() - count + 1);
    cells.push(makeCell(date, true));
  }
  return { year, month, cells };
}

function valueLabel(value: unknown, options: Option[]) {
  return options.find((item) => item.value === value)?.label ?? "";
}

function arrayText(value: unknown) {
  return Array.isArray(value) ? value.join("、") : "";
}

function boolText(value: unknown) {
  if (value === true || value === "true" || value === "yes") return "有";
  if (value === false || value === "false" || value === "no") return "没有";
  return "";
}

function leucorrheaText(record: any) {
  const values = Array.isArray(record.whiteLeucorrhea) ? record.whiteLeucorrhea : [];
  return [
    valueLabel(record.leuColor ?? values[0], leuColorOptions),
    valueLabel(record.leuState ?? values[1], leuStateOptions),
    valueLabel(record.leuOther ?? values[2], leuOtherOptions),
  ].filter(Boolean).join(" · ");
}

function changeMonth(delta: number) {
  periodStore.navigateMonth(delta);
}

function selectDate(key: string) {
  selectedDate.value = key;
  openSheet("dayDetail");
}

function goToday() {
  selectedDate.value = today;
  periodStore.monthOffset = 0;
}

function resetForm(record?: MenstrualRecord) {
  Object.keys(form).forEach((key) => delete form[key]);
  if (!record) return;
  Object.assign(form, record);
  const value: any = record;
  form.habitsArr = value.habitsArr ?? value.habits ?? [];
  form.symptomsArr = value.symptomsArr ?? value.symptoms ?? [];
  form.coc = value.coc ?? value.cocType;
  form.bloodStateTags = value.bloodStateTags ?? value.bloodState ?? [];
  if (Array.isArray(value.whiteLeucorrhea)) {
    form.leuColor ??= value.whiteLeucorrhea[0];
    form.leuState ??= value.whiteLeucorrhea[1];
    form.leuOther ??= value.whiteLeucorrhea[2];
  }
  if (record.sleepStart) form.sleepStartTime = localTime(record.sleepStart);
  if (record.sleepEnd) form.sleepEndTime = localTime(record.sleepEnd);
}

function openSheet(type: SheetType) {
  resetForm(selectedRecord.value);
  if (type === "symptomLog") Object.assign(form, { selectedTags: [], severity: 3 });
  periodStore.openSheet(type, selectedDate.value, { ...form });
}

function openQuickItem(item: { type: SheetType; primary?: boolean }) {
  openSheet(item.type);
}

function wakeAiAssistant() {
  // 周期页快捷记录项使用下面这些「推荐字段」时，对话中触发自动分析。
  // 点击「问小美」直接跳到聊天并自动生成一次当日分析报告；
  // 同时把当前日期写入待办，待用户在聊天里发送与今日经期相关的内容时再次触发分析。
  const today = new Date().toISOString().slice(0, 10);
  uni.setStorageSync("pendingAutoAnalysisDate", today);
  uni.removeStorageSync("pendingAiPrompt");
  uni.switchTab({ url: "/pages/chat/index" });
}

function openTodayPeriodConfirm() {
  openFromDay(selectedPeriodAction.value);
}

function openFromDay(type: SheetType) {
  closeSheet();
  openSheet(type);
}

function selectChoice(payload: { field: string; value: string | number; multi: boolean }) {
  if (!payload.multi) {
    form[payload.field] = payload.value;
    return;
  }
  const values = Array.isArray(form[payload.field]) ? [...form[payload.field]] : [];
  const index = values.indexOf(payload.value);
  if (index >= 0) values.splice(index, 1);
  else values.push(payload.value);
  form[payload.field] = values;
}

function pickMode(mode: Mode) {
  if (mode === currentMode.value) return closeSheet();
  if (mode === "PERIOD") {
    form.targetMode = mode;
    void saveMode();
    return;
  }
  Object.keys(form).forEach((key) => delete form[key]);
  form.targetMode = mode;
  const key = mode === "PREPARE" ? "prepareStartDate" : mode === "PREGNANT" ? "pregnantStartDate" : "babyBirthDate";
  form[key] = (periodStore.menstrualState as any)[key] ?? selectedDate.value;
  periodStore.openSheet("modeSettings", selectedDate.value, { ...form });
}

function closeSheet() {
  periodStore.closeSheet();
}

function setPickerValue(key: string, event: any) {
  form[key] = event.detail.value;
}

function localTime(value: string) {
  const date = new Date(value);
  return Number.isNaN(date.getTime()) ? "" : `${String(date.getHours()).padStart(2, "0")}:${String(date.getMinutes()).padStart(2, "0")}`;
}

function timeMinutes(value: string) {
  const [hours, minutes] = value.split(":").map(Number);
  return hours * 60 + minutes;
}

function sleepIso(date: string, time: string, nextDay = false) {
  const target = dateAt(date);
  if (nextDay) target.setDate(target.getDate() + 1);
  const [hours, minutes] = time.split(":").map(Number);
  target.setHours(hours, minutes, 0, 0);
  return target.toISOString();
}

async function saveMode() {
  const mode = (form.targetMode || currentMode.value) as Mode;
  const body: Record<string, string> = { mode };
  if (mode === "PREPARE" && form.prepareStartDate) body.prepareStartDate = form.prepareStartDate;
  if (mode === "PREGNANT" && form.pregnantStartDate) {
    body.pregnantStartDate = form.pregnantStartDate;
    body.dueDate = addDays(form.pregnantStartDate, 280);
  }
  if (mode === "MOM" && form.babyBirthDate) body.babyBirthDate = form.babyBirthDate;
  await periodStore.updateState(body);
  appStore.showToast("模式已切换", "success");
  closeSheet();
}

async function saveSheet() {
  if (saving.value) return;
  saving.value = true;
  try {
    if (periodStore.activeSheet === "modeSettings") {
      await saveMode();
    } else if (periodStore.activeSheet === "symptomLog") {
      await periodStore.createSymptomLog({
        occurredAt: `${selectedDate.value}T12:00:00`,
        tags: Array.isArray(form.selectedTags) ? form.selectedTags : [],
        severity: Number(form.severity || 3),
        category: "PHYSICAL",
        notes: "",
      });
    } else if (periodStore.activeSheet === "startConfirm") {
      // 经期开始:走 saveDailyRecord 合并已有记录(如首页已记的经期体感),不重复建
      const ongoing = ongoingPeriod.value;
      if (ongoing && selectedDate.value < recordStart(ongoing)) {
        await periodStore.updateRecord(ongoing.id, { startDate: selectedDate.value });
      } else {
        await periodStore.saveDailyRecord(selectedDate.value, {
          startDate: selectedDate.value,
          bloodState: [...new Set([...(selectedRecord.value && Array.isArray((selectedRecord.value as any).bloodState) ? (selectedRecord.value as any).bloodState : []), "PERIOD_CONFIRMED"])],
        }, today);
      }
    } else if (periodStore.activeSheet === "endConfirm") {
      const open = periodStore.records.find((record) => !record.endDate && recordStart(record) <= selectedDate.value);
      if (!open) throw new Error("当前没有进行中的经期");
      await periodStore.updateRecord(open.id, { endDate: selectedDate.value });
    } else {
      const body: Record<string, any> = { recordDate: selectedDate.value };
      const fields: Record<string, string[]> = {
        discomfort: ["flowLevel", "painLevel", "discomfort", "bloodColor"],
        recordFull: ["flowLevel", "painLevel", "discomfort", "bloodColor", "sleepQuality", "mood", "notes"],
        sleep: ["sleepQuality"], notes: ["mood", "notes"], weight: ["weight"], bodyTemp: ["bodyTemp"],
        ovuTest: ["ovuTestResult"], pregTest: ["pregTestResult"],
        habits: ["habitsArr"], symptoms: ["symptomsArr"], intercourse: ["intercourse"],
        leucorrhea: ["leuColor", "leuState", "leuOther"], coc: ["coc"],
        prepareDiet: ["dietTags", "dietNotes"], babyRecord: ["babyFeed", "babyCry", "babyDiaper", "babySleep", "babyNotes"],
      };
      for (const key of fields[periodStore.activeSheet ?? ""] ?? []) {
        if (form[key] !== undefined && form[key] !== "") body[key] = ["weight", "bodyTemp"].includes(key) ? Number(form[key]) : form[key];
      }
      if (periodStore.activeSheet === "sleep") {
        const start = String(form.sleepStartTime || "");
        const end = String(form.sleepEndTime || "");
        body.sleepStart = start ? sleepIso(selectedDate.value, start) : null;
        body.sleepEnd = end ? sleepIso(selectedDate.value, end, !!start && timeMinutes(end) <= timeMinutes(start)) : null;
      }
      if (periodStore.activeSheet === "intercourse") body.intercourse = form.intercourse === "yes";
      if (periodStore.activeSheet === "habits" || periodStore.activeSheet === "recordFull") body.habits = form.habitsArr ?? [];
      if (periodStore.activeSheet === "symptoms") body.symptoms = form.symptomsArr ?? [];
      if (periodStore.activeSheet === "leucorrhea") body.whiteLeucorrhea = [form.leuColor, form.leuState, form.leuOther].filter(Boolean);
      if (periodStore.activeSheet === "discomfort" || periodStore.activeSheet === "recordFull") body.bloodState = form.bloodStateTags ?? [];
      if (periodStore.activeSheet === "coc") body.cocType = form.coc;
      await periodStore.saveDailyRecord(selectedDate.value, body, today);
    }
    appStore.showToast("已保存", "success");
    closeSheet();
  } catch (error: any) {
    appStore.showToast(error?.message || "保存失败", "error");
  } finally {
    saving.value = false;
  }
}

onMounted(() => {
  if (!periodStore.records.length) periodStore.loadAll();
});

// 处理来自首页"点击补充"的请求:打开对应的 sheet,默认日期用请求中的日期
function consumeHomePendingSheet() {
  const sheet = String(uni.getStorageSync("homePendingSheet") || "");
  const date = String(uni.getStorageSync("homePendingSheetDate") || "");
  if (!sheet) return;
  uni.removeStorageSync("homePendingSheet");
  uni.removeStorageSync("homePendingSheetDate");
  if (date) selectedDate.value = date;
  periodStore.openSheet(sheet as SheetType, selectedDate.value, { ...form });
}
onShow(() => {
  consumeHomePendingSheet();
});
</script>

<style lang="scss" scoped>
.period-page { min-height: 100vh; padding: $gap-md $gap-md 120rpx; background: $surface; }
.status-pill { display: flex; justify-content: space-between; gap: 16rpx; padding: 16rpx 24rpx; border-radius: $radius-full; color: #fff; font-size: $font-xs; font-weight: 800; }
.status-pill__date { opacity: .88; }
.mode-card { display: flex; align-items: center; justify-content: space-between; margin-top: $gap-md; }
.mode-card__eyebrow, .mode-card__title, .mode-card__desc { display: block; }
.mode-card__eyebrow { color: $rose-dark; font-size: 20rpx; font-weight: 800; }
.mode-card__title { display:flex;align-items:center;gap:10rpx;margin-top:4rpx;font-size:$font-lg;font-weight:800; }
.mode-card__desc { color: $muted; font-size: $font-xs; }
.mode-card__action { padding: 12rpx 20rpx; border-radius: $radius-full; background: $blush; color: $rose-dark; font-size: $font-xs; font-weight: 800; }
.calendar { margin-top: $gap-md; padding: 24rpx; }
.calendar { position: relative; overflow: hidden; }
.calendar__header { display: flex; align-items: center; justify-content: space-between; text-align: center; }
.calendar__title, .calendar__subtitle { display: block; }
.calendar__title { font-size: $font-lg; font-weight: 800; }
.calendar__subtitle { color: $muted; font-size: 20rpx; }
.calendar__nav { display: grid; width: 58rpx; height: 58rpx; place-items: center; border-radius: 50%; background: $blush; color: $rose-dark; font-size: 42rpx; line-height: 1; }
.calendar__week, .calendar__grid { display: grid; grid-template-columns: repeat(7, 1fr); }
.calendar__week { margin-top: 22rpx; color: $muted; font-size: 20rpx; text-align: center; }
.calendar__grid { margin-top: 8rpx; row-gap: 8rpx; }
.calendar-day { position: relative; display: flex; height: 78rpx; align-items: center; justify-content: center; flex-direction: column; overflow: hidden; border-radius: 50%; color: $ink; transition: transform .18s ease; }
.calendar-day:active { transform: scale(.86); }
.calendar-day--out { opacity: .28; }
.calendar-day--today { box-shadow: inset 0 0 0 4rpx $rose-dark; }
.calendar-day--selected { outline: 4rpx solid rgba(236,72,153,.24); outline-offset: 2rpx; }
.calendar-day--period { background: $rose-dark; color: #fff; box-shadow: 0 8rpx 20rpx rgba(236,72,153,.28); }
.calendar-day--predicted { background: repeating-linear-gradient(135deg,#fff1f2 0,#fff1f2 8rpx,#fbcfe8 8rpx,#fbcfe8 13rpx); color: #be185d; box-shadow: inset 0 0 0 3rpx #f472b6; }
.calendar-day--ovulation { background: #ede9fe; color: #6d28d9; box-shadow: inset 0 0 0 3rpx #a78bfa; }
.calendar-day--peak { background: #8b5cf6; color: #fff; box-shadow: 0 8rpx 20rpx rgba(139,92,246,.3); }
.calendar-day--t1 { color: #be185d; }
.calendar-day--t1::after { content: ""; position: absolute; bottom: 8rpx; left: 50%; transform: translateX(-50%) rotate(-1.5deg); width: 22rpx; height: 4rpx; border-radius: 2rpx; background: #f9a8d4; }
.calendar-day--t2 { color: #7c3aed; }
.calendar-day--t2::after { content: ""; position: absolute; bottom: 8rpx; left: 50%; transform: translateX(-50%) rotate(1deg); width: 22rpx; height: 4rpx; border-radius: 2rpx; background: #c4b5fd; }
.calendar-day--t3 { color: #c2410c; }
.calendar-day--t3::after { content: ""; position: absolute; bottom: 8rpx; left: 50%; transform: translateX(-50%) rotate(-0.5deg); width: 22rpx; height: 4rpx; border-radius: 2rpx; background: #fdba74; }
.calendar-day--due { color: $rose-dark; }
.calendar-day--due::after { content: ""; position: absolute; bottom: 8rpx; left: 50%; transform: translateX(-50%) rotate(-1deg); width: 30rpx; height: 4rpx; border-radius: 2rpx; background: $rose-dark; }
.calendar-day__number { font-size: $font-sm; font-weight: 700; }
.calendar-day__meta { margin-top: -2rpx; font-size: 13rpx; font-weight: 700; opacity: .72; }
.calendar-day__meta--pregnant { font-size: 11rpx; opacity: .85; }
.calendar-day__pred { position: absolute; right: 6rpx; top: 4rpx; z-index: 2; font-size: 18rpx; line-height: 1; }
.calendar-day__record { position: absolute; right: 8rpx; top: 8rpx; z-index: 2; width: 8rpx; height: 8rpx; border: 2rpx solid #fff; border-radius: 50%; background: #f59e0b; }
.calendar-day__prob { position: absolute; right: 12rpx; bottom: 5rpx; left: 12rpx; height: 5rpx; border-radius: 99rpx; background: currentColor; }
.calendar__legend { display: flex; justify-content: center; gap: 22rpx; margin-top: 18rpx; color: $muted; font-size: 20rpx; }
.calendar__legend text { display: flex; align-items: center; gap: 6rpx; }
.legend-dot { display: inline-block; width: 12rpx; height: 12rpx; border-radius: 50%; }
.legend-dot--period { background: $rose-dark; }.legend-dot--predicted { border: 2rpx solid #f472b6; background: #fbcfe8; }.legend-dot--ovulation { border: 2rpx solid #8b5cf6; background: #ede9fe; }
.legend-dot--t1 { background: #fce7f3; border: 2rpx solid #f9a8d4; }.legend-dot--t2 { background: #ede9fe; border: 2rpx solid #c4b5fd; }.legend-dot--t3 { background: #fff7ed; border: 2rpx solid #fdba74; }.legend-dot--due { background: $rose-dark; }
.selected-card { display: flex; align-items: center; justify-content: space-between; margin-top: $gap-md; }
.selected-card__eyebrow, .selected-card__title, .selected-card__desc { display: block; }
.selected-card__eyebrow { color: $rose-dark; font-size: 20rpx; font-weight: 800; }
.selected-card__title { font-size: $font-lg; font-weight: 800; }
.selected-card__desc { color: $muted; font-size: $font-xs; }
.selected-card__today { padding: 12rpx 18rpx; border-radius: $radius-full; background: $blush; color: $rose-dark; font-size: $font-xs; font-weight: 700; }
.period-section-head { display:flex;align-items:center;justify-content:space-between;gap:20rpx;margin:26rpx 4rpx 12rpx; }
.period-section-title,.period-section-sub { display:block; }
.period-section-title { font-size:$font-lg;font-weight:800;line-height:1.35; }
.period-section-sub { margin-top:2rpx;color:$muted;font-size:20rpx; }
.period-ai-button { display:flex;min-width:148rpx;height:64rpx;flex:0 0 auto;align-items:center;justify-content:center;gap:8rpx;padding:0 18rpx;border-radius:$radius-full;background:$ink;color:#fff;font-size:$font-xs;font-weight:800;box-shadow:0 8rpx 18rpx rgba(48,48,57,.14); }
.period-ai-button__icon { color:#fbcfe8;font-size:26rpx; }
.quick-grid { display: grid; grid-template-columns: 1fr; gap: $gap-sm; }
.quick-item { display: grid; min-height: 104rpx; grid-template-columns: 54rpx 1fr auto 54rpx; align-items: center; gap: 16rpx; padding: 20rpx; border: 1px solid $line; border-radius: $radius-lg; background: $paper; box-shadow: $shadow-sm; }
.quick-item__icon { font-size: 42rpx; line-height: 1; }
.quick-item__label, .quick-item__value { display: block; }
.quick-item__label { font-size: $font-sm; font-weight: 800; }
.quick-item__value { max-width: 260rpx; overflow: hidden; color: $muted; font-size: 20rpx; text-overflow: ellipsis; white-space: nowrap; text-align: right; }
.quick-item__action { display:flex;width:54rpx;height:54rpx;align-items:center;justify-content:center;border-radius:50%;background:$surface;color:$ink;font-size:30rpx;font-weight:700; }
.quick-item--primary { border-color:#fbcfe8;background:linear-gradient(145deg,$blush,#fbcfe8); }
.quick-item--primary .quick-item__action { background:$rose-dark;color:#fff; }
.quick-item--editing .quick-item__action { background:transparent;color:$rose-dark; }
.sheet-backdrop { position: fixed; z-index: 100; inset: 0; display: flex; align-items: center; justify-content: center; padding: 28rpx; background: rgba(24,24,27,.5); backdrop-filter: blur(8px); }
.sheet-panel { display: flex; width: 100%; max-height: 86vh; flex-direction: column; padding: 24rpx 26rpx calc(24rpx + env(safe-area-inset-bottom)); border-radius: 36rpx; background: $paper; box-shadow: 0 36rpx 100rpx rgba(63,63,70,.24); animation: popup .28s ease-out; }
.sheet-header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 20rpx; }
.sheet-title, .sheet-date { display: block; }.sheet-title { font-size: $font-xl; font-weight: 800; }.sheet-date { color: $muted; font-size: $font-xs; }
.sheet-close { padding:12rpx;color:$muted;font-size:42rpx; }
.sheet-body { min-height: 0; flex: 1; max-height: 64vh; }
.mode-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14rpx; }
.mode-choice { display: grid; min-height: 128rpx; place-items: center; padding: 18rpx; border: 2rpx solid transparent; border-radius: $radius-lg; background: $surface; color: $ink; font-size: $font-sm; font-weight: 800; }
.mode-choice--active { border-color: $rose-dark; background: $blush; color: $rose-dark; }
.mode-choice__icon { font-size: 46rpx; line-height: 1; }
.form-section { margin-bottom: 18rpx; padding: 20rpx; border-radius: $radius-lg; background: $surface; }
.form-section__title { display: block; margin-bottom: 14rpx; font-size: $font-sm; font-weight: 800; }
.choice-grid { display: flex; flex-wrap: wrap; gap: 10rpx; }
.choice { flex: 1 0 27%; padding: 15rpx 10rpx; border: 2rpx solid transparent; border-radius: $radius-md; background: $paper; color: $ink; font-size: $font-xs; text-align: center; }
.choice--active { border-color: $rose-dark; background: $blush; color: $rose-dark; font-weight: 800; }
.choice-grid--icon { display: grid; grid-template-columns: repeat(4, 1fr); }
.choice--icon { display: flex; min-height: 104rpx; flex: initial; align-items: center; justify-content: center; flex-direction: column; gap: 5rpx; padding: 10rpx 5rpx; }
.choice-grid--level { display: grid; grid-template-columns: repeat(4, 1fr); }
.choice--level { display: flex; min-height: 88rpx; flex: initial; align-items: center; justify-content: center; flex-direction: column; gap: 4rpx; }
.choice__icon { color: $rose-dark; font-size: 26rpx; font-weight: 900; line-height: 1; }
.choice__label { font-size: 20rpx; font-weight: 700; }
.color-bar { display: flex; height: 72rpx; overflow: hidden; border-radius: $radius-full; }
.color-cell { display: flex; flex: 1; align-items: center; justify-content: center; color: #fff; font-size: 18rpx; font-weight: 800; opacity: .72; transition: flex .2s, opacity .2s; }
.color-cell--active { flex: 1.45; opacity: 1; box-shadow: inset 0 0 0 5rpx rgba(255,255,255,.85); }
.picker-field, .input-field, .text-area { width: 100%; border-radius: $radius-md; background: $paper; color: $ink; font-size: $font-md; }
.picker-field, .input-field { min-height: 76rpx; padding: 17rpx 20rpx; }
.text-area { height: 180rpx; padding: 18rpx; }
.form-hint { display: block; margin: -6rpx 0 18rpx; color: $rose-dark; font-size: $font-xs; text-align: center; }
.confirm-box { display: flex; align-items: center; flex-direction: column; padding: 40rpx 24rpx; border-radius: $radius-lg; background: $blush; color: $ink; text-align: center; }
.confirm-box__icon { font-size: 68rpx; line-height: 1; }
.confirm-box__title { margin-top: 20rpx; font-size: $font-lg; font-weight: 800; }
.confirm-box__copy { margin-top: 6rpx; color: $muted; font-size: $font-xs; }
.day-detail { padding: $gap-lg; border-radius: $radius-lg; background: $surface; }
.day-detail--period { background: #fff1f2; }.day-detail--predicted { background: $blush; }.day-detail--ovulation,.day-detail--peak { background: #f5f3ff; }
.day-detail--t1 { background: #fce7f3; }.day-detail--t2 { background: #ede9fe; }.day-detail--t3 { background: #fff7ed; }.day-detail--due { background: linear-gradient(135deg,#fff1f2,#fce7f3); }
.day-detail__status,.day-detail__summary,.day-detail__advice { display: block; }.day-detail__status { color: $ink; font-size: $font-xl; font-weight: 800; }.day-detail__summary { margin-top: 4rpx; color: $rose-dark; font-size: $font-sm; font-weight: 700; }.day-detail__advice { margin-top: $gap-md; color: $muted; font-size: $font-sm; line-height: 1.75; }
.day-hero { display: flex; align-items: center; gap: $gap-md; padding: $gap-lg; border-radius: $radius-lg; }
.day-hero--period { background: linear-gradient(135deg, #fff1f2, #fecdd3); }
.day-hero--predicted { background: linear-gradient(135deg, $blush, #fce7f3); }
.day-hero--ovulation { background: linear-gradient(135deg, #f5f3ff, #ede9fe); }
.day-hero--peak { background: linear-gradient(135deg, #ede9fe, #ddd6fe); }
.day-hero--t1 { background: linear-gradient(135deg, #fce7f3, #fbcfe8); }
.day-hero--t2 { background: linear-gradient(135deg, #ede9fe, #ddd6fe); }
.day-hero--t3 { background: linear-gradient(135deg, #fff7ed, #fed7aa); }
.day-hero--due { background: linear-gradient(135deg, #fff1f2, #fecdd3); }
.day-hero--none { background: linear-gradient(135deg, $surface, $paper); border: 1px solid $line; }
.day-hero__icon { display: grid; width: 110rpx; height: 110rpx; flex: 0 0 auto; place-items: center; border-radius: 50%; background: #fff; font-size: 56rpx; box-shadow: 0 6rpx 18rpx rgba(190, 18, 60, .08); }
.day-hero__body { min-width: 0; flex: 1; }
.day-hero__eyebrow { display: block; color: $rose-dark; font-size: $font-xs; font-weight: 800; letter-spacing: 1rpx; }
.day-hero__title { display: block; margin-top: 4rpx; color: $ink; font-size: $font-xl; font-weight: 800; line-height: 1.3; }
.day-hero__summary { display: block; margin-top: 4rpx; color: $muted; font-size: $font-sm; }
.day-advice { display: flex; align-items: flex-start; gap: $gap-sm; margin-top: $gap-sm; padding: $gap-md; background: $paper; border-radius: $radius-md; box-shadow: $shadow-sm; }
.day-advice__icon { font-size: 36rpx; line-height: 1.4; }
.day-advice__text { flex: 1; color: $ink; font-size: $font-sm; line-height: 1.75; }
.day-record-card { margin-top: $gap-sm; padding: $gap-md; }
.day-record-card__head { display: flex; align-items: center; justify-content: space-between; gap: $gap-sm; }
.day-record-card__title { color: $ink; font-size: $font-sm; font-weight: 800; }
.day-record-card__date { color: $muted; font-size: $font-xs; }
.day-record-card__copy { display: block; margin-top: $gap-xs; color: $muted; font-size: $font-sm; line-height: 1.7; }
.day-actions { display: flex; flex-direction: column; gap: $gap-sm; margin-top: $gap-md; }
.day-action { display: flex; align-items: center; gap: $gap-sm; padding: $gap-md; border: 1px solid $line; border-radius: $radius-md; background: $paper; color: $ink; font-size: $font-sm; box-shadow: $shadow-sm; }
.day-action__icon { display: grid; width: 64rpx; height: 64rpx; flex: 0 0 auto; place-items: center; border-radius: 50%; background: $blush; font-size: 30rpx; }
.day-action__body { min-width: 0; flex: 1; }
.day-action__title { display: block; color: $ink; font-size: $font-sm; font-weight: 800; }
.day-action__sub { display: block; margin-top: 2rpx; color: $muted; font-size: 20rpx; }
.day-action__arrow { color: $muted; font-size: 36rpx; line-height: 1; }
.day-action--primary { border-color: transparent; background: linear-gradient(135deg, $ink, #1f2937); color: #fff; box-shadow: 0 8rpx 18rpx rgba(0,0,0,.18); }
.day-action--primary .day-action__icon { background: rgba(255,255,255,.16); }
.day-action--primary .day-action__title { color: #fff; }
.day-action--primary .day-action__sub { color: rgba(255,255,255,.62); }
.day-action--primary .day-action__arrow { color: rgba(255,255,255,.7); }
.full-record-intro { display: block; margin-bottom: $gap-md; color: $muted; font-size: $font-sm; text-align: center; }
.sheet-actions { display: grid; grid-template-columns: 1fr 1.5fr; gap: 12rpx; margin-top: 18rpx; padding-top: 18rpx; border-top: 1px solid $line; }
.sheet-actions button { margin: 0; line-height: 1.2; }
@keyframes popup { from { transform: scale(.94); opacity: 0; } to { transform: scale(1); opacity: 1; } }
</style>
