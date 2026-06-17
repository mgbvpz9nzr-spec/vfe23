<template>
  <view class="page booking-page">
    <ClinicTopBar show-back />
    <view class="booking-hero">
      <text class="booking-hero__eyebrow">问诊预约</text>
      <text class="booking-hero__title">把合适的时间留给自己的健康</text>
      <text class="booking-hero__desc">选择医生或近期门诊活动，门店会尽快与您确认。</text>
    </view>

    <view class="section-head">
      <text class="section-head__title">本门诊出诊医生</text>
      <text class="section-head__meta">{{ doctors.length }} 位可预约</text>
    </view>
    <view class="doctor-list">
      <view v-for="doctor in doctors" :key="doctor.name" class="doctor-card card">
        <text class="doctor-card__avatar">{{ String(doctor.name || "医").slice(0, 1) }}</text>
        <view class="doctor-card__info">
          <text class="doctor-card__name">{{ doctor.name }}</text>
          <text class="doctor-card__desc">认证妇科医生 · 关注隐私与长期健康管理</text>
        </view>
        <text class="pill-action" @click="bookDoctor(doctor.name)">预约</text>
      </view>
    </view>

    <view class="section-head mt-lg">
      <text class="section-head__title">近期门诊活动</text>
      <text class="section-head__meta">{{ campaigns.length ? `${campaigns.length} 项进行中` : "暂无活动" }}</text>
    </view>
    <scroll-view v-if="campaigns.length" class="campaign-rail" scroll-x>
      <view class="campaign-rail__inner">
        <view v-for="campaign in campaigns.slice(0, 6)" :key="campaign.id" class="campaign-card" @click="showCampaign(campaign)">
          <text class="campaign-card__category">{{ campaign.category || "女性健康" }}</text>
          <text class="campaign-card__title">{{ campaign.title }}</text>
          <text class="campaign-card__target">{{ campaign.target || "适合关注日常妇科健康的女性" }}</text>
          <view class="campaign-card__footer">
            <text class="campaign-card__price">{{ money(campaign.promoPrice) }}</text>
            <text class="campaign-card__link">查看详情</text>
          </view>
        </view>
      </view>
    </scroll-view>
    <view v-else class="empty card">近期暂无开放活动，您仍可预约基础咨询。</view>

    <view class="section-head mt-lg">
      <text class="section-head__title">预约申请</text>
    </view>
    <view class="booking-form card">
      <view class="form-group">
        <text class="form-label">选择项目</text>
        <picker :range="itemLabels" :value="selectedItemIndex" @change="onItemChange">
          <view class="form-picker">{{ itemLabels[selectedItemIndex] }}</view>
        </picker>
      </view>
      <view class="form-group">
        <text class="form-label">期望医生</text>
        <picker :range="doctorNames" :value="selectedDoctorIndex" @change="onDoctorChange">
          <view class="form-picker">{{ selectedDoctor }}</view>
        </picker>
      </view>
      <view class="form-row">
        <view class="form-group form-row__item">
          <text class="form-label">到店日期</text>
          <picker mode="date" :value="selectedDate" :start="today" @change="onDateChange">
            <view class="form-picker">{{ selectedDate || "选择日期" }}</view>
          </picker>
        </view>
        <view class="form-group form-row__item">
          <text class="form-label">到店时间</text>
          <picker mode="time" :value="selectedTime" @change="onTimeChange">
            <view class="form-picker">{{ selectedTime || "选择时间" }}</view>
          </picker>
        </view>
      </view>
      <button class="btn-primary submit-button" :loading="submitting" @click="submitBooking">提交预约申请</button>
      <text class="business-hours">营业时间：{{ clinic.businessHours || "请联系诊所确认" }}</text>
    </view>

    <view class="section-head mt-lg">
      <view>
        <text class="section-head__eyebrow">预约进度</text>
        <text class="section-head__title">我的预约</text>
      </view>
      <text class="section-head__meta">{{ appointments.length }} 条记录</text>
    </view>
    <view class="appointment-list">
      <view v-for="appointment in orderedAppointments" :key="appointment.id" class="appointment-card card">
        <view class="appointment-card__top">
          <text class="status" :class="`status--${String(appointment.status).toLowerCase()}`">{{ statusLabel(appointment.status) }}</text>
          <text class="appointment-card__time">{{ formatDateTime(appointment.time) }}</text>
        </view>
        <text class="appointment-card__title">{{ appointment.item }}</text>
        <text class="appointment-card__meta">{{ appointment.doctor }} · {{ clinic.name || "门诊" }}</text>
        <text v-if="canCancel(appointment)" class="cancel-action" @click="cancelBooking(appointment)">取消预约</text>
      </view>
      <view v-if="appointments.length === 0" class="empty card">暂无预约记录，提交申请后会显示在这里。</view>
    </view>

    <view v-if="activeCampaign" class="detail-backdrop" @click="activeCampaign = null">
      <view class="detail-sheet" @click.stop>
        <view class="detail-sheet__handle" />
        <text class="detail-sheet__category">{{ activeCampaign.category || "健康活动" }}</text>
        <text class="detail-sheet__title">{{ activeCampaign.title }}</text>
        <text class="detail-sheet__summary">{{ activeCampaign.summary || activeCampaign.subtitle || activeCampaign.target || "关注女性日常健康，提供专业门诊支持。" }}</text>
        <text class="detail-sheet__content">{{ activeCampaign.content || activeCampaign.description || activeCampaign.activityInfo || "" }}</text>
        <view class="detail-sheet__footer">
          <text class="detail-sheet__price">{{ money(activeCampaign.promoPrice) }}</text>
          <button class="btn-primary detail-sheet__button" @click="selectCampaign(activeCampaign)">选择此项目</button>
        </view>
      </view>
    </view>

    <view v-if="cancelTarget" class="confirm-backdrop" @click="cancelTarget = null">
      <view class="confirm-dialog" @click.stop>
        <text class="confirm-dialog__icon">×</text>
        <text class="confirm-dialog__title">取消这次预约？</text>
        <text class="confirm-dialog__desc">{{ cancelTarget.item }} · {{ formatDateTime(cancelTarget.time) }}</text>
        <view class="confirm-dialog__actions">
          <button class="confirm-dialog__button confirm-dialog__button--quiet" @click="cancelTarget = null">保留预约</button>
          <button class="confirm-dialog__button confirm-dialog__button--danger" :loading="cancelling" @click="confirmCancel">确认取消</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import * as menstrualApi from "@/api/menstrual";
