<template>
  <view class="page chat-page">
    <ClinicTopBar />
    <!-- 顶部仅保留标题(去掉了原来的"AI 小美 / 姐姐的专属健康顾问"双行头部),直接进入对话 -->
    <view class="chat-titlebar">
      <text class="chat-titlebar__name">小美 · 姐姐的健康顾问</text>
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
        <text>正在加载聊天记录…</text>
      </view>

      <view v-else-if="!chatStore.hasMessages" class="chat-welcome">
        <text class="chat-welcome__icon">💬</text>
        <text class="chat-welcome__title">姐姐好，我是小美</text>
        <text class="chat-welcome__text">
          小美会倾听姐姐的健康困惑。可以点击快捷提问，或直接描述今天的经期、身体感受，小美会在对话中自动生成当日专属分析报告。
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
      <view class="chat-input-wrap">
        <textarea
          v-model="inputText"
          class="chat-input"
          auto-height
          :maxlength="500"
          :disabled="chatStore.isBusy"
          confirm-type="send"
          placeholder="姐姐，想问小美什么呀…"
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
          {{ chatStore.isSending ? "发送中" : "发送" }}
        </button>
      </view>
      <scroll-view class="quick-scroll" scroll-x :show-scrollbar="false" :enhanced="true">
        <view class="quick-list">
          <button
            v-for="item in quickQuestions"
            :key="item.text"
            class="quick-button"
            :disabled="chatStore.isBusy"
            @click="handleQuickQuestion(item.question)"
          >
            <text>{{ item.icon }} {{ item.text }}</text>
          </button>
          <button
            class="quick-button quick-button--primary"
            :disabled="chatStore.isBusy"
            @click="handleAnalysis"
          >
            <text>✨ 生成今日专属分析</text>
          </button>
        </view>
      </scroll-view>
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
  { text: "周期规律", icon: "📅", question: "小美帮我看看最近的周期规律" },
  { text: "痛经加重", icon: "😖", question: "小美，我最近为什么痛经加重？" },
  { text: "排卵期", icon: "🥚", question: "小美，排卵期怎么算？" },
  { text: "经期饮食", icon: "🥗", question: "小美，经期饮食需要注意什么？" },
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
  // 加载聊天记录不阻塞页面首次渲染，加载完再尝试自动分析
  chatStore.loadMessages().then(() => {
    scrollToBottom();
    // 若用户从周期页「问小美」进入，消息加载完后再自动生成分析
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
  padding: 18rpx $gap-lg;
  background: $paper;
  border-bottom: 1px solid $line;

  &__name {
    font-size: $font-md;
    font-weight: 800;
    color: $ink;
  }
}

.chat-messages {
  flex: 1;
  min-height: 0;
  padding: $gap-md $gap-md $gap-sm;
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
  color: #71717a;
  text-align: center;
}

.chat-loading { padding-top: 180rpx; }

.chat-welcome {
  padding: 72rpx 20rpx 40rpx;

  &__icon { margin-bottom:16rpx;font-size:80rpx; }
  &__title { margin-bottom: 10rpx; color: $ink; font-size: $font-lg; font-weight: 800; }
  &__text { max-width: 590rpx; font-size: $font-sm; line-height: 1.7; }
}

.quick-scroll {
  width: 100%;
  margin-top: 12rpx;
  white-space: nowrap;
  overflow: hidden;
}

.quick-list {
  display: inline-flex;
  gap: 10rpx;
  padding-right: 10rpx;
}

.quick-button {
  display: inline-flex;
  flex: 0 0 auto;
  align-items: center;
  justify-content: center;
  gap: 6rpx;
  height: 52rpx;
  margin: 0;
  padding: 0 18rpx;
  border: 1px solid rgba(236, 72, 153, 0.18);
  border-radius: $radius-full;
  background: $paper;
  color: #52525b;
  font-size: 22rpx;
  line-height: 52rpx;
  white-space: nowrap;

  &::after { border: 0; }
  &[disabled] { opacity: 0.5; }

  &--primary {
    background: linear-gradient(135deg, $rose, $rose-dark);
    color: #fff;
    font-weight: 700;
    box-shadow: 0 4rpx 12rpx rgba(236, 72, 153, .18);
    border-color: transparent;
  }
}

.chat-bubble {
  display: inline-block;
  max-width: min(560rpx, 78%);
  padding: 18rpx 22rpx;
  border-radius: $radius-lg;
  font-size: $font-md;
  line-height: 1.7;
  word-break: break-word;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
  box-sizing: border-box;

  &--user {
    background: linear-gradient(135deg, $rose, $rose-dark);
    color: #fff;
    border-radius: $radius-lg $radius-sm $radius-lg $radius-lg;
  }

  &--assistant {
    background: $paper;
    color: $ink;
    border-radius: $radius-sm $radius-lg $radius-lg $radius-lg;
    box-shadow: $shadow-sm;
  }

  &--pending { color: #71717a; }
  &__text {
    display: block;
    width: 100%;
    word-break: break-word;
    white-space: pre-wrap;
    overflow-wrap: anywhere;
  }
}

.chat-bottom-anchor { height: 2rpx; }

.chat-error {
  padding: 8rpx $gap-lg;
  background: #fff1f2;
  color: #be123c;
  font-size: $font-xs;
  text-align: center;
}

.chat-input-bar {
  padding: 14rpx $gap-md calc(14rpx + env(safe-area-inset-bottom));
  background: $paper;
  border-top: 1px solid $line;
}

.chat-input-wrap {
  display: flex;
  align-items: flex-end;
  gap: 12rpx;
}

.chat-input {
  flex: 1;
  min-height: 68rpx;
  max-height: 220rpx;
  padding: 16rpx 20rpx;
  background: $surface;
  border: 1px solid rgba(236, 72, 153, 0.12);
  border-radius: $radius-md;
  font-size: $font-md;
  line-height: 1.5;
  box-sizing: border-box;
  word-break: break-word;
}

.send-button {
  flex: 0 0 auto;
  align-self: flex-end;
  width: 120rpx;
  height: 68rpx;
  margin: 0;
  padding: 0;
  border: 0;
  border-radius: $radius-md;
  font-size: 24rpx;
  font-weight: 700;
  line-height: 68rpx;
  background: linear-gradient(135deg, $rose, $rose-dark);
  color: #fff;
  text-align: center;

  &::after { border: 0; }
  &[disabled] { opacity: 0.48; }
}
</style>
