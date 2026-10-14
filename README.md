# uni-video-demo

`video-player-xyyt` 跨端视频播放插件的**示例工程**，一份源码可编译为 **App（Android/iOS/鸿蒙）**、**H5**、**微信小程序**。

插件本体位于：`src/uni_modules/video-player-xyyt/`，示例页位于 `src/pages/index/index.vue`。

## 环境

- HBuilderX（App / 鸿蒙真机运行推荐）或 CLI（Node ≥ 16）
- Vue3 + Vite + `@dcloudio/*` 3.0.0 系列（见 `package.json`）

## 运行（CLI）

```bash
# 安装依赖
npm install

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
        └── video-player-xyyt/    # ← 插件（可整目录复制到其他工程）
            ├── package.json       # uni_modules 元信息
            ├── readme.md
            ├── changelog.md
            └── components/
                └── video-player-xyyt/
                    └── video-player-xyyt.vue
```

## 复用插件到其他工程

把 `src/uni_modules/video-player-xyyt` 整个目录拷贝到目标工程的 `src/uni_modules/`（或根 `uni_modules/`）即可，组件按 easycom 自动注册，直接用 `<video-player-xyyt />`。用法与 Props/Events/方法见插件 `readme.md`。

## 提示

- 示例页 `videoUrl` 为公网测试地址，请替换为你自己的可访问视频。
- 所有报错若为「找不到模块 vue / uni」，是尚未 `npm install`，安装依赖后消失。
