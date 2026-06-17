<template>
  <view class="page">
    <ClinicTopBar show-back />
    <text class="service-title">{{ title }}</text>
    <view v-if="loading" class="empty">正在加载...</view>
    <view v-else-if="error" class="empty error" @click="load">{{ error }}，点击重试</view>

    <template v-else-if="type === 'notifications'">
      <button v-if="items.some((item) => !item.read)" class="link" @click="readAll">全部标为已读</button>
      <view v-for="item in items" :key="item.id" class="card item" :class="{ unread: !item.read }" @click="readMessage(item)">
        <text class="item-title">{{ item.title || "门诊消息" }}</text><text class="copy">{{ item.content }}</text><text class="meta">{{ day(item.createdAt) }}</text>
      </view>
    </template>

    <template v-else-if="type === 'invite'">
      <view class="card invite">
        <text class="meta">我的邀请码</text><text class="code">{{ inviteInfo?.inviteCode || dashboard?.inviteCode || "暂未生成" }}</text>
        <button class="btn-secondary" @click="copy(inviteInfo?.inviteCode || dashboard?.inviteCode || '')">复制邀请码</button>
      </view>
      <image v-if="inviteInfo?.clinicSharePoster?.imageBase64" class="poster" :src="inviteInfo.clinicSharePoster.imageBase64" mode="widthFix" show-menu-by-longpress />
      <view v-else-if="inviteInfo?.qrDataUrl" class="card poster-fallback">
        <text class="poster-fallback__kicker">{{ inviteInfo?.clinic?.name || "诊所" }} 邀请卡</text>
        <image class="poster-fallback__qr" :src="inviteInfo.qrDataUrl" mode="aspectFit" show-menu-by-longpress />
        <text class="poster-fallback__code">邀请码 · {{ inviteInfo?.inviteCode || dashboard?.inviteCode || "—" }}</text>
        <text class="poster-fallback__hint">长按二维码保存图片，发送给好友扫码加入</text>
      </view>
      <view class="card form">
        <text class="section">分享健康活动给好友</text>
        <picker :range="campaigns.map((c:any) => c.title)" @change="referral.campaignId = campaigns[Number($event.detail.value)]?.id || ''"><view class="field">{{ campaignName || "选择健康活动" }}</view></picker>
        <input v-model="referral.name" class="field" placeholder="好友姓名" />
        <input v-model="referral.phone" class="field" type="number" maxlength="11" placeholder="好友手机号" />
        <button class="btn-primary" :loading="submitting" @click="submitReferral">发送健康活动信息</button>
      </view>
    </template>

    <template v-else-if="type === 'records'">
      <view class="card form">
        <text class="section">新增健康档案</text>
        <textarea v-model="content" class="textarea" maxlength="2000" placeholder="今天的感受、检查结果或想告诉医生的事..." />
        <view class="images"><view v-for="(file,i) in attachments" :key="file.url" class="image-wrap"><image :src="absolute(file.url)" mode="aspectFill" @click="preview(file.url)" /><text @click="attachments.splice(i,1)">删除</text></view><view v-if="attachments.length < 3" class="image-add" @click="chooseImages">+ 图片</view></view>
        <button class="btn-primary" :loading="submitting" @click="saveRecord">保存记录</button>
      </view>
      <view v-for="item in items" :key="item.id" class="card item">
        <view class="head"><text class="item-title">{{ item.authorRole === "PATIENT" ? "我的记录" : "医生记录" }}</text><text class="meta">{{ day(item.createdAt) }}</text></view>
        <text class="copy">{{ item.content }}</text>
        <view class="images"><image v-for="url in attachmentList(item.attachments)" :key="url" class="record-image" :src="absolute(url)" mode="aspectFill" @click="preview(url)" /></view>
        <button v-if="item.authorRole === 'PATIENT'" class="danger" @click="removeRecord(item.id)">删除</button>
      </view>
    </template>

    <template v-else-if="type === 'doctorRecords'">
      <view class="card doctor-banner">
        <text class="doctor-banner__icon">👨‍⚕️</text>
        <view>
          <text class="doctor-banner__title">我和医生的记录</text>
          <text class="doctor-banner__sub">共同维护的沟通与检查档案 · 共 {{ items.length }} 条</text>
        </view>
      </view>
      <view v-for="item in items" :key="item.id" class="card doctor-item">
        <view class="head">
          <text class="item-title" :class="{ 'item-title--mine': item.authorRole === 'PATIENT' }">{{ item.authorRole === "PATIENT" ? "我" : (item.authorName || "医生") }}</text>
          <text class="meta">{{ day(item.createdAt) }}</text>
        </view>
        <text v-if="item.content" class="copy">{{ item.content }}</text>
        <text v-else-if="attachmentList(item.attachments).length" class="copy copy--empty">图片档案记录</text>
        <view v-if="attachmentList(item.attachments).length" class="images"><image v-for="url in attachmentList(item.attachments)" :key="url" class="record-image" :src="absolute(url)" mode="aspectFill" @click="preview(url)" /></view>
      </view>
    </template>

    <template v-else-if="type === 'selfcheck'">
      <view v-if="selfLoading" class="card result result--loading">
        <text class="section">✨ AI 正在为您分析…</text>
        <text class="copy">小美根据您选择的症状，正在整理一份给医生参考的描述。</text>
      </view>
      <view v-else-if="selfResult" class="card result" :class="selfResult.level">
        <text class="section">{{ selfResult.title }}</text>
        <text v-if="selfResult.summary" class="copy">{{ selfResult.summary }}</text>
        <text class="meta">给医生看的描述</text>
        <text class="doctor-note">{{ selfResult.note }}</text>
        <button class="btn-primary" @click="copy(selfResult.note)">复制描述给医生</button>
        <button class="btn-secondary" @click="selfResult = null">返回修改</button>
      </view>
      <view v-else class="card form">
        <text class="section">妇科症状智能自查</text>

        <view class="form-row" @click="pickPregnant">
          <text class="form-row__label">是否怀孕或疑似怀孕</text>
          <view class="form-row__field"><text class="form-row__value">{{ selfForm.pregnant }}</text><text class="form-row__arrow">⌄</text></view>
        </view>

        <view class="form-row" @click="pickDuration">
          <text class="form-row__label">持续时间</text>
          <view class="form-row__field"><text class="form-row__value">{{ selfForm.duration }}</text><text class="form-row__arrow">⌄</text></view>
        </view>

        <text class="meta">主要不适（可多选）</text>
        <view class="chips">
          <text v-for="tag in symptomOptions" :key="tag" :class="{ active: selfForm.symptoms.includes(tag) }" @click="toggle(tag)">{{ tag }}</text>
        </view>

        <view class="form-row" @click="pickPain">
          <text class="form-row__label">下腹痛</text>
          <view class="form-row__field"><text class="form-row__value">{{ selfForm.pain }}</text><text class="form-row__arrow">⌄</text></view>
        </view>

        <view class="form-row" @click="pickBleeding">
          <text class="form-row__label">异常出血</text>
          <view class="form-row__field"><text class="form-row__value">{{ selfForm.bleeding }}</text><text class="form-row__arrow">⌄</text></view>
        </view>

        <view class="form-row" @click="pickFever">
          <text class="form-row__label">发热</text>
          <view class="form-row__field"><text class="form-row__value">{{ selfForm.fever }}</text><text class="form-row__arrow">⌄</text></view>
        </view>

        <button class="btn-primary" :loading="selfLoading" @click="runSelfCheck">AI 生成自查结果</button>
        <text class="disclaimer">结果由 AI 整理，仅用于健康信息参考，不可替代医生专业诊断。</text>
      </view>
    </template>

    <template v-else-if="type === 'treatments'">
      <view v-for="treatment in items" :key="treatment.id" class="card treatment">
        <view class="head"><text class="section">{{ treatment.name }}</text><text>{{ progress(treatment) }}%</text></view><view class="progress"><view :style="{width: progress(treatment)+'%'}" /></view>
        <view v-for="session in treatment.sessions" :key="session.id" class="session"><text class="item-title">{{ session.title }}</text><text class="meta">{{ day(session.plannedAt) }} · {{ session.status }}</text><view v-if="session.status !== 'COMPLETED'" class="actions"><button v-if="session.actionType === 'HOME_MEDICATION'" @click="treatmentAction(treatment.id,session.id,'COMPLETE')">完成打卡</button><button @click="feedback(treatment.id,session.id,'RESCHEDULE')">申请改期</button><button @click="feedback(treatment.id,session.id,'DISCOMFORT')">反馈不适</button></view></view>
      </view>
    </template>

    <template v-else>
      <view v-for="item in items" :key="item.id || item.title" class="card item"><view class="head"><text class="item-title">{{ itemTitle(item) }}</text><text class="meta">{{ itemStatus(item) }}</text></view><text class="copy">{{ itemCopy(item) }}</text><text class="meta">{{ itemDay(item) }}</text></view>
    </template>
    <view v-if="!loading && !error && !items.length && !['invite','selfcheck','doctorRecords'].includes(type)" class="empty">暂时没有{{ title }}</view>
    <view v-if="!loading && !error && !items.length && type==='doctorRecords'" class="empty">还没有任何记录<br/><text class="meta">您的记录和医生补充的检查档案都会显示在这里。</text></view>

    <view v-if="activeMessage" class="overlay" @click="activeMessage = null">
      <view class="sheet" @click.stop>
        <view class="sheet-handle" />
        <text class="sheet-kicker">诊所通知</text>
        <text class="sheet-title">{{ activeMessage.title || "门诊消息" }}</text>
        <text class="sheet-meta">{{ day(activeMessage.createdAt) }}</text>
        <scroll-view class="sheet-copy" scroll-y><text>{{ activeMessage.content || "暂无详细内容" }}</text></scroll-view>
        <button class="sheet-primary" @click="activeMessage = null">我知道了</button>
      </view>
    </view>

    <view v-if="deleteTarget" class="overlay overlay-center" @click="deleteTarget = null">
      <view class="dialog" @click.stop>
        <text class="dialog-icon">×</text>
        <text class="sheet-title">删除这条健康档案？</text>
        <text class="dialog-copy">删除后无法恢复，医生也将无法再查看这条记录。</text>
        <view class="dialog-actions">
          <button class="dialog-quiet" @click="deleteTarget = null">暂不删除</button>
          <button class="dialog-danger" :loading="submitting" @click="confirmRemoveRecord">确认删除</button>
        </view>
      </view>
    </view>

    <view v-if="feedbackForm" class="overlay" @click="feedbackForm = null">
      <view class="sheet" @click.stop>
        <view class="sheet-handle" />
        <text class="sheet-kicker">疗程反馈</text>
        <text class="sheet-title">{{ feedbackForm.action === "RESCHEDULE" ? "申请调整疗程时间" : "反馈本次不适" }}</text>
        <text class="dialog-copy">{{ feedbackForm.action === "RESCHEDULE" ? "请说明希望调整的日期或时间，诊所会尽快与您确认。" : "请描述不适表现和出现时间，便于医生及时判断。" }}</text>
        <textarea v-model="feedbackForm.content" class="feedback-textarea" maxlength="500" :placeholder="feedbackForm.action === 'RESCHEDULE' ? '例如：希望调整到本周五下午' : '例如：用药后出现轻微头晕，持续约半小时'" />
        <view class="feedback-count">{{ feedbackForm.content.length }}/500</view>
        <button class="sheet-primary" :loading="submitting" @click="submitFeedback">提交给诊所</button>
        <button class="sheet-secondary" @click="feedbackForm = null">取消</button>
      </view>
    </view>
  </view>
