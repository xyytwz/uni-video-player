# 变更记录

## 1.0.2

### 平台控制栏策略隔离（按端条件编译）

- Android 退回系统原生控件（`controls=true`），自定义控制栏整块用 `#ifndef APP-PLUS` 隔离，各端渲染路径互不影响：H5/微信小程序/鸿蒙=自定义控制栏（浏览器/真机已验证），Android=原生控件（真机已验证），iOS=原生控件（待真机验证）。
- 背景：1.0.1 之后曾尝试四端统一 cover-view + 原生 slider 方案，Android 真机实测原生 video 黑屏且无法播放（H5/鸿蒙正常），该实验方案已完整回退。
- 平台差异一律用条件编译隔离，后续针对某一端的样式调整不得改动其他端的渲染路径。

### 非 Android 端控制栏优化

- 控制栏容器改为 `cover-view`，自绘进度条替换为原生 `<slider>`（半透明：activeColor `rgba(255,125,0,0.35)`、backgroundColor `rgba(255,255,255,0.35)`）。
- 关闭 `show-center-play-btn`，消除 H5 端原生中央播放按钮与自定义按钮叠加的双层显示。该属性与 `controls` 相互独立，全平台固定为 `false`，不随 Android 原生控件开关联动（联动会重新引入 H5 双层按钮）。
- 暂停图标竖条改为方形样式（加宽、减小圆角）。
- 移除 Android 端不再使用的自绘拖拽链与进度状态（touch/mouse 双通路、dragging、progressPercent 等），组件代码明显精简。

### 多实例支持与发布合规

- video id 改为按组件实例唯一生成（`instance.uid` 后缀），同一页面多个播放器不再因固定 id 导致 `createVideoContext` 串扰。
- `package.json` 的 `uni_modules.platforms` 支持矩阵按实际验证结果填写（市场打包校验以 `uni_modules` 下声明为准）：vue3/app-vue/Android/iOS/鸿蒙/H5 Chrome&Safari/微信小程序为支持；nvue、uni-app-x 及其他小程序为不支持或未验证。
- 发布前检查：全仓无 `console.*` 调试语句、manifest.json 无 signingConfigs 凭据残留、无本机用户名泄漏；四端构建 + type-check 全部通过。

## 1.0.1

### Android 端控制栏方案调整

- Android 端放弃 cover-view 自定义控制栏，改用系统原生控件（`:controls="true"`）。
- 原因：Android 端原生 `<video>` 渲染在独立原生层（高于 WebView），HTML 控件被永久遮挡；`cover-view` 是唯一能覆盖原生层的方案，但其 CSS 支持极其有限——不支持 `position: absolute` 子元素、不支持 flexbox、`inline-block` 也不可靠，经过多轮尝试均无法实现可用的自定义控制栏。
- 影响：Android 端控制栏样式由系统决定，无法自定义；功能完整（播放/暂停/进度条拖拽/全屏/快进快退）。
- 其他平台（鸿蒙/微信小程序/H5）不受影响，仍使用自绘控制栏。

### 文档更新

- 平台验证状态表更新：鸿蒙/Android/微信小程序/H5 均已真机验证通过，iOS 待验证。
- 新增 FAQ 条目说明 Android 端控制栏差异。
- 关键实现说明补充 Android 端改用原生控件的原因。

## 1.0.0

### 功能与界面

- 首个版本：跨端视频播放组件（鸿蒙为主目标平台，兼容 App Android/iOS、微信小程序、H5）。
- 支持播放/暂停、全屏、快进快退（步长可配）、拖动进度条 seek；`ref` 暴露播放/暂停/seek/快进快退/全屏等 9 个方法。
- 自绘控制栏：CSS 绘制播放/暂停图标、自绘触摸进度条（替换原生 `<slider>`）、‹15/15› 步进胶囊、四角全屏图标、按压/悬停反馈，主题色接入 `uni.scss` 变量。

### 跨端适配与修复

- 鸿蒙 / App 统一走 `uni.createVideoContext`，作用域传 `instance.proxy`；以 `@fullscreenchange` 为全屏成功判据；自绘覆盖层补 `pointer-events:auto`；全屏过渡 pause 守卫。
- 微信小程序：平台判定改用 `uni.getDeviceInfo()`（条件编译，基础库 ≥ 2.20.1）并缓存，消除 `getSystemInfoSync` 废弃告警；全屏方向改为 `90`（向左旋转横屏，实测 `-90` 向右旋转）。
- H5：容器 `aspect-ratio:16/9` 兜底修复父级未定高时的高度塌陷；进度条拖动 touch + 鼠标双通路。
- 修复 App / 小程序端控制栏底部空隙：安全区 `padding-bottom` 改为仅 H5 条件编译生效（App/小程序全屏由原生层接管，自绘栏不到屏幕底）。
- 控制栏高度定为 84rpx；H5 全屏保留刘海屏/手势条安全区预留。
- 全屏模式控制栏两轮缩小（最终 60rpx）：按钮 44rpx、播放键 40rpx、进度条热区 40rpx、滑块 20rpx、时间/步进/全屏图标字号同步收紧，视觉更紧凑。

### 文档与元信息

- 产品定位调整为「鸿蒙为主目标平台、多端兼容」；readme 新增鸿蒙专章、平台验证状态表、示例工程说明、常见问题（FAQ）与反馈渠道。
- `repository.url` / `bugs.url` 填写 GitHub 仓库；displayName / description / keywords 同步鸿蒙定位。
- 文档补充：Android 真机调试必须先制作自定义基座（标准基座 SDK 与编译版本不一致时原生 video 渲染异常）；平台表 Android 行同步标注。
