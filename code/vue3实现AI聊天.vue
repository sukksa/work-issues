<template>
  <div >
    <div class="boxhead">
      <div>
        <span>AI助手
        </span>
      </div>
      <div>
        <img src="../assets/closew.png" alt="" @click="back" style="cursor: pointer;" />
      </div>
    </div>
    <div class="line"></div>
    <div class="md-main" ref="mdMainRef">
      <div class="md-box" v-for="(item,index) in records" :key="index">
        <div class="content">
          <span>{{ item.content }}</span>
        </div>
        <!-- <div class=""> -->
        <div v-html="md.render(item.markdown)" class="markdown-body"></div>
        <!-- </div> -->
      </div>

      <div class="md-box">
        <div class="content" v-if="oldContent"><span>{{ oldContent }}</span> </div>
        <div class="think" v-if="isThinking">思考中{{ dots }}</div>
        <!-- <div class=""> -->
        <div v-if="displayText" v-html="md.render(displayText)" class="markdown-body"></div>
        <!-- </div> -->
      </div>
    </div>
    <div v-if="userScrolledUp" class="scroll-bottom-btn" @click="forceScrollToBottom">
      <img src="../assets/to-bottom.png" alt="">
    </div>
    <div class="user-input">
      <div>
        <textarea placeholder="发送消息" class="send-msg" v-model="  content">

        </textarea>
        <div class="icon">
          <div v-if='content&&!isFetching'>
            <div @click="send" class="send"><img src="../assets/up-send.png" alt=""></div>
          </div>
          <div v-if="!content&&!isFetching">
            <el-tooltip class="box-item" effect="dark" content="请输入你的问题" placement="top-start">
              <div class="unsend"><img src="../assets/up-unsend.png" alt=""></div>
            </el-tooltip>
          </div>
          <div v-if="isFetching">
            <el-tooltip class="box-item" effect="dark" content="停止生成" placement="top-start">
              <div @click="stop" class="stop"><img src="../assets/stop.png" alt=""></div>
            </el-tooltip>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script setup>
  import { ref,  defineEmits, defineProps, watch, nextTick } from 'vue'
  import MarkdownIt from 'markdown-it'
  const md = new MarkdownIt()
  import { useIntervalFn } from '@vueuse/core'
  const contentText = ref('') // 原始文本内容
  const displayText = ref('') // 显示文本内容
  const cursorVisible = ref(true) // 光标状态
  const isThinking = ref(false)
  const records = ref([])
  const mdMainRef = ref(null) // 容器引用
  // 新增停止控制相关代码
  const controller = ref(null) // 存储AbortController
  const isFetching = ref(false) // 请求状态
  const props = defineProps({
    showDsChat: Boolean,
  })
  const content = ref('')
  const oldContent = ref('')
  const userScrolledUp = ref(false)

  // 思考中加载动画
  const dots = ref('...')
  const timer = ref(null)
  const start = () => {
    timer.value = setInterval(() => {
      if (dots.value.length === 6) {
        dots.value = '.'
      } else {
        dots.value += '.'
      }
    }, 1000)
  }
  const end = () => {
    if (timer.value) clearInterval(timer.value)
  }

  // 以前的对话记录
  const addRecords = () => {
    let oldText = {
      content: oldContent.value,
      markdown: contentText.value,
    }
    records.value.push(oldText)
    displayText.value = ''
    oldContent.value = ''
    contentText.value = ''
  }
