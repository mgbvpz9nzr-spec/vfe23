<template>
  <view class="page chat-page">
    <ClinicTopBar />
    <view class="chat-titlebar">
      <text class="chat-titlebar__name">AI 辅助咨询系统</text>
    </view>

    <scroll-view
      class="chat-messages"
      scroll-y
      :scroll-into-view="scrollToId"
      :scroll-with-animation="true"
      :enhanced="true"
      :show-scrollbar="false"
      :bounces="true"
    >
      <view v-if="chatStore.isLoading" class="chat-loading">
        <text>正在建立安全连接并加载记录...</text>
      </view>

      <view v-else-if="!chatStore.hasMessages" class="chat-welcome">
        <text class="chat-welcome__title">在线咨询辅助</text>
        <text class="chat-welcome__text">
          欢迎使用AI健康咨询系统。请描述您的症状、疑惑或提供当天的体征数据，系统将进行初步的数据分析与分诊建议。
        </text>
      </view>

      <view
        v-for="message in chatStore.messages"
        :id="'msg-' + message.id"
        :key="message.id"
        class="chat-row"
        :class="message.role === 'user' ? 'chat-row--user' : 'chat-row--assistant'"
      >
        <view
          class="chat-bubble"
          :class="[
            message.role === 'user' ? 'chat-bubble--user' : 'chat-bubble--assistant',
            { 'chat-bubble--pending': message.pending },
          ]"
        >
          <text class="chat-bubble__text" :selectable="true" :user-select="true">{{ chatStore.displayContent(message) }}</text>
        </view>
      </view>
      <view id="chat-bottom" class="chat-bottom-anchor" />
    </scroll-view>

    <view v-if="chatStore.error" class="chat-error">
      <text>{{ chatStore.error }}</text>
    </view>

    <view class="chat-input-bar">
      <scroll-view class="quick-scroll" scroll-x :show-scrollbar="false" :enhanced="true">
        <view class="quick-list">
          <button
            class="quick-button quick-button--primary"
            :disabled="chatStore.isBusy"
            @click="handleAnalysis"
          >
            <text>执行综合指征分析</text>
          </button>
          <button
            v-for="item in quickQuestions"
            :key="item.text"
            class="quick-button"
            :disabled="chatStore.isBusy"
            @click="handleQuickQuestion(item.question)"
          >
            <text>{{ item.text }}</text>
          </button>
        </view>
      </scroll-view>
      
      <view class="chat-input-wrap">
        <textarea
          v-model="inputText"
          class="chat-input"
          auto-height
          :maxlength="500"
          :disabled="chatStore.isBusy"
          confirm-type="send"
          placeholder="请输入症状描述或咨询问题..."
          :show-confirm-bar="false"
          :adjust-position="true"
          :hold-keyboard="true"
          @confirm="handleSend"
          @focus="scrollToBottom"
        />
        <button
          class="send-button"
          :disabled="!inputText.trim() || chatStore.isBusy"
          @click="handleSend"
        >
          {{ chatStore.isSending ? "处理中" : "发送" }}
        </button>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { nextTick, onMounted, ref } from "vue";
import { onShow } from "@dcloudio/uni-app";
import { useChatStore } from "@/stores/useChatStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";

const chatStore = useChatStore();
const inputText = ref("");
const scrollToId = ref("");

const quickQuestions = [
  { text: "评估周期规律", question: "请根据我的历史数据评估近期的周期规律。" },
  { text: "排卵期计算", question: "请说明排卵期的临床计算方法及我的预测日期。" },
  { text: "经期注意事项", question: "请列出经期需要注意的医学常识与生活干预建议。" },
];

async function scrollToBottom() {
  await nextTick();
  scrollToId.value = "";
  await nextTick();
  scrollToId.value = "chat-bottom";
}

async function handleSend() {
  const text = inputText.value.trim();
  if (!text || chatStore.isBusy) return;
  inputText.value = "";
  const sending = chatStore.sendMessage(text);
  await scrollToBottom();
  await sending;
  await scrollToBottom();
}

async function handleQuickQuestion(question: string) {
  if (chatStore.isBusy) return;
  const sending = chatStore.sendMessage(question);
  await scrollToBottom();
  await sending;
  await scrollToBottom();
}

function restorePendingPrompt() {
  const prompt = String(uni.getStorageSync("pendingAiPrompt") || "");
  if (!prompt) return;
  inputText.value = prompt;
  uni.removeStorageSync("pendingAiPrompt");
  void scrollToBottom();
}

