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
    "无不适": "○", "腹胀": "●", "痉挛": "⚡", "头痛": "▲", "腹痛": "■", "腰酸": "≈",
    "头晕": "◎", "乏力": "↓", "乳房胀痛": "◆", "恶心": "×", "腹泻": "▤", "便秘": "▥",
  };
  if (exact[label]) return exact[label];
  if (/开心/.test(label)) return "●";
  if (/平静|冥想|放松/.test(label)) return "○";
  if (/焦虑/.test(label)) return "▲";
  if (/低落|哭/.test(label)) return "↓";
  if (/烦躁|生气/.test(label)) return "×";
  if (/正常|没有|阴性/.test(label)) return "✓";
  if (/阳/.test(label)) return "✚";
  return "·";
}
</script>

<style lang="scss" scoped>
.form-section { margin-bottom: $gap-md; }
.form-section__title { display: block; margin-bottom: 12rpx; font-size: $font-sm; font-weight: 600; color: $muted; }
.choice-grid { display: flex; flex-wrap: wrap; gap: 16rpx; }
.choice { 
  flex: 1 0 22%; 
  padding: 16rpx 12rpx; 
  border: 1px solid $line; 
  border-radius: $radius-sm; 
  background: $paper; 
  color: $ink; 
  font-size: $font-xs; 
  text-align: center; 
  transition: all 0.2s ease;
}
.choice--active { 
  border-color: $rose-dark; 
  background: $blush; 
  color: $rose-dark; 
  font-weight: 600; 
  box-shadow: 0 0 0 1px $rose-dark;
}
.choice-grid--icon { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16rpx; }
.choice--icon { display: flex; min-height: 96rpx; align-items: center; justify-content: center; flex-direction: column; gap: 8rpx; }
.choice-grid--level { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16rpx;}
.choice--level { display: flex; min-height: 80rpx; align-items: center; justify-content: center; flex-direction: column; gap: 4rpx; }
.choice__icon { font-size: 28rpx; line-height: 1; color: $muted; }
.choice--active .choice__icon { color: $rose-dark; }
.choice__label { font-size: 24rpx; }
</style>
