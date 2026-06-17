<template>
  <view class="page period-page">
    <ClinicTopBar />
    
    <view class="status-panel card">
      <view class="status-panel__info">
        <text class="status-panel__label">系统当前追踪模式</text>
        <view class="status-panel__title">
          <text>{{ currentModeInfo.label }}</text>
        </view>
        <text class="status-panel__desc">{{ modeDateText }}</text>
      </view>
      <button class="btn-outline status-panel__action" @click="openSheet('stateSwitch')">切换模式</button>
    </view>

    <view class="calendar card">
      <view class="calendar__header">
        <view class="calendar__nav" @click="changeMonth(-1)">‹</view>
        <view class="calendar__title-group">
          <text class="calendar__title">{{ calendar.year }} 年 {{ calendar.month + 1 }} 月</text>
          <text class="calendar__subtitle">点击日期登记体征数据</text>
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
          <view v-if="cell.hasRecord" class="calendar-day__record" />
          <view v-if="cell.probability > 0 && !cell.isOut && currentMode !== 'PREGNANT'" class="calendar-day__prob" :style="{ opacity: Math.max(.18, cell.probability / 100) }" />
        </view>
      </view>
      
      <view v-if="currentMode === 'PREGNANT'" class="calendar__legend">
        <text><i class="legend-dot legend-dot--t1" />孕早期</text>
        <text><i class="legend-dot legend-dot--t2" />孕中期</text>
        <text><i class="legend-dot legend-dot--t3" />孕晚期</text>
        <text><i class="legend-dot legend-dot--due" />预产期</text>
      </view>
      <view v-else class="calendar__legend">
        <text><i class="legend-dot legend-dot--period" />经期记录</text>
        <text><i class="legend-dot legend-dot--predicted" />系统预测</text>
        <text><i class="legend-dot legend-dot--ovulation" />易孕/排卵</text>
      </view>
    </view>

    <view class="day-summary card">
      <view class="day-summary__header">
        <text class="day-summary__title">选中日期：{{ selectedDateLabel }}</text>
        <button class="btn-secondary day-summary__today" @click="goToday">返回今日</button>
      </view>
      <text class="day-summary__desc">{{ selectedDayDescription }}</text>
    </view>

    <view class="section-header">
      <view>
        <text class="section-title">临床数据录入</text>
        <text class="section-sub">选择相应的临床评估表单进行记录</text>
      </view>
      <button class="btn-primary btn-sm" @click="wakeAiAssistant">AI 咨询助手</button>
    </view>
    
    <view class="quick-grid">
      <view v-for="item in quickItems" :key="item.type || item.label" class="quick-item card" :class="{ 'quick-item--primary': item.primary, 'quick-item--editing': item.value }" @click="openQuickItem(item)">
        <view class="quick-item__info">
          <text class="quick-item__label">{{ item.label }}</text>
          <text class="quick-item__value">{{ item.value || '未录入数据' }}</text>
        </view>
        <view class="quick-item__action">{{ item.value ? '编辑' : '录入' }}</view>
      </view>
    </view>

    <view v-if="periodStore.activeSheet" class="overlay" @click="closeSheet">
      <view class="sheet-dialog" @click.stop>
        <view class="sheet-dialog__header">
          <text class="sheet-dialog__title">{{ sheetTitle }}</text>
          <text class="sheet-dialog__close" @click="closeSheet">×</text>
        </view>

        <scroll-view class="sheet-dialog__body" scroll-y>
          <view v-if="periodStore.activeSheet === 'startConfirm' || periodStore.activeSheet === 'endConfirm'" class="confirm-box">
            <text class="confirm-box__title">{{ periodStore.activeSheet === 'startConfirm' ? '确认登记：月经期开始' : '确认登记：月经期结束' }}</text>
            <text class="confirm-box__copy">{{ periodStore.activeSheet === 'startConfirm' ? '将该日期标记为本次周期的起点，后续评估将以此为基准。' : '将该日期标记为出血完全停止日，系统将归档本次周期数据。' }}</text>
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
                <text>{{ mode.label }}</text>
              </view>
            </view>
          </template>

          <template v-else-if="periodStore.activeSheet === 'modeSettings'">
            <view class="form-section">
              <text class="form-section__title">{{ modeSettingLabel }}</text>
              <picker mode="date" :value="form[modeSettingKey]" @change="setPickerValue(modeSettingKey, $event)">
                <view class="form-input">{{ form[modeSettingKey] || '请选择日期' }}</view>
              </picker>
            </view>
          </template>

          <template v-else-if="periodStore.activeSheet === 'dayDetail'">
            <view class="day-hero" :class="`day-hero--${selectedKind}`">
              <view class="day-hero__body">
                <text class="day-hero__eyebrow">{{ dayDetailKindLabel }}阶段</text>
                <text class="day-hero__title">{{ selectedDayStatus }}</text>
                <text class="day-hero__summary">{{ selectedRecord ? '该日存在临床记录' : selectedDate > today ? '未来预估数据' : '该日无临床记录' }}</text>
              </view>
            </view>

            <view class="day-advice card">
              <text class="day-advice__label">临床建议：</text>
              <text class="day-advice__text">{{ selectedDayAdvice }}</text>
            </view>

            <view v-if="selectedRecord && selectedRecordSummary" class="day-record-card card">
              <view class="day-record-card__head">
                <text class="day-record-card__title">已归档指征</text>
              </view>
              <text class="day-record-card__copy">{{ selectedRecordSummary }}</text>
            </view>

            <view v-if="selectedDate <= today" class="day-actions">
              <view v-if="canRecordPeriod" class="day-action card" @click="openTodayPeriodConfirm">
                <view class="day-action__body">
                  <text class="day-action__title">{{ selectedPeriodAction === 'endConfirm' ? '结束当前周期' : '标记新周期开始' }}</text>
                </view>
              </view>
              <view class="day-action day-action--primary card" @click="openFromDay('recordFull')">
                <view class="day-action__body">
                  <text class="day-action__title">综合指征评估</text>
                  <text class="day-action__sub">系统化录入多维度体征数据</text>
                </view>
              </view>
            </view>
          </template>

          <template v-else-if="periodStore.activeSheet === 'recordFull'">
            <text class="full-record-intro text-muted">综合评估表单</text>
            <choice-section title="疼痛评估量表 (VAS)" :options="painOptions" field="painLevel" :value="form.painLevel" visual="level" @select="selectChoice" />
            <choice-section title="出血量评估" :options="flowOptions" field="flowLevel" :value="form.flowLevel" visual="level" @select="selectChoice" />
            <choice-section title="伴随症状 (多选)" :options="discomfortOptions" field="discomfort" :value="form.discomfort" multi visual="icon" @select="selectChoice" />
            <choice-section title="睡眠质量评估" :options="sleepOptions" field="sleepQuality" :value="form.sleepQuality" visual="icon" @select="selectChoice" />
            <choice-section title="心理状态评估" :options="moodOptions" field="mood" :value="form.mood" visual="icon" @select="selectChoice" />
            <view class="form-section">
              <text class="form-section__title">附加临床备注</text>
              <textarea v-model="form.notes" class="form-textarea" maxlength="500" placeholder="请输入其他值得关注的临床指征..." />
            </view>
          </template>

          <template v-else-if="periodStore.activeSheet === 'discomfort'">
            <choice-section title="疼痛评估量表 (VAS)" :options="painOptions" field="painLevel" :value="form.painLevel" visual="level" @select="selectChoice" />
            <choice-section title="出血量评估" :options="flowOptions" field="flowLevel" :value="form.flowLevel" visual="level" @select="selectChoice" />
            <choice-section title="血液性状 (多选)" :options="bloodStateOptions" field="bloodStateTags" :value="form.bloodStateTags" multi visual="icon" @select="selectChoice" />
            <choice-section title="伴随症状" :options="discomfortOptions" field="discomfort" :value="form.discomfort" visual="icon" @select="selectChoice" />
          </template>

          <template v-else-if="periodStore.activeSheet === 'sleep'">
            <choice-section title="睡眠质量评估" :options="sleepOptions" field="sleepQuality" :value="form.sleepQuality" visual="icon" @select="selectChoice" />
            <view class="form-section">
              <text class="form-section__title">入睡时间</text>
              <picker mode="time" :value="form.sleepStartTime" @change="setPickerValue('sleepStartTime', $event)">
                <view class="form-input">{{ form.sleepStartTime || '未选择' }}</view>
              </picker>
            </view>
            <view class="form-section">
              <text class="form-section__title">唤醒时间</text>
              <picker mode="time" :value="form.sleepEndTime" @change="setPickerValue('sleepEndTime', $event)">
                <view class="form-input">{{ form.sleepEndTime || '未选择' }}</view>
              </picker>
            </view>
            <text v-if="sleepHoursText" class="text-rose text-center" style="display:block; margin-top:8rpx;">睡眠时长：{{ sleepHoursText }}</text>
          </template>

          <template v-else-if="periodStore.activeSheet === 'notes'">
            <choice-section title="心理状态评估" :options="moodOptions" field="mood" :value="form.mood" visual="icon" @select="selectChoice" />
            <view class="form-section">
              <text class="form-section__title">附加备注</text>
              <textarea v-model="form.notes" class="form-textarea" maxlength="500" placeholder="请输入..." />
            </view>
          </template>

          <template v-else-if="periodStore.activeSheet === 'weight' || periodStore.activeSheet === 'bodyTemp'">
            <view class="form-section">
              <text class="form-section__title">{{ periodStore.activeSheet === 'weight' ? '体重数值（kg）' : '基础体温数值（°C）' }}</text>
              <input v-model="numericValue" class="form-input" type="digit" placeholder="请输入精确数值" />
            </view>
          </template>

          <template v-else-if="periodStore.activeSheet === 'ovuTest'">
            <choice-section title="排卵监测试纸定性结果" :options="ovuOptions" field="ovuTestResult" :value="form.ovuTestResult" visual="level" @select="selectChoice" />
          </template>

          <template v-else-if="periodStore.activeSheet === 'pregTest'">
            <choice-section title="早孕试纸 (HCG) 检测结果" :options="pregOptions" field="pregTestResult" :value="form.pregTestResult" visual="level" @select="selectChoice" />
          </template>

          <template v-else-if="periodStore.activeSheet === 'symptomLog'">
            <choice-section title="症状分类 (多选)" :options="symptomLogOptions" field="selectedTags" :value="form.selectedTags" multi visual="icon" @select="selectChoice" />
            <choice-section title="严重程度评级 (1-5)" :options="severityOptions" field="severity" :value="form.severity" visual="level" @select="selectChoice" />
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
              <text class="form-section__title">附加说明</text>
              <textarea v-model="form[simpleSheet.noteKey]" class="form-textarea" maxlength="500" placeholder="请输入..." />
            </view>
          </template>
        </scroll-view>

        <view v-if="!['stateSwitch', 'dayDetail'].includes(periodStore.activeSheet || '')" class="sheet-dialog__footer">
          <button class="btn-primary sheet-dialog__btn" :loading="saving" @click="saveSheet">{{ isPeriodConfirm ? '确认提交' : '保存数据' }}</button>
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
  { value: "PERIOD", icon: "", label: "常规周期监测" },
  { value: "PREPARE", icon: "", label: "备孕管理" },
  { value: "PREGNANT", icon: "", label: "孕期管理" },
  { value: "MOM", icon: "", label: "产后/育儿管理" },
];

