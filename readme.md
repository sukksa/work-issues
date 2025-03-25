## chrome 插件

[What Font - find font](https://link.juejin.cn/?target=https%3A%2F%2Fchrome.google.com%2Fwebstore%2Fdetail%2Fwhat-font-find-font%2Fdjgfpbegnihdgbngpmhjnlchgglngcdn)

[CSS Peeper](https://link.juejin.cn/?target=https%3A%2F%2Fchrome.google.com%2Fwebstore%2Fdetail%2Fcss-peeper%2Fmbnbehikldjhnfehhnaidhjhoofhpehk)

[Image downloader](https://link.juejin.cn/?target=https%3A%2F%2Fchrome.google.com%2Fwebstore%2Fdetail%2Fimage-downloader%2Fkdbfjpagopjjaiofmgodphiklmjhcnok)

[FeHelper](https://link.juejin.cn/?target=https%3A%2F%2Fchrome.google.com%2Fwebstore%2Fdetail%2Ffehelper%E5%89%8D%E7%AB%AF%E5%8A%A9%E6%89%8B%2Fpkgccpejnmalmdinmhkkfafefagiiiad)

## index.html通过script加载js文件

在vue3中，index.html通过script标签引入天地图的js文件，在network中可以见到js文件是被成功引入了的。然而在组件中获取，结果是undifined

```js
console.log(window.T) // undifined
```

问题是vue实例化之后，在加载的js文件，可以在main.js中通过promise异步加载

```js
const app = createApp(App)
const loadTMap = () => {
  return new Promise((resolve, reject) => {
    if (window.T) return resolve(true)

    const script = document.createElement('script')
    script.src = `https://api.tianditu.gov.cn/api?v=3.0&tk=ceaba4c55e6f7b1ef64c662f1348c90d`
    script.async = true
    script.defer = true

    script.onload = () => {
      resolve(true)
    }

    script.onerror = (error) => {
      console.error('加载失败:', error)
      reject(error)
    }

    document.head.appendChild(script)
  })
}

// 在 Vue 实例化前加载
loadTMap()
  .then(() => {
    app.config.globalProperties.$echarts = echarts

    app.use(router)
    app.use(ElementPlus)
    app.use(scroll)
    app.mount('#app')
  })
  .catch((error) => {
    console.error('应用初始化失败:', error)
    // 可在此处显示错误提示
  })
```

## vue3 列表无缝滚动组件 vue3-seamless-scroll

[vue3-seamless-scroll](https://github.com/xfy520/vue3-seamless-scroll)

## 小程序原生实现ai问答

markdown插件：towxml

html

```html
<towxml nodes="{{article}}" />
```

js

```js
aiSearch() {
  this.setData({
    isCreate: true
  })
  this.data.aibasetext = ''
  let token = wx.getStorageSync("token")
  let userid = wx.getStorageSync('mainInfo').user_id

  this.setData({
    article: '',

  })
  let param = {
    userId: userid,
    model: "deepseek-chat",
    content: this.data.content
  }
  // 点击发送自动下滑
  if (this.data.historyList.length > 0) wx.pageScrollTo({
    scrollTop: 99999, // 设置足够大的数值确保滚动到底部
    duration: 300 // 滚动动画时长（ms）
  })

  const requestTask = wx.request({
    url: 'https://llm.sjrt.net/api/LLMCenter/DsChat',
    method: 'POST',
    responseType: "arraybuffer",
    enableChunked: true, // 分块
    header: {
      'content-type': 'application/json'
    },
    data: param,
    success: function (res) {
      console.log(res);
    },
    fail: (err) => {
      console.log("request fail", err);
    },
  })
  // 全局 requestTask，点击停止时清除
  this.setData({
    currentRequestTask: requestTask // 保存请求任务
  });
  // 监听请求头接受事件
  requestTask.onHeadersReceived(r => {});
    
  // 监听数据分块接收事件
  requestTask.onChunkReceived(async (response) => {

    // 收到流式数据，根据返回值进行相对应数据解码
    let data16 = new Uint8Array(response.data)
    const str = new TextEncoding.TextDecoder('utf-8').decode(data16);
    if (str.indexOf('会话完成') != -1) {
      this.data.currentRequestTask.abort()
      this.data.currentRequestTask = null
    }
    const trimSessionMarkers = (result) => {
      result = result.replace(/会话开始/g, "");
      result = result.replace(/会话完成/g, "");
      result = result.replace(/data:/g, "");
      return result;
    }

    let trimmed = trimSessionMarkers(str);
    if (trimmed.trim() == '') {
      return
    }

    this.data.aibasetext = this.data.aibasetext + trimmed

    if (!this.data.isTyping && this.data.isCreate) {
      this.setData({
        isTyping: true
      });
      // 等待打字完成
      await this.startTyping(this.data.aibasetext, 0);
      this.setData({
        isTyping: false,
        isCreate: false
      });
      this.data.aibasetext = ''
    }
    return
  });
}

// 停止
stopSearch() {
  if (this.data.currentRequestTask) {
    this.data.currentRequestTask.abort(); // 终止请求
    this.data.currentRequestTask = null
  }
  clearInterval(this.data.typingIntervalData);
  if (this.data.article.length == 0) {
    this.data.article.push(app.towxml('思考已停止', 'markdown'))
    this.setData({
      article: this.data.article
    })
  }
  this.data.aibasetext = ''
  this.data.isTyping = false
  this.data.content = ''
}

// 打字机效果
startTyping(text, index) {
  return new Promise((resolve) => {
    let currentIndex = 0;
    // 打字速度
    const intervalTime = 50;
    let msg = '';
      
    const typingInterval = setInterval(() => {
      // aibasetext 是通过接口拿到的文本，在分块的接口中将后续的text拼接到aibasetext上，整个流程只有打开一次计时器
      if (currentIndex < this.data.aibasetext.length) {
        let currentChar = this.data.aibasetext.charAt(currentIndex);
        msg += currentChar;
        // towxml插件解析markdown语法
        this.data.aitext = app.towxml(msg, 'markdown');
        this.setData({
          article: this.data.aitext
        });
        currentIndex++;
        // isScroll 判断是否在屏幕底部，下滑滚动
        if (this.data.isScroll) {
          wx.pageScrollTo({
            scrollTop: 99999,
            duration: 0
          });
        }
      } else {
        clearInterval(typingInterval);
        this.setData({
          isTyping: false,
        });
        resolve(); // 完成时resolve
      }
    }, intervalTime);
    // 全局定时器，点击 stop 清除停止打字
    this.data.typingIntervalData = typingInterval
  });
}

// 监听页面滚动，在底部时自动滚动，向上滑停止滚动
onPageScroll: function (e) {
  const scrollTop = e.scrollTop // 当前滚动位置
  const windowHeight = wx.getSystemInfoSync().windowHeight // 屏幕高度
  const query = wx.createSelectorQuery()
  query.select('.subM').boundingClientRect() // 滚动容器
  query.exec(res => {
    const contentHeight = res[0].height // 内容总高度
    // 判断逻辑（+10是为了容错）
    const isBottom = scrollTop + windowHeight >= contentHeight - 10
    if (isBottom) {
      // 触底处理逻辑
      this.data.isScroll = true
    } else {
      this.data.isScroll = false
    }
  })
}
```

