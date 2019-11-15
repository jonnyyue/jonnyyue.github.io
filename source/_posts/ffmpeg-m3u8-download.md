---
title: "[音视频]下载m3u8格式的视频文件"
top: false
cover: false
toc: true
mathjax: false
date: 2019-11-12 11:37:27
password:
summary:
tags:
 - ffmpeg
 - m3u8
categories:
 - 周边
---

#### 0. 网页视频播放器的演进

&nbsp;&nbsp;&nbsp;&nbsp;随着浏览器的逐步迭代，网页播放视频方案也一直不断演进。从早期的MediaPlayer到Flash，再到video标签，我们可以更方便的在网页中播放视频。


#### 1. HLS协议

&nbsp;&nbsp;&nbsp;&nbsp;在浏览器中，HLS协议的video标签是没有保存功能的([源码地址](https://cs.chromium.org/chromium/src/third_party/blink/renderer/core/html/media/html_media_element.cc?rcl=a0829135f51b120470ab845efc586058d1d96d57&l=1996))。所以不能直接在浏览器中下载媒体文件，但是仅仅这一点是不够的，用户可以方便的拿到云存储上的视频地址，下载到本地即可以播放。所以HLS又提供了一套视频流加密的方案，这样云存储中存放的视频流也是加密的，由浏览器负责一边进行解密一边播放。

&nbsp;&nbsp;&nbsp;&nbsp;M3U8作为HLS协议的载体，也是我们后续分析的主要对象。详细的M3U8文件格式的介绍网上有很多，可以参照文档[m3u8文件格式详解](https://www.jianshu.com/p/e97f6555a070)

下面列出一段未加密的样例[传送门](https://node.imgio.in/demo/birds.m3u8)

![没有加密的M3U8视频流](1.png)

可以使用ffmpeg工具将视频保存到本地

```shell
ffmpeg -protocol_whitelist crypto,file,tcp,http,https,tls -i "https://node.imgio.in/demo/birds.m3u8" -c  copy  -copyts "birds.ts"
```

含有加密KEY的样例[传送门](https://cdn.theoplayer.com/video/big_buck_bunny_encrypted/stream-800/index.m3u8)

![加密的M3U8视频流](2.png)

因为样例中的解密KEY没有做鉴权，我们同样可以将视频保存到本地

```shell
ffmpeg -protocol_whitelist crypto,file,tcp,http,https,tls -i "https://cdn.theoplayer.com/video/big_buck_bunny_encrypted/stream-800/index.m3u8" -c  copy  -copyts "index.ts"
```


#### 2. 加密策略

&nbsp;&nbsp;&nbsp;&nbsp;浏览器在解析M3U8协议文件时候，如果发现文件中指定了加密策略，就使用当前的网络环境访问该地址，获取对应的解密KEY。从云存储中读取到ts流之后，使用KEY和IV进行解密，然后在网页上播放。由于视频流一般是部署在OSS或者CDN，方便让用户下载，通常这类服务器是不会做鉴权策略的，于是对视频的保护就转变成了对解密KEY的保护。

##### 2.1 refer检测

&nbsp;&nbsp;&nbsp;&nbsp;这是比较粗糙的方案，如果请求来源不是来自可信的域名，就直接返回错误。我们直接用ffmpeg工具操作M3U8时，访问解密KEY的URI是没有refer的，所以不能直接下载。

##### 2.2 用户鉴权

&nbsp;&nbsp;&nbsp;&nbsp;很多网校通常会用这种方案进行加密，这种方式可以根据权限管理来决策哪些用户可以获取到解密KEY。推荐一种比较方便的方案，通过fiddler进行抓包，然后用python脚本解析对应的saz，将m3u8中的URI换成对应的KEY，然后使用ffmpeg将媒体文件保存到本地。

##### 2.3 自定义KeyLoader

&nbsp;&nbsp;&nbsp;&nbsp;可以使用这个[组件](https://github.com/video-dev/hls.js)，支持自定义KeyLoader，通过URI从服务端请求回来一串字符串，经过定制的KeyLoader来解密之后，得到真正的解密KEY。这种需要我们的分析KeyLoader源码，百度云提供的加密视频方案就用了这个策略[练手地址](http://cyberplayer.bcelive.com/demo/new/index.html)，对key的解密用到了AES-ecb，对解密KEY进行解密的密钥写死在js代码中。

##### 2.4 自定义PlaylistLoader

&nbsp;&nbsp;&nbsp;&nbsp;除了可以自定义KeyLoader，这个组件也支持指定PlaylistLoader，这样M3U8文件也是动态解密出来的，增强了安全性[练手地址](https://coding.imooc.com/class/321.html)。通过动态调试的方式可以拿到明文的M3U8和对应的解密KEY。由于网站的解密算法比较复杂，笔者没有尝试提取算法，感兴趣的大神可以分析下。

#### 3. 实战下载百度的加密视频

> 这里使用Chrome浏览器进行演示，页面地址同目录2.3

##### 3.1 抓取关键信息

![打开开发者工具，定位到Network的tab页](3.png)

![点击播放token加密视频](4.png)

![触发了对m3u8文件的拉取](5.png)

![拉取视频流的解密KEY](6.png)

![通过关键字找到js文件中的KeyLoader](7.png)

##### 可以看到这里用到了AES-ecb解密，并且密钥也暴露在js文件中

![下断点，重新加载视频，即可看到解密KEY](8.png)

##### 3.2 构造可以方便下载视频的M3U8

&nbsp;&nbsp;&nbsp;&nbsp;如何让ffmpeg工具方便的拿到这个key？我们搭建一个服务端接口：接收一个base64编码后的参数，解码成二进制流返回给客户端。(可能比较搓的方案，但是好在还很通用)

##### 构造解密KEY的URI：

![将复制下来的解密KEY做base64编码](9.png)

![替换M3U8文件中的URI部分](10.png)

##### 3.3 使用ffmpeg工具下载媒体文件

```shell
ffmpeg -protocol_whitelist crypto,file,tcp,http,https,tls -i "baidu.m3u8" -c  copy  -copyts "baidu.ts"
```

![下载后的视频可以正常播放](11.png)

#### 4. 总结

&nbsp;&nbsp;&nbsp;&nbsp;在线教育成为了互联网的风口，HLS协议也很好的契合了这个行业。最后呼吁大家多多尊重版权，从正规渠道购买视频资源。创作不易。



仅技术交流，请勿使用于非法用途。
