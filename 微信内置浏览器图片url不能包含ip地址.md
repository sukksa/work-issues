在开发微信小程序时发现在内嵌的web-view中, 图片的url包含ip地址时, 会导致web-view通过img标签引入的图片无法显示. 但是在小程序里通过image引入时能正常显示. 

```
// 无法显示
https://iotapplet.sjrt.net/camera/127.0.0.1_i.jpg 

// 正常显示
https://iotapplet.sjrt.net/camera/1.1.jpg
```

原因是微信官方更新了安卓微信浏览器内核, 新内核对http资源请求要求更为严格，导致部分开发者升级新内核后，出现使用 http 带 ip 地址时、或者不支持 https 升级的图片资源无法显示的问题。小程序中的web-view本质就是在微信内置浏览器中打开网页, 所以无法显示图片

[关于安卓微信 XWeb 内核版本升级到 111 新内核的公告](https://developers.weixin.qq.com/community/develop/doc/00002624d34d5098f1cf773e356001?blockType=1)

打印出安卓微信浏览器的UA可以看到, 目前浏览器的版本为`Chrome/138.0.7204.180`, 微信仍然未修复这个问题. 解决方案只有给图片重命名. 

```
UA: Mozilla/5.0 (Linux; Android 14; 2206123SC Build/UKQ1.231003.002; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/138.0.7204.180 Mobile Safari/537.36 XWEB/1380159 MMWEBSDK/20250201 MMWEBID/488 MicroMessenger/8.0.57.2820(0x28003956) WeChat/arm64 Weixin NetType/WIFI Language/zh_CN ABI/arm64 MiniProgramEnv/android
```

我原以为微信中图片url是host不能使用ip地址, 没想到是整个url路径都不能包含ip地址, 甚至是文件名. 由于图片是海康摄像头自动拍照上传到ftp服务器中的, 修改配置比较麻烦. 只有另起一台ftp再将保存后的图片重新重命名后上传到ftp上, 由此问题解决