</template>

<script setup lang="ts">
import { computed, reactive, ref } from "vue";
import { onLoad } from "@dcloudio/uni-app";
import * as api from "@/api/menstrual";
import { absoluteApiUrl, uploadFile } from "@/api/client";
import { useAppStore } from "@/stores/useAppStore";
import ClinicTopBar from "@/components/ClinicTopBar.vue";
const app = useAppStore(); const type=ref("notifications"),loading=ref(true),submitting=ref(false),error=ref(""),dashboard=ref<any>(),inviteInfo=ref<any>(),items=ref<any[]>([]),content=ref(""),attachments=ref<Array<{url:string;name:string;type:string;size:number}>>([]);
const activeMessage=ref<any|null>(null),deleteTarget=ref<string|null>(null),feedbackForm=ref<{treatmentId:string;sessionId:string;action:"RESCHEDULE"|"DISCOMFORT";content:string}|null>(null);
const referral=reactive({name:"",phone:"",campaignId:""}); const selfResult=ref<any>(null); const selfLoading=ref(false); const selfForm=reactive({pregnant:"否",duration:"1天以内",symptoms:[] as string[],pain:"没有",bleeding:"没有",fever:"没有"});
const titles:Record<string,string>={notifications:"通知与提醒",invite:"邀请好友",visits:"就诊记录",purchases:"疗程记录",reminders:"用药提醒",enrollments:"活动报名",coupons:"健康关怀",records:"健康档案",treatments:"我的疗程",selfcheck:"症状智能自查",doctorRecords:"我和医生的记录"}; const title=computed(()=>titles[type.value]||"健康服务"); const campaigns=computed(()=>dashboard.value?.campaigns??[]); const campaignName=computed(()=>campaigns.value.find((c:any)=>c.id===referral.campaignId)?.title);
const durations=["1天以内","2-3天","4-7天","超过7天","反复发作"], symptomOptions=["外阴瘙痒","灼热感","白带异常","异味","同房疼痛","小便疼痛或尿频","下腹痛","外阴有疙瘩或破溃"], pains=["没有","轻微","明显","持续加重","剧烈疼痛"], bleedings=["没有","同房后少量出血","非月经期出血","褐色分泌物","出血量较多","绝经后出血"], fevers=["没有","低热","发热明显","不确定"];
onLoad((q)=>{type.value=String(q?.type||"notifications");load()}); const day=(v:any)=>v?String(v).slice(0,10):""; const copy=(v:string)=>uni.setClipboardData({data:v}); const absolute=absoluteApiUrl; const preview=(u:string)=>uni.previewImage({urls:[absolute(u)]});
function attachmentList(v:any){if(Array.isArray(v))return v;try{return JSON.parse(v||"[]")}catch{return[]}} function toggle(tag:string){const i=selfForm.symptoms.indexOf(tag);i>=0?selfForm.symptoms.splice(i,1):selfForm.symptoms.push(tag)}
function withTimeout<T>(promise: Promise<T>, ms = 8000, label = "请求"): Promise<T> {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error(`${label}超时（${ms / 1000}s），请检查网络后重试`)), ms);
    promise.then(
      (value) => { clearTimeout(timer); resolve(value); },
      (error) => { clearTimeout(timer); reject(error); },
    );
  });
}

