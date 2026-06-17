<template>
  <view class="login-page">
    <view class="login-card">
      <view class="login-header">
        <text class="login-logo">🌷</text>
        <text class="login-title">女性健康助手</text>
        <text class="login-sub">登录后开始记录您的健康</text>
      </view>

      <!-- 登录表单 -->
      <view class="login-form" v-if="!isRegistering">
        <view class="form-group">
          <text class="form-label">手机号</text>
          <input
            class="form-input"
            v-model="phone"
            type="number"
            maxlength="11"
            placeholder="请输入手机号"
          />
        </view>
        <view class="form-group">
          <text class="form-label">密码</text>
          <input
            class="form-input"
            v-model="password"
            type="password"
            placeholder="请输入密码"
          />
        </view>
        <view class="form-error" v-if="userStore.error">{{ userStore.error }}</view>
        <button
          class="btn-primary login-btn"
          :loading="userStore.isLoggingIn"
          @click="handleLogin"
        >
          登录
        </button>
        <view class="login-switch" @click="isRegistering = true">
          还没有账号？<text class="text-rose">立即注册</text>
        </view>
      </view>

      <!-- 注册表单 -->
      <view class="login-form" v-else>
        <view class="form-group">
          <text class="form-label">手机号</text>
          <input class="form-input" v-model="phone" type="number" maxlength="11" placeholder="请输入手机号" />
        </view>
        <view class="form-group form-group--row">
          <input class="form-input form-input--flex" v-model="code" type="number" maxlength="6" placeholder="验证码" />
          <button class="btn-code" :disabled="codeSending" @click="handleRequestCode">
            {{ codeSending ? `${countdown}s` : "获取验证码" }}
          </button>
        </view>
        <view class="form-group">
          <text class="form-label">姓名</text>
          <input class="form-input" v-model="name" placeholder="请输入您的姓名" />
        </view>
        <view class="form-group">
          <text class="form-label">年龄</text>
          <input class="form-input" v-model="age" type="number" placeholder="选填" />
        </view>
        <view class="form-group">
          <text class="form-label">选择诊所 <text class="text-muted" style="font-size:20rpx">（必选）</text></text>
          <view class="clinic-search">
            <text class="clinic-search__icon">⌕</text>
            <input
              v-model="clinicQuery"
              class="clinic-search__input"
              confirm-type="search"
              placeholder="搜索为您服务的诊所、城市或地址"
              @input="scheduleClinicSearch"
              @confirm="loadClinics"
            />
            <text v-if="clinicQuery" class="clinic-search__clear" @click="clearClinicSearch">×</text>
          </view>
          <view class="clinic-hint">
            <text v-if="clinicLoading">正在查找诊所...</text>
            <text v-else-if="clinicError" class="clinic-hint__error" @click="loadClinics">{{ clinicError }}，点击重试</text>
            <text v-else>{{ clinicQuery ? `找到 ${clinics.length} 家匹配诊所` : `共 ${clinics.length} 家可选诊所` }}</text>
          </view>
          <view v-if="selectedClinic" class="clinic-selected">
            <view>
              <text class="clinic-selected__eyebrow">已选择诊所</text>
              <text class="clinic-selected__name">{{ selectedClinic.name }}</text>
              <text class="clinic-selected__location">{{ clinicLocation(selectedClinic) }}</text>
            </view>
            <text class="clinic-selected__clear" @click="clearSelectedClinic">重新选择</text>
          </view>
          <scroll-view v-if="!clinicLoading && clinics.length" class="clinic-list" scroll-y>
            <view
              v-for="clinic in clinics"
              :key="clinic.id"
              class="clinic-option"
              :class="{ 'clinic-option--active': selectedClinicId === clinic.id }"
              @click="selectClinic(clinic)"
            >
              <view class="clinic-option__mark">{{ selectedClinicId === clinic.id ? '✓' : String(clinic.name).slice(0, 1) }}</view>
              <view class="clinic-option__body">
                <text class="clinic-option__name">{{ clinic.name }}</text>
                <text class="clinic-option__location">{{ clinicLocation(clinic) }}</text>
                <text v-if="clinic.address" class="clinic-option__address">{{ clinic.address }}</text>
              </view>
            </view>
          </scroll-view>
          <view v-else-if="!clinicLoading && !clinicError" class="clinic-empty">
            <text>没有找到匹配诊所</text>
            <text>可尝试输入诊所名称、城市、区县或地址</text>
          </view>
        </view>
        <view class="form-group">
          <text class="form-label">密码</text>
          <input class="form-input" v-model="password" type="password" placeholder="设置登录密码" />
        </view>
        <view class="form-group">
          <text class="form-label">确认密码</text>
          <input class="form-input" v-model="confirmPassword" type="password" placeholder="再次输入密码" />
        </view>
        <view class="form-error" v-if="userStore.error">{{ userStore.error }}</view>
        <button
          class="btn-primary login-btn"
          :loading="userStore.isLoggingIn"
          @click="handleRegister"
        >
          注册
        </button>
        <view class="login-switch" @click="isRegistering = false">
          已有账号？<text class="text-rose">去登录</text>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, watch } from "vue";
