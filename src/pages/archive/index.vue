<template>
  <view class="page archive-page">
    <ClinicTopBar show-back />
    <text class="archive-title">健康档案</text>
    <text class="archive-desc">您和医生共同维护的记录，双方都能看到</text>

    <!-- 新增记录 -->
    <view class="archive-form card">
      <textarea v-model="content" class="archive-form__textarea" maxlength="2000" placeholder="写下想告诉医生的内容，或记录今天的身体感受…" />
      <view class="archive-form__images">
        <view v-for="(file, i) in attachments" :key="file.url" class="archive-form__thumb" @click="preview(file.url)">
          <image :src="absoluteUrl(file.url)" mode="aspectFill" />
          <text class="archive-form__thumb-del" @click.stop="attachments = attachments.filter((_, idx) => idx !== i)">×</text>
        </view>
        <view v-if="attachments.length < 3" class="archive-form__add" @click="chooseImages">
          <text class="archive-form__add-icon">+</text>
          <text>图片</text>
        </view>
      </view>
      <button class="archive-form__submit" :loading="submitting" :disabled="!canSubmit" @click="saveRecord">保存记录</button>
    </view>

    <!-- 记录列表 -->
    <view v-if="loading" class="archive-loading">
      <text>加载中…</text>
    </view>
    <template v-else-if="records.length">
      <view class="archive-list">
        <view v-for="item in records" :key="item.id" class="archive-item card">
          <view class="archive-item__head">
            <view class="archive-item__author" :class="{ 'is-patient': item.authorRole === 'PATIENT' }">
              <text class="archive-item__avatar">{{ item.authorRole === 'PATIENT' ? '我' : '医' }}</text>
              <view>
                <text class="archive-item__name">{{ item.authorRole === 'PATIENT' ? '我' : (item.authorName || '医生') }}</text>
                <text class="archive-item__role">{{ item.authorRole === 'PATIENT' ? '患者' : '医生' }}</text>
              </view>
            </view>
            <view class="archive-item__meta">
              <text class="archive-item__time">{{ formatDate(item.createdAt) }}</text>
              <text v-if="item.authorRole === 'PATIENT'" class="archive-item__delete" @click="removeRecord(item.id)">删除</text>
            </view>
          </view>
          <text v-if="item.content" class="archive-item__content">{{ item.content }}</text>
          <view v-if="imageList(item.attachments).length" class="archive-item__images">
            <image v-for="url in imageList(item.attachments)" :key="url" class="archive-item__img" :src="absoluteUrl(url)" mode="aspectFill" @click="preview(url)" />
          </view>
        </view>
      </view>
    </template>
    <view v-else class="archive-empty">
      <text class="archive-empty__icon">📋</text>
      <text>还没有任何记录</text>
      <text class="archive-empty__sub">写下第一条记录，和医生一起管理您的健康。</text>
    </view>

    <!-- 删除二次确认 -->
    <view v-if="deleteTarget" class="archive-overlay" @click="cancelRemove">
      <view class="archive-dialog" @click.stop>
        <text class="archive-dialog__icon">×</text>
        <text class="archive-dialog__title">删除这条健康档案？</text>
        <text class="archive-dialog__copy">删除后无法恢复，医生也将无法再查看这条记录。</text>
        <view class="archive-dialog__actions">
          <button class="archive-dialog__quiet" @click="cancelRemove">暂不删除</button>
          <button class="archive-dialog__danger" :loading="removing" @click="confirmRemove">确认删除</button>
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

function absoluteUrl(path: string): string {
  return absoluteApiUrl(path);
}

function formatDate(iso?: string): string {
  return iso?.slice(0, 10) || "—";
}

function imageList(raw: unknown): string[] {
  if (Array.isArray(raw)) return raw.map((item: any) => typeof item === "string" ? item : item.url);
  try { return JSON.parse(raw as string || "[]"); } catch { return []; }
}

function preview(url: string) {
  uni.previewImage({ urls: [absoluteApiUrl(url)] });
}

