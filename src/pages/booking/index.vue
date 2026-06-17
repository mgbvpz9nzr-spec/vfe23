<template>
  <view class="page booking-page">
    <ClinicTopBar show-back />
    
    <view class="booking-header">
      <text class="booking-title">门诊挂号与服务预约</text>
      <text class="booking-sub">在线提交预约申请，分诊台将尽快为您确认排期。</text>
    </view>

    <!-- 预约表单 -->
    <view class="booking-form card">
      <view class="form-section">
        <text class="form-label">预约项目分类</text>
        <picker :range="itemLabels" :value="selectedItemIndex" @change="onItemChange">
          <view class="form-input">{{ itemLabels[selectedItemIndex] }}</view>
        </picker>
      </view>
      
      <view class="form-section">
        <text class="form-label">指定接诊医师</text>
        <picker :range="doctorNames" :value="selectedDoctorIndex" @change="onDoctorChange">
          <view class="form-input">{{ selectedDoctor }}</view>
        </picker>
      </view>
      
      <view class="form-row">
        <view class="form-section flex-1">
          <text class="form-label">期望就诊日期</text>
          <picker mode="date" :value="selectedDate" :start="today" @change="onDateChange">
            <view class="form-input">{{ selectedDate || "请选择日期" }}</view>
          </picker>
        </view>
        <view class="form-section flex-1">
          <text class="form-label">期望就诊时段</text>
          <picker mode="time" :value="selectedTime" @change="onTimeChange">
            <view class="form-input">{{ selectedTime || "请选择时间" }}</view>
          </picker>
        </view>
      </view>
      
      <button class="btn-primary submit-btn" :loading="submitting" @click="submitBooking">提交预约申请</button>
      <text class="clinic-info">机构门诊时间：{{ clinic.businessHours || "请详询导诊台" }}</text>
    </view>

    <!-- 预约记录 -->
    <view class="section-title">我的预约记录</view>
    <view class="appointment-list">
      <view v-for="appointment in orderedAppointments" :key="appointment.id" class="appointment-card card">
        <view class="appointment-card__head">
          <text class="appointment-status" :class="`status--${String(appointment.status).toLowerCase()}`">{{ statusLabel(appointment.status) }}</text>
          <text class="appointment-time">{{ formatDateTime(appointment.time) }}</text>
        </view>
        <view class="appointment-card__body">
          <text class="appointment-item">{{ appointment.item }}</text>
          <text class="appointment-meta">主诊医生：{{ appointment.doctor }} | {{ clinic.name || "门诊" }}</text>
        </view>
        <view class="appointment-card__actions" v-if="canCancel(appointment)">
          <button class="btn-outline btn-cancel" @click="cancelBooking(appointment)">取消预约</button>
        </view>
      </view>
      <view v-if="appointments.length === 0" class="empty-state">
        <text>暂无挂号或预约记录</text>
      </view>
    </view>

    <!-- 取消确认弹窗 -->
    <view v-if="cancelTarget" class="overlay" @click="cancelTarget = null">
      <view class="dialog" @click.stop>
        <view class="dialog-header">
          <text class="dialog-title">确认取消预约</text>
        </view>
        <view class="dialog-body">
          <text>您确定要取消以下预约吗？</text>
          <text class="dialog-detail">{{ cancelTarget.item }} · {{ formatDateTime(cancelTarget.time) }}</text>
        </view>
        <view class="dialog-footer">
          <button class="btn-secondary flex-1" @click="cancelTarget = null">暂不取消</button>
          <button class="btn-danger flex-1" :loading="cancelling" @click="confirmCancel">确认取消</button>
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
const BASIC_ITEM = "常规临床问诊";
const STATUS_LABELS: Record<string, string> = {
  PENDING: "等待分诊确认",
  CONFIRMED: "已确认排期",
  ARRIVED: "患者已到诊",
  CANCELLED: "已取消",
};

const doctors = ref<any[]>([]);
const campaigns = ref<any[]>([]);
const appointments = ref<any[]>([]);
const clinic = ref<any>({});
const selectedItemIndex = ref(0);
const selectedDoctor = ref("急诊/值班医师");
const selectedDate = ref("");
const selectedTime = ref("09:00");
const cancelTarget = ref<any | null>(null);
const submitting = ref(false);
const cancelling = ref(false);

const today = new Date().toISOString().slice(0, 10);
const doctorNames = computed(() => doctors.value.map((doctor) => doctor.name || "急诊/值班医师"));
const selectedDoctorIndex = computed(() => Math.max(0, doctorNames.value.indexOf(selectedDoctor.value)));
const itemLabels = computed(() => [
  BASIC_ITEM,
  ...campaigns.value.map((campaign) => `${campaign.title} (活动项目)`),
]);
const selectedCampaign = computed(() => campaigns.value[selectedItemIndex.value - 1]);
const orderedAppointments = computed(() => [...appointments.value].reverse());

function formatDateTime(value: string) {
  if (!value) return "排期确认中";
  return value.replace("T", " ").slice(0, 16);
}

function statusLabel(status: string) {
  return STATUS_LABELS[status] || status || "等待确认";
}

function canCancel(appointment: any) {
  return ["PENDING", "CONFIRMED"].includes(appointment.status);
}

function onItemChange(event: any) { selectedItemIndex.value = Number(event.detail.value); }
function onDoctorChange(event: any) { selectedDoctor.value = doctorNames.value[Number(event.detail.value)] || "急诊/值班医师"; }
function onDateChange(event: any) { selectedDate.value = event.detail.value; }
function onTimeChange(event: any) { selectedTime.value = event.detail.value; }

