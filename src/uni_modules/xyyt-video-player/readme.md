# xyyt-video-player 鸿蒙跨端视频播放器

一个基于 uni-app（Vue3 `<script setup>`）的自定义视频播放组件。**主目标平台为鸿蒙（HarmonyOS NEXT / App-Harmony）**，同时兼容 App Android/iOS、微信小程序、H5。各端控制栏策略：

- **鸿蒙 / 微信小程序 / H5**：自定义控制栏（`cover-view` 容器 + 原生 `<slider>` 半透明进度条，含播放/暂停、快进快退、全屏按钮；未播放时叠加居中播放按钮）
- **App Android**：使用系统原生控件（无自定义控制栏与居中按钮，进度条/播放暂停/全屏均由系统提供）。真机实测：Android 上自定义控制栏（cover-view + slider 混合）会导致原生 video 黑屏且无法播放，已固定为原生控件方案
- **App iOS**：同 Android，使用原生控件（待真机验证）

各端渲染路径通过条件编译（`#ifdef APP-PLUS` / `#ifndef APP-PLUS`）完全隔离，针对某一端的样式调整不会影响其他端。

- 播放 / 暂停
- 全屏（鸿蒙 / Android / iOS / 微信小程序 / H5）
- +Ns / -Ns 快进快退（步长可配；Android 端由原生控件提供）
- 拖动进度条 seek（非 Android 端用原生 `<slider>`；Android 端由原生控件提供）
- 通过 `ref` 外部调用播放 / 暂停 / 跳转 / 全屏（全平台可用）

## 平台支持

**最低兼容版本：HBuilderX / uni-app 5.34**（`engines.HBuilderX`、`engines.uni-app` 与 `dcloudext.minHBuilderX` 同口径）。之所以不按官方的鸿蒙 Next 支持线（4.27）声明，是因为本插件仅在 HBuilderX 5.34 完成四端编译与真机验证，4.27 ~ 5.34 区间未实测，故不对外承诺兼容。

| 平台 | 支持 | 验证状态 | 说明 |
| --- | --- | --- | --- |
| App（鸿蒙 HarmonyOS NEXT） | ✅ 主目标 | **真机验证通过** | 原生渲染无 DOM，统一走 `createVideoContext`，作用域传 `instance.proxy`；横屏全屏 `direction:-90`；自定义控制栏（cover-view + 原生 slider） |
| App（Android） | ✅ 兼容 | **真机验证通过**，**必须使用自定义基座** | 使用系统原生控件（播放/暂停/进度条/全屏）；标准基座 SDK 版本与编译版本不一致时原生 video 可能渲染异常（视频界面不显示） |
| App（iOS） | ✅ 兼容 | 编译通过，**待真机验证** | 同 Android 使用原生控件；默认竖屏全屏 `direction:0`，可覆盖 |
| 微信小程序 | ✅ 兼容 | **开发者工具 + 真机验证通过** | 依赖 video 同层渲染，基础库最低 2.20.1（平台判定用 `getDeviceInfo`，见下方「最低运行环境版本」），自定义控制栏 + 原生 slider 进度条 |
| H5 | ✅ 兼容 | **浏览器验证通过** | 自定义控制栏补 `pointer-events:auto`，避免点击穿透；原生 slider 进度条 |
| vue2 | ❌ | - | 组件使用 `<script setup>`，需 vue3 |
| uni-app x 项目 | ❌ | - | 本组件是 uni-app（Vue3 SFC）实现，依赖 `cover-view` 内嵌 `slider`、`uni.createVideoContext(id, instance.proxy)` 等在 uni-app x 下未验证的能力，**uni-app x 项目请勿安装**（`engines.uni-app-x` 已置空，平台矩阵中 uni-app x 各端均声明为不支持） |

### 最低运行环境版本

指用户设备/基础库的下限，与上面的编译器版本（5.34）是两个维度：

| 平台 | 最低版本 | 依据 |
| --- | --- | --- |
| 鸿蒙 HarmonyOS NEXT | **API 12**（对应 HarmonyOS NEXT 5.0） | uni-app 鸿蒙项目要求系统版本 API 12 以上。DevEco Studio 模拟器需 API 19+ 镜像属工具限制，不是设备下限，勿误填 |
| App Android | **5.0**（API level 21） | uni-app 云端打包 `minSdkVersion` 默认 21，本示例工程未改写该配置 |
| 微信小程序 | **基础库 2.20.1** | 组件微信分支调用 `uni.getDeviceInfo()`，官方标注微信小程序 2.20.1+。低于该版本取不到 `platform`，iOS 机型的全屏方向会退化为横屏 `-90` |
| App iOS | 不声明 | 仅编译通过，**无真机验证**，不对外承诺具体 iOS 版本 |

