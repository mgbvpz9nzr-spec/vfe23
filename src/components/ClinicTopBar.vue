<template>
  <view class="clinic-top-spacer" :style="{ height: `${barHeight}px` }" />
  <view class="clinic-top" :style="{ height: `${barHeight}px`, paddingTop: `${statusBarHeight}px`, paddingRight: `${rightInset}px` }">
    <view v-if="showBack" class="clinic-top__back" aria-label="返回" @click="goBack">‹</view>
    <view class="clinic-top__brand">✚</view>
    <view class="clinic-top__identity">
      <text class="clinic-top__kicker">{{ welcomeText }}</text>
      <text class="clinic-top__name">{{ appStore.clinicName || '临床管理中心' }}</text>
    </view>
    <view class="clinic-top__notice" aria-label="通知" @click="openNotifications">
      <text class="clinic-top__notice-icon">🔔</text>
      <text v-if="appStore.unreadCount" class="clinic-top__badge">{{ appStore.unreadCount > 9 ? "9+" : appStore.unreadCount }}</text>
    </view>
  </view>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import { useAppStore } from "@/stores/useAppStore";
import { useUserStore } from "@/stores/useUserStore";

withDefaults(defineProps<{ showBack?: boolean }>(), { showBack: false });

const appStore = useAppStore();
const userStore = useUserStore();
const statusBarHeight = ref(24);
const barHeight = ref(72);
const rightInset = ref(96);
const welcomeText = computed(() => {
  const hour = new Date().getHours();
  const greeting = hour < 6 ? "夜间值守" : hour < 11 ? "上午好" : hour < 14 ? "中午好" : hour < 18 ? "下午好" : "晚上好";
  return `${greeting}，${userStore.displayName || "用户"}`;
});

function measureNavigation() {
  const system = uni.getSystemInfoSync();
  statusBarHeight.value = system.statusBarHeight || 24;
  let capsule: UniApp.GetMenuButtonBoundingClientRectRes | null = null;
  try {
    capsule = uni.getMenuButtonBoundingClientRect();
  } catch {
    capsule = null;
  }
  const navigationHeight = capsule?.height
    ? capsule.height + Math.max(8, (capsule.top - statusBarHeight.value) * 2)
    : 48;
  barHeight.value = statusBarHeight.value + navigationHeight;
  rightInset.value = capsule?.left
    ? Math.max(88, (system.windowWidth || 375) - capsule.left + 8)
    : 88;
}

function openNotifications() {
  uni.navigateTo({ url: "/pages/services/index?type=notifications" });
}

function goBack() {
  uni.navigateBack({
    fail: () => uni.switchTab({ url: "/pages/home/index" }),
  });
}

onMounted(() => {
  measureNavigation();
  void appStore.loadDashboard().catch(() => undefined);
});
</script>

<style lang="scss" scoped>
.clinic-top-spacer { width: 100%; flex: 0 0 auto; }
.clinic-top {
  position: fixed;
  z-index: 20;
  top: 0;
  left: 0;
  display: flex;
  width: 100%;
  align-items: center;
  gap: 16rpx;
  padding-left: 24rpx;
  border-bottom: 1px solid $line;
  background: rgba(255, 255, 255, 0.98);
  backdrop-filter: blur(10px);
}
.clinic-top__back,
.clinic-top__brand,
.clinic-top__notice {
  display: flex;
  width: 56rpx;
  height: 56rpx;
  flex: 0 0 auto;
  align-items: center;
  justify-content: center;
  border-radius: $radius-sm;
}
.clinic-top__back { color: $muted; font-size: 48rpx; line-height: 1; }
.clinic-top__brand { 
  background: $rose-dark; 
  color: #fff;
  font-size: 32rpx;
  font-weight: bold;
}
.clinic-top__identity { min-width: 0; flex: 1; display: flex; flex-direction: column; justify-content: center;}
.clinic-top__kicker,
.clinic-top__name { display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.clinic-top__kicker { color: $muted; font-size: 20rpx; font-weight: 500; line-height: 1.2; }
.clinic-top__name { color: $ink; font-size: 26rpx; font-weight: 700; line-height: 1.4; }
.clinic-top__notice { position: relative; background: $surface; border: 1px solid $line; color: $muted;}
.clinic-top__notice-icon { font-size: 28rpx; line-height: 1; }
.clinic-top__badge {
  position: absolute;
  top: -6rpx;
  right: -6rpx;
  min-width: 24rpx;
  height: 24rpx;
  padding: 0 4rpx;
  border-radius: $radius-full;
  background: #ef4444; /* 医疗警告红 */
  color: #fff;
  font-size: 16rpx;
  font-weight: bold;
  line-height: 24rpx;
  text-align: center;
}
</style>
