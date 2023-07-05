<script setup lang="ts">
import { onMounted, ref, computed } from 'vue';
import { NEmpty, NButton, useMessage, NResult, NInput } from 'naive-ui';
import conversationCssText from '@/assets/css/conversation.css?raw';
import { usePromptStore, type IPrompt } from '@/stores/modules/prompt';
import { storeToRefs } from 'pinia';
import VirtualList from 'vue3-virtual-scroll-list';
import ChatPromptItem from './ChatPromptItem.vue';
import { isMobile } from '@/utils/utils';
import LoadingSpinner from '@/components/LoadingSpinner/LoadingSpinner.vue';
import { ApiResultCode } from '@/api/model/ApiResult';
import type { SysConfig } from '@/api/model/sysconf/SysConfig';
import { useChatStore } from '@/stores/modules/chat';
import ChatServiceSelect from '@/components/ChatServiceSelect/ChatServiceSelect.vue';
import { useUserStore } from '@/stores/modules/user';

const message = useMessage();
const isShowLoading = ref(true);

const promptStore = usePromptStore();
const { isShowPromptSotre, isShowChatPrompt, keyword, promptList, searchPromptList, selectedPromptIndex } = storeToRefs(promptStore);

const chatStore = useChatStore();
const { isShowChatServiceSelectModal, sydneyConfigs, selectedSydneyBaseUrl } = storeToRefs(chatStore);

const userStore = useUserStore();

const scrollbarRef = ref<{
  scrollToIndex: (index: number) => {};
  getOffset: () => number;
  getClientSize: () => number;
  getScrollSize: () => number;
}>();
const isInput = ref(false);
const isPromptScrolling = ref(false);
const promptItemHeight = 130;

const isShowUnauthorizedModal = ref(false);
const authKey = ref('');
const isAuthBtnLoading = ref(false);

const isShowHistory = computed(() => {
  return (CIB.vm.isMobile && CIB.vm.sidePanel.isVisibleMobile) || (!CIB.vm.isMobile && CIB.vm.sidePanel.isVisibleDesktop);
});

onMounted(async () => {
  await initChat();
  hackDevMode();
  // CIB.vm.isMobile = isMobile();
  // show conversion
  SydneyFullScreenConv.initWithWaitlistUpdate({ cookLoc: {} }, 10);
  initSysConfig();

  isShowLoading.value = false;
  hackStyle();
  initChatPrompt();
});

const hackDevMode = () => {
  if (import.meta.env.DEV) {
    CIB.manager.chat.api.bing._endpoint = location.origin;
  }
};

const initChatService = () => {
  if (selectedSydneyBaseUrl.value) {
    CIB.config.sydney.baseUrl = selectedSydneyBaseUrl.value;
    isShowChatServiceSelectModal.value = false;
  } else {
    isShowChatServiceSelectModal.value = true;
    selectedSydneyBaseUrl.value = CIB.config.sydney.baseUrl;
    const isCus = sydneyConfigs.value.filter((x) => !x.isCus).every((x) => x.baseUrl !== selectedSydneyBaseUrl.value);
    if (isCus) {
      const cusSydneyConfig = sydneyConfigs.value.find((x) => x.isCus);
      if (cusSydneyConfig) {
        cusSydneyConfig.baseUrl = selectedSydneyBaseUrl.value;
      }
    }
    chatStore.checkAllSydneyConfig();
  }
};

const initSysConfig = async () => {
  const res = await userStore.getSysConfig();
  switch (res.code) {
    case ApiResultCode.OK:
      {
        if (!res.data.isAuth) {
          isShowUnauthorizedModal.value = true;
          return;
        }
        afterAuth(res.data);
      }
      break;
    default:
      message.error(`[${res.code}] ${res.message}`);
      break;
  }
};

const afterAuth = (data: SysConfig) => {
  if (!data.isSysCK) {
    userStore.checkUserToken();
  }
  initChatService();
};

const initChat = async () => {
  return new Promise((resolve, reject) => {
    sj_evt.bind('sydFSC.init', resolve, true);
    sj_evt.fire('showSydFSC');
  });
};

