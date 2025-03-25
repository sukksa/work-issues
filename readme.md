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
