<template>
  <div class="h-screen w-full bg-gray-50 flex flex-col items-center">
    <!-- 顶部导航栏 -->
    <div
      class="w-full max-w-3xl bg-white border-b border-gray-200 px-6 py-4 flex justify-between items-center shadow-sm z-10"
    >
      <div class="flex items-center gap-4">
        <h1 class="font-bold text-gray-800 text-lg">🎓 智能助教</h1>
        <!-- 🔴 改进：使用 v-for 循环渲染题目列表 -->
        <select
          v-model="selectedQuestionId"
          @change="init"
          class="text-xs border rounded p-1 outline-none"
        >
          <option
            v-for="item in problemList"
            :key="item.problemId"
            :value="item.problemId"
          >
            题目 {{ item.problemId }}: {{ item.title }}
          </option>
        </select>
      </div>
      <div class="flex items-center gap-2">
        <span
          class="bg-blue-600 text-white text-xs font-bold px-3 py-1 rounded-full"
          >Step {{ currentStep }}</span
        >
      </div>
    </div>

    <!-- 聊天区域 -->
    <div
      class="flex-1 w-full max-w-3xl bg-white shadow-xl border-x flex flex-col relative overflow-hidden"
    >
      <div
        class="flex-1 overflow-y-auto p-6 space-y-6 scroll-smooth"
        ref="chatRef"
      >
        <div
          v-for="(msg, i) in messages"
          :key="i"
          class="flex flex-col w-full"
          :class="msg.sender === 'user' ? 'items-end' : 'items-start'"
        >
          <div
            class="flex w-full"
            :class="msg.sender === 'user' ? 'justify-end' : 'justify-start'"
          >
            <div
              v-if="msg.sender === 'bot'"
              class="w-8 h-8 rounded-full bg-blue-100 flex items-center justify-center mr-3 shrink-0 text-lg"
            >
              🤖
            </div>
            <div
              class="px-5 py-3.5 max-w-[90%] rounded-2xl text-sm shadow-sm"
              :class="
                msg.sender === 'user'
                  ? 'bg-blue-600 text-white rounded-tr-sm'
                  : 'bg-gray-100 text-gray-800 rounded-tl-sm'
              "
            >
              <!-- Markdown 渲染区 -->
              <div
                class="prose prose-sm max-w-none media-container"
                :class="
                  msg.sender === 'user'
                    ? 'text-white prose-invert'
                    : 'text-gray-800'
                "
                v-html="renderText(msg.text)"
              ></div>
            </div>
          </div>
        </div>
        <div v-if="loading" class="ml-11 text-gray-400 text-xs animate-pulse">
          助教正在思考...
        </div>
      </div>

      <!-- 输入区 -->
      <div class="p-4 border-t bg-white">
        <div class="relative">
          <input
            v-model="inputMsg"
            @keyup.enter="send"
            type="text"
            placeholder="输入你的问题..."
            class="w-full bg-gray-50 border border-gray-200 rounded-xl px-4 py-3.5 focus:ring-2 focus:ring-blue-500 outline-none"
          />
          <button
            @click="send"
            :disabled="loading || !inputMsg"
            class="absolute right-2 top-2 bottom-2 bg-blue-600 text-white px-4 rounded-lg text-sm font-medium hover:bg-blue-700 disabled:opacity-50 transition"
          >
            发送
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from "vue";
import axios from "axios";
import { marked } from "marked";

const messages = ref([]);
const inputMsg = ref("");
const currentStep = ref(1);
const loading = ref(false);
const chatRef = ref(null);
const userId = "user_001";
const selectedQuestionId = ref(1);
// 🔴 新增：存储题目列表
const problemList = ref([]);

const renderer = new marked.Renderer();

/**
 * 🟢 兼容各版本 marked 的参数传递
 */
renderer.link = function (hrefOrObj, title, text) {
  let finalHref, finalTitle, finalText;
  if (typeof hrefOrObj === "object" && hrefOrObj !== null) {
    finalHref = hrefOrObj.href;
    finalTitle = hrefOrObj.title;
    finalText = hrefOrObj.text;
  } else {
    finalHref = hrefOrObj;
    finalTitle = title;
    finalText = text;
  }
  if (!finalText || finalText === "undefined") finalText = "查看链接";

  return `<a href="${finalHref}" target="_blank" class="inline-block mt-1 mb-1 px-3 py-1 bg-white border border-blue-400 text-blue-600 rounded-md no-underline font-bold hover:bg-blue-50 transition">${finalText}</a>`;
};

renderer.image = function (hrefOrObj, title, text) {
  let finalHref, finalAlt;
  if (typeof hrefOrObj === "object" && hrefOrObj !== null) {
    finalHref = hrefOrObj.href;
    finalAlt = hrefOrObj.text;
  } else {
    finalHref = hrefOrObj;
    finalAlt = text;
  }
  return `<div class="image-box"><img src="${finalHref}" alt="${finalAlt}" style="width: 100%; height: auto; display: block; border-radius: 8px; margin: 10px 0;" /></div>`;
};

// 🔴 新增：获取所有题目
const fetchProblems = async () => {
  try {
    const res = await axios.get("http://localhost:3000/api/problems");
    problemList.value = res.data;
    if (problemList.value.length > 0) {
      selectedQuestionId.value = problemList.value[0].problemId;
    }
  } catch (e) {
    console.error("获取题目列表失败:", e);
  }
};

const init = async () => {
  messages.value = [];
  loading.value = true;
  try {
    const res = await axios.post("http://localhost:3000/api/init", {
      userId,
      questionId: selectedQuestionId.value,
    });
    messages.value.push({ sender: "bot", text: res.data.systemMsg });
    currentStep.value = res.data.currentStep;
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
  }
};

const send = async () => {
  if (!inputMsg.value.trim()) return;
  const text = inputMsg.value;
  messages.value.push({ sender: "user", text });
  inputMsg.value = "";
  loading.value = true;
  scrollToBottom();
  try {
    const res = await axios.post("http://localhost:3000/api/chat", {
      userId,
      message: text,
    });
    messages.value.push({ sender: "bot", text: res.data.text });
    if (res.data.currentStep) currentStep.value = res.data.currentStep;
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
    scrollToBottom();
  }
};

const scrollToBottom = () => {
  nextTick(() => {
    if (chatRef.value) chatRef.value.scrollTop = chatRef.value.scrollHeight;
  });
};

const renderText = (t) => marked.parse(t || "", { renderer: renderer });

// 🔴 修改：页面挂载先获取题目，再初始化聊天
onMounted(async () => {
  await fetchProblems();
  init();
});
</script>

<style scoped>
:deep(.prose p) {
  margin-bottom: 0.5rem;
}
:deep(.prose p:last-child) {
  margin-bottom: 0;
}
:deep(.media-container) {
  overflow: visible !important;
}
:deep(.image-box) {
  width: 100%;
  overflow: hidden;
  border-radius: 8px;
}
</style>
