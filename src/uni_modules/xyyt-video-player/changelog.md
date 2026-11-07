# 变更记录

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

### 文档与元信息

- 产品定位调整为「鸿蒙为主目标平台、多端兼容」；readme 新增鸿蒙专章、平台验证状态表、示例工程说明、常见问题（FAQ）与反馈渠道。
- `repository.url` / `bugs.url` 填写 GitHub 仓库；displayName / description / keywords 同步鸿蒙定位。