import { useUserStore } from "@/stores/useUserStore";
import { useAppStore } from "@/stores/useAppStore";
import * as authApi from "@/api/auth";
import type { ClinicLookupItem } from "@/api/auth";

const userStore = useUserStore();
const appStore = useAppStore();

const isRegistering = ref(false);
const phone = ref("");
const password = ref("");
const confirmPassword = ref("");
const code = ref("");
const name = ref("");
const age = ref("");

const codeSending = ref(false);
const countdown = ref(0);
let timer: any = null;

// === 诊所选择 ===
const clinics = ref<ClinicLookupItem[]>([]);
const selectedClinicId = ref("");
const selectedClinic = ref<ClinicLookupItem | null>(null);
const clinicQuery = ref("");
const clinicLoading = ref(false);
const clinicError = ref("");
let clinicSearchTimer: any = null;

async function loadClinics() {
  clinicLoading.value = true;
  clinicError.value = "";
  try {
    clinics.value = await authApi.lookupClinics(clinicQuery.value);
  } catch (error: any) {
    clinics.value = [];
    clinicError.value = error.message || "诊所列表加载失败";
  } finally {
    clinicLoading.value = false;
  }
}

function scheduleClinicSearch() {
  clearTimeout(clinicSearchTimer);
  clinicSearchTimer = setTimeout(loadClinics, 280);
}

function clearClinicSearch() {
  clinicQuery.value = "";
  void loadClinics();
}

function selectClinic(clinic: ClinicLookupItem) {
  selectedClinicId.value = clinic.id;
  selectedClinic.value = clinic;
}

function clearSelectedClinic() {
  selectedClinicId.value = "";
  selectedClinic.value = null;
}

function clinicLocation(clinic: ClinicLookupItem) {
  return [clinic.province, clinic.city, clinic.district].filter(Boolean).join(" · ") || "诊所地址待完善";
}

// 注册页显示时加载诊所列表
watch(isRegistering, (val) => {
  if (val && clinics.value.length === 0) loadClinics();
});

async function handleLogin() {
  if (!phone.value || !password.value) {
    appStore.showToast("请填写手机号和密码", "error");
    return;
  }
  try {
    await userStore.login(phone.value, password.value);
    uni.switchTab({ url: "/pages/home/index" });
  } catch {
    // error shown in store
  }
}

async function handleRegister() {
  if (!phone.value || !code.value || !name.value) {
    appStore.showToast("请填写必填信息", "error");
    return;
  }
  if (password.value !== confirmPassword.value) {
    appStore.showToast("两次密码不一致", "error");
    return;
  }
  if (!selectedClinicId.value) {
    appStore.showToast("请选择您所在的诊所", "error");
    return;
  }
  try {
    await userStore.register({
      phone: phone.value,
      code: code.value,
      name: name.value,
      age: age.value ? Number(age.value) : undefined,
      password: password.value,
      clinicId: selectedClinicId.value,
    });
    uni.switchTab({ url: "/pages/home/index" });
  } catch {
    // error shown in store
  }
}

async function handleRequestCode() {
  if (!phone.value) {
    appStore.showToast("请先填写手机号", "error");
    return;
  }
  codeSending.value = true;
  countdown.value = 60;
  timer = setInterval(() => {
    countdown.value--;
    if (countdown.value <= 0) {
      clearInterval(timer);
      codeSending.value = false;
    }
  }, 1000);
  try {
    await userStore.requestCode(phone.value);
    if (userStore.devCode) {
      code.value = userStore.devCode;
      appStore.showToast("开发模式：验证码已自动填入", "info");
    }
  } catch {
    clearInterval(timer);
    codeSending.value = false;
  }
}
</script>

<style lang="scss" scoped>
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 48rpx;
  background: linear-gradient(180deg, $blush 0%, $surface 60%);
}

.login-card {
  width: 100%;
  max-width: 600rpx;
}

.login-header {
  text-align: center;
  margin-bottom: 64rpx;
}

.login-logo { display:block;margin-bottom:$gap-md;font-size:80rpx; }

.login-title {
  font-size: $font-2xl;
  font-weight: 800;
  color: $ink;
  display: block;
  margin-bottom: $gap-xs;
}

.login-sub {
  font-size: $font-sm;
  color: $muted;
}

.login-form {
  background: $paper;
  border-radius: $radius-lg;
  padding: $gap-lg $gap-xl;
  box-shadow: $shadow-md;
}

