<template>
  <view class="page">
    <ClinicTopBar show-back />
    <view class="header-section">
      <text class="service-title">{{ title }}</text>
    </view>
    
    <view v-if="loading" class="empty-state">系统数据同步中...</view>
    <view v-else-if="error" class="empty-state error-text" @click="load">{{ error }}，点击重试</view>

    <!-- 消息与通知 -->
    <template v-else-if="type === 'notifications'">
      <view class="actions-bar" v-if="items.some((item) => !item.read)">
        <text class="btn-text" @click="readAll">全部标记已读</text>
      </view>
      <view class="list-container">
        <view v-for="item in items" :key="item.id" class="list-item card" :class="{ 'is-unread': !item.read }" @click="readMessage(item)">
          <view class="list-item__head">
            <text class="item-title">{{ item.title || "系统通告" }}</text>
            <text class="item-meta">{{ day(item.createdAt) }}</text>
          </view>
          <text class="item-desc">{{ item.content }}</text>
        </view>
      </view>
    </template>

    <!-- 推广与邀请 -->
    <template v-else-if="type === 'invite'">
      <view class="card info-card">
        <text class="info-card__label">转介服务标识码 (ID)</text>
        <text class="info-card__value">{{ inviteInfo?.inviteCode || dashboard?.inviteCode || "暂未分配" }}</text>
        <button class="btn-secondary btn-sm mt-md" @click="copy(inviteInfo?.inviteCode || dashboard?.inviteCode || '')">复制标识码</button>
      </view>
      
      <image v-if="inviteInfo?.clinicSharePoster?.imageBase64" class="poster-image" :src="inviteInfo.clinicSharePoster.imageBase64" mode="widthFix" show-menu-by-longpress />
      <view v-else-if="inviteInfo?.qrDataUrl" class="card qr-card">
        <text class="qr-card__title">{{ inviteInfo?.clinic?.name || "临床机构" }} 专属入口</text>
        <image class="qr-card__img" :src="inviteInfo.qrDataUrl" mode="aspectFit" show-menu-by-longpress />
        <text class="qr-card__code">ID: {{ inviteInfo?.inviteCode || dashboard?.inviteCode || "—" }}</text>
        <text class="qr-card__hint">此二维码含加密标识，长按保存可发送给其他需建档的患者。</text>
      </view>
      
      <view class="card form-card">
        <text class="form-title">健康随访计划下发</text>
        <picker :range="campaigns.map((c:any) => c.title)" @change="referral.campaignId = campaigns[Number($event.detail.value)]?.id || ''">
          <view class="form-input">{{ campaignName || "请选择医学干预/宣教计划" }}</view>
        </picker>
        <input v-model="referral.name" class="form-input" placeholder="目标对象姓名" />
        <input v-model="referral.phone" class="form-input" type="number" maxlength="11" placeholder="目标对象联系电话" />
        <button class="btn-primary" :loading="submitting" @click="submitReferral">执行计划下发</button>
      </view>
    </template>

    <!-- 健康档案 (旧逻辑入口兼容) -->
    <template v-else-if="type === 'records'">
      <view class="card form-card">
        <text class="form-title">归档新数据</text>
        <textarea v-model="content" class="form-textarea" maxlength="2000" placeholder="临床表现、查体结果..." />
        <view class="image-grid">
          <view v-for="(file,i) in attachments" :key="file.url" class="image-thumb">
            <image :src="absolute(file.url)" mode="aspectFill" @click="preview(file.url)" />
            <view class="image-del" @click="attachments.splice(i,1)">×</view>
          </view>
          <view v-if="attachments.length < 3" class="image-add" @click="chooseImages">
            <text class="image-add__icon">+</text>
          </view>
        </view>
        <button class="btn-primary" :loading="submitting" @click="saveRecord">存入数据库</button>
      </view>
      <view class="list-container">
        <view v-for="item in items" :key="item.id" class="list-item card">
          <view class="list-item__head">
            <text class="item-title">{{ item.authorRole === "PATIENT" ? "自述记录" : "主治医师记录" }}</text>
            <text class="item-meta">{{ day(item.createdAt) }}</text>
          </view>
          <text class="item-desc">{{ item.content }}</text>
          <view class="record-images" v-if="attachmentList(item.attachments).length">
            <image v-for="url in attachmentList(item.attachments)" :key="url" class="record-image" :src="absolute(url)" mode="aspectFill" @click="preview(url)" />
          </view>
          <view class="list-item__footer" v-if="item.authorRole === 'PATIENT'">
            <text class="btn-text-danger" @click="removeRecord(item.id)">撤销</text>
          </view>
        </view>
      </view>
    </template>

    <!-- 医患共享档案 -->
    <template v-else-if="type === 'doctorRecords'">
      <view class="list-container mt-md">
        <view v-for="item in items" :key="item.id" class="list-item card">
          <view class="list-item__head">
            <text class="item-title" :class="{ 'text-rose': item.authorRole === 'PATIENT' }">
              {{ item.authorRole === "PATIENT" ? "患者自述" : (item.authorName || "医师诊断") }}
            </text>
            <text class="item-meta">{{ day(item.createdAt) }}</text>
          </view>
          <text v-if="item.content" class="item-desc">{{ item.content }}</text>
          <text v-else-if="attachmentList(item.attachments).length" class="item-desc text-muted italic">附加影像资料</text>
          <view class="record-images" v-if="attachmentList(item.attachments).length">
            <image v-for="url in attachmentList(item.attachments)" :key="url" class="record-image" :src="absolute(url)" mode="aspectFill" @click="preview(url)" />
          </view>
        </view>
        <view v-if="!items.length" class="empty-state">目前没有医患共享的医疗记录。</view>
      </view>
    </template>

  </view>