async function load(){
  loading.value=true;
  error.value="";
  try {
    // 先加载当前类型需要的核心数据，避免 dashboard 慢/失败时本页一直转圈
    if (type.value === "doctorRecords") {
      try {
        const res = await withTimeout(api.fetchRecords(), 8000, "我和医生的记录");
        items.value = res.records ?? [];
      } catch (e: any) {
        error.value = e?.message || "我和医生的记录加载失败";
        items.value = [];
      }
      // 后台异步加载 dashboard（用于 campaigns 等），不影响本页显示
      withTimeout(api.fetchDashboard(), 8000, "诊所信息").then((d) => { dashboard.value = d; }).catch(() => undefined);
    } else if (type.value === "notifications") {
      const [d, msg] = await Promise.allSettled([
        withTimeout(api.fetchDashboard(), 8000, "诊所信息"),
        withTimeout(api.fetchMessages(), 8000, "消息"),
      ]);
      if (d.status === "fulfilled") dashboard.value = d.value;
      if (msg.status === "fulfilled") items.value = msg.value.messages ?? [];
      if (msg.status === "rejected") error.value = msg.reason?.message || "消息加载失败";
    } else if (type.value === "invite") {
      const [d, inv] = await Promise.allSettled([
        withTimeout(api.fetchDashboard(), 8000, "诊所信息"),
        withTimeout(api.fetchInviteInfo(), 8000, "邀请信息"),
      ]);
      if (d.status === "fulfilled") dashboard.value = d.value;
      if (inv.status === "fulfilled") {
        inviteInfo.value = inv.value;
        referral.campaignId = (dashboard.value?.campaigns?.[0] as any)?.id || referral.campaignId;
      } else {
        inviteInfo.value = {};
      }
    } else if (type.value === "treatments") {
      const [d, t] = await Promise.allSettled([
        withTimeout(api.fetchDashboard(), 8000, "诊所信息"),
        withTimeout(api.fetchTreatments(), 8000, "疗程"),
      ]);
      if (d.status === "fulfilled") dashboard.value = d.value;
      if (t.status === "fulfilled") items.value = t.value.treatments ?? [];
    } else {
      const d = await withTimeout(api.fetchDashboard(), 8000, "诊所信息").catch(() => null);
      if (d) dashboard.value = d;
      const map: Record<string, string> = {
        visits: "visitRecords",
        purchases: "purchases",
        reminders: "reminders",
        enrollments: "enrollments",
        coupons: "coupons",
      };
      items.value = dashboard.value?.[map[type.value]] ?? [];
      if (type.value === "records") {
        const r = await withTimeout(api.fetchRecords(), 8000, "档案").catch(() => null);
        if (r) items.value = r.records ?? [];
      }
    }
  } catch (e: any) {
    error.value = e?.message || "加载失败";
  } finally {
    loading.value = false;
  }
}
async function readMessage(m:any){if(!m.read){await api.markMessageRead(m.id);m.read=true}activeMessage.value=m} async function readAll(){await api.markAllMessagesRead();items.value.forEach(i=>i.read=true)}
async function submitReferral(){if(!referral.name||!/^1\d{10}$/.test(referral.phone)||!referral.campaignId)return app.showToast("请完善好友和活动信息","error");submitting.value=true;try{await api.createReferral({...referral});app.showToast("活动信息已发送","success");referral.name="";referral.phone=""}catch(e:any){app.showToast(e.message,"error")}finally{submitting.value=false}}
function chooseImages(){uni.chooseImage({count:3-attachments.value.length,success:async(r:any)=>{for(const p of r.tempFilePaths)attachments.value.push(await uploadFile(p))}})} async function saveRecord(){if(!content.value.trim()&&!attachments.value.length)return app.showToast("请填写文字或上传图片","error");submitting.value=true;try{await api.createRecord({content:content.value,attachments:attachments.value});content.value="";attachments.value=[];await load();app.showToast("档案已保存","success")}catch(e:any){app.showToast(e.message,"error")}finally{submitting.value=false}} function removeRecord(id:string){deleteTarget.value=id} async function confirmRemoveRecord(){if(!deleteTarget.value)return;submitting.value=true;try{await api.deleteRecord(deleteTarget.value);items.value=items.value.filter(i=>i.id!==deleteTarget.value);deleteTarget.value=null;app.showToast("档案已删除","success")}catch(e:any){app.showToast(e.message||"删除失败","error")}finally{submitting.value=false}}
function pickOption(current: string, options: string[], key: keyof typeof selfForm) {
  return new Promise<string>((resolve) => {
    uni.showActionSheet({
      itemList: options,
      success: (res) => {
        const chosen = options[res.tapIndex];
        (selfForm as any)[key] = chosen;
        resolve(chosen);
      },
      fail: () => resolve(current),
    });
  });
}
function pickPregnant() { void pickOption(selfForm.pregnant, ["否","是","不确定"], "pregnant"); }
function pickDuration() { void pickOption(selfForm.duration, durations, "duration"); }
function pickPain() { void pickOption(selfForm.pain, pains, "pain"); }
function pickBleeding() { void pickOption(selfForm.bleeding, bleedings, "bleeding"); }
function pickFever() { void pickOption(selfForm.fever, fevers, "fever"); }

