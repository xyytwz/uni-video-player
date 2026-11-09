<template>
  <view class="qvp-container">
    <video
      :id="videoId"
      class="qvp-video"
      :src="src"
      :poster="poster"
      :autoplay="autoplay"
      :muted="muted"
      :loop="loop"
      :initial-time="initialTime"
      :object-fit="objectFit"
      :controls="appNativeControls"
      :show-center-play-btn="false"
      @play="onPlay"
      @pause="onPause"
      @ended="onEnded"
      @timeupdate="onTimeUpdate"
      @loadedmetadata="onLoadedMetadata"
      @fullscreenchange="onFullScreenChange"
      @error="onError"
      @click="togglePlay"
    >
      <!-- 自定义控制栏仅用于非 Android 平台（H5/鸿蒙/微信小程序）；
           Android 真机实测：cover-view + slider 混合方案导致原生 video 黑屏且无法播放，
           改用系统原生控件（appNativeControls），样式由系统决定、不可自定义。 -->
      <!-- #ifndef APP-PLUS -->
      <cover-view v-if="!isPlaying" class="qvp-center" @click.stop="togglePlay">
        <cover-view class="qvp-center-btn">
          <cover-view class="qvp-ico-play" />
        </cover-view>
      </cover-view>
      <!-- #endif -->

      <!-- #ifndef APP-PLUS -->
      <cover-view class="qvp-bar" :class="{ 'qvp-bar--fs': isFullScreen }" @click.stop>
        <cover-view class="qvp-btn qvp-btn--play" @click.stop="togglePlay">
          <cover-view v-if="isPlaying" class="qvp-ico-pause">
            <cover-view class="qvp-pause-bar" />
            <cover-view class="qvp-pause-bar" />
          </cover-view>
          <cover-view v-else class="qvp-ico-play qvp-ico-play--sm" />
        </cover-view>

        <text class="qvp-time" :class="{ 'qvp-time--fs': isFullScreen }">
          {{ fmt(currentTime) }} / {{ fmt(duration) }}
        </text>

        <slider
          class="qvp-slider"
          :class="{ 'qvp-slider--fs': isFullScreen }"
          :value="sliderValue"
          :max="duration || 100"
          activeColor="rgba(255,125,0,0.35)"
          backgroundColor="rgba(255,255,255,0.35)"
          :block-size="isFullScreen ? 12 : 14"
          @change="onSliderChange"
        />

        <template v-if="showSpeedButtons">
          <cover-view class="qvp-btn qvp-btn--step" @click.stop="fastRewind">
            <text class="qvp-step-label"><text class="qvp-step-arrow">‹</text>{{ speedStep }}</text>
          </cover-view>
          <cover-view class="qvp-btn qvp-btn--step" @click.stop="fastForward">
            <text class="qvp-step-label">{{ speedStep }}<text class="qvp-step-arrow">›</text></text>
          </cover-view>
        </template>

        <cover-view v-if="showFullScreen" class="qvp-btn qvp-btn--fs" @click.stop="toggleFullScreen">
          <text class="qvp-ico qvp-ico--fs-label">{{ isFullScreen ? '退出' : '全屏' }}</text>
        </cover-view>
      </cover-view>
      <!-- #endif -->
    </video>
  </view>
</template>

