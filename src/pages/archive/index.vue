<template>
  <view class="page archive-page">
    <ClinicTopBar show-back />
    <view class="archive-header">
      <text class="archive-title">临床健康档案</text>
      <text class="archive-desc">供医患双方共同阅览的临床记录资料库</text>
    </view>

    <!-- 新增记录表单 -->
    <view class="archive-form card">
      <text class="form-label">补充病史或临床表现</text>
      <textarea v-model="content" class="form-textarea" maxlength="2000" placeholder="请输入详实的病史、症状描述或化验结果说明..." />
      
      <text class="form-label">附件影像资料 (最多3张)</text>
      <view class="image-grid">
        <view v-for="(file, i) in attachments" :key="file.url" class="image-thumb" @click="preview(file.url)">
          <image :src="absoluteUrl(file.url)" mode="aspectFill" />
          <view class="image-del" @click.stop="attachments.splice(i, 1)">×</view>
        </view>
        <view v-if="attachments.length < 3" class="image-add" @click="chooseImages">
          <text class="image-add__icon">+</text>
          <text class="image-add__text">上传资料</text>
        </view>
      </view>
      
      <button class="btn-primary submit-btn" :loading="submitting" :disabled="!canSubmit" @click="saveRecord">提交归档</button>
    </view>

    <!-- 记录列表 -->
    <view class="section-title">档案记录明细</view>
    <view v-if="loading" class="empty-state">数据拉取中...</view>
    <template v-else-if="records.length">
      <view class="record-list">
        <view v-for="item in records" :key="item.id" class="record-card card">
          <view class="record-card__head">
            <view class="record-author">
              <text class="author-badge" :class="item.authorRole === 'PATIENT' ? 'author-badge--patient' : 'author-badge--doctor'">
                {{ item.authorRole === 'PATIENT' ? '患者' : '医师' }}
              </text>
              <text class="author-name">{{ item.authorRole === 'PATIENT' ? '我' : (item.authorName || '主治医师') }}</text>
            </view>
            <text class="record-time">{{ formatDate(item.createdAt) }}</text>
          </view>
          
          <text v-if="item.content" class="record-content">{{ item.content }}</text>
          
          <view v-if="imageList(item.attachments).length" class="record-images">
            <image v-for="url in imageList(item.attachments)" :key="url" class="record-image" :src="absoluteUrl(url)" mode="aspectFill" @click="preview(url)" />
          </view>
          
          <view class="record-card__footer" v-if="item.authorRole === 'PATIENT'">
            <text class="btn-text-danger" @click="removeRecord(item.id)">撤销此记录</text>
          </view>
        </view>
      </view>
    </template>
    <view v-else class="empty-state">
      <text>数据库中暂无存档资料。</text>
    </view>

    <!-- 删除二次确认 -->
    <view v-if="deleteTarget" class="overlay" @click="cancelRemove">
      <view class="dialog" @click.stop>
        <view class="dialog-header">
          <text class="dialog-title">警告：撤销临床记录</text>
        </view>
        <view class="dialog-body">
          <text>确认将此条记录移出临床档案库？操作不可逆。</text>
        </view>
        <view class="dialog-footer">
          <button class="btn-secondary flex-1" @click="cancelRemove">取消操作</button>
          <button class="btn-danger flex-1" :loading="removing" @click="confirmRemove">确认撤销</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import * as menstrualApi from "@/api/menstrual";
import { absoluteApiUrl, uploadFile } from "@/api/client";
import { useAppStore } from "@/stores/useAppStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

const app = useAppStore();
const records = ref<any[]>([]);
const content = ref("");
const attachments = ref<Array<{ url: string; name: string; type: string; size: number }>>([]);
const submitting = ref(false);
const loading = ref(true);
const deleteTarget = ref<string | null>(null);
const removing = ref(false);

const canSubmit = computed(() => content.value.trim() || attachments.value.length > 0);

function absoluteUrl(path: string): string { return absoluteApiUrl(path); }
function formatDate(iso?: string): string { return iso?.slice(0, 16).replace("T", " ") || "—"; }

function imageList(raw: unknown): string[] {
  if (Array.isArray(raw)) return raw.map((item: any) => typeof item === "string" ? item : item.url);
  try { return JSON.parse(raw as string || "[]"); } catch { return []; }
}

function preview(url: string) { uni.previewImage({ urls: [absoluteApiUrl(url)] }); }

function chooseImages() {
  uni.chooseImage({
    count: 3 - attachments.value.length,
    success: async (res: any) => {
      for (const path of res.tempFilePaths) {
        try {
          const uploaded = await uploadFile(path);
          attachments.value = [...attachments.value, uploaded];
        } catch (e: any) {
          app.showToast(e?.message || "影像上传错误", "error");
        }
      }
    },
  });
}

async function saveRecord() {
  if (!canSubmit.value) return;
  submitting.value = true;
  try {
    await menstrualApi.createRecord({
      content: content.value.trim() || undefined,
      attachments: attachments.value.length ? attachments.value : undefined,
    });
    content.value = "";
    attachments.value = [];
    await loadRecords();
    app.showToast("资料已归档", "success");
  } catch (e: any) {
    app.showToast(e?.message || "网络存储异常", "error");
  } finally {
    submitting.value = false;
  }
}