import { api } from "@/api/client";
import { useAppStore } from "@/stores/useAppStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

const appStore = useAppStore();
const BASIC_ITEM = "基础妇科健康咨询";
const STATUS_LABELS: Record<string, string> = {
  PENDING: "待确认",
  CONFIRMED: "已确认",
  ARRIVED: "已到店",
  CANCELLED: "已取消",
};

const doctors = ref<any[]>([]);
const campaigns = ref<any[]>([]);
const appointments = ref<any[]>([]);
const clinic = ref<any>({});
const selectedItemIndex = ref(0);
const selectedDoctor = ref("门店值班医生");
const selectedDate = ref("");
const selectedTime = ref("09:00");
const activeCampaign = ref<any | null>(null);
const cancelTarget = ref<any | null>(null);
const submitting = ref(false);
const cancelling = ref(false);

const today = new Date().toISOString().slice(0, 10);
const doctorNames = computed(() => doctors.value.map((doctor) => doctor.name || "门店值班医生"));
const selectedDoctorIndex = computed(() => Math.max(0, doctorNames.value.indexOf(selectedDoctor.value)));
const itemLabels = computed(() => [
  BASIC_ITEM,
  ...campaigns.value.map((campaign) => `${campaign.title} · ${money(campaign.promoPrice)}`),
]);
const selectedCampaign = computed(() => campaigns.value[selectedItemIndex.value - 1]);
const orderedAppointments = computed(() => [...appointments.value].reverse());

function money(value: unknown) {
  const amount = Number(value);
  return Number.isFinite(amount) ? `¥${amount.toFixed(2)}` : "到店咨询";
}

function formatDateTime(value: string) {
  if (!value) return "时间待确认";
  return value.replace("T", " ").slice(0, 16);
}

function statusLabel(status: string) {
  return STATUS_LABELS[status] || status || "待确认";
}

function canCancel(appointment: any) {
  return ["PENDING", "CONFIRMED"].includes(appointment.status);
}

function onItemChange(event: any) {
  selectedItemIndex.value = Number(event.detail.value);
}

function onDoctorChange(event: any) {
  selectedDoctor.value = doctorNames.value[Number(event.detail.value)] || "门店值班医生";
}

function onDateChange(event: any) {
  selectedDate.value = event.detail.value;
}

function onTimeChange(event: any) {
  selectedTime.value = event.detail.value;
}

