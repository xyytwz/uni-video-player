# 变更记录

## 1.0.0

### 功能与界面

- 首个对外发布版本：跨端视频播放组件（鸿蒙为主目标平台，兼容 App Android/iOS、微信小程序、H5）。
- 支持播放/暂停、全屏、快进快退（步长可配）、拖动进度条 seek；`ref` 暴露播放/暂停/seek/快进快退/全屏等 9 个方法。
- H5 / 鸿蒙 / 微信小程序：自定义控制栏（`cover-view` 容器 + 原生 `<slider>` 半透明进度条），CSS 绘制播放/暂停图标、‹15/15› 步进胶囊、全屏切换按钮、按压/悬停反馈；未播放时叠加居中播放按钮。
- App Android / iOS：使用系统原生控件（真机实测 Android 自定义控制栏会导致原生 video 黑屏无法播放，故固定原生控件方案）；样式由系统决定、功能完整。
- 各端渲染路径通过条件编译完全隔离，针对某一端的调整不影响其他端。

### 跨端适配与修复

- 鸿蒙 / App 统一走 `uni.createVideoContext`，作用域传 `instance.proxy`；以 `@fullscreenchange` 回调为全屏成功唯一判据；全屏过渡 pause 守卫；自绘覆盖层补 `pointer-events:auto`。
- 全屏方向按端内置：iOS 竖屏(0)、微信小程序横屏 90（向左旋转）、其余横屏 -90；可用 `fullScreenDirection` 覆盖。
- 微信小程序平台判定使用 `uni.getDeviceInfo()`（基础库 ≥ 2.20.1），无 `getSystemInfoSync` 废弃告警。
- video id 按组件实例唯一生成，同一页面多播放器互不串扰；seek 对原生层静默失败有一次延迟重试（定时器随组件卸载清理）。
- 容器 `padding-bottom: 56.25%` 内置 16:9 宽高比兜底；安全区 `padding-bottom` 仅 H5 条件编译生效，App/小程序端无底部空隙。
- `show-center-play-btn` 全平台固定关闭，避免原生中央按钮与自定义按钮叠加。

### 文档与元信息

- readme 含平台验证状态表、鸿蒙专章、Props/Events/方法、FAQ（10 条）与跨端踩坑说明；`uni_modules.platforms` 支持矩阵按实际验证结果声明。
- 明确对外兼容边界：`engines.HBuilderX` / `engines.uni-app` / `dcloudext.minHBuilderX` 统一声明为 5.34（唯一完成四端编译与真机验证的编译器版本）；`engines.uni-app-x` 置空并将 uni-app x 各端声明为不支持（无 uvue/UTS 实现）；nvue 同样声明不支持。
- readme 新增「最低运行环境版本」表：鸿蒙 HarmonyOS NEXT API 12+、App Android 5.0+（API 21）、微信小程序基础库 2.20.1+（受 `getDeviceInfo` 约束）；iOS 因未真机验证不声明版本。`minVersion` 由市场上传面板写入，仓库内 `platforms` 保持旧结构。
- 示例工程 `manifest.json` 的 `app-harmony.distribute.bundleName` 由其他项目遗留的占位值修正为 `uni.video.player.demo`，并在 readme「注意」中说明实际项目需自行替换。
- Android 真机调试必须先制作自定义基座（标准基座 SDK 与编译版本不一致时原生 video 渲染异常），已在「注意」章节说明。
- `repository.url` / `bugs.url` 填写 GitHub 仓库。