async function handleAnalysis() {
  if (chatStore.isBusy) return;
  const analyzing = chatStore.analyzeToday();
  await scrollToBottom();
  await analyzing;
  await scrollToBottom();
}

onMounted(() => {
  chatStore.loadMessages().then(() => {
    scrollToBottom();
    return chatStore.runPendingAutoAnalysis();
  }).then(() => scrollToBottom());
});

onShow(restorePendingPrompt);
</script>

<style lang="scss" scoped>
.chat-page {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  background: $surface;
}

.chat-titlebar {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20rpx $gap-lg;
  background: $paper;
  border-bottom: 1px solid $line;
}
.chat-titlebar__name { font-size: $font-md; font-weight: 700; color: $ink; }

.chat-messages {
  flex: 1;
  min-height: 0;
  padding: $gap-md;
  overflow-x: hidden;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
  box-sizing: border-box;
}

.chat-row {
  display: flex;
  width: 100%;
  margin-bottom: $gap-md;

  &--user { justify-content: flex-end; }
  &--assistant { justify-content: flex-start; }
}

.chat-loading,
.chat-welcome {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: $muted;
  text-align: center;
}

.chat-loading { padding-top: 80rpx; font-size: 24rpx;}

.chat-welcome {
  padding: 60rpx 40rpx;
  background: $paper;
  border: 1px solid $line;
  border-radius: $radius-md;
  margin: $gap-lg 0;
}
.chat-welcome__title { margin-bottom: 16rpx; color: $ink; font-size: $font-lg; font-weight: 700; }
.chat-welcome__text { font-size: $font-sm; line-height: 1.6; color: $muted;}

.quick-scroll {
  width: 100%;
  margin-bottom: 16rpx;
  white-space: nowrap;
  overflow: hidden;
}

.quick-list {
  display: inline-flex;
  gap: 16rpx;
}

.quick-button {
  display: inline-flex;
  align-items: center;
  height: 56rpx;
  margin: 0;
  padding: 0 24rpx;
  border: 1px solid $line;
  border-radius: $radius-sm;
  background: $paper;
  color: $ink;
  font-size: 24rpx;
  line-height: 54rpx;

  &::after { border: 0; }
  &[disabled] { opacity: 0.5; }

  &--primary {
    background: $rose-dark;
    color: #fff;
    border-color: $rose-dark;
    font-weight: 600;
  }
}

.chat-bubble {
  display: inline-block;
  max-width: 85%;
  padding: 20rpx 24rpx;
  border-radius: $radius-md;
  font-size: 28rpx;
  line-height: 1.6;
  word-break: break-word;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
  box-sizing: border-box;

  &--user {
    background: $rose-dark;
    color: #fff;
    border-bottom-right-radius: 4rpx;
  }

  &--assistant {
    background: $paper;
    color: $ink;
    border: 1px solid $line;
    border-bottom-left-radius: 4rpx;
    box-shadow: $shadow-sm;
  }

  &--pending { opacity: 0.7; }
  &__text {
    display: block;
    width: 100%;
  }
}

.chat-bottom-anchor { height: 1px; }

.chat-error {
  padding: 12rpx $gap-md;
  background: #fef2f2;
  color: #b91c1c;
  font-size: $font-xs;
  text-align: center;
  border-top: 1px solid #fecaca;
}

.chat-input-bar {
  padding: 20rpx $gap-md calc(20rpx + env(safe-area-inset-bottom));
  background: $surface;
  border-top: 1px solid $line;
}

.chat-input-wrap {
  display: flex;
  align-items: flex-end;
  gap: 16rpx;
}

.chat-input {
  flex: 1;
  min-height: 80rpx;
  max-height: 240rpx;
  padding: 20rpx 24rpx;
  background: $paper;
  border: 1px solid $line;
  border-radius: $radius-sm;
  font-size: 28rpx;
  box-sizing: border-box;
}
.chat-input:focus { border-color: $rose-dark; }

.send-button {
  flex: 0 0 auto;
  height: 80rpx;
  margin: 0;
  padding: 0 32rpx;
  border-radius: $radius-sm;
  font-size: 26rpx;
  font-weight: 600;
  line-height: 80rpx;
  background: $rose-dark;
  color: #fff;
  border: none;

  &::after { border: 0; }
  &[disabled] { background: #cbd5e1; color: #f8fafc; }
}
</style>