</template>

<script setup lang="ts">
import { ref, computed } from "vue";
import { onLoad } from "@dcloudio/uni-app";
import { api, absoluteApiUrl, uploadFile } from "@/api/client";
import { useAppStore } from "@/stores/useAppStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

const app = useAppStore();
const type = ref("");
const title = ref("服务加载中...");
const loading = ref(true);
const error = ref("");
const items = ref<any[]>([]);

interface NotificationsResponse {
  notifications?: any[];
}

interface DashboardResponse {
  inviteCode?: string;
  campaigns?: any[];
}

interface RecordsResponse {
  records?: any[];
}

// 通知
async function loadNotifications() {
  const res = await api<NotificationsResponse>("/api/patient/notifications");
  items.value = res.notifications || [];
  app.loadDashboard(true);
}
async function readMessage(item: any) {
  if (item.read) return;
  await api(`/api/patient/notifications/${item.id}/read`, { method: "POST" });
  item.read = true;
  app.loadDashboard(true);
}
async function readAll() {
  await api("/api/patient/notifications/read-all", { method: "POST" });
  items.value.forEach(i => i.read = true);
  app.loadDashboard(true);
}

// 推广与邀请
const inviteInfo = ref<any>(null);
const dashboard = ref<any>(null);
const campaigns = ref<any[]>([]);
const referral = ref({ campaignId: "", name: "", phone: "" });
const submitting = ref(false);
const campaignName = computed(() => campaigns.value.find(c => c.id === referral.value.campaignId)?.title || "");

async function loadInvite() {
  try {
    inviteInfo.value = await api("/api/patient/invite-info");
  } catch (e: any) {
    if (e?.statusCode !== 404) throw e;
  }
  const db = await api<DashboardResponse>("/api/patient/dashboard");
  dashboard.value = db;
  campaigns.value = db.campaigns || [];
}

async function submitReferral() {
  if (!referral.value.name || !referral.value.phone) return app.showToast("缺少必填信息", "error");
  submitting.value = true;
  try {
    await api("/api/patient/referrals", { method: "POST", body: referral.value });
    app.showToast("已成功录入系统", "success");
    referral.value = { campaignId: "", name: "", phone: "" };
  } catch (e: any) {
    app.showToast(e.message || "请求失败", "error");
  } finally {
    submitting.value = false;
  }
}

function copy(text: string) {
  uni.setClipboardData({ data: text, success: () => app.showToast("已复制", "success") });
}

// 记录相关
const content = ref("");
const attachments = ref<any[]>([]);

async function loadRecords() {
  const res = await api<RecordsResponse>("/api/patient/menstrual-records");
  items.value = res.records || [];
}
async function loadDoctorRecords() {
  const res = await api<RecordsResponse>("/api/patient/doctor-records");
  items.value = res.records || [];
}
function absolute(url: string) { return absoluteApiUrl(url); }
function attachmentList(raw: any) {
  if (Array.isArray(raw)) return raw.map(i => typeof i === "string" ? i : i.url);
  try { return JSON.parse(raw || "[]"); } catch { return []; }
}
function preview(url: string) { uni.previewImage({ urls: [absolute(url)] }); }

function chooseImages() {
  uni.chooseImage({
    count: 3 - attachments.value.length,
    success: async (res: any) => {
      for (const path of res.tempFilePaths) {
        try { attachments.value.push(await uploadFile(path)); }
        catch (e: any) { app.showToast(e.message || "上传失败", "error"); }
      }
    }
  });
}
async function saveRecord() {
  if (!content.value && !attachments.value.length) return;
  submitting.value = true;
  try {
    await api("/api/patient/menstrual-records", {
      method: "POST", body: { content: content.value, attachments: attachments.value }
    });
    content.value = ""; attachments.value = [];
    app.showToast("已归档", "success");
    await loadRecords();
  } catch (e: any) { app.showToast(e.message || "失败", "error"); }
  finally { submitting.value = false; }
}
async function removeRecord(id: string) {
  try {
    await api(`/api/patient/menstrual-records/${id}`, { method: "DELETE" });
    items.value = items.value.filter(i => i.id !== id);
    app.showToast("已撤销", "success");
  } catch (e: any) { app.showToast(e.message || "失败", "error"); }
}

