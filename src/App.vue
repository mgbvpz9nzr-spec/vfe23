<script setup lang="ts">
import { onLaunch, onShow, onHide } from "@dcloudio/uni-app";
import { useUserStore } from "@/stores/useUserStore";

const userStore = useUserStore();

onLaunch(async () => {
  await userStore.restoreSession();
  // 有 token 且有效 → 进首页，否则留在登录页
  if (userStore.isLoggedIn) {
    uni.switchTab({ url: "/pages/home/index" });
  }
});

onShow(() => {
  // 前台显示时刷新状态
});

onHide(() => {
  // 后台隐藏
});
</script>

<style lang="scss">
/* uni.scss is auto-imported by uni-app as global style */
</style>
