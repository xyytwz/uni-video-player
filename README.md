# uni-video-demo

`xyyt-video-player` 是以**鸿蒙（HarmonyOS NEXT / App-Harmony）为主目标平台**的跨端视频播放插件的**示例工程**，同一份源码兼容 **App（Android/iOS）**、**H5**、**微信小程序**。

插件本体位于：`src/uni_modules/xyyt-video-player/`，示例页位于 `src/pages/index/index.vue`。

## 平台定位与验证状态

| 平台 | 定位 | 验证状态 |
| --- | --- | --- |
| App（鸿蒙 HarmonyOS NEXT） | 主目标 | 编译验证通过（`build:app-harmony`），建议真机复验 |
| App（Android / iOS） | 兼容 | 编译验证通过，建议真机复验 |
| 微信小程序 | 兼容 | 开发者工具 + 真机实测通过 |
| H5 | 兼容 | 浏览器实测通过 |

各端差异与踩坑说明见插件文档：[`src/uni_modules/xyyt-video-player/readme.md`](src/uni_modules/xyyt-video-player/readme.md)。

## 环境

- HBuilderX（App / 鸿蒙真机运行推荐）或 CLI（Node ≥ 16）
- Vue3 + Vite + `@dcloudio/*` 3.0.0 系列（见 `package.json`）

## 运行（CLI）

```bash
# 安装依赖
npm install

# 鸿蒙（主目标平台，配合 HBuilderX 鸿蒙基座 / DevEco Studio）
npm run dev:app-harmony

# H5
npm run dev:h5            # 浏览器访问 vite 输出的本地地址

# 微信小程序（产物 dist/dev/mp-weixin，用微信开发者工具打开）
npm run dev:mp-weixin

# App（Android/iOS，需 HBuilderX 基座或自定义基座）
npm run dev:app
```

## 构建

```bash
npm run build:h5
npm run build:mp-weixin
npm run build:app
npm run build:app-harmony   # 鸿蒙（配合 DevEco / HBuilderX 鸿蒙基座）
```

## 用 HBuilderX 打开（App / 鸿蒙推荐）

1. HBuilderX「文件 → 打开目录」选择本工程 `uni-video-demo`。
2. 运行 → 运行到手机或模拟器 → 选择 Android / iOS / 鸿蒙。
3. App 与鸿蒙的完整原生能力（含全屏横竖屏）建议用 HBuilderX 真机运行验证。

## 目录结构

```
uni-video-demo/
├── package.json
├── vite.config.ts
├── tsconfig.json
├── index.html
├── project.config.json          # 微信小程序工程配置
└── src/
    ├── main.ts
    ├── App.vue
    ├── manifest.json
    ├── pages.json
    ├── uni.scss
    ├── env.d.ts
    ├── pages/
    │   └── index/index.vue       # 使用插件的示例页
    └── uni_modules/
        └── xyyt-video-player/    # ← 插件（可整目录复制到其他工程）
            ├── package.json       # uni_modules 元信息
            ├── readme.md
            ├── changelog.md
            └── components/
                └── xyyt-video-player/
                    └── xyyt-video-player.vue
```

## 复用插件到其他工程

把 `src/uni_modules/xyyt-video-player` 整个目录拷贝到目标工程的 `src/uni_modules/`（或根 `uni_modules/`）即可，组件按 easycom 自动注册，直接用 `<xyyt-video-player />`。用法与 Props/Events/方法见插件 `readme.md`。

## 提示

- 示例页 `videoUrl` 为公网测试地址，请替换为你自己的可访问视频。
- 所有报错若为「找不到模块 vue / uni」，是尚未 `npm install`，安装依赖后消失。
- 鸿蒙真机运行时 HBuilderX 会把本机签名配置（`manifest.json` 的 `app-harmony.distribute.signingConfigs`，含证书路径与密码）自动写入工程；**提交/上传前请删除该节**（下次真机运行会自动重建），避免敏感信息入库入市。

## 文档与反馈

- 插件完整文档（Props / Events / 方法 / 平台说明 / FAQ）：[`src/uni_modules/xyyt-video-player/readme.md`](src/uni_modules/xyyt-video-player/readme.md)
- 版本变更记录：[`src/uni_modules/xyyt-video-player/changelog.md`](src/uni_modules/xyyt-video-player/changelog.md)
- 问题反馈：[GitHub Issues](https://github.com/xyytwz/uni-video-player/issues)
- 插件市场：DCloud 插件市场搜索 `xyyt-video-player`