// 判断是否在底部
  const handleScroll = () => {
    const { scrollTop, scrollHeight, clientHeight } = mdMainRef.value
    userScrolledUp.value = scrollTop + clientHeight < scrollHeight - 20
  }

  // 手动点击停止
  const stop = () => {
    end()

    if (controller.value) {
      controller.value.abort() // 终止请求
      controller.value = null
      isFetching.value = false
      isThinking.value = false
      console.log('请求已中止')
    }
  }
  // 在setup()中添加方法
  const forceScrollToBottom = () => {
    userScrolledUp.value = false // 重置滚动状态
    scrollToBottom() // 复用已有滚动方法
  }
  // 滚动到底部方法
  const scrollToBottom = () => {
    nextTick(() => {
      if (mdMainRef.value) {
        mdMainRef.value.scrollTo({
          top: mdMainRef.value.scrollHeight,
          // behavior: 'smooth',
        })
      }
    })
  }

  // 修改打字机效果，添加滚动调用
  const { pause, resume } = useIntervalFn(() => {
    if (contentText.value.length > displayText.value.length) {
      displayText.value = contentText.value.slice(0, displayText.value.length + 1)
      if (!userScrolledUp.value) scrollToBottom()
    } else {
      pause()
    }
  }, 15)

  // 光标闪烁效果
  // useIntervalFn(() => {
  //   cursorVisible.value = !cursorVisible.value
  // }, 500)

  const emits = defineEmits(['showDsChat'])
  const back = () => {
    emits('closeDsChat')
  }
  const send = () => {
    isThinking.value = true
    start()
    oldContent.value = content.value
    content.value = ''
    fetchStream()
    scrollToBottom()
  }

  const fetchStream = async () => {
    try {
      controller.value = new AbortController() // 创建新控制器
      isFetching.value = true

      let data = {
        userId: '00022a00-11ae-d045-0a41-3511ff8b9fc0',
        model: 'deepseek-chat',
        content: oldContent.value,
      }
      let url = 'https://llm.sjrt.net/api/LLMCenter/AiChat'
      // 1. 发起请求并获取响应
      const response = await fetch(url, {
        method: 'POST',
        cache: 'no-cache',
        headers: {
          'Content-Type': 'application/json',
        },
        signal: controller.value.signal, // 添加中止信号
        body: JSON.stringify(data),
      })

      // 2. 获取可读流
      const reader = response.body.getReader()
      const decoder = new TextDecoder() // 用于将二进制数据解码为字符串

      // 循环读取数据
      while (true) {
        const { done, value } = await reader.read()

        // 如果流结束，退出循环
        if (done) break

        // 3. 处理数据块（此处将二进制数据转为字符串）
        const chunk = decoder.decode(value, { stream: true }).replace(/data: /g, '')
        // console.log('Received chunk:', chunk)

        if (chunk.indexOf('会话开始') != -1 || chunk.indexOf('会话完成') != -1) continue
        isThinking.value = false
        end()
        contentText.value += chunk // 追加到原始文本
        resume()
      }

      console.log('Stream finished')
    } catch (err) {
      if (err.name === 'AbortError') {
        console.log('请求被用户终止', contentText.value)
        if (contentText.value == '') contentText.value = '思考已停止'
      } else {
        console.error('请求出错:', err)
      }
    } finally {
      console.log('finally', contentText.value)
      isFetching.value = false
      controller.value = null
      isThinking.value = false
      addRecords()
      end()
    }
  }

  watch(
    () => props.showDsChat,
    newVal => {
      if (newVal) {
        nextTick(() => {
          if (mdMainRef.value) {
            mdMainRef.value.addEventListener('scroll', handleScroll)
            handleScroll()
          }
        })
        //
      } else {
        if (mdMainRef.value) {
          mdMainRef.value.removeEventListener('scroll', handleScroll)
        }
      }
    },
  )
</script>
<style lang="scss" scoped>
  .md-main {
    /* height: calc(100% - 100px);  */
    height: 560px;
    overflow-y: auto; /* 关键样式 */
    padding: 10px;
    display: flex;
    flex-direction: column;
    gap: 20px;
    width: 1000px;
    margin: 0 auto;
  }

  .user-input {
    /* position: absolute; */
    /* bottom: 0; */
    /* width: 100%; */
    padding: 10px;
    background: #1b2c3c;
  }
  .valvebox {
    width: 1300px;
    height: 804px;
    background: #1b2c3c;
    border: 2px solid #2776cf;
    position: absolute;
    top: 10%;
    left: 16%;
    z-index: 9999;
  }


  .markdown-body {
    box-sizing: border-box;
    min-width: 200px;
    max-width: 980px;
    margin: 0 auto 0 0;
    color: #fff;
    background: #1b2c3c;
    padding: 12px;
    table tr {
      background-color: #1b2c3c !important;
    }
  }

  @media (max-width: 767px) {
    .markdown-body {
      padding: 15px;
    }
  }
  .think {
    color: #fff;
    font-size: 16px;
    margin-left: 6px;
  }
  .md-box {
    position: relative;

    .content {
      width: 100%;
      text-align: end;
      height: 32px;

      position: relative;
      display: inline-block;
      right: 0;
      > span {
        max-width: 80%;
        display: inline-block;
        border-radius: 10px;
        padding: 2px 12px;
        line-height: 32px;
        font-weight: 400;
        font-size: 16px;
        color: #333333;
        background: #ffffff;
        max-width: 800px;
      }
    }
  }

  .user-input {
    width: 1000px;
    height: 120px;
    background: rgba(46, 129, 238, 0.1);
    border-radius: 20px;
    border: 1px solid #2e81ee;
    margin: 0 auto;
    position: relative;
    .send-msg {
      /* 移除边框和背景 */
      border: none;
      background: transparent;
      outline: none;

      /* 设置字体样式 */
      font-size: 14px;
      color: #fff;
      line-height: 1.5;
      font-weight: 400;

      /* 调整内边距 */
      padding: 8px;
      width: 94%;
      height: 80px;
      resize: none;
    }
    /* 移除焦点时的轮廓 */
    .send-msg:focus {
      box-shadow: none;
      border-color: transparent;
    }

    /* 可选：调整placeholder颜色 */
    .send-msg::placeholder {
      color: #a0aec0;
      font-size: 14px;
    }
    .icon {
      right: 12px;
      bottom: 6px;
      position: absolute;
      .send,
      .stop {
        cursor: pointer;
      }
      .unsend {
        cursor: not-allowed;
      }
    }
  }


  .scroll-bottom-btn {
    position: absolute;
    right: 24px;
    bottom: 192px; // 根据输入框高度调整
    width: 32px;
    height: 32px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.3s;
    z-index: 10000;

    &:hover {
      transform: scale(1.1);
    }
  }
  img {
    width: 32px;
    height: 32px;
  }
</style>
