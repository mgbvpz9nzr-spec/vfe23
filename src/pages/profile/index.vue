<template>
  <view class="page profile-page">
    <ClinicTopBar />
    <view class="profile-card">
      <text class="profile-card__avatar">{{ String(patient.name || userStore.displayName || "您").slice(0, 1) }}</text>
      <view class="profile-card__info">
        <text class="profile-card__eyebrow">我的专属健康空间</text>
        <text class="profile-card__name">{{ patient.name || userStore.displayName || "您好" }}</text>
        <text class="profile-card__meta">{{ patient.phone || "" }}{{ patient.phone && clinic.name ? " · " : "" }}{{ clinic.name || "" }}</text>
      </view>
    </view>

    <view class="profile-stats">
      <view v-for="stat in stats" :key="stat.type" class="profile-stat" @click="openService(stat.type)">
        <text class="profile-stat__value">{{ stat.value }}</text>
        <text class="profile-stat__label">{{ stat.label }}</text>
      </view>
    </view>

    <view class="section-head">
      <view>
        <text class="section-head__eyebrow">消息关怀</text>
        <text class="section-head__title">通知与提醒</text>
      </view>
      <text class="section-head__meta">{{ unreadCount }} 条未读</text>
    </view>
    <view class="message-list">
      <view v-for="message in messages.slice(0, 2)" :key="message.id" class="message-card card" :class="{ 'message-card--unread': !message.read }" @click="openService('notifications')">
        <view class="message-card__dot" />
        <view class="message-card__body">
          <text class="message-card__title">{{ message.title || "门诊消息" }}</text>
          <text class="message-card__content">{{ message.content || "" }}</text>
          <text class="message-card__time">{{ formatDate(message.createdAt) }}</text>
        </view>
        <text class="message-card__arrow">›</text>
      </view>
      <view v-if="messages.length === 0" class="empty card">暂无消息，诊所关怀通知会显示在这里。</view>
      <text v-if="messages.length > 2" class="view-all" @click="openService('notifications')">查看全部通知</text>
    </view>

    <view class="invite-card" @click="openService('invite')">
      <view>
        <text class="invite-card__label">我的专属邀请码</text>
        <text class="invite-card__code">{{ inviteCode }}</text>
        <text class="invite-card__desc">分享健康陪伴给身边的人</text>
      </view>
      <text class="invite-card__action">邀请好友</text>
    </view>

    <button class="logout-button" @click="handleLogout">退出当前账号</button>

    <view v-if="showLogoutConfirm" class="logout-backdrop" @click="showLogoutConfirm = false">
      <view class="logout-dialog" @click.stop>
        <text class="logout-dialog__icon">🚪</text>
        <text class="logout-dialog__title">退出当前账号？</text>
        <text class="logout-dialog__desc">退出后，本机将清除登录状态。您的健康档案仍会安全保存在诊所。</text>
        <view class="logout-dialog__actions">
          <button class="logout-dialog__button logout-dialog__button--quiet" @click="showLogoutConfirm = false">继续使用</button>
          <button class="logout-dialog__button logout-dialog__button--danger" @click="confirmLogout">退出登录</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import { useUserStore } from "@/stores/useUserStore";
import { usePeriodStore } from "@/stores/usePeriodStore";
import * as menstrualApi from "@/api/menstrual";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

const userStore = useUserStore();
const periodStore = usePeriodStore();
const dashboard = ref<any>({});
const inviteInfo = ref<any>({});
const showLogoutConfirm = ref(false);

const patient = computed(() => dashboard.value.patient ?? {});
const clinic = computed(() => dashboard.value.clinic ?? {});
const messages = computed<any[]>(() => dashboard.value.messages ?? []);
const unreadCount = computed(() => messages.value.filter((message) => !message.read).length);
const inviteCode = computed(() => inviteInfo.value.inviteCode || dashboard.value.inviteCode || "暂未生成");
const stats = computed(() => [
  { type: "booking", label: "预约", value: dashboard.value.appointments?.length ?? 0 },
  { type: "enrollments", label: "报名", value: dashboard.value.enrollments?.length ?? 0 },
  { type: "period", label: "周期记录", value: periodStore.recordCount },
  { type: "coupons", label: "健康关怀", value: dashboard.value.coupons?.length ?? 0 },
]);

function formatDate(value: string) {
  return value?.replace("T", " ").slice(0, 16) || "";
}

function openService(type: string) {
  if (type === "period") {
    uni.switchTab({ url: "/pages/period/index" });
    return;
  }
  if (type === "booking") {
    uni.navigateTo({ url: "/pages/booking/index" });
    return;
  }
  uni.navigateTo({ url: `/pages/services/index?type=${type}` });
}

function handleLogout() {
  showLogoutConfirm.value = true;
}

async function confirmLogout() {
  showLogoutConfirm.value = false;
  await userStore.doLogout();
  uni.reLaunch({ url: "/pages/login/index" });
}