function chooseImages() {
  uni.chooseImage({
    count: 3 - attachments.value.length,
    success: async (res: any) => {
      for (const path of res.tempFilePaths) {
        try {
          const uploaded = await uploadFile(path);
          attachments.value = [...attachments.value, uploaded];
        } catch (e: any) {
          app.showToast(e?.message || "图片上传失败", "error");
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
    app.showToast("记录已保存", "success");
  } catch (e: any) {
    app.showToast(e?.message || "保存失败", "error");
  } finally {
    submitting.value = false;
  }
}

function removeRecord(id: string) {
  deleteTarget.value = id;
}

function cancelRemove() {
  if (removing.value) return;
  deleteTarget.value = null;
}

async function confirmRemove() {
  if (!deleteTarget.value || removing.value) return;
  removing.value = true;
  const id = deleteTarget.value;
  try {
    await menstrualApi.deleteRecord(id);
    records.value = records.value.filter((r) => r.id !== id);
    deleteTarget.value = null;
    app.showToast("记录已删除", "success");
  } catch (e: any) {
    app.showToast(e?.message || "删除失败", "error");
  } finally {
    removing.value = false;
  }
}

async function loadRecords() {
  try {
    const res = await menstrualApi.fetchRecords();
    records.value = res.records ?? [];
  } catch (e: any) {
    app.showToast(e?.message || "加载失败", "error");
  } finally {
    loading.value = false;
  }
}

onMounted(() => {
  loadRecords();
});
</script>

<style lang="scss" scoped>
.archive-page { min-height: 100vh; padding: 0 $gap-md 80rpx; }
.archive-title { display: block; padding-top: $gap-md; color: $ink; font-size: $font-xl; font-weight: 800; }
.archive-desc { display: block; margin-top: 4rpx; color: $muted; font-size: $font-xs; }

.archive-form { padding: $gap-lg; margin-top: $gap-lg; }
.archive-form__textarea { width: 100%; min-height: 180rpx; box-sizing: border-box; margin-bottom: $gap-sm; padding: $gap-md; border: 1px solid $line; border-radius: $radius-md; background: $surface; color: $ink; font-size: $font-sm; line-height: 1.7; }
.archive-form__images { display: flex; flex-wrap: wrap; gap: 12rpx; margin-bottom: $gap-md; }
.archive-form__thumb { position: relative; width: 144rpx; height: 144rpx; border-radius: $radius-md; overflow: hidden; }
.archive-form__thumb image { width: 100%; height: 100%; }
.archive-form__thumb-del { position: absolute; top: 0; right: 0; display: grid; width: 40rpx; height: 40rpx; place-items: center; border-radius: 0 0 0 $radius-md; background: rgba(0,0,0,.55); color: #fff; font-size: 28rpx; line-height: 1; }
.archive-form__add { display: flex; width: 144rpx; height: 144rpx; flex-direction: column; align-items: center; justify-content: center; gap: 2rpx; border: 2rpx dashed #e4e4e7; border-radius: $radius-md; color: $rose-dark; }
.archive-form__add-icon { font-size: 36rpx; font-weight: 700; }
.archive-form__add text:last-child { font-size: 20rpx; }
.archive-form__submit { width: 100%; border-radius: $radius-full; background: $ink; color: #fff; font-size: $font-sm; font-weight: 700; }
.archive-form__submit[disabled] { background: #e4e4e7; color: $muted; }

.archive-list { display: flex; flex-direction: column; gap: $gap-sm; margin-top: $gap-md; }
.archive-item { padding: $gap-lg; }
.archive-item__head { display: flex; align-items: center; justify-content: space-between; gap: $gap-sm; }
.archive-item__author { display: flex; align-items: center; gap: $gap-sm; }
.archive-item__author.is-patient .archive-item__avatar { background: $rose-dark; color: #fff; }
.archive-item__avatar { display: grid; width: 64rpx; height: 64rpx; flex: 0 0 auto; place-items: center; border-radius: 50%; background: #e4e4e7; color: $ink; font-size: 24rpx; font-weight: 800; }
.archive-item__name { display: block; color: $ink; font-size: $font-sm; font-weight: 700; }
.archive-item__role { display: block; color: $muted; font-size: 20rpx; }
.archive-item__meta { display: flex; flex-direction: column; align-items: flex-end; gap: 2rpx; }
.archive-item__time { color: $muted; font-size: 20rpx; }
.archive-item__delete { color: #e11d48; font-size: 20rpx; }
.archive-item__content { display: block; margin-top: $gap-md; color: $ink; font-size: $font-sm; line-height: 1.8; white-space: pre-wrap; }
.archive-item__images { display: flex; flex-wrap: wrap; gap: 12rpx; margin-top: $gap-md; }
.archive-item__img { width: 160rpx; height: 160rpx; border-radius: $radius-md; background: $surface; }

.archive-empty { display: flex; flex-direction: column; align-items: center; padding: 80rpx $gap-lg; margin-top: $gap-md; border-radius: $radius-lg; background: $paper; box-shadow: $shadow-sm; color: $muted; font-size: $font-sm; text-align: center; }
.archive-empty__icon { margin-bottom: $gap-sm; font-size: 48rpx; }
.archive-empty__sub { margin-top: $gap-xs; color: $muted; font-size: $font-xs; }
.archive-loading { padding: 80rpx 0; text-align: center; color: $muted; font-size: $font-sm; }

.archive-overlay { position: fixed; inset: 0; z-index: 40; display: flex; align-items: center; justify-content: center; padding: $gap-lg; background: rgba(39, 39, 42, .46); backdrop-filter: blur(8rpx); }
.archive-dialog { display: flex; width: 100%; max-width: 560rpx; flex-direction: column; align-items: center; padding: $gap-xl $gap-lg $gap-lg; border-radius: $radius-lg; background: $paper; box-shadow: $shadow-lg; }
.archive-dialog__icon { display: grid; width: 96rpx; height: 96rpx; flex: 0 0 auto; place-items: center; border-radius: 50%; background: $blush; color: $rose-dark; font-size: 56rpx; line-height: 1; }
.archive-dialog__title { display: block; margin-top: $gap-md; color: $ink; font-size: $font-lg; font-weight: 800; text-align: center; }
.archive-dialog__copy { display: block; margin-top: $gap-sm; color: $muted; font-size: $font-sm; line-height: 1.7; text-align: center; }
.archive-dialog__actions { display: flex; width: 100%; gap: $gap-sm; margin-top: $gap-lg; }
.archive-dialog__quiet, .archive-dialog__danger { flex: 1; height: 88rpx; line-height: 88rpx; border-radius: $radius-full; font-size: $font-sm; font-weight: 700; }
.archive-dialog__quiet { background: $surface; color: $ink; }
.archive-dialog__danger { background: #e11d48; color: #fff; }
</style>
