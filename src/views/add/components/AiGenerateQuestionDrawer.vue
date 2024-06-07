<template>
  <a-button type="outline" @click="handleClick">AI 生成题目</a-button>
  <a-drawer
    :width="340"
    :visible="visible"
    @ok="handleOk"
    @cancel="handleCancel"
    unmountOnClose
  >
    <template #title>AI 生成题目</template>
    <div>
      <a-form
        :model="form"
        label-align="left"
        auto-label-width
        @submit="handleSubmit"
      >
        <a-form-item label="应用 id">
          {{ appId }}
        </a-form-item>
        <a-form-item field="questionNumber" label="题目数量">
          <a-input-number
            min="0"
            max="20"
            v-model="form.questionNumber"
            placeholder="请输入题目数量"
          />
        </a-form-item>
        <a-form-item field="optionNumber" label="选项数量">
          <a-input-number
            min="0"
            max="6"
            v-model="form.optionNumber"
            placeholder="请输入选项数量"
          />
        </a-form-item>
        <a-form-item>
          <a-space>
            <a-button
              :loading="submitting"
              type="primary"
              html-type="submit"
              style="width: 120px"
            >
              {{ submitting ? "生成中" : "一键生成" }}
            </a-button>
            <a-button
              :loading="submitting"
              type="primary"
              @click="handleSSESubmit"
              style="width: 120px"
            >
              {{ submitting ? "生成中" : "实时生成" }}
            </a-button>
          </a-space>
        </a-form-item>
      </a-form>
    </div>
  </a-drawer>
</template>

<script setup lang="ts">
import { defineProps, reactive, ref, withDefaults } from "vue";
import API from "@/api";
import { aiGenerateQuestionUsingPost } from "@/api/questionController";
import message from "@arco-design/web-vue/es/message";

interface Props {
  appId: string;
  onSuccess?: (result: API.QuestionContentDTO[]) => void;
  onSSESuccess?: (result: API.QuestionContentDTO) => void;
  onSSEStart?: (event: any) => void;
  onSSEClose?: (event: any) => void;
}

const props = withDefaults(defineProps<Props>(), {
  appId: () => {
    return "";
  },
});

const form = reactive({
  optionNumber: 2,
  questionNumber: 10,
} as API.AiGenerateQuestionRequest);

const visible = ref(false);
const submitting = ref(false);

const handleClick = () => {
  visible.value = true;
};
const handleOk = () => {
  visible.value = false;
};
const handleCancel = () => {
  visible.value = false;
};

/**
 * 提交
 */
const handleSubmit = async () => {
  if (!props.appId) {
    return;
  }
  submitting.value = true;
  const res = await aiGenerateQuestionUsingPost({
    appId: props.appId as any,
    ...form,
  });
  if (res.data.code === 0 && res.data.data.length > 0) {
    if (props.onSuccess) {
      props.onSuccess(res.data.data);
    } else {
      message.success("生成题目成功");
    }
    // 关闭抽屉
    handleCancel();
  } else {
    message.error("操作失败，" + res.data.message);
  }
  submitting.value = false;
};
/**
 * 提交(SSE实时生成)
 */
const isDev = process.env.NODE_ENV === "development";
const handleSSESubmit = async () => {
  if (!props.appId) {
    return;
  }
  submitting.value = true;
  //创建SSE请求
  const eventSource = new EventSource(
    isDev
      ? `http://localhost:8081/api/question/ai_generate/sse?appId=${props.appId}&optionNumber=${form.optionNumber}&questionNumber=${form.questionNumber}`
      : `http://121.40.229.102:8081/api/question/ai_generate/sse?appId=${props.appId}&optionNumber=${form.optionNumber}&questionNumber=${form.questionNumber}`
  );
  let first = true;
  // 监听sse消息
  eventSource.onmessage = (event) => {
    if (first) {
      props.onSSEStart?.(event);
      handleCancel();
      first = false;
    }
    // 解析sse消息
    props.onSSESuccess?.(JSON.parse(event.data));
  };
  eventSource.onerror = (event) => {
    // 处理错误
    if (event.eventPhase === EventSource.CLOSED) {
      console.log("sse连接已关闭");
      eventSource.close();
      props.onSSEClose?.(event);
    } else {
      eventSource.close();
    }
  };
  submitting.value = false;
};
</script>
