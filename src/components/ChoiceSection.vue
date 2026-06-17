<template>
  <view class="form-section">
    <text class="form-section__title">{{ title }}</text>
    <view class="choice-grid" :class="`choice-grid--${visual}`">
      <view
        v-for="option in options"
        :key="String(option.value)"
        class="choice"
        :class="[`choice--${visual}`, { 'choice--active': isSelected(option.value) }]"
        @click="$emit('select', { field, value: option.value, multi })"
      >
        <text v-if="visual === 'icon'" class="choice__icon">{{ option.icon || emojiFor(option.label) }}</text>
        <text class="choice__label">{{ option.label }}</text>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
type Option = { value: string | number; label: string; icon?: string; color?: string };

const props = withDefaults(defineProps<{
  title: string;
  options: Option[];
  field: string;
  value?: unknown;
  multi?: boolean;
  visual?: string;
}>(), {
  value: undefined,
  multi: false,
  visual: "choice",
});

defineEmits<{
  select: [payload: { field: string; value: string | number; multi: boolean }];
}>();

function isSelected(value: string | number) {
  return props.multi ? Array.isArray(props.value) && props.value.includes(value) : props.value === value;
}

function emojiFor(raw: string | number) {
  const label = String(raw);
  const exact: Record<string, string> = {
    "无不适": "👌", "腹胀": "🎈", "痉挛": "⚡", "头痛": "🤕", "腹痛": "🫃", "腰酸": "🧘",
    "头晕": "💫", "乏力": "🪫", "乳房胀痛": "🎀", "恶心": "🤢", "腹泻": "🚽", "便秘": "🧱",
    "长痘": "🫧", "浮肿": "🌊", "异常出血": "🩸", "白带异常": "💧", "瘙痒": "🪶", "发热": "🤒",
    "透明无色": "💎", "乳白色": "🥛", "偏黄色": "🌕", "褐/咖啡色": "🟤",
    "正常状态": "✅", "像蛋清": "🥚", "有点粘稠": "🍯", "像水一样": "💦",
    "完全没有": "⭕", "有异味": "👃", "发痒": "🪶", "灼热感": "🔥",
    "多喝水": "🥤", "吃水果": "🍎", "清淡饮食": "🥣", "补充蛋白": "🥚", "规律运动": "🏋️",
    "散步": "🚶", "早睡": "🌙", "午休": "🛌", "热敷": "♨️", "泡脚": "🦶", "放松冥想": "🧘", "不喝冰饮": "🧊",
    "叶酸打卡": "🍀", "复合维他命": "💊", "吃钙片": "🦴", "补铁啦": "🧲", "高蛋白": "🍳",
    "绿叶菜": "🥬", "鲜水果": "🍓", "全谷物": "🌾", "小坚果": "🥜", "戒了冷饮": "🚫", "今天没喝咖啡": "☕", "远离酒精": "🍷", "远离二手烟": "🚭",
    "开心": "😊", "平静": "😌", "疲倦": "🥱", "焦虑": "😰", "低落": "😢", "烦躁": "😤",
  };
  if (exact[label]) return exact[label];
  if (/开心/.test(label)) return "😊";
  if (/平静|冥想|放松/.test(label)) return "😌";
  if (/焦虑/.test(label)) return "😰";
  if (/低落|哭/.test(label)) return "😢";
  if (/烦躁|生气/.test(label)) return "😤";
  if (/睡|休|早起/.test(label)) return "😴";
  if (/痛|不适|症状|头晕|乏力|恶心|发热|痒|灼热/.test(label)) return "🩹";
  if (/吃|饮|水|营养|蛋白|水果|维他命|叶酸|钙|铁|谷物|坚果/.test(label)) return "🥗";
  if (/运动|散步|习惯|打卡/.test(label)) return "🏃";
  if (/宝宝|母乳|奶|辅食|便便/.test(label)) return "🍼";
  if (/排卵|试纸/.test(label)) return "🔬";
  if (/孕/.test(label)) return "🤰";
  if (/体温/.test(label)) return "🌡️";
  if (/体重/.test(label)) return "⚖️";
  if (/白带|颜色|蛋清|粘稠|水一样/.test(label)) return "💧";
  if (/避孕|药/.test(label)) return "💊";
  if (/同房|甜蜜/.test(label)) return "💕";
  if (/正常|没有|阴性/.test(label)) return "✅";
  if (/阳/.test(label)) return "➕";
  return "✨";
}
</script>

<style lang="scss" scoped>
.form-section { margin-bottom: 18rpx; padding: 20rpx; border-radius: $radius-lg; background: $surface; }
.form-section__title { display: block; margin-bottom: 14rpx; font-size: $font-sm; font-weight: 800; }
.choice-grid { display: flex; flex-wrap: wrap; gap: 10rpx; }
.choice { flex: 1 0 27%; padding: 15rpx 10rpx; border: 2rpx solid transparent; border-radius: $radius-md; background: $paper; color: $ink; font-size: $font-xs; text-align: center; }
.choice--active { border-color: $rose-dark; background: $blush; color: $rose-dark; font-weight: 800; }
.choice-grid--icon { display: grid; grid-template-columns: repeat(4, 1fr); }
.choice--icon { display: flex; min-height: 104rpx; flex: initial; align-items: center; justify-content: center; flex-direction: column; gap: 5rpx; padding: 10rpx 5rpx; }
.choice-grid--level { display: grid; grid-template-columns: repeat(4, 1fr); }
.choice--level { display: flex; min-height: 88rpx; flex: initial; align-items: center; justify-content: center; flex-direction: column; gap: 4rpx; }
.choice__icon { font-size: 34rpx; line-height: 1; }
.choice__label { font-size: 20rpx; font-weight: 700; }
</style>