.form-group {
  margin-bottom: $gap-md;

  &--row {
    display: flex;
    gap: $gap-sm;
  }
}

.form-label {
  font-size: $font-sm;
  color: $muted;
  margin-bottom: $gap-xs;
  display: block;
}

.form-input {
  width: 100%;
  height: 80rpx;
  background: $surface;
  border: 1px solid $line;
  border-radius: $radius-sm;
  padding: 0 $gap-md;
  font-size: $font-md;
  color: $ink;
  box-sizing: border-box;

  &--flex {
    flex: 1;
  }
}

.form-error {
  color: #ef4444;
  font-size: $font-xs;
  margin-bottom: $gap-sm;
}

.btn-code {
  height: 80rpx;
  padding: 0 $gap-md;
  background: $blush;
  border: none;
  border-radius: $radius-sm;
  font-size: $font-xs;
  color: $rose-dark;
  font-weight: 600;
  white-space: nowrap;
  line-height: 80rpx;

  &[disabled] {
    opacity: 0.5;
  }
}

.login-btn {
  width: 100%;
  margin-top: $gap-sm;
}

.form-picker {
  width: 100%;
  height: 80rpx;
  background: $surface;
  border: 1px solid $line;
  border-radius: $radius-sm;
  padding: 0 $gap-md;
  font-size: $font-md;
  color: $ink;
  display: flex;
  align-items: center;
  box-sizing: border-box;

  &--placeholder { color: $muted; }
}

.clinic-search { display: flex; height: 80rpx; align-items: center; gap: $gap-xs; padding: 0 $gap-md; border: 1px solid $line; border-radius: $radius-sm; background: $surface; }
.clinic-search:focus-within { border-color: $rose; background: $paper; }
.clinic-search__icon { color: $rose-dark; font-size: 32rpx; font-weight: 800; }
.clinic-search__input { min-width: 0; flex: 1; color: $ink; font-size: $font-sm; }
.clinic-search__clear { display: grid; width: 40rpx; height: 40rpx; place-items: center; border-radius: 50%; background: #e4e4e7; color: $muted; font-size: 28rpx; }
.clinic-hint { min-height: 38rpx; padding: 8rpx 4rpx 4rpx; color: $muted; font-size: 20rpx; }
.clinic-hint__error { color: #dc2626; }
.clinic-list { max-height: 390rpx; margin-top: $gap-xs; border: 1px solid $line; border-radius: $radius-md; background: $surface; }
.clinic-selected { display: flex; align-items: center; justify-content: space-between; gap: $gap-sm; margin: $gap-xs 0; padding: $gap-sm $gap-md; border: 2rpx solid $rose; border-radius: $radius-md; background: $blush; }
.clinic-selected__eyebrow,.clinic-selected__name,.clinic-selected__location { display: block; }.clinic-selected__eyebrow { color: $rose-dark; font-size: 18rpx; font-weight: 800; }.clinic-selected__name { color: $ink; font-size: $font-sm; font-weight: 800; }.clinic-selected__location { color: $muted; font-size: 20rpx; }.clinic-selected__clear { flex: 0 0 auto; color: $rose-dark; font-size: 20rpx; font-weight: 800; }
.clinic-option { display: flex; align-items: center; gap: $gap-sm; padding: $gap-md; border-bottom: 1px solid $line; background: $paper; }
.clinic-option:last-child { border-bottom: 0; }
.clinic-option--active { background: $blush; box-shadow: inset 5rpx 0 0 $rose-dark; }
.clinic-option__mark { display: grid; width: 52rpx; height: 52rpx; flex: 0 0 auto; place-items: center; border-radius: 50%; background: $blush; color: $rose-dark; font-size: $font-sm; font-weight: 800; }
.clinic-option--active .clinic-option__mark { background: $rose-dark; color: #fff; }
.clinic-option__body { min-width: 0; flex: 1; }
.clinic-option__name,.clinic-option__location,.clinic-option__address { display: block; }
.clinic-option__name { color: $ink; font-size: $font-sm; font-weight: 800; }
.clinic-option__location { margin-top: 2rpx; color: $rose-dark; font-size: 20rpx; }
.clinic-option__address { overflow: hidden; margin-top: 2rpx; color: $muted; font-size: 20rpx; text-overflow: ellipsis; white-space: nowrap; }
.clinic-empty { display: flex; align-items: center; flex-direction: column; gap: 4rpx; margin-top: $gap-xs; padding: $gap-lg; border-radius: $radius-md; background: $surface; color: $ink; font-size: $font-sm; text-align: center; }
.clinic-empty text:last-child { color: $muted; font-size: 20rpx; }

.login-switch {
  text-align: center;
  margin-top: $gap-lg;
  font-size: $font-sm;
  color: $muted;
}
</style>
