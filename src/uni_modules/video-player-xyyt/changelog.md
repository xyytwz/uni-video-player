# 变更记录

## 1.0.0

- 首个版本：跨端视频播放组件。
- 支持播放/暂停、全屏、快进快退、拖动进度。
- 适配 Android / iOS / 鸿蒙（App-Harmony）/ 微信小程序 / H5。
- 鸿蒙 / App 统一走 `uni.createVideoContext`，作用域传 `instance.proxy`；以 `@fullscreenchange` 为全屏成功判据。
- 自绘控制栏补 `pointer-events:auto`，全屏过渡 pause 守卫，进度用 `<slider>` 跨端统一。