const hackStyle = () => {
  if (location.hostname === 'localhost') {
    CIB.config.sydney.hostnamesToBypassSecureConnection = CIB.config.sydney.hostnamesToBypassSecureConnection.filter((x) => x !== location.hostname);
  }
  const serpEle = document.querySelector('cib-serp');
  // 居中
  serpEle?.setAttribute('alignment', 'center');
  const conversationEle = serpEle?.shadowRoot?.querySelector('cib-conversation') as HTMLElement;
  // todo 反馈暂时无法使用，先移除
  const welcomeEle = conversationEle?.shadowRoot?.querySelector('cib-welcome-container');
  welcomeEle?.shadowRoot?.querySelector('.learn-tog-item')?.remove();
  serpEle?.shadowRoot?.querySelector('cib-serp-feedback')?.remove();
  if (isMobile()) {
    welcomeEle?.shadowRoot?.querySelector('.container-item')?.remove();
    CIB.vm.actionBar.input.placeholder = '有问题尽管问我...（"/" 触发提示词）';
  }
  // 加入css
  const conversationStyleEle = document.createElement('style');
  conversationStyleEle.innerText = conversationCssText;
  conversationEle.shadowRoot?.append(conversationStyleEle);
};

interface IActionBarElement extends HTMLElement {
  handleInputTextKey: (ev: KeyboardEvent) => void;
}

const initChatPrompt = () => {
  const actionBarEle = document.querySelector('#b_sydConvCont > cib-serp')?.shadowRoot?.querySelector('#cib-action-bar-main') as IActionBarElement;
  const oldHandleInputTextKey = actionBarEle.handleInputTextKey;
  actionBarEle.handleInputTextKey = function (ev: KeyboardEvent) {
    // 有提示词时，优先选择提示词
    if (ev.key === 'Enter' && isShowChatPrompt.value) {
      return;
    }
    return oldHandleInputTextKey.apply(this, [ev]);
  };

  CIB.vm.actionBar.input.addEventListener('compositionstart', handleInputStart);
  CIB.vm.actionBar.input.addEventListener('compositionend', handleInputEnd);
  CIB.vm.actionBar.input.addEventListener('change', handleInputTextChanged);
  CIB.vm.actionBar.input.addEventListener('input', handleInputTextChanged);
  // CIB.vm.actionBar.input.addEventListener('keyup', handleInputTextKey);
  CIB.vm.actionBar.input.addEventListener('keydown', handleInputTextKey);
  CIB.vm.actionBar.input.addEventListener('focus', handleInputFocus);
  CIB.vm.actionBar.input.addEventListener('blur', handleInputBlur);
};

const handleInputStart = (ev: Event) => {
  // console.log('compositionstart : ', ev);
  isInput.value = true;
};

const handleInputEnd = (ev: Event) => {
  // console.log('compositionend : ', ev);
  isInput.value = false;
  handleInputTextChanged(ev);
};

const handleInputTextChanged = (ev: Event) => {
  // console.log('ev : ', ev);
  if (isInput.value) {
    return;
  }
  if ((ev instanceof InputEvent || ev instanceof CompositionEvent) && ev.target instanceof HTMLTextAreaElement) {
    if (ev.target.value?.startsWith('/')) {
      isShowChatPrompt.value = true;
      keyword.value = ev.target.value.slice(1);
      selectedPromptIndex.value = 0;
    } else {
      keyword.value = '';
      isShowChatPrompt.value = false;
    }
  }
};

const handleInputFocus = (ev: FocusEvent) => {
  // console.log('获取焦点:', ev);
};
const handleInputBlur = (ev: FocusEvent) => {
  // 简单解决失焦与点击冲突
  setTimeout(() => {
    isShowChatPrompt.value = false;
  }, 200);
};