function day(iso: string) { return iso ? iso.slice(0, 10) : ""; }

async function load() {
  loading.value = true;
  error.value = "";
  try {
    if (type.value === "notifications") await loadNotifications();
    else if (type.value === "invite") await loadInvite();
    else if (type.value === "records") await loadRecords();
    else if (type.value === "doctorRecords") await loadDoctorRecords();
  } catch (e: any) {
    error.value = e.message || "加载异常";
  } finally {
    loading.value = false;
  }
}

onLoad((options: any) => {
  type.value = options.type || "";
  const map: Record<string, string> = {
    notifications: "系统通知", invite: "转介与分享", records: "历史归档", doctorRecords: "医患档案"
  };
  title.value = map[type.value] || "医疗服务";
  load();
});
</script>

<style lang="scss" scoped>
.page { min-height: 100vh; padding: $gap-md; padding-bottom: 120rpx;}
.header-section { padding: $gap-md 0; }
.service-title { font-size: $font-xl; font-weight: 700; color: $ink; }
.empty-state { text-align: center; padding: 120rpx 40rpx; color: $muted; font-size: $font-sm; }
.error-text { color: #ef4444; }

.actions-bar { display: flex; justify-content: flex-end; margin-bottom: $gap-sm; }
.btn-text { font-size: 24rpx; color: $rose-dark; padding: 8rpx; font-weight: 600; }

.list-container { display: flex; flex-direction: column; gap: $gap-sm; }
.list-item { padding: $gap-md; }
.is-unread { border-left: 4px solid $rose-dark; }
.list-item__head { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8rpx; }
.item-title { font-size: $font-sm; font-weight: 600; color: $ink; }
.item-meta { font-size: 22rpx; color: $muted; }
.item-desc { display: block; font-size: 24rpx; color: $ink; line-height: 1.6; margin-top: 8rpx;}

.info-card { text-align: center; margin-bottom: $gap-md; }
.info-card__label { display: block; font-size: 24rpx; color: $muted; margin-bottom: 8rpx; }
.info-card__value { display: block; font-size: 40rpx; font-weight: 700; color: $rose-dark; letter-spacing: 2px;}

.qr-card { text-align: center; margin-bottom: $gap-md; }
.qr-card__title { display: block; font-size: $font-md; font-weight: 700; margin-bottom: $gap-md; color: $ink; }
.qr-card__img { width: 400rpx; height: 400rpx; margin: 0 auto $gap-md; }
.qr-card__code { display: block; font-size: 24rpx; color: $muted; font-family: monospace; }
.qr-card__hint { display: block; font-size: 22rpx; color: $muted; margin-top: $gap-sm; }

.poster-image { width: 100%; border-radius: $radius-md; margin-bottom: $gap-md; }

.form-card { margin-bottom: $gap-md; }
.form-title { display: block; font-size: $font-md; font-weight: 700; margin-bottom: $gap-md; color: $ink; }
.form-input { width: 100%; height: 80rpx; background: $surface; border: 1px solid $line; border-radius: $radius-sm; padding: 0 24rpx; display: flex; align-items: center; font-size: $font-sm; margin-bottom: $gap-md; color: $ink; box-sizing: border-box; }
.form-textarea { width: 100%; height: 160rpx; background: $surface; border: 1px solid $line; border-radius: $radius-sm; padding: 24rpx; font-size: $font-sm; margin-bottom: $gap-md; color: $ink; box-sizing: border-box; }

.image-grid { display: flex; flex-wrap: wrap; gap: 16rpx; margin-bottom: $gap-md; }
.image-thumb { position: relative; width: 140rpx; height: 140rpx; border-radius: $radius-sm; overflow: hidden; border: 1px solid $line; }
.image-thumb image { width: 100%; height: 100%; }
.image-del { position: absolute; top: 0; right: 0; width: 40rpx; height: 40rpx; background: rgba(0,0,0,0.5); color: #fff; display: flex; align-items: center; justify-content: center; font-size: 32rpx; }
.image-add { width: 140rpx; height: 140rpx; display: flex; align-items: center; justify-content: center; background: $surface; border: 1px dashed $muted; border-radius: $radius-sm; color: $muted; }
.image-add__icon { font-size: 40rpx; }

.record-images { display: flex; flex-wrap: wrap; gap: 12rpx; margin-top: $gap-md; }
.record-image { width: 120rpx; height: 120rpx; border-radius: $radius-sm; border: 1px solid $line; }

.list-item__footer { margin-top: $gap-md; text-align: right; }
.btn-text-danger { font-size: 24rpx; color: #ef4444; padding: 8rpx; }
.btn-sm { margin: 0; padding: 12rpx 24rpx; font-size: 24rpx;}
</style>
