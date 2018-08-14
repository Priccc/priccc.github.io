---
title: 利用 canvas 实现字符流视频
categories: [javascript]
tags: [抖音, canvas, javascript]
date: 2018-8-14
---

最近刷抖音越来越频繁，在里面页发现了不少技术相关的视频，今天这里说到的*字符流视频*就是其中之一

一开始我也是没有思路的，毕竟 canvas 基本没用过，只能在 百度、Google 上寻求答案...

### 实现思路
1. 利用 `input` 获取到视频，并转成 `HTMLVideDOM`
2. 将视频通过 `drawImage()` 画到 canvas 上，并通过 `getImageData()` 获取画布上的信息，计算灰度值，并替换成字符，越深替换的字符越密集
3. 将替换好的字符视频画到画布上
4. 更改视频的 `currentTime` 属性，通过 `window.requestAnimationFrame()` 重复 2、3 步

> 思路借鉴自 [js视频转字符画 —— 写一个属于自己的字符转换器](https://juejin.im/post/5b5ec60d6fb9a04f8a219a1d)

<!-- more -->
### 页面初始化

初始化页面大体布局，全局变量，CSS 样式看个人喜好

```
  ...
  <div id="container">
    <input type="file" id="input-file" accept=".mp4" />
    <div id="video-box">
        <canvas id="canvas"></canvas>
        <canvas id="canvas-show"></canvas>
    </div>
  </div>

  <script>
    const inputFile = document.getElementById('input-file'); // input
    const canvas = document.getElementById('canvas'); // 画原始视频的 canvas
    const canvasShow = document.getElementById('canvas-show'); // 画字符流视频的 canvas
    const ctx = canvas.getContext('2d');
    const ctxShow = canvasShow.getContext('2d');
    const size = { w: 0, h: 0 }; // 视频大小
    let afid = null; // requestAnimationFrame 的 id
    let videoDom = document.createElement("VIDEO");

    ...
  </script>
```

### 利用 `input` 获取到视频，并转成 `HTMLVideDOM`

通过 `input` 的 `change` 事件，监听文件是否被上传了，并将上传的视频信息获取到，通过 `URL.createObjectURL()` 生成一个视频地址，并赋值给穿件好的 `video` 元素

`await new Promise(res => videoDom.addEventListener('canplay', res));` 是为了等待视频被加载完成

最后规定一下视频的大小，开始执行动画操作

```
  inputFile.addEventListener('change', async ({ target: { files } }) => {
    const file = files[0];
    const url = URL.createObjectURL(file);

    videoDom.src = url;
    await new Promise(res => videoDom.addEventListener('canplay', res));
    
    const { videoHeight, videoWidth } = videoDom;
 
    size.w = videoWidth * 0.5;
    size.h = videoHeight * 0.5;

    canvas.width = size.w;
    canvas.height = size.h;
    canvasShow.width = size.w;
    canvasShow.height = size.h;

    await playVideo();
  });
```

### 执行动画操作
此部分是动画部分，主要是修改视频的当前时间，执行函数，并调用下次动画

```
  const playVideo = async ({
    currentTime = 0,
    curTime = Date.now(),
    prevTime = Date.now(),
    prevProgress,
  } = {}) => {
    videoDom.currentTime = currentTime;
    await new Promise(res => videoDom.addEventListener('canplay', res));
    ctx.drawImage(videoDom, 0, 0, size.w, size.h);
    replaceImage();

    let progress = Math.max(curTime - prevTime, 16) / 1000;
    const nextTime = currentTime + progress;

    if (nextTime >= videoDom.duration) {
      return clearVideo();
    }

    afid = window.requestAnimationFrame(() => playVideo({
      currentTime: nextTime,
      curTime: Date.now(),
      prevTime: curTime,
      prevProgress: progress,
    }));
  }
```

### 读取画布信息，并替换
这是核心内容，大体思路是获取画布信息，然后计算灰度值，根据灰度值替换成相应的字符

但是如何计算灰度值呢 ？我也是百思不得其解啊，只能靠他人了，还别说，网上关于 canvas 计算灰度值的信息一大堆，公式为：`gray color = 0.299 × red color + 0.578 × green color + 0.114 * blue color` ，大体如下
```
  for (let _h = 0; _h < h; _h += 6) {
    for (let _w= 0; _w< w; _w += 6) {
      const index = (_w + w * _h) * 4;
      const r = data[index + 0];
      const g = data[index + 1];
      const b = data[index + 2];
      const gray = .299 * r + .587 * g + .114 * b;
    }
  }
```

然后再根据当前的灰度值替换成相应的字符 `replaceText()`

```
  const replaceImage = () => {
    const { w, h } = size;
    const { data } = ctx.getImageData(0, 0, w, h);

    ctxShow.clearRect(0, 0, w, h);
    for (let _h = 0; _h < h; _h += 6) {
      for (let _w= 0; _w< w; _w += 6) {
        const index = (_w + w * _h) * 4;
        const r = data[index + 0];
        const g = data[index + 1];
        const b = data[index + 2];
        const gray = .299 * r + .587 * g + .114 * b;
        ctxShow.fillText(replaceText(gray), _w, _h + 8);
      }
    }
  }

  const replaceText = (g) => {
    const textList = ['#', '&', '@', '%', '$', 'w', '*', '+', 'o', '?', '!', ';', '^', ',', '.', ' '];
    const i = g % 16 === 0 ? parseInt(g / 16) - 1 : parseInt(g / 16);
    return textList[i];
  }
```

### 效果图
到现在为止，功能就已经实现了，小伙伴们可以去尝试一下效果吧...
![效果图](/../images/post/string-video-1.png)