const handleInputTextKey = (ev: KeyboardEvent) => {
  switch (ev.key) {
    case 'ArrowUp':
      {
        // ev.preventDefault();
        if (selectedPromptIndex.value > 0) {
          selectedPromptIndex.value--;
          if (scrollbarRef.value) {
            scrollbarRef.value.scrollToIndex(selectedPromptIndex.value);
          }
        }
      }
      break;
    case 'ArrowDown':
      {
        // ev.preventDefault();
        if (selectedPromptIndex.value < searchPromptList.value.length - 1) {
          selectedPromptIndex.value++;
          if (scrollbarRef.value) {
            scrollbarRef.value.scrollToIndex(selectedPromptIndex.value);
          }
        }
      }
      break;
    case 'Tab':
    case 'Enter':
      {
        // ev.preventDefault();
        if (!CIB.vm.actionBar.inputText || !CIB.vm.actionBar.inputText.startsWith('/')) {
          return;
        }
        selectPrompt(searchPromptList.value[selectedPromptIndex.value]);
      }
      break;
  }
};

const selectPrompt = (item: IPrompt) => {
  // console.log('select prompt : ', item);
  if (!item) {
    return;
  }
  keyword.value = '';
  CIB.vm.actionBar.inputText = item.prompt;
  isShowChatPrompt.value = false;
};

const handlePromptListScroll = () => {
  isPromptScrolling.value = true;
  setTimeout(() => {
    if (isPromptScrolling.value === true) {
      isPromptScrolling.value = false;
      // 滚动结束设置选中
      const offset = scrollbarRef.value?.getOffset() || 0;
      selectedPromptIndex.value = Math.round(offset / promptItemHeight);
    }
  }, 100);
};

const auth = async () => {
  if (!authKey.value) {
    message.error('请先输入授权码');
    return;
  }
  isAuthBtnLoading.value = true;
  userStore.setAuthKey(authKey.value);
  const res = await userStore.getSysConfig();
  if (res.data.isAuth) {
    message.success('授权成功');
    isShowUnauthorizedModal.value = false;
    afterAuth(res.data);
  } else {
    message.error('授权码有误');
  }
  isAuthBtnLoading.value = false;
};
</script>

<template>
  <LoadingSpinner :is-show="isShowLoading" />
  <main>
    <div
      v-if="isShowChatPrompt"
      class="box-border fixed bottom-[110px] w-full flex justify-center px-[14px] md:px-[34px] z-999"
      :class="{
        'md:px-[170px]': isShowHistory,
        'xl:px-[220px]': isShowHistory,
      }"
    >
      <div class="w-0 md:w-[60px]"></div>
      <VirtualList
        ref="scrollbarRef"
        v-if="promptList.length > 0"
        class="bg-white w-full max-w-[1060px] max-h-[390px] rounded-xl overflow-y-auto"
        :data-key="'prompt'"
        :data-sources="searchPromptList"
        :data-component="ChatPromptItem"
        :keeps="10"
        @scroll="handlePromptListScroll"
      />
      <NEmpty v-else class="bg-white w-full max-w-[1060px] max-h-[390px] rounded-xl py-6" description="暂未设置提示词数据">
        <template #extra>
          <NButton secondary type="info" @click="isShowPromptSotre = true">去提示词库添加</NButton>
        </template>
      </NEmpty>
    </div>
  </main>
  <footer>
    <!-- 服务器选择 -->
    <ChatServiceSelect />
    <!-- 授权 -->
    <div v-if="isShowUnauthorizedModal" class="fixed top-0 left-0 w-screen h-screen flex justify-center items-center bg-black/40 z-50">
      <NResult class="box-border w-11/12 lg:w-[400px] px-4 py-4 bg-white rounded-md" status="403" title="">
        <template #icon>
          <div style="display: flex; flex-direction: column; align-items: center; justify-content: center;">
                <svg version="1.1" id="Layer_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" width="120px" height="120px" viewBox="0 0 120 120" enable-background="new 0 0 120 120" xml:space="preserve">  
                  <image id="image0" width="120" height="120" x="0" y="0"
          href="data:image/png;base64,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" />
            </svg>
            <span>扫码关注公众号:农村自强少年，回复gpt获取授权码</span>
          </div>
        </template>
        <template #footer>
          <NInput class="w-11/12" v-model:value="authKey" type="password" placeholder="请输入授权码" maxlength="60" clearable></NInput>
          <n-button class="mt-4" secondary type="info" :loading="isAuthBtnLoading" @click="auth">授权</n-button>
        </template>
      </NResult>
    </div>
  </footer>
</template>