> 上传插件市场时这三个数值填进 HBuilderX「平台兼容性」面板，由 IDE 按 4.71+ 新结构将 `minVersion` 回写到本插件 `package.json`；仓库内的 `platforms` 仍为旧结构裸标记（`y`/`n`），**不要手工改写为新结构**，以免与 IDE 预期层级不符被打包校验拦截。

## 鸿蒙（主目标平台）说明

- **运行形态**：uni-app App-Harmony（HarmonyOS NEXT），页面为原生渲染、不存在可脚本化 DOM，因此组件全程不使用 `document` / `requestFullscreen`，播放控制统一走 `uni.createVideoContext(id, instance?.proxy)`。
- **全屏方向**：横屏 `direction:-90`（与 App Android 一致），可用 `fullScreenDirection` 覆盖；全屏成功以 `@fullscreenchange` 回调为唯一判据。
- **系统下限**：HarmonyOS NEXT **API 12** 以上（uni-app 鸿蒙项目的统一要求）；DevEco Studio 内置模拟器需 API 19+ 镜像才能运行，属工具限制。
- **运行与验证**：HBuilderX「运行 → 运行到手机或模拟器 → 鸿蒙」真机运行；或 CLI `npm run dev:app-harmony` / `npm run build:app-harmony`（产物 `dist/build/app-harmony`，配合 DevEco Studio / 鸿蒙基座导入）。
- **当前验证状态**：真机验证通过；全屏横竖屏与触摸拖动均正常工作。

## 安装

### 方式一：uni-app 插件市场
在 HBuilderX 中「插件市场」搜索 `xyyt-video-player`，下载到当前项目的 `src/uni_modules` 目录。需 HBuilderX ≥ 5.34 的 uni-app（非 uni-app x）工程。

### 方式二：手动拷贝
将 `uni_modules/xyyt-video-player` 整个目录复制到你的项目 `src/uni_modules/`（或根 `uni_modules/`）下即可。
组件遵循 easycom 规范，`components/xyyt-video-player/xyyt-video-player.vue` 会被自动识别，**无需手动 import**。

## 使用

