<template>
  <view class="login-page">
    <view class="login-card card">
      <view class="login-header">
        <view class="login-logo">✚</view>
        <text class="login-title">临床健康管理工作站</text>
        <text class="login-sub">请输入凭据以访问系统</text>
      </view>

      <!-- 登录表单 -->
      <view class="login-form" v-if="!isRegistering">
        <view class="form-group">
          <text class="form-label">账号 (手机号)</text>
          <input
            class="form-input"
            v-model="phone"
            type="number"
            maxlength="11"
            placeholder="请输入您的手机号"
          />
        </view>
        <view class="form-group">
          <text class="form-label">安全密码</text>
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
          系统登录
        </button>
        <view class="login-switch" @click="isRegistering = true">
          首次使用？<text class="text-rose">注册建档</text>
        </view>
      </view>

      <!-- 注册表单 -->
      <view class="login-form" v-else>
        <view class="form-group">
          <text class="form-label">账号 (手机号)</text>
          <input class="form-input" v-model="phone" type="number" maxlength="11" placeholder="请输入手机号" />
        </view>
        <view class="form-group form-group--row">
          <input class="form-input form-input--flex" v-model="code" type="number" maxlength="6" placeholder="验证码" />
          <button class="btn-code" :disabled="codeSending" @click="handleRequestCode">
            {{ codeSending ? `${countdown}s` : "获取验证码" }}
          </button>
        </view>
        <view class="form-group">
          <text class="form-label">患者/医生姓名</text>
          <input class="form-input" v-model="name" placeholder="请输入真实姓名" />
        </view>
        <view class="form-group">
          <text class="form-label">年龄 <text class="text-muted" style="font-size:20rpx">（选填）</text></text>
          <input class="form-input" v-model="age" type="number" placeholder="请输入年龄" />
        </view>
        <view class="form-group">
          <text class="form-label">所属机构/诊所 <text class="text-muted" style="font-size:20rpx">（必选）</text></text>
          <view class="clinic-search">
            <text class="clinic-search__icon">🔍</text>
            <input
              v-model="clinicQuery"
              class="clinic-search__input"
              confirm-type="search"
              placeholder="搜索机构名称或地址..."
              @input="scheduleClinicSearch"
              @confirm="loadClinics"
            />
            <text v-if="clinicQuery" class="clinic-search__clear" @click="clearClinicSearch">×</text>
          </view>
          <view class="clinic-hint">
            <text v-if="clinicLoading">正在检索机构数据库...</text>
            <text v-else-if="clinicError" class="clinic-hint__error" @click="loadClinics">{{ clinicError }}，点击重试</text>
            <text v-else>{{ clinicQuery ? `找到 ${clinics.length} 家匹配机构` : `共 ${clinics.length} 家可选机构` }}</text>
          </view>
          <view v-if="selectedClinic" class="clinic-selected">
            <view>
              <text class="clinic-selected__eyebrow">当前选择</text>
              <text class="clinic-selected__name">{{ selectedClinic.name }}</text>
              <text class="clinic-selected__location">{{ clinicLocation(selectedClinic) }}</text>
            </view>
            <text class="clinic-selected__clear" @click="clearSelectedClinic">更改</text>
          </view>
          <scroll-view v-if="!clinicLoading && clinics.length" class="clinic-list" scroll-y>
            <view
              v-for="clinic in clinics"
              :key="clinic.id"
              class="clinic-option"
              :class="{ 'clinic-option--active': selectedClinicId === clinic.id }"
              @click="selectClinic(clinic)"
            >
              <view class="clinic-option__body">
                <text class="clinic-option__name">{{ clinic.name }}</text>
                <text class="clinic-option__location">{{ clinicLocation(clinic) }}</text>
                <text v-if="clinic.address" class="clinic-option__address">{{ clinic.address }}</text>
              </view>
              <view v-if="selectedClinicId === clinic.id" class="clinic-option__mark">✓</view>
            </view>
          </scroll-view>
          <view v-else-if="!clinicLoading && !clinicError" class="clinic-empty">
            <text>未匹配到相关机构</text>
            <text>请尝试输入更精确的信息</text>
          </view>
        </view>
        <view class="form-group">
          <text class="form-label">设置密码</text>
          <input class="form-input" v-model="password" type="password" placeholder="设置安全密码" />
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
          提交注册
        </button>
        <view class="login-switch" @click="isRegistering = false">
          已有通行证？<text class="text-rose">返回登录</text>
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
  background: $surface;
}