function removeRecord(id: string) { deleteTarget.value = id; }
function cancelRemove() { if (removing.value) return; deleteTarget.value = null; }

async function confirmRemove() {
  if (!deleteTarget.value || removing.value) return;
  removing.value = true;
  const id = deleteTarget.value;
  try {
    await menstrualApi.deleteRecord(id);
    records.value = records.value.filter((r) => r.id !== id);
    deleteTarget.value = null;
    app.showToast("档案已撤销", "success");
  } catch (e: any) {
    app.showToast(e?.message || "服务器操作失败", "error");
  } finally {
    removing.value = false;
  }
}

async function loadRecords() {
  try {
    const res = await menstrualApi.fetchRecords();
    records.value = res.records ?? [];
  } catch (e: any) {
    app.showToast(e?.message || "同步失败", "error");
  } finally {
    loading.value = false;
  }
}

onMounted(() => { loadRecords(); });
</script>

<style lang="scss" scoped>
.archive-page { min-height: 100vh; padding: $gap-md $gap-md 80rpx; }
.archive-header { padding: $gap-sm 0 $gap-md; }
.archive-title { display: block; font-size: $font-xl; font-weight: 700; color: $ink; margin-bottom: 8rpx;}
.archive-desc { font-size: $font-sm; color: $muted; }

.form-label { display: block; font-size: 24rpx; font-weight: 600; color: $ink; margin-bottom: 12rpx; margin-top: $gap-sm;}
.form-label:first-child { margin-top: 0; }
.form-textarea { width: 100%; min-height: 200rpx; padding: 20rpx; background: $surface; border: 1px solid $line; border-radius: $radius-sm; font-size: $font-sm; color: $ink; box-sizing: border-box; }

.image-grid { display: flex; flex-wrap: wrap; gap: 16rpx; margin-bottom: $gap-md; }
.image-thumb { position: relative; width: 160rpx; height: 160rpx; border-radius: $radius-sm; overflow: hidden; border: 1px solid $line;}
.image-thumb image { width: 100%; height: 100%; }
.image-del { position: absolute; top: 0; right: 0; width: 40rpx; height: 40rpx; background: rgba(0,0,0,0.6); color: #fff; display: flex; align-items: center; justify-content: center; font-size: 32rpx; border-bottom-left-radius: 8rpx;}
.image-add { width: 160rpx; height: 160rpx; display: flex; flex-direction: column; align-items: center; justify-content: center; background: $surface; border: 1px dashed $muted; border-radius: $radius-sm; color: $muted; }
.image-add__icon { font-size: 40rpx; margin-bottom: 4rpx; }
.image-add__text { font-size: 20rpx; }

.submit-btn { width: 100%; }

.section-title { font-size: $font-md; font-weight: 700; color: $ink; margin: $gap-lg 0 $gap-sm; }

.record-list { display: flex; flex-direction: column; gap: $gap-md; }
.record-card { padding: $gap-lg; }
.record-card__head { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16rpx; padding-bottom: 16rpx; border-bottom: 1px solid $surface;}
.record-author { display: flex; align-items: center; gap: 12rpx; }
.author-badge { padding: 4rpx 12rpx; border-radius: 4rpx; font-size: 20rpx; font-weight: bold; }
.author-badge--patient { background: $blush; color: $rose-dark; }
.author-badge--doctor { background: #e0f2fe; color: #0284c7; }
.author-name { font-size: $font-sm; font-weight: 700; color: $ink; }
.record-time { font-size: 22rpx; color: $muted; }

.record-content { display: block; font-size: $font-sm; color: $ink; line-height: 1.6; white-space: pre-wrap; }
.record-images { display: flex; flex-wrap: wrap; gap: 12rpx; margin-top: $gap-md; }
.record-image { width: 140rpx; height: 140rpx; border-radius: $radius-sm; border: 1px solid $line;}

.record-card__footer { margin-top: $gap-md; text-align: right; }
.btn-text-danger { font-size: 24rpx; color: #ef4444; font-weight: 600; padding: 8rpx; }

.empty-state { padding: $gap-xl; text-align: center; background: $paper; border-radius: $radius-md; border: 1px dashed $line; color: $muted; font-size: $font-sm; }

.overlay { position: fixed; inset: 0; z-index: 50; background: rgba(15, 23, 42, 0.4); display: flex; align-items: center; justify-content: center; padding: $gap-xl; }
.dialog { width: 100%; background: $paper; border-radius: $radius-md; overflow: hidden; box-shadow: $shadow-lg; }
.dialog-header { padding: $gap-md $gap-lg; border-bottom: 1px solid $line; background: $surface; }
.dialog-title { font-size: $font-md; font-weight: 700; color: #ef4444; }
.dialog-body { padding: $gap-lg; font-size: $font-sm; color: $ink; line-height: 1.6; }
.dialog-footer { display: flex; padding: $gap-md $gap-lg; gap: $gap-md; border-top: 1px solid $line; }
.btn-danger { background: #ef4444; color: #fff; border: none; border-radius: $radius-md; font-size: $font-md; font-weight: 500; }
.flex-1 { flex: 1; }
</style>