async function submitBooking() {
  if (!selectedDoctor.value || !selectedDate.value || !selectedTime.value) {
    appStore.showToast("请完善预约日期和时段", "error");
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
        body: { campaignId: campaign.id, source: "系统在线预约" },
      }).catch(() => undefined);
    }
    appStore.showToast("预约申请已进入分诊队列", "success");
    await loadData();
  } catch (error: any) {
    appStore.showToast(error.message || "系统接口错误，提交失败", "error");
  } finally {
    submitting.value = false;
  }
}

function cancelBooking(appointment: any) { cancelTarget.value = appointment; }

async function confirmCancel() {
  if (!cancelTarget.value || cancelling.value) return;
  cancelling.value = true;
  try {
    await menstrualApi.cancelAppointment(cancelTarget.value.id);
    cancelTarget.value = null;
    appStore.showToast("已成功撤销该预约申请", "success");
    await loadData();
  } catch (error: any) {
    appStore.showToast(error.message || "撤销操作失败", "error");
  } finally {
    cancelling.value = false;
  }
}

async function loadData() {
  try {
    const dashboard = await menstrualApi.fetchDashboard();
    doctors.value = dashboard.doctors?.length ? dashboard.doctors : [{ name: "急诊/值班医师" }];
    campaigns.value = dashboard.campaigns ?? [];
    appointments.value = dashboard.appointments ?? [];
    clinic.value = dashboard.clinic ?? {};
    if (!doctorNames.value.includes(selectedDoctor.value)) selectedDoctor.value = doctorNames.value[0];
  } catch (error: any) {
    appStore.showToast(error.message || "核心数据加载失败", "error");
  }
}

onMounted(loadData);
</script>

<style lang="scss" scoped>
.booking-page { min-height: 100vh; padding: $gap-md $gap-md 120rpx; }

.booking-header { padding: $gap-md 0 $gap-lg; }
.booking-title { display: block; font-size: $font-xl; font-weight: 700; color: $ink; margin-bottom: 8rpx;}
.booking-sub { font-size: $font-sm; color: $muted; }

.booking-form { padding: $gap-lg; margin-bottom: $gap-lg; }
.form-section { margin-bottom: $gap-md; }
.form-label { display: block; font-size: 24rpx; font-weight: 600; color: $muted; margin-bottom: 8rpx; }
.form-input { 
  width: 100%; 
  height: 80rpx; 
  background: $surface; 
  border: 1px solid $line; 
  border-radius: $radius-sm; 
  padding: 0 24rpx; 
  display: flex; 
  align-items: center; 
  font-size: $font-md; 
  color: $ink; 
}
.form-row { display: flex; gap: $gap-md; }
.flex-1 { flex: 1; min-width: 0; }

.submit-btn { width: 100%; margin-top: $gap-md; }
.clinic-info { display: block; text-align: center; font-size: 22rpx; color: $muted; margin-top: $gap-md; }

.section-title { font-size: $font-md; font-weight: 700; color: $ink; margin-bottom: $gap-sm; }

.appointment-list { display: flex; flex-direction: column; gap: $gap-sm; }
.appointment-card { padding: $gap-md; }
.appointment-card__head { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12rpx; border-bottom: 1px dashed $line; padding-bottom: 12rpx;}
.appointment-status { font-size: 20rpx; font-weight: bold; padding: 4rpx 12rpx; border-radius: 4rpx; }
.status--pending { background: #fef08a; color: #854d0e; }
.status--confirmed { background: #dcfce7; color: #166534; }
.status--arrived { background: #e0f2fe; color: #075985; }
.status--cancelled { background: $surface; color: $muted; }
.appointment-time { font-size: 24rpx; color: $ink; font-weight: 600; }
.appointment-card__body { display: flex; flex-direction: column; gap: 4rpx; }
.appointment-item { font-size: $font-sm; font-weight: 700; color: $ink; }
.appointment-meta { font-size: 24rpx; color: $muted; }
.appointment-card__actions { margin-top: $gap-md; text-align: right; }
.btn-cancel { display: inline-block; padding: 8rpx 24rpx; font-size: 22rpx; margin: 0; border-color: #cbd5e1; color: $muted;}

.empty-state { padding: $gap-xl; text-align: center; background: $paper; border-radius: $radius-md; border: 1px dashed $line; color: $muted; font-size: $font-sm; }

.overlay { position: fixed; inset: 0; z-index: 50; background: rgba(15, 23, 42, 0.4); display: flex; align-items: center; justify-content: center; padding: $gap-xl; }
.dialog { width: 100%; background: $paper; border-radius: $radius-md; overflow: hidden; box-shadow: $shadow-lg; }
.dialog-header { padding: $gap-md $gap-lg; border-bottom: 1px solid $line; background: $surface; }
.dialog-title { font-size: $font-md; font-weight: 700; color: $ink; }
.dialog-body { padding: $gap-lg; font-size: $font-sm; color: $ink; line-height: 1.6; }
.dialog-detail { display: block; margin-top: 8rpx; font-weight: 600; color: #ef4444; }
.dialog-footer { display: flex; padding: $gap-md $gap-lg; gap: $gap-md; border-top: 1px solid $line; }
.btn-danger { background: #ef4444; color: #fff; border: none; border-radius: $radius-md; font-size: $font-md; font-weight: 500; }
</style>