function bookDoctor(name: string) {
  selectedDoctor.value = name;
  appStore.showToast(`已选择${name}`, "info");
}

function showCampaign(campaign: any) {
  activeCampaign.value = campaign;
}

function selectCampaign(campaign: any) {
  const index = campaigns.value.findIndex((item) => item.id === campaign.id);
  selectedItemIndex.value = index + 1;
  activeCampaign.value = null;
  appStore.showToast("已选择活动项目", "success");
}

async function submitBooking() {
  if (!selectedDoctor.value || !selectedDate.value || !selectedTime.value) {
    appStore.showToast("请完善预约日期和时间", "error");
    return;
  }
  submitting.value = true;
  try {
    const campaign = selectedCampaign.value;
    await menstrualApi.createAppointment({
      doctor: selectedDoctor.value,
      item: campaign?.title || BASIC_ITEM,
      time: `${selectedDate.value}T${selectedTime.value}`,
    });
    if (campaign?.id) {
      await api("/api/patient/enrollments", {
        method: "POST",
        body: { campaignId: campaign.id, source: "在线预约" },
      }).catch(() => undefined);
    }
    appStore.showToast("预约申请已提交", "success");
    await loadData();
  } catch (error: any) {
    appStore.showToast(error.message || "预约提交失败", "error");
  } finally {
    submitting.value = false;
  }
}

function cancelBooking(appointment: any) {
  cancelTarget.value = appointment;
}

async function confirmCancel() {
  if (!cancelTarget.value || cancelling.value) return;
  cancelling.value = true;
  try {
    await menstrualApi.cancelAppointment(cancelTarget.value.id);
    cancelTarget.value = null;
    appStore.showToast("预约已取消", "success");
    await loadData();
  } catch (error: any) {
    appStore.showToast(error.message || "取消失败", "error");
  } finally {
    cancelling.value = false;
  }
}

async function loadData() {
  try {
    const dashboard = await menstrualApi.fetchDashboard();
    doctors.value = dashboard.doctors?.length ? dashboard.doctors : [{ name: "门店值班医生" }];
    campaigns.value = dashboard.campaigns ?? [];
    appointments.value = dashboard.appointments ?? [];
    clinic.value = dashboard.clinic ?? {};
    if (!doctorNames.value.includes(selectedDoctor.value)) selectedDoctor.value = doctorNames.value[0];
  } catch (error: any) {
    appStore.showToast(error.message || "预约信息加载失败", "error");
  }
}

onMounted(loadData);
</script>

