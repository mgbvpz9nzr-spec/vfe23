<template>
  <view class="clinic-top-spacer" :style="{ height: `${barHeight}px` }" />
  <view class="clinic-top" :style="{ height: `${barHeight}px`, paddingTop: `${statusBarHeight}px`, paddingRight: `${rightInset}px` }">
    <view v-if="showBack" class="clinic-top__back" aria-label="返回" @click="goBack">‹</view>
    <view class="clinic-top__brand">🌸</view>
    <view class="clinic-top__identity">
      <text class="clinic-top__kicker">{{ welcomeText }}</text>
      <text class="clinic-top__name">{{ appStore.clinicName }}</text>
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
  const greeting = hour < 6 ? "夜深了" : hour < 11 ? "早上好" : hour < 14 ? "中午好" : hour < 18 ? "下午好" : "晚上好";
  return `${greeting}，${userStore.displayName || "姐姐"}`;
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
  gap: 14rpx;
  padding-left: 24rpx;
  border-bottom: 1rpx solid rgba(236, 72, 153, .08);
  background: rgba(255, 255, 255, .97);
  box-shadow: 0 4rpx 12rpx rgba(177, 68, 124, .05);
}
.clinic-top__back,
.clinic-top__brand,
.clinic-top__notice {
  display: flex;
  width: 64rpx;
  height: 64rpx;
  flex: 0 0 auto;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: $blush;
}
.clinic-top__back { color: $ink; font-size: 48rpx; line-height: 1; }
.clinic-top__brand { width:58rpx;height:58rpx;background:$blush;font-size:30rpx;box-shadow:inset 0 0 0 1rpx rgba(236,72,153,.08); }
.clinic-top__identity { min-width: 0; flex: 1; }
.clinic-top__kicker,
.clinic-top__name { display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.clinic-top__kicker { color: $rose-dark; font-size: 20rpx; font-weight: 700; line-height: 1.3; }
.clinic-top__name { color: $ink; font-size: 27rpx; font-weight: 800; line-height: 1.35; }
.clinic-top__notice { position: relative; background: $paper; box-shadow: $shadow-sm; }
.clinic-top__notice-icon { font-size: 32rpx; line-height: 1; }
.clinic-top__badge {
  position: absolute;
  top: -4rpx;
  right: -7rpx;
  min-width: 26rpx;
  height: 26rpx;
  padding: 0 5rpx;
  border: 3rpx solid $paper;
  border-radius: $radius-full;
  background: $rose-dark;
  color: #fff;
  font-size: 16rpx;
  font-weight: 800;
  line-height: 26rpx;
  text-align: center;
}
</style>