function buildSelfCheckPrompt() {
  const symptomText = selfForm.symptoms.length ? selfForm.symptoms.join("、") : "无明显症状";
  const urgent = selfForm.pain === "剧烈疼痛" || ["出血量较多", "绝经后出血"].includes(selfForm.bleeding) || selfForm.fever === "发热明显" || selfForm.pregnant === "是";
  const score = selfForm.symptoms.length * 2 + (["超过7天", "反复发作"].includes(selfForm.duration) ? 2 : 0);
  const level = urgent ? "urgent" : score >= 5 ? "appointment" : "observe";
  const levelTitle = level === "urgent" ? "建议尽快就医" : level === "appointment" ? "建议安排妇科检查" : "建议继续观察";
  const summary = urgent
    ? "存在需要优先关注的表现，请尽快联系医生。"
    : "已为您整理症状信息，可用于和医生的预约沟通。";
  const note =
    `症状持续 ${selfForm.duration}；主要不适：${symptomText}；` +
    `下腹痛：${selfForm.pain}；异常出血：${selfForm.bleeding}；` +
    `发热：${selfForm.fever}；怀孕或疑似怀孕：${selfForm.pregnant}。`;
  return { level, levelTitle, summary, note };
}

async function runSelfCheck() {
  if (selfLoading.value) return;
  selfLoading.value = true;
  selfResult.value = null;
  const fallback = buildSelfCheckPrompt();
  try {
    const prompt =
      `我是女性患者，请根据以下症状给出简短的就医建议与提示，不要做任何确诊，只做信息整理。\n` +
      `症状持续时间：${selfForm.duration}\n` +
      `主要不适：${selfForm.symptoms.join("、") || "无"}\n` +
      `下腹痛程度：${selfForm.pain}\n` +
      `异常出血情况：${selfForm.bleeding}\n` +
      `发热：${selfForm.fever}\n` +
      `是否怀孕或疑似怀孕：${selfForm.pregnant}\n` +
      `请用 1-2 句话回复，并明确是否需要尽快就医。`;
    const state = await api.fetchMenstrualState().catch(() => ({ state: { mode: "PERIOD" } }));
    const ai = await api.generateChatReply({ message: prompt, mode: state.state?.mode || "PERIOD" });
    const aiText = ai.reply?.trim() || "";
    selfResult.value = {
      level: fallback.level,
      title: fallback.levelTitle,
      summary: aiText || fallback.summary,
      note: fallback.note,
    };
  } catch (e: any) {
    // AI 失败时使用兜底静态结果，确保用户仍能拿到可用描述
    selfResult.value = {
      level: fallback.level,
      title: fallback.levelTitle,
      summary: fallback.summary,
      note: fallback.note,
    };
    app.showToast(e?.message || "AI 暂时无法访问，已显示本地整理结果", "error");
  } finally {
    selfLoading.value = false;
  }
}
const progress=(t:any)=>t.sessions?.length?Math.round(t.sessions.filter((s:any)=>s.status==="COMPLETED").length/t.sessions.length*100):0; async function treatmentAction(t:string,s:string,a:"COMPLETE"|"RESCHEDULE"|"DISCOMFORT",c=""){await api.updateTreatment({treatmentId:t,sessionId:s,type:a,content:c});await load();app.showToast("操作已提交","success")} function feedback(t:string,s:string,a:"RESCHEDULE"|"DISCOMFORT"){feedbackForm.value={treatmentId:t,sessionId:s,action:a,content:""}} async function submitFeedback(){if(!feedbackForm.value?.content.trim())return app.showToast("请填写具体情况","error");const form=feedbackForm.value;submitting.value=true;try{await treatmentAction(form.treatmentId,form.sessionId,form.action,form.content.trim());feedbackForm.value=null}catch(e:any){app.showToast(e.message||"提交失败","error")}finally{submitting.value=false}}
function itemTitle(i:any){if(type.value==="visits")return i.visitType||"就诊记录";if(type.value==="purchases")return i.kitName||"健康疗程";if(type.value==="reminders")return i.title||"用药提醒";if(type.value==="enrollments")return campaigns.value.find((c:any)=>c.id===i.campaignId)?.title||"健康活动报名";return i.title||"健康关怀"} const itemStatus=(i:any)=>type.value==="reminders"?(i.done?"已完成":"待完成"):(i.status||""); const itemCopy=(i:any)=>i.summary||i.content||i.description||i.source||(type.value==="coupons"?`关怀金额：${i.amount??0}`:"记录已归档"); const itemDay=(i:any)=>day(i.visitedAt||i.purchasedAt||i.createdAt||i.expiresAt||i.time);
</script>

