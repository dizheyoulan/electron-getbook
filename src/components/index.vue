<template>
  <div class="warp">
    <h1>{{ msg }}</h1>
    <n-form label-placement="left" label-width="200" label-align="right" require-mark-placement="right-hanging" :style="{
      maxWidth: '640px'
    }">
      <n-form-item label="网络状态：">
        <p class="item-content">测试中... </p>
      </n-form-item>
      <n-form-item label="进度：">
        <n-progress type="line" :percentage="60" indicator-placement="inside" :processing="processing" status="success" />
      </n-form-item>
      <n-form-item label="URL前缀：">
        <n-input v-model:value="formValue.url" />
      </n-form-item>
      <n-form-item label="下载章节数量：">
        <n-input-number v-model:value="formValue.count" />
        <n-button attr-type="button" @click="getNum" class="ml" type="primary">获取总章节数量</n-button>
      </n-form-item>
      <n-form-item label="下载出错时重试间隔：">
        <n-input-number v-model:value="formValue.sleep" />
      </n-form-item>
      <n-form-item label="切分章节：">
        <n-switch v-model:value="formValue.clip" size="large" />
      </n-form-item>
      <n-form-item label="每几章切分章节：" v-if="formValue.clip">
        <n-input-number v-model:value="formValue.clipNum" />
      </n-form-item>
      <n-form-item label="下载模式：">
        <n-radio-group v-model:value="formValue.model" name="radiobuttongroup1">
          <n-radio-button :value="1" label="单线程下载" />
          <n-radio-button :value="2" label="多线程下载" />
        </n-radio-group>
      </n-form-item>
    </n-form>
    <n-button type="info" @click="download">开始下载</n-button>
  </div>
</template>
<script setup lang="ts">
import { ref } from 'vue'
import { NButton, NInput, NInputNumber, NForm, NFormItem, NSwitch, NRadioGroup, NRadioButton, NProgress, NSpin, NAlert } from 'naive-ui'
const Crawler = require("crawler");
import { ipcRenderer } from 'electron'
let proxy = ''
ipcRenderer.on('proxy', (_event, ...args) => {
  proxy = 'http://'+args[0].split(' ')[1]
})
export interface Props {
  msg?: string
}
const props = withDefaults(defineProps<Props>(), {
  msg: '输入参数，点击按钮开始下载',
})
const formValue = ref({
  status: '测试中',
  progress: '0/1',
  url: 'https://ncode.syosetu.com/n5677cl/',
  count: 1,
  sleep: 5,
  clip: false,
  clipNum: 10,
  model: 1
})
const processing = ref(true)
const getNum = (e: MouseEvent) => {

}
const download = () => {
  const baseURL = "https://ncode.syosetu.com/n5677cl/";

  const crawler = new Crawler({
    maxConnections: 10,
    timeout: 1000,
    // rateLimit: 1000,
    callback: (error:any, res:any, done:any) => {
      if (error) {
        console.error(error);
      } else {
        const $ = res.$;

        // 在这里处理页面数据，根据实际需要进行解析和提取

        // 示例：打印页面标题
        const title = $("title").text();
        console.log(`Page Title: ${title}`);
      }

      done();
    },
  });

  // 生成要爬取的URL列表
  const urls = Array.from({ length: 1 }, (_, i) => `${baseURL}${i + 1}/`);
  console.log(proxy);
  
  // 启动爬虫
  crawler.queue({
    uri:baseURL,
    proxy
  });
}
</script>

<style scoped>
.n-form {
  margin-top: 40px;
}

h1 {
  font-size: 20px;
}

.warp {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.item-content {
  margin-left: 100px;
}

.ml {
  margin-left: 20px;
}
</style>