<style lang="scss" scoped>
.booking-page { min-height: 100vh; padding: $gap-md $gap-md 120rpx; }
.booking-hero { padding: $gap-xl $gap-sm $gap-lg; }
.booking-hero__eyebrow, .section-head__eyebrow { color: $rose-dark; font-size: $font-xs; font-weight: 700; display: block; }
.booking-hero__title { color: $ink; font-size: $font-xl; font-weight: 800; line-height: 1.35; display: block; margin: 6rpx 0; }
.booking-hero__desc, .business-hours { color: $muted; font-size: $font-sm; display: block; }
.section-head { display: flex; align-items: flex-end; justify-content: space-between; gap: $gap-md; margin-bottom: $gap-sm; }
.section-head__title { color: $ink; font-size: $font-md; font-weight: 800; display: block; }
.section-head__meta { color: $muted; font-size: $font-xs; white-space: nowrap; }
.doctor-list, .appointment-list { display: flex; flex-direction: column; gap: $gap-sm; }
.doctor-card { display: flex; align-items: center; gap: $gap-md; }
.doctor-card__avatar { width: 72rpx; height: 72rpx; border-radius: 50%; background: $blush; color: $rose-dark; font-size: $font-lg; font-weight: 800; display: flex; align-items: center; justify-content: center; }
.doctor-card__info { flex: 1; min-width: 0; }
.doctor-card__name, .appointment-card__title { color: $ink; font-size: $font-md; font-weight: 700; display: block; }
.doctor-card__desc, .appointment-card__meta, .appointment-card__time { color: $muted; font-size: $font-xs; }
.pill-action { padding: 10rpx 24rpx; border-radius: $radius-full; background: $blush; color: $rose-dark; font-size: $font-sm; font-weight: 700; }
.campaign-rail { width: 100%; white-space: nowrap; }
.campaign-rail__inner { display: inline-flex; gap: $gap-sm; padding-bottom: 4rpx; }
.campaign-card { width: 520rpx; padding: $gap-lg; border-radius: $radius-lg; background: linear-gradient(145deg, #fff, $blush); box-shadow: $shadow-sm; white-space: normal; }
.campaign-card__category { color: $rose-dark; font-size: $font-xs; font-weight: 700; }
.campaign-card__title { color: $ink; font-size: $font-lg; font-weight: 800; display: block; margin: $gap-xs 0; }
.campaign-card__target { color: $muted; font-size: $font-sm; display: block; min-height: 72rpx; }
.campaign-card__footer, .appointment-card__top, .detail-sheet__footer { display: flex; align-items: center; justify-content: space-between; gap: $gap-md; margin-top: $gap-md; }
.campaign-card__price, .detail-sheet__price { color: $ink; font-size: $font-md; font-weight: 800; }
.campaign-card__link { color: $rose-dark; font-size: $font-sm; font-weight: 700; }
.empty { color: $muted; font-size: $font-sm; text-align: center; }
.form-group { margin-bottom: $gap-md; }
.form-label { color: $muted; font-size: $font-sm; display: block; margin-bottom: $gap-xs; }
.form-picker { min-height: 80rpx; padding: 0 $gap-md; border: 1px solid $line; border-radius: $radius-sm; background: $surface; color: $ink; font-size: $font-sm; display: flex; align-items: center; }
.form-row { display: flex; gap: $gap-sm; }
.form-row__item { flex: 1; min-width: 0; }
.submit-button { width: 100%; margin-top: $gap-xs; }
.business-hours { margin-top: $gap-md; text-align: center; }
.status { padding: 6rpx 16rpx; border-radius: $radius-full; background: $blush; color: $rose-dark; font-size: $font-xs; font-weight: 700; }
.status--arrived { background: #ecfdf5; color: #059669; }
.status--cancelled { background: #f4f4f5; color: $muted; }
.appointment-card__title { margin-top: $gap-sm; }
.cancel-action { color: #e11d48; font-size: $font-sm; font-weight: 700; display: inline-block; margin-top: $gap-md; }
.detail-backdrop { position: fixed; inset: 0; z-index: 20; background: rgba(39, 39, 42, .42); display: flex; align-items: flex-end; }
.detail-sheet { width: 100%; max-height: 78vh; overflow-y: auto; padding: $gap-md $gap-lg 64rpx; border-radius: 36rpx 36rpx 0 0; background: $paper; }
.detail-sheet__handle { width: 80rpx; height: 8rpx; margin: 0 auto $gap-lg; border-radius: $radius-full; background: #e4e4e7; }
.detail-sheet__category { color: $rose-dark; font-size: $font-xs; font-weight: 700; }
.detail-sheet__title { color: $ink; font-size: $font-xl; font-weight: 800; display: block; margin: $gap-xs 0 $gap-sm; }
.detail-sheet__summary { color: $ink; font-size: $font-md; font-weight: 600; display: block; }
.detail-sheet__content { color: $muted; font-size: $font-sm; display: block; margin-top: $gap-md; white-space: pre-wrap; }
.detail-sheet__button { margin: 0; font-size: $font-sm; padding: 18rpx 32rpx; }
.confirm-backdrop { position: fixed; inset: 0; z-index: 30; display: flex; align-items: center; justify-content: center; padding: $gap-xl; background: rgba(39, 39, 42, .48); backdrop-filter: blur(8rpx); }
.confirm-dialog { width: 100%; padding: 52rpx $gap-lg $gap-lg; border-radius: 36rpx; background: $paper; box-shadow: 0 30rpx 90rpx rgba(39, 39, 42, .22); text-align: center; }
.confirm-dialog__icon { display: flex; width: 84rpx; height: 84rpx; align-items: center; justify-content: center; margin: 0 auto $gap-md; border-radius: 50%; background: #fff1f2; color: #e11d48; font-size: 52rpx; line-height: 1; }
.confirm-dialog__title { display: block; color: $ink; font-size: $font-lg; font-weight: 800; }
.confirm-dialog__desc { display: block; margin-top: $gap-xs; color: $muted; font-size: $font-sm; line-height: 1.6; }
.confirm-dialog__actions { display: flex; gap: $gap-sm; margin-top: $gap-xl; }
.confirm-dialog__button { flex: 1; margin: 0; border-radius: $radius-full; font-size: $font-sm; font-weight: 700; }
.confirm-dialog__button--quiet { background: $surface; color: $ink; }
.confirm-dialog__button--danger { background: #e11d48; color: #fff; }
</style>