```vue
<template>
  <xyyt-video-player
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

## 示例工程（本仓库）

仓库根目录即**完整项目示例**（可作为「完整项目示例」形态直接编译运行）：

- 示例页：`src/pages/index/index.vue`，覆盖全部事件监听与 `ref` 外部调用（播放 / 暂停 / seek）
- 运行：`npm install` 后按平台执行 `npm run dev:app-harmony`（鸿蒙，主目标）/ `dev:app` / `dev:h5` / `dev:mp-weixin`；构建用对应 `build:*` 脚本
- 示例页 `videoUrl` 为公网测试地址，实际使用请替换为业务视频地址

## 常见问题（FAQ）

1. **视频区域不显示 / 高度为 0**：组件容器以 `padding-bottom: 56.25%` 内置 16:9 宽高比兜底（比 `aspect-ratio` 在 Android 原生 video 上更可靠），父级未定高时也能正常撑开；需自定义比例或高度时，直接覆盖外层容器的 `height` / `padding-bottom` 即可。
2. **H5 / 小程序 / 鸿蒙端进度条能否拖动**：支持。这三端进度条为原生 `<slider>` 组件（松手 `change` 事件触发 seek）；Android 端进度条由系统原生控件提供。
3. **微信开发者工具点全屏无效并报 `Permissions policy violation: fullscreen is not allowed`**：工具环境限制（Chromium iframe 未授予 fullscreen），非组件缺陷，详见「注意」第 1 条；请真机验证。
4. **App / 小程序端控制栏与底部有空隙**：安全区 `padding-bottom` 仅 H5 条件编译生效，App/小程序端无此问题；若仍见空隙请确认插件版本与构建产物为最新。
5. **微信端控制台 `wx.getSystemInfoSync is deprecated` 告警**：微信分支已改用 `uni.getDeviceInfo()` 并缓存判定结果，本组件不会触发该告警。
6. **全屏旋转方向与预期不符**：各端 direction 语义不一致（微信 `90` 为向左旋转横屏，App/鸿蒙 `-90` 为横屏），组件已按端内置正确取值；特殊需求用 `fullScreenDirection` 覆盖。
7. **Android 端控制栏样式与 H5/鸿蒙不一致 / 没有自定义进度条**：设计如此。Android 端使用系统原生控件，样式由系统决定、不可自定义（cover-view + 原生 slider 的自定义栏在 Android 真机会导致原生 video 黑屏无法播放）。H5/鸿蒙/微信小程序仍为自定义控制栏。
8. **vue2 项目能否使用**：不能。组件使用 `<script setup>`，仅支持 vue3。
9. **同一页面能否放多个播放器**：可以。video id 按组件实例唯一生成（instance.uid 后缀），`createVideoContext` 配合实例作用域各自定位自己的 video，互不串扰。
10. **uni-app x 项目能否使用**：不能。本组件未提供 uvue/UTS 实现，`uni-app x` 维度已在元信息中声明为不支持；如确有 uni-app x 需求请开 Issue 说明场景。

## 关键实现说明（跨端踩坑）

1. **鸿蒙 / App 原生渲染没有可脚本化 DOM**：`.nvue` 或鸿蒙原生渲染下 `document` 不存在，不能用 `getElementById` / `requestFullscreen`。统一用 `uni.createVideoContext(id, scope)` 控制。
2. **`<script setup>` 里作用域必须传组件代理实例**：`uni.createVideoContext(id, instance?.proxy)`。若传内部 `instance`，`requestFullScreen` 等原生方法会**静默失效且无任何报错**。
3. **全屏方向**：iOS 竖屏(0)；微信小程序横屏用 `90`（向左旋转，实测 `-90` 为向右旋转），其余非 iOS 端横屏用 `-90`；均可用 `fullScreenDirection` 覆盖。
4. **自绘覆盖层需 `pointer-events:auto`**：uni-app H5/App-webview 的视频容器默认 `pointer-events:none`，插槽内按钮/进度条需显式 `auto` 才能命中点击，否则会穿透到 video。
5. **全屏过渡 pause 守卫**：进出全屏时原生会派发中间态 `pause`，用时间锁吞掉，过渡后按“切换前意图”恢复播放，避免状态抖动或卡在暂停。
6. **进度条方案**：H5/鸿蒙/微信小程序在 `cover-view` 容器内使用原生 `<slider>` 拖动 seek（`activeColor` 需硬编码色值——模板属性绑定不经过 SCSS 编译，变量不生效）。
7. **平台隔离结构**：`appNativeControls` 是 `<script setup>` 顶层条件编译常量（APP-PLUS 为 true，其余为 false，不可写进函数体内）；video 的 `controls` 直接绑该常量，自定义控制栏整块由模板 `<!-- #ifndef APP-PLUS -->` 包裹。针对某一端的调整只改对应分支，不得改动另一端的渲染路径。
8. **`show-center-play-btn` 是独立开关**：与 `controls` 互不影响，`controls=false` 不会关闭原生中央播放按钮。本组件全平台固定 `false`：非 Android 端避免与自定义居中按钮叠加成双层；Android 原生控件自带中央按钮、不受该属性影响。不要把它写成 `!appNativeControls` 之类的联动表达式。

## 注意

- **Android 真机调试必须先制作自定义基座**：标准基座（手机上的 HBuilderX App）SDK 版本固定，若与项目编译版本不一致（控制台提示"本应用使用 HBuilderX x.xx 编译，手机端 SDK 版本是 y.yy"），原生 `<video>` 组件会出现渲染异常（视频界面不显示）。解决：HBuilderX → 运行 → 运行到手机或模拟器 → **制作自定义调试基座** → 再选择"运行到 Android App 自定义基座"。
- **微信开发者工具（Windows 模拟器）无法验证全屏**：工具的页面渲染运行在 iframe 中，Chromium Permissions Policy 未授予 fullscreen，任何 `requestFullScreen` 调用（包括微信原生 video 控件自带的全屏按钮）都会被拦截并报 `Permissions policy violation: fullscreen is not allowed in this document`。这是开发者工具的环境限制而非组件缺陷，组件调用链已正常到达渲染层且会安全降级（`isFullScreen` 保持 false，播放状态不受影响）。**全屏功能请通过「预览」扫码或「真机调试」在手机上验证**。
- **示例工程的鸿蒙 bundleName 需替换**：`src/manifest.json` 的 `app-harmony.distribute.bundleName`（当前为 `uni.video.player.demo`）仅用于本示例，实际项目请改成你自己应用的包名；bundleName 一旦上架华为侧不可变更。
- 微信小程序对 `<video>` 上覆盖普通组件依赖**同层渲染**，请使用较新基础库；极旧版本可能需改用 `cover-view`。
- 示例工程根 `index.html` / `manifest.json` 未内置任何真实后端与 appid，请自行替换。

## 反馈与版本

- 问题反馈：[GitHub Issues](https://github.com/xyytwz/uni-video-player/issues)
- 源码仓库：[github.com/xyytwz/uni-video-player](https://github.com/xyytwz/uni-video-player)
- 版本历史：见同目录 `changelog.md`；版本号以 `package.json` 的 `version` 为准，二者同步维护