<script setup lang="ts">
  import { getCurrentInstance, onMounted, onUnmounted, ref } from 'vue';

  interface XyytVideoPlayerProps {
    /** 视频地址（必填） */
    src: string;
    /** 封面图 */
    poster?: string;
    /** 是否自动播放 */
    autoplay?: boolean;
    /** 是否静音 */
    muted?: boolean;
    /** 是否循环 */
    loop?: boolean;
    /** 初始播放位置（秒） */
    initialTime?: number;
    /** 视频填充模式 */
    objectFit?: 'contain' | 'fill' | 'cover';
    /** 是否显示全屏按钮 */
    showFullScreen?: boolean;
    /** 是否显示快退/快进按钮 */
    showSpeedButtons?: boolean;
    /** 快退/快进步长（秒） */
    speedStep?: number;
    /**
     * 全屏方向：0 竖屏，-90 横屏（逆时针），90 横屏（顺时针）。
     * 不传时按平台自动选择：iOS 竖屏(0)，Android/鸿蒙/小程序/H5 横屏(-90)。
     */
    fullScreenDirection?: number;
  }

  const props = withDefaults(defineProps<XyytVideoPlayerProps>(), {
    poster: '',
    autoplay: false,
    muted: false,
    loop: false,
    initialTime: 0,
    objectFit: 'contain',
    showFullScreen: true,
    showSpeedButtons: true,
    speedStep: 15,
    fullScreenDirection: undefined,
  });

  const emit = defineEmits<{
    (e: 'play'): void;
    (e: 'pause'): void;
    (e: 'ended'): void;
    (e: 'timeupdate', detail: { currentTime: number; duration: number }): void;
    (e: 'fullscreenchange', detail: { fullScreen: boolean }): void;
    (e: 'error', detail: any): void;
  }>();

  // 关键：script setup 下必须传组件代理实例(instance.proxy)作作用域，
  // 传内部 instance 会让 requestFullScreen 等原生方法在各端(App/鸿蒙)静默失效。
  const instance = getCurrentInstance();

  // 实例级唯一 video id：同一页面多实例时固定 id 会让 createVideoContext 定位到错误的 video。
  // 用 Vue 全局递增的 instance.uid 作后缀（不能用 script setup 内的模块计数器——
  // script setup 整体就是 setup 函数体，每次实例化都会重新初始化）
  const videoId = 'xyytVideoPlayer-' + instance?.uid;

  let ctx: any = null;

  // Android(App) 端使用系统原生控件，其余平台显示自定义控制栏。
  // 条件编译必须写在脚本顶层：此前写在 computed 函数体内曾导致 App 端取值异常。
  // 单声明 + 分支赋值（而非两份 const 声明）：vue-tsc 不做条件编译剥离，重复声明会报 TS2451
  let appNativeControls: boolean;
  // #ifdef APP-PLUS
  appNativeControls = true;
  // #endif
  // #ifndef APP-PLUS
  appNativeControls = false;
  // #endif
  // 注意：show-center-play-btn 与 controls 是相互独立的开关，不随 appNativeControls 联动：
  // 非 Android 端需固定 false（否则原生中央按钮与自定义按钮叠加）；Android 原生控件自带
  // 中央按钮，不受该属性影响（Android 真机验证行为）。

  const isPlaying = ref(false);
  const isFullScreen = ref(false);
  const currentTime = ref(0);
  const duration = ref(0);
  const sliderValue = ref(0);
  const isEnded = ref(false);
  let seekRetryTimer: ReturnType<typeof setTimeout> | null = null;

  /** 平台判定：iOS 全屏用竖屏(0)，其余用横屏(-90)。鸿蒙不属于 ios，走横屏。 */
  // 平台值在生命周期内不变，只查一次并缓存
  let iosPlatformCached: boolean | null = null;
  function isIOSPlatform(): boolean {
    if (iosPlatformCached !== null) return iosPlatformCached;
    try {
      // #ifdef MP-WEIXIN
      // getSystemInfoSync 在微信端已废弃（控制台告警），改用 getDeviceInfo（基础库 ≥ 2.20.1）
      iosPlatformCached = uni.getDeviceInfo().platform?.toLowerCase() === 'ios';
      // #endif
      // #ifndef MP-WEIXIN
      iosPlatformCached = uni.getSystemInfoSync().platform?.toLowerCase() === 'ios';
      // #endif
    } catch {
      iosPlatformCached = false;
    }
    return iosPlatformCached;
  }

  // ========== 全屏过渡守卫 ==========
  // 进入/退出全屏时，多数原生实现会派发一次中间态 pause；
  // 用时间锁吞掉它，避免误置 isPlaying，并在过渡后按“切换前意图”恢复播放。
  let guardUntil = 0;
  let wasPlayingBeforeFs = false;
  function setGuard(ms = 700) {
    guardUntil = Date.now() + ms;
  }

  function fmt(sec: number): string {
    const s = Math.max(0, Math.floor(sec || 0));
    const m = Math.floor(s / 60);
    const r = s % 60;
    return (m < 10 ? '0' + m : '' + m) + ':' + (r < 10 ? '0' + r : '' + r);
  }

  onMounted(() => {
    ctx = uni.createVideoContext(videoId, instance?.proxy);
  });

  onUnmounted(() => {
    ctx = null;
    if (seekRetryTimer) clearTimeout(seekRetryTimer);
  });

  // ========== 播放控制 ==========
  function play() {
    ctx?.play();
  }
  function pause() {
    ctx?.pause();
  }
  function togglePlay() {
    if (Date.now() < guardUntil) return; // 全屏过渡期忽略误触
    if (isPlaying.value) pause();
    else play();
  }
  function seek(seconds: number) {
    if (!ctx) return;
    const target = Math.max(0, seconds);
    // 原生层（Android/iOS/鸿蒙）seek 可能因未就绪而静默失败，加一次延迟重试；
    // 重试前比对 currentTime（由 timeupdate 回填），目标已达成则不重复 seek，各端安全无副作用
    ctx.seek(target);
    currentTime.value = target;
    sliderValue.value = Math.floor(target);
    // 500ms 后重试一次，确保原生层已处理
    if (seekRetryTimer) clearTimeout(seekRetryTimer);
    seekRetryTimer = setTimeout(() => {
      seekRetryTimer = null;
      if (Math.abs(currentTime.value - target) > 1) {
        ctx?.seek(target);
      }
    }, 500);
  }
  function fastForward() {
    seek(currentTime.value + props.speedStep);
  }
  function fastRewind() {
    seek(currentTime.value - props.speedStep);
  }

  // ========== 全屏 ==========
  function enterFullScreen() {
    if (!ctx) return;
    let dir = props.fullScreenDirection;
    if (dir == null) {
      // #ifdef MP-WEIXIN
      // 微信端实测：-90 为向右旋转横屏，90 才是向左旋转横屏（与 iOS 竖屏 0 组合）
      dir = isIOSPlatform() ? 0 : 90;
      // #endif
      // #ifndef MP-WEIXIN
      dir = isIOSPlatform() ? 0 : -90;
      // #endif
    }
    ctx.requestFullScreen({ direction: dir });
  }
  function exitFullScreen() {
    ctx?.exitFullScreen();
  }
  function toggleFullScreen() {
    wasPlayingBeforeFs = isPlaying.value;
    setGuard(900);
    if (!isFullScreen.value) enterFullScreen();
    else exitFullScreen();
  }

  // ========== 原生 slider 拖动 ==========
  function onSliderChange(e: any) {
    const val = Number(e?.detail?.value ?? 0);
    sliderValue.value = val;
    seek(val);
  }

  // ========== 视频事件 ==========
  function onPlay() {
    isPlaying.value = true;
    isEnded.value = false;
    emit('play');
  }
  function onPause() {
    // 全屏过渡期的中间态 pause 不上报，避免状态抖动
    if (Date.now() < guardUntil) return;
    isPlaying.value = false;
    emit('pause');
  }
  function onEnded() {
    isPlaying.value = false;
    isEnded.value = true;
    emit('ended');
  }
  function onTimeUpdate(e: any) {
    const ct = Number(e?.detail?.currentTime ?? 0);
    const du = Number(e?.detail?.duration ?? 0);
    currentTime.value = ct;
    if (du > 0) duration.value = du;
    sliderValue.value = Math.floor(ct);
    emit('timeupdate', { currentTime: ct, duration: du });
  }
  function onLoadedMetadata(e: any) {
    const du = Number(e?.detail?.duration ?? 0);
    if (du > 0) duration.value = du;
  }
  function onError(e: any) {
    emit('error', e?.detail ?? e);
    uni.showToast({ title: '视频加载失败', icon: 'none' });
  }
  function onFullScreenChange(e: any) {
    // 全屏是否成功的唯一判据是回调，而非“API 已调用”
    isFullScreen.value = !!e?.detail?.fullScreen;
    emit('fullscreenchange', { fullScreen: isFullScreen.value });
    // 过渡确认后按切换前意图补发 play()（幂等，已播完则不恢复）
    if (wasPlayingBeforeFs && !isPlaying.value && !isEnded.value) {
      setTimeout(() => {
        if (!isEnded.value) play();
      }, 200);
    }
  }

  defineExpose({
    play,
    pause,
    seek,
    fastForward,
    fastRewind,
    enterFullScreen,
    exitFullScreen,
    toggleFullScreen,
    togglePlay,
  });
