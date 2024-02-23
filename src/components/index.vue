<template>
  <div class="warp">
    <h1>{{ msg }}</h1>
    <n-form label-placement="left" label-width="200" label-align="right" require-mark-placement="right-hanging" :style="{
      maxWidth: '640px'
    }">
      <n-form-item label="网络状态：">
        <div class="item-content">
          <p v-if="formValue.status===0">测试中... </p>
          <p class="green" v-if="formValue.status===1">可连接 </p>
          <p class="red" v-if="formValue.status===2">无法连接 </p>
        </div>
      </n-form-item>
      <n-form-item label="进度：">
        <n-progress type="line" :percentage="progress" indicator-placement="inside" :processing="processing"
          status="success" />
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
      <n-form-item label="下载模式(多线程比较快，但是可能会被ban ip)：">
        <n-radio-group v-model:value="formValue.model" name="radiobuttongroup1">
          <n-radio-button :value="1" label="单线程下载" />
          <n-radio-button :value="10" label="多线程下载" />
        </n-radio-group>
      </n-form-item>
    </n-form>
    <n-button type="info" @click="download">开始下载</n-button>
  </div>
</template>
<script setup lang="ts">
import { ref, computed, watch, VueElement } from 'vue'
import { NButton, NInput, NInputNumber, NForm, NFormItem, NSwitch, NRadioGroup, NRadioButton, NProgress, useNotification } from 'naive-ui'
const Crawler = require("crawler");
import { ipcRenderer } from 'electron'
const fs = require('fs');
const path = require('path');
let userDataPath = path.resolve()
console.log('userDataPath：',userDataPath);
let proxy = ''
ipcRenderer.on('proxy', (_event, ...args) => {
  proxy = 'http://' + args[0].split(' ')[1]
  console.log('args',args);
  if(args[1]){
    userDataPath = args[1]
    console.log('userDataPath：',userDataPath);
  }
})
export interface Props {
  msg?: string
}
const props = withDefaults(defineProps<Props>(), {
  msg: '输入参数，点击按钮开始下载',
})
const formValue = ref({
  status: 0,
  url: 'https://ncode.syosetu.com/n5677cl/',
  count: 10,
  sleep: 5,
  clip: false,
  clipNum: 10,
  model: 10,
})
let completeNum = ref(0)
let temp: any[] = []
let title = ''
let name = ''
const progress = computed(() => Math.floor((completeNum.value / formValue.value.count) * 100))
const processing = ref(false)
const headers = {
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.3',
  'Cookie': 'over18=yes'
}
const timeout = 5000
const init = ()=>{
  completeNum.value = 0
  temp = []
  title = ''
  name = ''
}
const getNum = (e: MouseEvent) => {
  if(processing.value)return
  init()
  const crawler = new Crawler({
    timeout,
    headers,
    proxy,
    callback: (error: any, res: any, done: any) => {
      if (error) {
        console.error(error);
      } else {
        const $ = res.$;
        formValue.value.count = $('.novel_sublist2').length
        if(res.body.includes('Too many access')){
          notification['error']({
            content: '短时间访问次数过多',
            meta: `ip被网站ban了，暂无法爬取，请以后再试`,
            duration: 10000,
            keepAliveOnHover: true
          })
          return
        }
        console.log('获取到页面数据：',res);
        console.log('获取到章节数量：',$('.novel_sublist2').length);
      }
      done();
    },
  });
  crawler.queue(formValue.value.url);
}
const download = () => {
  
  if(processing.value)return
  init()
  let urls:string[] = []
  for (let i = 0; i < formValue.value.count; i++) {
    urls[i] = `${formValue.value.url}${i + 1}/`
  }
  temp.length = formValue.value.count
  processing.value = true
  // console.log('urls', urls);
  // console.log('proxy', proxy);
  const firstcrawler = new Crawler({
    timeout,
    headers,
    proxy,
    callback: (error: any, res: any, done: any) => {
      if (error) {
        console.error(error);
      } else {
        const $ = res.$;
        title = $('.novel_title').text();
        name = $('.novel_writername').text();
        console.log('获取元信息',title,name);
        crawler.queue(urls);
      }
      done();
    },
  });

  const crawler = new Crawler({
    maxConnections: formValue.value.model,
    timeout,
    // rateLimit: 5000,
    retryTimeout:formValue.value.sleep*1000,
    headers,
    proxy,
    callback: (error: any, res: any, done: any) => {
      if (error) {
        console.error(error);
      } else {
        const $ = res.$;
        const index = +res.options.uri.replace(formValue.value.url, '').replace('/', '')
        const stit = $('.novel_subtitle').text();
        const content = $('#novel_honbun').text();
        temp[index - 1] = { stit, content }
        // console.log(stit,content);
        completeNum.value++
      }
      done();
    },
  });


  // 启动爬虫
  firstcrawler.queue(formValue.value.url);
  
}
watch(progress, val => {
  if (val===100&&title) {
    processing.value = false
    if(formValue.value.clip){
      writeClip()
      write()
    }else{
      write()
    }
  }
})
const notification = useNotification()
const write = () => {
  let data = temp.map((item, index) => `第${index + 1}章 ${item.stit}\n\n${item.content}`).join('\n')
  data = `${title}\n${name}\n` + data
  
  fs.writeFile(path.join(userDataPath, `${title}.txt`), data, (err: any) => {
    if (err) {
      console.log(err);
    } else {
      notification['success']({
        content: '下载完成',
        meta: `${title} 下载完成`,
        duration: 10000,
        keepAliveOnHover: true
      })
    }
  });
}
function chunkArray(arr:any[], n:number) {
  const chunkedArray = [];
  let index = 0;

  while (index < arr.length) {
    chunkedArray.push(arr.slice(index, index + n));
    index += n;
  }
  return chunkedArray;
}

const writeClip = () => {
  let data = temp.map((item, index) => `第${index + 1}章 ${item.stit}\n\n${item.content}`)
  let dataList = chunkArray(data,formValue.value.clipNum)
  dataList[0][0]=`${title}\n${name}\n`+dataList[0][0]

  dataList.forEach((item,index)=>{
    fs.writeFile(path.join(userDataPath, `${title}-${index+1}.txt`), item.join('\n'), (err: any) => {
      if (err) {
        console.log(err);
      } else {
        if(index===dataList.length-1){
          notification['success']({
            content: '下载完成',
            meta: `${title} 下载完成`,
            duration: 10000,
            keepAliveOnHover: true
          })
        }
      }
    });
  })
}
let errCount = 0
const loopPing = (link:string)=>{
  const crawler = new Crawler({
    timeout,
    retries:0,
    headers,
    proxy,
    callback: (error: any, res: any, done: any) => {
      
      
      if (error) {
        errCount+=1
        if(errCount===3){
          formValue.value.status = 2
        }
      } else {
        errCount = 0
        formValue.value.status = 1
      }
      done();
    },
  });
  crawler.queue(link);
  setTimeout(() => {
    loopPing(link);
  }, 5000);
}
loopPing('https://ncode.syosetu.com/')
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
.green{
  color: #18a058;
}
.red{
  color: red;
}

.ml {
  margin-left: 20px;
}
</style>