.login-card {
  width: 100%;
  max-width: 600rpx;
  padding: $gap-xl;
  box-shadow: $shadow-lg;
}

.login-header {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 48rpx;
}

.login-logo {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 96rpx;
  height: 96rpx;
  background: $rose-dark;
  color: #fff;
  border-radius: 16rpx;
  font-size: 56rpx;
  font-weight: bold;
  margin-bottom: $gap-md;
}

.login-title {
  font-size: $font-xl;
  font-weight: 700;
  color: $ink;
  margin-bottom: 8rpx;
}

.login-sub {
  font-size: $font-sm;
  color: $muted;
}

.form-group {
  margin-bottom: $gap-md;
  &--row { display: flex; gap: $gap-sm; }
}

.form-label {
  font-size: 24rpx;
  font-weight: 600;
  color: $ink;
  margin-bottom: 12rpx;
  display: block;
}

.form-input {
  width: 100%;
  height: 80rpx;
  background: $paper;
  border: 1px solid $line;
  border-radius: $radius-sm;
  padding: 0 $gap-md;
  font-size: $font-md;
  color: $ink;
  box-sizing: border-box;
  transition: border-color 0.2s;
  
  &:focus {
    border-color: $rose-dark;
  }

  &--flex { flex: 1; }
}

.form-error {
  color: #ef4444;
  font-size: $font-xs;
  margin-bottom: $gap-sm;
}

.btn-code {
  height: 80rpx;
  padding: 0 $gap-md;
  background: $surface;
  border: 1px solid $line;
  border-radius: $radius-sm;
  font-size: $font-sm;
  color: $ink;
  font-weight: 500;
  line-height: 76rpx;
  &[disabled] { opacity: 0.5; }
}

.login-btn { width: 100%; margin-top: $gap-md; }

.clinic-search { display: flex; height: 80rpx; align-items: center; gap: $gap-xs; padding: 0 $gap-md; border: 1px solid $line; border-radius: $radius-sm; background: $paper; }
.clinic-search__icon { color: $muted; font-size: 28rpx; }
.clinic-search__input { min-width: 0; flex: 1; color: $ink; font-size: $font-sm; }
.clinic-search__clear { display: grid; width: 40rpx; height: 40rpx; place-items: center; border-radius: 50%; background: $line; color: $muted; font-size: 28rpx; }
.clinic-hint { min-height: 38rpx; padding: 8rpx 0; color: $muted; font-size: 22rpx; }
.clinic-hint__error { color: #dc2626; }
.clinic-list { max-height: 360rpx; margin-top: $gap-xs; border: 1px solid $line; border-radius: $radius-sm; background: $paper; }
.clinic-selected { display: flex; align-items: center; justify-content: space-between; gap: $gap-sm; margin: $gap-xs 0; padding: $gap-sm $gap-md; border: 1px solid $rose-dark; border-radius: $radius-sm; background: $blush; }
.clinic-selected__eyebrow { color: $rose-dark; font-size: 20rpx; font-weight: 600; display: block;}
.clinic-selected__name { color: $ink; font-size: $font-sm; font-weight: 600; display: block;}
.clinic-selected__location { color: $muted; font-size: 22rpx; display: block;}
.clinic-selected__clear { flex: 0 0 auto; color: $rose-dark; font-size: 24rpx; font-weight: 600; }
.clinic-option { display: flex; align-items: center; justify-content: space-between; gap: $gap-sm; padding: $gap-md; border-bottom: 1px solid $line; }
.clinic-option:last-child { border-bottom: 0; }
.clinic-option--active { background: $surface; }
.clinic-option__body { min-width: 0; flex: 1; }
.clinic-option__name { color: $ink; font-size: $font-sm; font-weight: 600; display: block;}
.clinic-option__location { margin-top: 4rpx; color: $muted; font-size: 22rpx; display: block;}
.clinic-option__address { overflow: hidden; margin-top: 4rpx; color: $muted; font-size: 22rpx; text-overflow: ellipsis; white-space: nowrap; display: block;}
.clinic-option__mark { color: $rose-dark; font-size: 32rpx; font-weight: bold; }
.clinic-empty { padding: $gap-lg; border: 1px dashed $line; border-radius: $radius-sm; background: $surface; color: $muted; font-size: $font-sm; text-align: center; }

.login-switch {
  text-align: center;
  margin-top: $gap-lg;
  font-size: $font-sm;
  color: $muted;
}
</style>