</script>

<style lang="scss" scoped>
  .qvp-container {
    width: 100%;
    position: relative;
    background: #000;
    overflow: hidden;
    /* padding-bottom 56.25% = 16:9 宽高比（比 aspect-ratio 在 Android 原生 video 上更可靠） */
    height: 0;
    padding-bottom: 56.25%;
  }
  .qvp-video {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
  }
  /* H5 端 uni-video 内层原生 video 需同步铺满，否则绝对定位后内容为 0 尺寸 */
  .qvp-video :deep(video) {
    width: 100%;
    height: 100%;
  }

  /* 播放/暂停图标：CSS 绘制，替代各端渲染参差的 Unicode 字符 */
  .qvp-ico-play {
    width: 0;
    height: 0;
    border-style: solid;
    /* 三角形：左边界着色，宽高差构成视觉比例 */
    border-width: 16rpx 0 16rpx 26rpx;
    border-color: transparent transparent transparent #fff;
    /* 视觉重心补偿：三角形几何中心偏左，右移微调 */
    margin-left: 6rpx;
  }
  .qvp-ico-play--sm {
    border-width: 12rpx 0 12rpx 20rpx;
    margin-left: 4rpx;
  }
  /* 控制栏播放键：圆形衬底 + 图标居中，触控目标更明确 */
  .qvp-btn--play {
    width: 60rpx;
    height: 60rpx;
    min-width: 60rpx;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.12);
  }
  .qvp-bar--fs .qvp-btn--play {
    width: 40rpx;
    height: 40rpx;
    min-width: 40rpx;
  }
  .qvp-ico-pause {
    display: flex;
    flex-direction: row;
    gap: 10rpx;
  }
  .qvp-pause-bar {
    width: 10rpx;
    height: 20rpx;
    border-radius: 2rpx;
    background: #fff;
  }

  /* 居中播放按钮 */
  .qvp-center {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    /* 关键：uni-app H5/App-webview 视频容器默认 pointer-events:none，
       插槽内元素需显式 auto 才能命中点击，否则点击穿透到 video */
    pointer-events: auto;
  }
  .qvp-center-btn {
    width: 116rpx;
    height: 116rpx;
    border-radius: 58rpx;
    background: rgba(0, 0, 0, 0.4);
    border: 2rpx solid rgba(255, 255, 255, 0.35);
    box-shadow: 0 8rpx 32rpx rgba(0, 0, 0, 0.35);
    display: flex;
    align-items: center;
    justify-content: center;
    transition: opacity 0.2s ease;
  }
  .qvp-center-btn:active {
    opacity: 0.7;
  }

  /* 底部控制栏：cover-view 在 H5/小程序端等同 div，flex 完全可用；Android 端整栏不渲染（原生控件） */
  .qvp-bar {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    height: 84rpx;
    padding: 0 24rpx;
    /* #ifdef H5 */
    padding-bottom: constant(safe-area-inset-bottom);
    padding-bottom: env(safe-area-inset-bottom);
    /* #endif */
    display: flex;
    flex-direction: row;
    align-items: center;
    background: rgba(0, 0, 0, 0.35);
    pointer-events: auto;
    box-sizing: content-box;
  }
  /* 全屏模式：控制栏整体缩小一档 */
  .qvp-bar--fs {
    height: 60rpx;
    padding: 0 16rpx;
  }
  .qvp-btn {
    min-width: 64rpx;
    height: 64rpx;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    border-radius: 32rpx;
    pointer-events: auto;
  }
  .qvp-bar--fs .qvp-btn {
    min-width: 44rpx;
    height: 44rpx;
    border-radius: 22rpx;
  }
  .qvp-btn:active {
    background: rgba(255, 255, 255, 0.18);
  }
  /* H5 桌面端悬停反馈 */
  @media (hover: hover) {
    .qvp-btn:hover {
      background: rgba(255, 255, 255, 0.14);
    }
  }
  .qvp-ico {
    color: #fff;
    font-size: 30rpx;
    line-height: 1;
  }

  /* 快退/快进：底色描边胶囊 + 方向箭头 */
  .qvp-btn--step {
    margin-left: 8rpx;
    padding: 0 8rpx;
  }
  .qvp-bar--fs .qvp-btn--step {
    margin-left: 6rpx;
    padding: 0 6rpx;
  }
  .qvp-step-label {
    display: inline-block;
    color: #fff;
    font-size: 22rpx;
    line-height: 1;
    padding: 10rpx 12rpx;
    border: 1rpx solid rgba(255, 255, 255, 0.4);
    border-radius: 26rpx;
    background: rgba(255, 255, 255, 0.1);
    white-space: nowrap;
  }
  .qvp-bar--fs .qvp-step-label {
    font-size: 18rpx;
    padding: 6rpx 8rpx;
    border-radius: 18rpx;
  }
  .qvp-step-arrow {
    font-size: 26rpx;
    line-height: 1;
  }

  /* 全屏按钮 */
  .qvp-btn--fs {
    margin-left: 12rpx;
    gap: 8rpx;
  }
  .qvp-bar--fs .qvp-btn--fs {
    margin-left: 8rpx;
    gap: 6rpx;
  }
  .qvp-ico--fs-label {
    font-size: 22rpx;
    line-height: 1;
    color: #fff;
  }
  .qvp-bar--fs .qvp-ico--fs-label {
    font-size: 18rpx;
  }
  .qvp-time {
    display: inline-block;
    color: rgba(255, 255, 255, 0.92);
    font-size: 22rpx;
    line-height: 1;
    margin: 0 16rpx;
    white-space: nowrap;
    font-variant-numeric: tabular-nums;
  }
  .qvp-time--fs {
    font-size: 20rpx;
  }

  /* 原生 slider 进度条：flex:1 占满剩余空间 */
  .qvp-slider {
    flex: 1;
    margin: 0 16rpx;
  }
  .qvp-slider--fs {
    margin: 0 8rpx;
  }
</style>