const flowOptions: Option[] = [
  { value: "LIGHT", label: "偏少" }, { value: "MEDIUM", label: "适中" }, { value: "HEAVY", label: "偏多" },
];
const painOptions: Option[] = [
  { value: "NONE", label: "无" }, { value: "MILD", label: "轻微" }, { value: "MODERATE", label: "中度" }, { value: "SEVERE", label: "严重" },
];
const bloodColorOptions: Option[] = [
  { value: "light_pink", label: "淡粉"}, { value: "red", label: "鲜红"},
  { value: "dark_red", label: "暗红"}, { value: "brown", label: "褐色"}, { value: "black", label: "深色"},
];
const bloodStateOptions: Option[] = [
  { value: "nothing", label: "无异常" }, { value: "bad_smell", label: "异味" },
  { value: "block", label: "血块" }, { value: "zhazhuang", label: "渣状" },
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
const habitsOptions = options(["多饮水", "摄入水果", "清淡饮食", "补充蛋白", "规律运动", "散步", "按时睡眠", "午休", "热敷", "泡脚", "冥想放松", "忌冰饮"]);
const symptomsOptions = options(["腹胀", "腹痛", "腰酸", "头痛", "眩晕", "乏力", "乳房胀痛", "恶心", "腹泻", "便秘", "痤疮", "浮肿"]);
const intercourseOptions: Option[] = [{ value: "yes", label: "是" }, { value: "no", label: "否" }];
const cocOptions: Option[] = [{ value: "condom", label: "屏障避孕" }, { value: "emergency", label: "紧急避孕药" }, { value: "short_term", label: "短效避孕药" }, { value: "none", label: "无防护" }];
const leuColorOptions: Option[] = [{ value: "clear", label: "透明" }, { value: "white", label: "乳白" }, { value: "yellow", label: "偏黄" }, { value: "brown", label: "褐色" }];
const leuStateOptions: Option[] = [{ value: "normal", label: "正常" }, { value: "egg_white", label: "拉丝状" }, { value: "thick", label: "粘稠" }, { value: "watery", label: "水样" }];
const leuOtherOptions: Option[] = [{ value: "none", label: "无症状" }, { value: "smell", label: "异味" }, { value: "itchy", label: "瘙痒" }, { value: "burning", label: "灼热" }];
const dietOptions = options(["补充叶酸", "复合维生素", "补钙", "补铁", "高蛋白膳食", "绿叶蔬菜", "鲜果", "全谷物", "坚果", "忌冷饮", "忌咖啡因", "忌酒精", "忌二手烟"]);
const babyFeedOptions = options(["母乳喂养", "配方奶", "混合喂养", "辅食添加"]);
const babyCryOptions = options(["情绪稳定", "轻度哭闹", "剧烈哭闹", "持续烦躁"]);
const babyDiaperOptions = options(["排便正常", "排便量少", "排便量多", "未排便"]);
const babySleepOptions = options(["睡眠安稳", "睡眠一般", "易惊醒", "入睡困难"]);
const symptomLogOptions = options(["腹痛", "头痛", "头晕", "乏力", "恶心", "异常出血", "分泌物异常", "瘙痒", "发热", "情绪低落"]);
const severityOptions: Option[] = [1, 2, 3, 4, 5].map((value) => ({ value, label: `等级 ${value}` }));

const simpleSheets: Record<string, SimpleSheet> = {
  habits: { fields: [{ key: "habitsArr", label: "生活方式干预项", options: habitsOptions, multi: true }] },
  symptoms: { fields: [{ key: "symptomsArr", label: "临床体征", options: symptomsOptions, multi: true }] },
  intercourse: { fields: [{ key: "intercourse", label: "性生活记录", options: intercourseOptions }] },
  leucorrhea: { fields: [
    { key: "leuColor", label: "颜色特征", options: leuColorOptions },
    { key: "leuState", label: "性状特征", options: leuStateOptions },
    { key: "leuOther", label: "伴随症状", options: leuOtherOptions },
  ] },
  coc: { fields: [{ key: "coc", label: "避孕方式", options: cocOptions }] },
  prepareDiet: { fields: [{ key: "dietTags", label: "备孕期营养摄入", options: dietOptions, multi: true }], noteKey: "dietNotes" },
  babyRecord: { fields: [
    { key: "babyFeed", label: "喂养情况", options: babyFeedOptions },
    { key: "babyCry", label: "情绪状态", options: babyCryOptions },
    { key: "babyDiaper", label: "排便情况", options: babyDiaperOptions },
    { key: "babySleep", label: "睡眠质量", options: babySleepOptions },
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
const modeSettingLabel = computed(() => form.targetMode === "PREPARE" ? "备孕建档日期" : form.targetMode === "PREGNANT" ? "末次月经日期 (LMP)" : "分娩日期");
const modeDateText = computed(() => {
  const state = periodStore.menstrualState;
  if (currentMode.value === "PREPARE") return state.prepareStartDate ? `建档起始日: ${state.prepareStartDate}` : "备孕状态监测";
  if (currentMode.value === "PREGNANT") return state.dueDate ? `预产期 (EDC): ${state.dueDate}` : "孕期生理监测";
  if (currentMode.value === "MOM") return state.babyBirthDate ? `分娩日期: ${state.babyBirthDate}` : "产后评估阶段";
  return "常规周期体征跟踪";
});
const selectedDateLabel = computed(() => `${selectedDate.value.slice(0, 4)}-${selectedDate.value.slice(5, 7)}-${selectedDate.value.slice(8, 10)}`);
const selectedKind = computed(() => calendarKindForDate(selectedDate.value));
const selectedDayDescription = computed(() => ({
  period: "当前处于月经期，请重点关注出血量及疼痛体征。", predicted: "系统预测的月经期，请留意早期体征。", ovulation: "易孕窗口期，适用于受孕指导。", peak: "排卵预估高峰，激素水平变化关键期。",
  t1: "孕早期监测期 (T1, 1-13周)", t2: "孕中期监测期 (T2, 14-27周)", t3: "孕晚期监测期 (T3, 28-40周)",
  due: "系统计算的预产期 (EDC)", none: selectedRecord.value ? "已存在健康随访记录" : "常规监测日",
}[selectedKind.value]));

const selectedDayStatus = computed(() => {
  const ck = selectedCheckup.value;
  if (ck) return `第${ck.week}周产检 · ${fullCheckupLabel(ck.week)}`;
  return selectedKind.value === "period" ? "月经期" : selectedKind.value === "predicted" ? "预测月经期" : selectedKind.value === "peak" ? "排卵高峰" : selectedKind.value === "ovulation" ? "易孕期" : selectedKind.value === "t1" ? "孕早期 (T1)" : selectedKind.value === "t2" ? "孕中期 (T2)" : selectedKind.value === "t3" ? "孕晚期 (T3)" : selectedKind.value === "due" ? "预产期 (EDC)" : "无特殊临床事件";
});
const selectedDayAdvice = computed(() => {
  const ck = selectedCheckup.value;
  if (ck) return `建议在医疗机构进行${fullCheckupLabel(ck.week)}，请准备相关检查病历。`;
  return selectedKind.value === "period"
    ? "建议记录镇痛需求及出血量以供后续临床评估。"
    : selectedKind.value === "predicted"
      ? "即将进入周期初期，请留意生理症状变化。"
      : ["peak", "ovulation"].includes(selectedKind.value)
        ? "当前处于易受孕期，符合备孕或避孕管理的关键节点。"
        : selectedKind.value === "t1"
          ? "孕早期需规律补充叶酸，避免剧烈运动及致畸因素暴露。"
          : selectedKind.value === "t2"
            ? "孕中期平稳，可进行适度活动，开始关注胎动频率。"
            : selectedKind.value === "t3"
              ? "孕晚期需密切监测胎动，准备分娩相关医疗物资。"
              : selectedKind.value === "due"
                ? "到达预产期，如出现规律宫缩、破水或见红请立即就医。"
                : selectedDate.value > today ? "未来日期数据待采集。" : "建议完成日常体征、睡眠和心理状态录入。";
});

const selectedCheckup = computed(() => {
  const c = calendar.value?.cells?.find((cell: CalendarCell) => cell.key === selectedDate.value);
  if (!c?.pregnancyWeek || !c.checkupText) return null;
  return prenatalCheckups.find((p) => p.week === c.pregnancyWeek!) ?? null;
});
function fullCheckupLabel(week: number): string {
  const map: Record<number, string> = { 6: "早孕超声确认", 12: "NT筛查与建档", 16: "唐氏筛查评估", 20: "中孕期排畸超声", 24: "口服葡萄糖耐量试验 (OGTT)", 30: "晚孕期超声评估", 34: "胎心监护起始", 36: "分娩方式评估", 37: "足月常规产检" };
  return map[week] || "常规产检";
}
const dayDetailKindLabel = computed(() => ({
  period: "经期", predicted: "预测", ovulation: "易孕", peak: "排卵",
  t1: "孕早", t2: "孕中", t3: "孕晚", due: "预产",
  none: "常规",
}[selectedKind.value] || "常规"));
const dayRecordShortDate = computed(() => {
  const d = selectedDate.value;
  if (!d || d.length < 10) return "";
  return `${Number(d.slice(5, 7))}-${Number(d.slice(8, 10))}`;
});
const selectedRecordSummary = computed(() => {
  const record: any = selectedRecord.value;
  if (!record) return "";
  return [
    valueLabel(record.painLevel, painOptions) && `VAS评分:${valueLabel(record.painLevel, painOptions)}`,
    valueLabel(record.flowLevel, flowOptions) && `血量:${valueLabel(record.flowLevel, flowOptions)}`,
    valueLabel(record.sleepQuality, sleepOptions) && `睡眠:${valueLabel(record.sleepQuality, sleepOptions)}`,
    record.notes && `备注有记录`,
  ].filter(Boolean).join(" | ");
});

const periodCommon = (record: any): QuickItem[] => [
  { type: "sleep" as SheetType, icon: "", label: "睡眠质量评估", value: valueLabel(record?.sleepQuality, sleepOptions) },
  { type: "notes" as SheetType, icon: "", label: "心理状态", value: valueLabel(record?.mood, moodOptions) },
  { type: "habits" as SheetType, icon: "", label: "生活方式干预", value: arrayText(record.habits ?? record.habitsArr) },
  { type: "symptomLog" as SheetType, icon: "", label: "体征日志", value: symptomLogSummary.value },
];
const periodBlock = (record: any): QuickItem[] => [
  { type: selectedPeriodAction.value as SheetType, icon: "", label: selectedPeriodAction.value === "endConfirm" ? "月经期结束" : "月经期开始", primary: true },
  ...(selectedPeriodAction.value === "endConfirm" ? [{ type: "discomfort" as SheetType, icon: "", label: "经期指征详情", value: valueLabel(record.painLevel, painOptions) || valueLabel(record.flowLevel, flowOptions) }] : []),
];

const quickItems = computed<QuickItem[]>(() => {
  const record: any = selectedRecord.value ?? {};
  if (currentMode.value === "PREGNANT") {
    return [
      { type: "weight" as SheetType, icon: "", label: "孕期体重监测", value: record.weight ? `${record.weight} kg` : "" },
      ...periodCommon(record),
    ];
  }
  if (currentMode.value === "MOM") {
    return [
      ...periodBlock(record),
      { type: "babyRecord" as SheetType, icon: "", label: "婴儿体征记录", value: record.babyFeed || record.babyCry },
      ...periodCommon(record),
    ];
  }
  if (currentMode.value === "PREPARE") {
    return [
      ...periodBlock(record),
      { type: "prepareDiet" as SheetType, icon: "", label: "营养摄入记录", value: arrayText(record.dietTags) },
      { type: "ovuTest" as SheetType, icon: "", label: "排卵定性检测", value: valueLabel(record.ovuTestResult, ovuOptions) },
      { type: "pregTest" as SheetType, icon: "", label: "HCG定性检测", value: valueLabel(record.pregTestResult, pregOptions) },
      { type: "bodyTemp" as SheetType, icon: "", label: "基础体温(BBT)", value: record.bodyTemp ? `${record.bodyTemp} °C` : "" },
      { type: "intercourse" as SheetType, icon: "", label: "性生活日志", value: boolText(record.intercourse) },
      ...periodCommon(record),
    ];
  }
  return [
    ...periodBlock(record),
    { type: "leucorrhea" as SheetType, icon: "", label: "分泌物表征", value: leucorrheaText(record) },
    { type: "coc" as SheetType, icon: "", label: "避孕措施", value: valueLabel(record.coc ?? record.cocType, cocOptions) },
    { type: "intercourse" as SheetType, icon: "", label: "性生活日志", value: boolText(record.intercourse) },
    { type: "weight" as SheetType, icon: "", label: "体重监测", value: record.weight ? `${record.weight} kg` : "" },
    ...periodCommon(record),
  ];
});
const symptomLogSummary = computed(() => {
  const count = periodStore.symptomLogs.filter((log) => String(log.occurredAt).slice(0, 10) === selectedDate.value).length;
  return count ? `${count} 项异常` : "";
});

const sheetTitle = computed(() => {
  const titles: Record<string, string> = {
  dayDetail: "当日临床概况", recordFull: "综合指征录入",
  discomfort: "经期指征录入", sleep: "睡眠质量评估", notes: "心理评估记录", weight: "体重录入", bodyTemp: "BBT录入",
  ovuTest: "排卵检测录入", pregTest: "HCG检测录入", startConfirm: "确认周期起点", endConfirm: "确认周期终点",
  stateSwitch: "模式配置", modeSettings: "参数设置", symptomLog: "临床日志", habits: "依从性干预",
  symptoms: "体征录入", intercourse: "性生活录入", leucorrhea: "分泌物监测", coc: "避孕档案",
  prepareDiet: "营养干预", babyRecord: "育儿记录",
  };
  return titles[periodStore.activeSheet ?? ""] ?? "数据录入";
});

const numericValue = computed({
  get: () => String(periodStore.activeSheet === "weight" ? form.weight ?? "" : form.bodyTemp ?? ""),
  set: (value: string) => { form[periodStore.activeSheet === "weight" ? "weight" : "bodyTemp"] = value; },
});

const sleepHoursText = computed(() => {
  if (!form.sleepStartTime || !form.sleepEndTime) return "";
  let minutes = timeMinutes(form.sleepEndTime) - timeMinutes(form.sleepStartTime);
  if (minutes <= 0) minutes += 24 * 60;
  return `${(minutes / 60).toFixed(1)} h`;
});

function isoDate(date: Date) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  const d = String(date.getDate()).padStart(2, "0");
  return `${y}-${m}-${d}`;
}

function dateAt(key: string) { return new Date(`${key}T00:00:00`); }
function addDays(key: string, amount: number) {
  const date = dateAt(key);
  date.setDate(date.getDate() + amount);
  return isoDate(date);
}
function options(values: string[]): Option[] { return values.map((value) => ({ value, label: value })); }
function recordStart(record: MenstrualRecord) { return String((record as any).recordDate || record.startDate).slice(0, 10); }

function isPeriodStartRecord(record: any): boolean {
  const dailyFields = [record.mood, record.sleepQuality, record.sleepHours, record.weight,
    record.bodyTemp, record.ovuTestResult, record.pregTestResult, record.leuColor,
    record.notes, record.babyFeed, record.babyCry, record.intercourse, record.exercise,
    record.bloodColor, record.cocType, record.dietTags];
  const allDefaults = (record.painLevel ?? "NONE") === "NONE" && (record.flowLevel ?? "MEDIUM") === "MEDIUM" && (record.discomfort ?? "NONE") === "NONE";
  return allDefaults && dailyFields.every((v) => v == null) && !record.endDate;
}

function hasPeriodSymptoms(record: any): boolean {
  return (record.flowLevel && record.flowLevel !== "MEDIUM") || (record.painLevel && record.painLevel !== "NONE") || (record.discomfort && record.discomfort !== "NONE") || !!record.bloodColor || (Array.isArray(record.bloodState) && record.bloodState.includes("PERIOD_CONFIRMED"));
}

function isPeriodEpisode(record: any): boolean { return isPeriodStartRecord(record) || hasPeriodSymptoms(record); }
function findRecordForDate(key: string) { return periodStore.findRecordForDate(key, today); }
function normalizedProbability(value: unknown) {
  const raw = Number(value || 0);
  return raw > 1 ? raw / 100 : raw;
}

const periodSets = computed(() => {
  const period = new Set<string>();
  const predicted = new Set<string>();
  const ovulation = new Set<string>();
  let peak = String(periodStore.cyclePrediction?.fertileWindow?.peakDay || periodStore.assessment?.predictedOvulationDate || "");

  periodStore.records.forEach((record) => {
    if (!isPeriodEpisode(record)) return;
    const start = recordStart(record);
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
  const avgDuration2 = Number(periodStore.assessment?.averageDurationDays || 5);
  const horizonDays = 120;
  let next = String(periodStore.cyclePrediction?.predictedNextStart || periodStore.assessment?.predictedNextStart || "").slice(0, 10);
  if (!next && latestStart) next = addDays(latestStart, avgCycle);
  while (next && next < today) next = addDays(next, avgCycle);
  if (next) {
    for (let offset = 0; offset < horizonDays; offset += 1) {
      const key = addDays(next, offset);
      const inCycleWindow = (offset % avgCycle) < avgDuration2;
      if (inCycleWindow && !period.has(key)) predicted.add(key);
    }
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

const prenatalCheckups: Array<{ week: number; label: string }> = [
  { week: 6, label: "早孕确认" },
  { week: 12, label: "NT筛查" },
  { week: 16, label: "唐氏筛查" },
  { week: 20, label: "排畸检查" },
  { week: 24, label: "OGTT" },
  { week: 30, label: "小排畸" },
  { week: 34, label: "胎心监护" },
  { week: 36, label: "分娩评估" },
  { week: 37, label: "足月产检" },
];
function checkupForWeek(w: number) { return prenatalCheckups.find((c) => c.week === w)?.label ?? ""; }

const pregnancySets = computed(() => {
  const empty = { t1: new Set<string>(), t2: new Set<string>(), t3: new Set<string>(), due: "" };
  if (currentMode.value !== "PREGNANT") return empty;
  const state: any = periodStore.menstrualState ?? {};
  const lmp = state.pregnantStartDate ? String(state.pregnantStartDate).slice(0, 10) : "";
  if (!lmp) return empty;
  const due = state.dueDate ? String(state.dueDate).slice(0, 10) : addDays(lmp, 280);
  const t1 = new Set<string>(); const t2 = new Set<string>(); const t3 = new Set<string>();
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
  const allRecordDates = new Set<string>();
  for (const r of periodStore.records) { allRecordDates.add(recordStart(r)); }
  const symptomDateSet = new Set<string>();
  for (const log of periodStore.symptomLogs) { symptomDateSet.add(String(log.occurredAt).slice(0, 10)); }

  const makeCell = (date: Date, isOut: boolean): CalendarCell => {
    const key = isoDate(date);
    const cycleDay = latestStart && key >= latestStart ? Math.floor((dateAt(key).getTime() - dateAt(latestStart).getTime()) / 86400000) + 1 : undefined;
    let pregnancyDay: number | undefined;
    let pregnancyWeek: number | undefined;
    if (currentMode.value === "PREGNANT") {
      const lmp = (periodStore.menstrualState as any)?.pregnantStartDate;
      if (lmp) {
        pregnancyDay = Math.floor((dateAt(key).getTime() - dateAt(String(lmp).slice(0, 10)).getTime()) / 86400000) + 1;
        if (pregnancyDay >= 1 && pregnancyDay <= 280) {
          pregnancyWeek = Math.floor((pregnancyDay - 1) / 7) + 1;
        } else {
          pregnancyDay = undefined; pregnancyWeek = undefined;
        }
      }
    }
    return {
      key, day: date.getDate(), kind: calendarKindForDate(key), isToday: key === today, isOut,
      probability: probabilities.get(key) ?? 0, hasRecord: allRecordDates.has(key) || symptomDateSet.has(key),
      cycleDay: cycleDay && cycleDay > 0 && cycleDay <= 45 ? cycleDay : undefined,
      pregnancyWeek, pregnancyDay, checkupText: currentMode.value === "PREGNANT" && pregnancyWeek ? checkupForWeek(pregnancyWeek) : "",
    };
  };
  for (let i = first.getDay(); i > 0; i -= 1) { const date = new Date(year, month, 1 - i); cells.push(makeCell(date, true)); }
  const count = new Date(year, month + 1, 0).getDate();
  for (let day = 1; day <= count; day += 1) { cells.push(makeCell(new Date(year, month, day), false)); }
  while (cells.length % 7) { const date = new Date(year, month + 1, cells.length - first.getDay() - count + 1); cells.push(makeCell(date, true)); }
  return { year, month, cells };
}

function valueLabel(value: unknown, options: Option[]) { return options.find((item) => item.value === value)?.label ?? ""; }
function arrayText(value: unknown) { return Array.isArray(value) ? value.join("、") : ""; }
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

function changeMonth(delta: number) { periodStore.navigateMonth(delta); }
function selectDate(key: string) { selectedDate.value = key; openSheet("dayDetail"); }
function goToday() { selectedDate.value = today; periodStore.monthOffset = 0; }

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

function openQuickItem(item: { type: SheetType; primary?: boolean }) { openSheet(item.type); }

function wakeAiAssistant() {
  const today = new Date().toISOString().slice(0, 10);
  uni.setStorageSync("pendingAutoAnalysisDate", today);
  uni.removeStorageSync("pendingAiPrompt");
  uni.switchTab({ url: "/pages/chat/index" });
}

function openTodayPeriodConfirm() { openFromDay(selectedPeriodAction.value); }
function openFromDay(type: SheetType) { closeSheet(); openSheet(type); }

function selectChoice(payload: { field: string; value: string | number; multi: boolean }) {
  if (!payload.multi) { form[payload.field] = payload.value; return; }
  const values = Array.isArray(form[payload.field]) ? [...form[payload.field]] : [];
  const index = values.indexOf(payload.value);
  if (index >= 0) values.splice(index, 1); else values.push(payload.value);
  form[payload.field] = values;
}

function pickMode(mode: Mode) {
  if (mode === currentMode.value) return closeSheet();
  if (mode === "PERIOD") { form.targetMode = mode; void saveMode(); return; }
  Object.keys(form).forEach((key) => delete form[key]);
  form.targetMode = mode;
  const key = mode === "PREPARE" ? "prepareStartDate" : mode === "PREGNANT" ? "pregnantStartDate" : "babyBirthDate";
  form[key] = (periodStore.menstrualState as any)[key] ?? selectedDate.value;
  periodStore.openSheet("modeSettings", selectedDate.value, { ...form });
}

function closeSheet() { periodStore.closeSheet(); }
function setPickerValue(key: string, event: any) { form[key] = event.detail.value; }
function localTime(value: string) { const date = new Date(value); return Number.isNaN(date.getTime()) ? "" : `${String(date.getHours()).padStart(2, "0")}:${String(date.getMinutes()).padStart(2, "0")}`; }
function timeMinutes(value: string) { const [hours, minutes] = value.split(":").map(Number); return hours * 60 + minutes; }
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
  appStore.showToast("系统模式已更新", "success");
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
      if (!open) throw new Error("无进行中的经期记录");
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
    appStore.showToast("数据已成功归档", "success");
    closeSheet();
  } catch (error: any) {
    appStore.showToast(error?.message || "数据保存失败", "error");
  } finally {
    saving.value = false;
  }
}

onMounted(() => { if (!periodStore.records.length) periodStore.loadAll(); });

function consumeHomePendingSheet() {
  const sheet = String(uni.getStorageSync("homePendingSheet") || "");
  const date = String(uni.getStorageSync("homePendingSheetDate") || "");
  if (!sheet) return;
  uni.removeStorageSync("homePendingSheet");
  uni.removeStorageSync("homePendingSheetDate");
  if (date) selectedDate.value = date;
  periodStore.openSheet(sheet as SheetType, selectedDate.value, { ...form });
}
onShow(() => { consumeHomePendingSheet(); });
</script>

<style lang="scss" scoped>
.period-page { min-height: 100vh; padding: $gap-md $gap-md 120rpx; }

.status-panel {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: $gap-md;
}
.status-panel__info { display: flex; flex-direction: column; }
.status-panel__label { font-size: 22rpx; color: $muted; font-weight: 600; margin-bottom: 4rpx;}
.status-panel__title { font-size: $font-lg; font-weight: 700; color: $rose-dark; }
.status-panel__desc { font-size: 22rpx; color: $muted; margin-top: 4rpx; }
.status-panel__action { margin: 0; font-size: 24rpx; padding: 12rpx 24rpx; }

.calendar { margin-bottom: $gap-md; }
.calendar__header { display: flex; align-items: center; justify-content: space-between; margin-bottom: $gap-sm;}
.calendar__title-group { display: flex; flex-direction: column; align-items: center; }
.calendar__title { font-size: $font-md; font-weight: 700; color: $ink; }
.calendar__subtitle { font-size: 22rpx; color: $muted; }
.calendar__nav { padding: 10rpx 20rpx; background: $surface; border-radius: $radius-sm; color: $muted; font-size: 32rpx; font-weight: bold; }

.calendar__week { display: grid; grid-template-columns: repeat(7, 1fr); text-align: center; font-size: 24rpx; color: $muted; font-weight: 600; margin-bottom: 8rpx; }
.calendar__grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 4rpx; }
.calendar-day {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 80rpx;
  border-radius: $radius-sm;
  font-size: $font-sm;
  color: $ink;
  border: 1px solid transparent;
}
.calendar-day--out { color: #cbd5e1; }
.calendar-day--today { border-color: $rose-dark; font-weight: bold;}
.calendar-day--selected { background: $surface; box-shadow: 0 0 0 2px $rose-dark inset; }

/* 医疗风格日历颜色 */
.calendar-day--period { background: #e11d48; color: #fff; }
.calendar-day--predicted { background: #f1f5f9; color: #64748b; border-color: #cbd5e1; }
.calendar-day--ovulation { background: #f0fdfa; color: #0f766e; border-color: #99f6e4; }
.calendar-day--peak { background: #0f766e; color: #fff; }
.calendar-day--t1 { background: #fdf2f8; color: #be185d; border-color: #fbcfe8;}
.calendar-day--t2 { background: #faf5ff; color: #7e22ce; border-color: #e9d5ff;}
.calendar-day--t3 { background: #fff7ed; color: #c2410c; border-color: #fed7aa;}
.calendar-day--due { background: #e11d48; color: #fff; }

.calendar-day__number { z-index: 1; }
.calendar-day__meta { font-size: 18rpx; margin-top: -4rpx; opacity: 0.8; z-index: 1;}
.calendar-day__record { position: absolute; top: 4rpx; right: 4rpx; width: 8rpx; height: 8rpx; background: #f59e0b; border-radius: 50%; }

.calendar__legend { display: flex; justify-content: center; gap: 24rpx; margin-top: $gap-md; font-size: 22rpx; color: $muted; }
.calendar__legend text { display: flex; align-items: center; gap: 8rpx; }
.legend-dot { width: 12rpx; height: 12rpx; border-radius: 2rpx; }
.legend-dot--period { background: #e11d48; }
.legend-dot--predicted { background: #f1f5f9; border: 1px solid #cbd5e1; }
.legend-dot--ovulation { background: #f0fdfa; border: 1px solid #99f6e4; }
.legend-dot--t1 { background: #fdf2f8; border: 1px solid #fbcfe8; }
.legend-dot--t2 { background: #faf5ff; border: 1px solid #e9d5ff; }
.legend-dot--t3 { background: #fff7ed; border: 1px solid #fed7aa; }
.legend-dot--due { background: #e11d48; }

.day-summary { margin-bottom: $gap-md; padding: $gap-md; }
.day-summary__header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8rpx; }
.day-summary__title { font-size: $font-sm; font-weight: 700; color: $ink; }
.day-summary__today { margin: 0; padding: 10rpx 20rpx; font-size: 22rpx; }
.day-summary__desc { font-size: 24rpx; color: $muted; }

.section-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: $gap-sm; }
.section-title { display: block; font-size: $font-md; font-weight: 700; color: $ink; }
.section-sub { font-size: 22rpx; color: $muted; }
.btn-sm { padding: 12rpx 24rpx; font-size: 24rpx; }

.quick-grid { display: grid; gap: $gap-sm; }
.quick-item { 
  display: flex; 
  align-items: center; 
  justify-content: space-between; 
  padding: $gap-md; 
  border-radius: $radius-sm;
  transition: all 0.2s;
}
.quick-item__info { display: flex; flex-direction: column; }
.quick-item__label { font-size: $font-sm; font-weight: 700; color: $ink; }
.quick-item__value { font-size: 22rpx; color: $muted; margin-top: 4rpx; }
.quick-item__action { font-size: 24rpx; color: $rose-dark; font-weight: 600; background: $surface; padding: 6rpx 16rpx; border-radius: 4rpx;}
.quick-item--primary { border-color: $rose-dark; background: $blush; }
.quick-item--primary .quick-item__action { background: $rose-dark; color: #fff; }

/* 覆盖层与表单 */
.overlay { position: fixed; inset: 0; z-index: 100; background: rgba(15, 23, 42, 0.4); backdrop-filter: blur(4px); display: flex; align-items: flex-end; padding: $gap-md;}
.sheet-dialog { width: 100%; max-height: 85vh; background: $paper; border-radius: $radius-lg; display: flex; flex-direction: column; box-shadow: $shadow-lg; overflow: hidden;}
.sheet-dialog__header { display: flex; justify-content: space-between; align-items: center; padding: $gap-md $gap-lg; border-bottom: 1px solid $line; background: $surface; }
.sheet-dialog__title { font-size: $font-md; font-weight: 700; color: $ink; }
.sheet-dialog__close { font-size: 40rpx; color: $muted; padding: 0 10rpx;}
.sheet-dialog__body { padding: $gap-lg; flex: 1; min-height: 0; overflow-y: auto; }
.sheet-dialog__footer { padding: $gap-md $gap-lg; border-top: 1px solid $line; }
.sheet-dialog__btn { width: 100%; margin: 0; }

.confirm-box { text-align: center; padding: $gap-lg 0; }
.confirm-box__title { display: block; font-size: $font-lg; font-weight: 700; color: $ink; margin-bottom: $gap-xs; }
.confirm-box__copy { font-size: $font-sm; color: $muted; }

.mode-grid { display: grid; grid-template-columns: 1fr 1fr; gap: $gap-sm; }
.mode-choice { padding: $gap-md; border: 1px solid $line; border-radius: $radius-sm; text-align: center; font-size: $font-sm; font-weight: 600; color: $muted; background: $surface;}
.mode-choice--active { border-color: $rose-dark; color: $rose-dark; background: $blush; }

.form-section { margin-bottom: $gap-md; }
.form-section__title { display: block; font-size: $font-sm; font-weight: 600; color: $muted; margin-bottom: 12rpx; }
.form-input { width: 100%; padding: 20rpx; border: 1px solid $line; border-radius: $radius-sm; background: $paper; font-size: $font-sm; }
.form-textarea { width: 100%; padding: 20rpx; border: 1px solid $line; border-radius: $radius-sm; background: $paper; font-size: $font-sm; height: 160rpx; box-sizing: border-box;}

.day-hero { padding: $gap-lg; border: 1px solid $line; border-radius: $radius-md; background: $surface; margin-bottom: $gap-sm; }
.day-hero__eyebrow { display: block; font-size: 22rpx; color: $muted; font-weight: 600; }
.day-hero__title { display: block; font-size: $font-xl; font-weight: 700; color: $ink; margin-top: 4rpx;}
.day-hero__summary { display: block; font-size: 24rpx; color: $muted; margin-top: 8rpx;}

.day-advice { margin-bottom: $gap-sm; }
.day-advice__label { font-weight: 600; color: $rose-dark; font-size: $font-sm; }
.day-advice__text { color: $ink; font-size: $font-sm; }

.day-record-card { margin-bottom: $gap-md; }
.day-record-card__head { margin-bottom: 8rpx; }
.day-record-card__title { font-weight: 600; font-size: $font-sm; color: $ink; }
.day-record-card__copy { font-size: 24rpx; color: $muted; }

.day-actions { display: flex; flex-direction: column; gap: $gap-sm; }
.day-action { display: flex; align-items: center; padding: $gap-md; }
.day-action__body { display: flex; flex-direction: column; }
.day-action__title { font-size: $font-sm; font-weight: 700; color: $ink; }
.day-action__sub { font-size: 22rpx; color: $muted; }
.day-action--primary { background: $rose-dark; color: #fff; border-color: $rose-dark; }
.day-action--primary .day-action__title, .day-action--primary .day-action__sub { color: #fff; }
</style>