<style lang="scss" scoped>
.page{min-height:100vh;padding:0 $gap-md 80rpx;background:$surface}.service-title{display:block;padding:24rpx 0 18rpx;font-size:$font-xl;font-weight:800}.meta,.disclaimer{color:$muted;font-size:$font-xs}.empty,.card{background:$paper;border-radius:$radius-lg;padding:$gap-lg;margin-bottom:$gap-md;box-shadow:$shadow-sm}.empty{text-align:center;color:$muted;padding:80rpx 20rpx}.empty .meta{display:block;margin-top:8rpx;font-size:20rpx}.error,.danger{color:#b91c1c}
.doctor-banner{display:flex;align-items:center;gap:$gap-md;background:linear-gradient(135deg,$blush,#fce7f3);border:1px solid #fbcfe8}
.doctor-banner__icon{display:grid;width:80rpx;height:80rpx;flex:0 0 auto;place-items:center;border-radius:50%;background:#fff;font-size:42rpx}
.doctor-banner__title{display:block;color:$ink;font-size:$font-md;font-weight:800}
.doctor-banner__sub{display:block;margin-top:4rpx;color:$rose-dark;font-size:$font-xs;line-height:1.55}
.doctor-item{background:$paper}.item-title--mine{color:$rose-dark;font-weight:800}
.item.unread{background:$blush}.head{display:flex;justify-content:space-between;gap:16rpx}.item-title,.section{display:block;font-size:$font-md;font-weight:800}.copy{display:block;font-size:$font-sm;line-height:1.7;margin:10rpx 0;white-space:pre-wrap}
.copy--empty{color:$muted;font-style:italic}.link,.danger{background:transparent;border:0;font-size:$font-sm}.invite{text-align:center}.code{display:block;font-size:60rpx;font-weight:900;letter-spacing:8rpx;margin:20rpx}.poster{width:100%;border-radius:$radius-lg;margin-bottom:$gap-md}
.poster-fallback{display:flex;flex-direction:column;align-items:center;padding:$gap-xl $gap-lg;background:linear-gradient(180deg,$blush,#fff5f9);text-align:center}
.poster-fallback__kicker{display:block;color:$rose-dark;font-size:$font-xs;font-weight:800;letter-spacing:2rpx}
.poster-fallback__qr{width:360rpx;height:360rpx;margin:$gap-md 0;background:$paper;padding:18rpx;border-radius:$radius-md;box-shadow:$shadow-sm}
.poster-fallback__code{display:block;color:$ink;font-size:$font-md;font-weight:800;letter-spacing:4rpx}
.poster-fallback__hint{display:block;margin-top:$gap-xs;color:$muted;font-size:$font-xs;line-height:1.6}.section{font-size:$font-lg;margin-bottom:$gap-md}.field,.textarea{display:block;width:100%;box-sizing:border-box;background:$surface;border:1px solid $line;border-radius:$radius-sm;padding:20rpx;margin-bottom:$gap-sm;font-size:$font-sm}
.field--input{padding:24rpx 20rpx;line-height:1.4}.textarea{min-height:180rpx}.images{display:flex;flex-wrap:wrap;gap:12rpx;margin:12rpx 0}.image-wrap,.image-add,.record-image,.image-wrap image{width:150rpx;height:150rpx;border-radius:$radius-sm}.image-add{display:grid;place-items:center;background:$blush;color:$rose-dark}.image-wrap text{display:block;text-align:center;color:#b91c1c;font-size:20rpx}.chips{display:flex;flex-wrap:wrap;gap:10rpx;margin:12rpx 0 20rpx}.chips text{padding:12rpx 20rpx;border-radius:$radius-full;background:$surface;font-size:$font-xs}.chips .active{background:$rose-dark;color:#fff}
.form-row{display:flex;align-items:center;justify-content:space-between;gap:12rpx;margin-bottom:14rpx;padding:18rpx 20rpx;border-radius:$radius-sm;background:$surface}
.form-row__label{color:$muted;font-size:$font-xs;font-weight:700}
.form-row__field{display:flex;align-items:center;gap:8rpx;color:$ink;font-size:$font-sm;font-weight:700}
.form-row__value::before{content:"";display:inline-block;width:8rpx;height:8rpx;margin-right:8rpx;border-radius:50%;background:$rose-dark}
.form-row__arrow{color:$rose-dark;font-size:24rpx;font-weight:700}
.result--loading{background:linear-gradient(135deg,$blush,#fce7f3);border:1px solid #fbcfe8;text-align:center}.doctor-note{display:block;padding:20rpx;background:$surface;margin:12rpx 0;white-space:pre-wrap}.result.urgent{border-top:8rpx solid #dc2626}.result.appointment{border-top:8rpx solid #f59e0b}.result.observe{border-top:8rpx solid #16a34a}.progress{height:12rpx;background:$surface;border-radius:99rpx;overflow:hidden;margin:16rpx 0}.progress view{height:100%;background:$rose-dark}.session{padding:18rpx 0;border-top:1px solid $line}.actions{display:flex;gap:8rpx;margin-top:12rpx}.actions button{flex:1;font-size:20rpx;padding:8rpx;background:$blush;color:$rose-dark}
.overlay{position:fixed;inset:0;z-index:40;display:flex;align-items:flex-end;background:rgba(39,39,42,.48);backdrop-filter:blur(8rpx)}.overlay-center{align-items:center;justify-content:center;padding:$gap-xl}.sheet{width:100%;max-height:86vh;box-sizing:border-box;padding:$gap-md $gap-lg calc(36rpx + env(safe-area-inset-bottom));border-radius:40rpx 40rpx 0 0;background:$paper}.sheet-handle{width:80rpx;height:8rpx;margin:0 auto $gap-lg;border-radius:99rpx;background:#e4e4e7}.sheet-kicker{display:block;color:$rose-dark;font-size:$font-xs;font-weight:800}.sheet-title{display:block;margin-top:4rpx;color:$ink;font-size:$font-xl;font-weight:800;line-height:1.35}.sheet-meta{display:block;margin-top:6rpx;color:$muted;font-size:$font-xs}.sheet-copy{max-height:42vh;margin:$gap-lg 0;color:$ink;font-size:$font-sm;line-height:1.9;white-space:pre-wrap}.sheet-primary,.sheet-secondary{width:100%;margin-top:$gap-sm;border-radius:$radius-full;font-size:$font-sm;font-weight:700}.sheet-primary{background:$ink;color:#fff}.sheet-secondary{background:$surface;color:$ink}.dialog{width:100%;box-sizing:border-box;padding:52rpx $gap-lg $gap-lg;border-radius:36rpx;background:$paper;text-align:center}.dialog-icon{display:block;width:72rpx;height:72rpx;margin:0 auto $gap-md}.dialog-copy{display:block;margin-top:$gap-sm;color:$muted;font-size:$font-sm;line-height:1.7}.dialog-actions{display:flex;gap:$gap-sm;margin-top:$gap-xl}.dialog-actions button{flex:1;margin:0;border-radius:$radius-full;font-size:$font-sm;font-weight:700}.dialog-quiet{background:$surface;color:$ink}.dialog-danger{background:#e11d48;color:#fff}.feedback-textarea{width:100%;min-height:220rpx;box-sizing:border-box;margin-top:$gap-lg;padding:$gap-md;border:1px solid $line;border-radius:$radius-md;background:$surface;color:$ink;font-size:$font-sm;line-height:1.7}.feedback-count{margin-top:6rpx;color:$muted;font-size:20rpx;text-align:right}
</style>