onMounted(async () => {
  try {
    const [dash, invite] = await Promise.all([
      menstrualApi.fetchDashboard(),
      menstrualApi.fetchInviteInfo().catch(() => ({})),
    ]);
    dashboard.value = dash;
    inviteInfo.value = invite;
  } catch {
    // Profile remains usable with locally cached identity when dashboard loading fails.
  }
  if (!periodStore.records.length) await periodStore.loadAll().catch(() => undefined);
});
</script>

<style lang="scss" scoped>
.profile-page { min-height: 100vh; padding: $gap-lg $gap-md 120rpx; }
.profile-card { display: flex; align-items: center; gap: $gap-md; padding: $gap-xl $gap-lg; border-radius: $radius-lg; background: linear-gradient(145deg, #fff, $blush); box-shadow: $shadow-md; }
.profile-card__avatar { width: 104rpx; height: 104rpx; flex: 0 0 auto; border-radius: 50%; background: linear-gradient(135deg, $rose, $rose-dark); color: #fff; font-size: $font-2xl; font-weight: 800; display: flex; align-items: center; justify-content: center; }
.profile-card__info { min-width: 0; }
.profile-card__eyebrow, .section-head__eyebrow { color: $rose-dark; font-size: $font-xs; font-weight: 700; display: block; }
.profile-card__name { color: $ink; font-size: $font-xl; font-weight: 800; display: block; line-height: 1.35; }
.profile-card__meta { color: $muted; font-size: $font-sm; display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.profile-stats { display: grid; grid-template-columns: repeat(4, 1fr); gap: $gap-sm; margin: $gap-lg 0 $gap-xl; }
.profile-stat { min-width: 0; padding: $gap-md $gap-xs; border-radius: $radius-md; background: $paper; box-shadow: $shadow-sm; text-align: center; }
.profile-stat__value { color: $ink; font-size: $font-xl; font-weight: 800; display: block; }
.profile-stat__label { color: $muted; font-size: 20rpx; white-space: nowrap; }
.section-head { display: flex; align-items: flex-end; justify-content: space-between; gap: $gap-md; margin-bottom: $gap-sm; }
.section-head__title { color: $ink; font-size: $font-lg; font-weight: 800; display: block; }
.section-head__meta { color: $muted; font-size: $font-xs; }
.message-list { display: flex; flex-direction: column; gap: $gap-sm; }
.message-card { display: flex; align-items: center; gap: $gap-sm; }
.message-card__dot { width: 12rpx; height: 12rpx; flex: 0 0 auto; border-radius: 50%; background: transparent; }
.message-card--unread .message-card__dot { background: $rose-dark; }
.message-card__body { flex: 1; min-width: 0; }
.message-card__title { color: $ink; font-size: $font-sm; font-weight: 700; display: block; }
.message-card__content { color: $muted; font-size: $font-xs; display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.message-card__time { color: $muted; font-size: 20rpx; display: block; margin-top: 4rpx; }
.message-card__arrow { color: $muted; font-size: $font-xl; }
.empty { color: $muted; font-size: $font-sm; text-align: center; }
.view-all { color: $rose-dark; font-size: $font-sm; font-weight: 700; text-align: center; padding: $gap-sm; }
.invite-card { display: flex; align-items: center; justify-content: space-between; gap: $gap-md; margin-top: $gap-xl; padding: $gap-lg; border-radius: $radius-lg; background: #27272a; box-shadow: $shadow-md; }
.invite-card__label { color: #d4d4d8; font-size: $font-xs; display: block; }
.invite-card__code { color: #fff; font-size: $font-xl; font-weight: 800; letter-spacing: 2rpx; display: block; }
.invite-card__desc { color: #a1a1aa; font-size: $font-xs; display: block; }
.invite-card__action { flex: 0 0 auto; padding: 12rpx 24rpx; border-radius: $radius-full; background: #fff; color: #27272a; font-size: $font-sm; font-weight: 700; }
.logout-button { width: 100%; margin-top: $gap-xl; padding: 22rpx; border: 1px solid $line; border-radius: $radius-md; background: $paper; color: #e11d48; font-size: $font-sm; }
.logout-backdrop { position: fixed; inset: 0; z-index: 30; display: flex; align-items: center; justify-content: center; padding: $gap-xl; background: rgba(39, 39, 42, .48); backdrop-filter: blur(8rpx); }
.logout-dialog { width: 100%; padding: 52rpx $gap-lg $gap-lg; border-radius: 36rpx; background: $paper; box-shadow: 0 30rpx 90rpx rgba(39, 39, 42, .22); text-align: center; }
.logout-dialog__icon { display:block;margin:0 auto $gap-md;font-size:64rpx;line-height:1; }
.logout-dialog__title { display: block; color: $ink; font-size: $font-lg; font-weight: 800; }
.logout-dialog__desc { display: block; margin-top: $gap-xs; color: $muted; font-size: $font-sm; line-height: 1.7; }
.logout-dialog__actions { display: flex; gap: $gap-sm; margin-top: $gap-xl; }
.logout-dialog__button { flex: 1; margin: 0; border-radius: $radius-full; font-size: $font-sm; font-weight: 700; }
.logout-dialog__button--quiet { background: $surface; color: $ink; }
.logout-dialog__button--danger { background: #e11d48; color: #fff; }
</style>
