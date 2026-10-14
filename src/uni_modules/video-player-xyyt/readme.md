# video-player-xyyt 跨端视频播放器

一个基于 uni-app（Vue3 `<script setup>`）的自定义视频播放组件，隐藏各端原生控件、统一自绘控制栏，支持：

- 播放 / 暂停
- 全屏（Android / iOS / 鸿蒙 App-Harmony / 微信小程序 / H5）
- +Ns / -Ns 快进快退（步长可配）
- 拖动进度条 seek
- 通过 `ref` 外部调用播放 / 暂停 / 跳转 / 全屏

## 平台支持

| 平台 | 支持 | 说明 |
| --- | --- | --- |
| App（Android） | ✅ | 走 `uni.createVideoContext`，横屏全屏 `direction:-90` |
| App（iOS） | ✅ | 默认竖屏全屏 `direction:0`，可覆盖 |
| App（鸿蒙 Harmony） | ✅ | 原生渲染无 DOM，统一走 `createVideoContext`，作用域传 `instance.proxy` |
| H5 | ✅ | 自绘覆盖层补 `pointer-events:auto`，避免点击穿透 |
| 微信小程序 | ✅ | 依赖 video 同层渲染（基础库建议 ≥ 2.10.0），进度用 `<slider>` |
| vue2 | ❌ | 组件使用 `<script setup>`，需 vue3 |

## 安装

### 方式一：uni-app 插件市场
在 HBuilderX 中「插件市场」搜索 `video-player-xyyt`，下载到当前项目的 `src/uni_modules` 目录。

### 方式二：手动拷贝
将 `uni_modules/video-player-xyyt` 整个目录复制到你的项目 `src/uni_modules/`（或根 `uni_modules/`）下即可。
组件遵循 easycom 规范，`components/video-player-xyyt/video-player-xyyt.vue` 会被自动识别，**无需手动 import**。

## 使用

```vue
<template>
  <video-player-xyyt
    ref="player"
    :src="url"
    :poster="poster"
    :autoplay="false"
    :speed-step="15"
    @fullscreenchange="onFs"
    @timeupdate="onTime"
  />
  <button @click="player?.seek(30)">跳到 30s</button>
</template>

<script setup lang="ts">
import { ref } from 'vue';
const player = ref();
const url = ref('https://your-cdn/xxx.mp4');
function onFs(e) { console.log('全屏状态', e.fullScreen); }
function onTime(e) { console.log(e.currentTime, e.duration); }
</script>
```

## Props

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| src | string | 必填 | 视频地址 |
| poster | string | '' | 封面图 |
| autoplay | boolean | false | 是否自动播放 |
| muted | boolean | false | 是否静音 |
| loop | boolean | false | 是否循环 |
| initialTime | number | 0 | 初始播放位置（秒） |
| objectFit | 'contain'\|'fill'\|'cover' | 'contain' | 视频填充模式 |
| showFullScreen | boolean | true | 是否显示全屏按钮 |
| showSpeedButtons | boolean | true | 是否显示快退/快进按钮 |
| speedStep | number | 15 | 快退/快进步长（秒） |
| fullScreenDirection | number | 自动 | 全屏方向：0 竖屏，-90/90 横屏；不传按平台自动选择 |

## Events

| 事件 | 回调参数 | 说明 |
| --- | --- | --- |
| play | - | 开始播放 |
| pause | - | 暂停 |
| ended | - | 播放结束 |
| timeupdate | `{ currentTime, duration }` | 播放进度更新 |
| fullscreenchange | `{ fullScreen }` | 全屏状态变化（全屏成功的唯一判据） |
| error | 原生错误对象 | 加载失败 |

## 暴露方法（通过 ref 调用）

`play()` / `pause()` / `seek(seconds)` / `fastForward()` / `fastRewind()` / `enterFullScreen()` / `exitFullScreen()` / `toggleFullScreen()` / `togglePlay()`

## 关键实现说明（跨端踩坑）

1. **鸿蒙 / App 原生渲染没有可脚本化 DOM**：`.nvue` 或鸿蒙原生渲染下 `document` 不存在，不能用 `getElementById` / `requestFullscreen`。统一用 `uni.createVideoContext(id, scope)` 控制。
2. **`<script setup>` 里作用域必须传组件代理实例**：`uni.createVideoContext(id, instance?.proxy)`。若传内部 `instance`，`requestFullScreen` 等原生方法会**静默失效且无任何报错**。
3. **全屏方向**：iOS 竖屏(0)、Android/鸿蒙/小程序/H5 横屏(-90)，可用 `fullScreenDirection` 覆盖。
4. **自绘覆盖层需 `pointer-events:auto`**：uni-app H5/App-webview 的视频容器默认 `pointer-events:none`，插槽内按钮/进度条需显式 `auto` 才能命中点击，否则会穿透到 video。
5. **全屏过渡 pause 守卫**：进出全屏时原生会派发中间态 `pause`，用时间锁吞掉，过渡后按“切换前意图”恢复播放，避免状态抖动或卡在暂停。
6. **拖动进度用 `<slider>`**：各端一致，避免在原生组件上手写触摸坐标换算。

## 注意

- 微信小程序对 `<video>` 上覆盖普通组件依赖**同层渲染**，请使用较新基础库；极旧版本可能需改用 `cover-view`。
- 示例工程根 `index.html` / `manifest.json` 未内置任何真实后端与 appid，请自行替换。
