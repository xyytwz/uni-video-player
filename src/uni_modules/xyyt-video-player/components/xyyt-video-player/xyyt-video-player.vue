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
      :enable-progress-gesture="false"
      :show-progress="false"
      :controls="false"
      :show-center-play-btn="false"
      :show-play-btn="false"
      :show-fullscreen-btn="false"
      @play="onPlay"
      @pause="onPause"
      @ended="onEnded"
      @timeupdate="onTimeUpdate"
      @loadedmetadata="onLoadedMetadata"
      @fullscreenchange="onFullScreenChange"
      @waiting="onWaiting"
      @error="onError"
      @click="togglePlay"
    >
      <!-- 居中播放按钮：暂停 / 未开始 / 播放结束时显示 -->
      <view v-if="!isPlaying" class="qvp-center" @click.stop="togglePlay">
        <view class="qvp-center-btn">
          <view class="qvp-ico-play" />
        </view>
      </view>

      <!-- 底部控制栏：播放/暂停 + 时间 + 拖动进度 + 快退/快进 + 全屏 -->
      <view class="qvp-bar" @click.stop>
        <view class="qvp-btn qvp-btn--play" @click.stop="togglePlay">
          <view v-if="isPlaying" class="qvp-ico-pause"><view class="qvp-pause-bar" /><view class="qvp-pause-bar" /></view>
          <view v-else class="qvp-ico-play qvp-ico-play--sm" />
        </view>

        <text class="qvp-time" :class="{ 'qvp-time--fs': isFullScreen }">
          {{ fmt(currentTime) }} / {{ fmt(duration) }}
        </text>

        <!-- 自绘进度条：移动端走 touch 链（小程序会忽略鼠标绑定），
             H5 桌面端由 @mousedown + window 鼠标事件驱动（运行时按 window 存在性守卫）；
             避开微信原生 slider 同层渲染在部分基础库上的层级/样式问题 -->
        <view
          :id="barId"
          class="qvp-progress"
          :class="{ 'qvp-progress--fs': isFullScreen }"
          @touchstart="onBarTouchStart"
          @touchmove.stop.prevent="onBarTouchMove"
          @touchend="onBarTouchEnd"
          @mousedown="onBarMouseDown"
        >
          <view class="qvp-track">
            <view class="qvp-filled" :style="{ width: progressPercent + '%' }" />
            <view class="qvp-knob" :style="{ left: progressPercent + '%' }" />
          </view>
        </view>

        <template v-if="showSpeedButtons">
          <view class="qvp-btn qvp-btn--step" @click.stop="fastRewind">
            <text class="qvp-step-label"><text class="qvp-step-arrow">‹</text>{{ speedStep }}</text>
          </view>
          <view class="qvp-btn qvp-btn--step" @click.stop="fastForward">
            <text class="qvp-step-label">{{ speedStep }}<text class="qvp-step-arrow">›</text></text>
          </view>
        </template>

        <view v-if="showFullScreen" class="qvp-btn qvp-btn--fs" @click.stop="toggleFullScreen">
          <view class="qvp-ico-fs">
            <view class="qvp-fs-corner qvp-fs-tl" /><view class="qvp-fs-corner qvp-fs-tr" />
            <view class="qvp-fs-corner qvp-fs-bl" /><view class="qvp-fs-corner qvp-fs-br" />
          </view>
          <text class="qvp-ico qvp-ico--fs-label">{{ isFullScreen ? '退出' : '全屏' }}</text>
        </view>
      </view>
    </video>
  </view>
</template>

<script setup lang="ts">
  import { computed, getCurrentInstance, onMounted, onUnmounted, ref } from 'vue';

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

  // 组件内唯一 video id；配合 createVideoContext 的 scope 精确定位到本组件实例
  const videoId = 'xyytVideoPlayer';

  // 关键：script setup 下必须传组件代理实例(instance.proxy)作作用域，
  // 传内部 instance 会让 requestFullScreen 等原生方法在各端(App/鸿蒙)静默失效。
  const instance = getCurrentInstance();
  let ctx: any = null;

  const isPlaying = ref(false);
  const isFullScreen = ref(false);
  const currentTime = ref(0);
  const duration = ref(0);
  const sliderValue = ref(0);
  const dragging = ref(false);
  const isEnded = ref(false);

  const durationReady = computed(() => duration.value > 0);
  const progressPercent = computed(() =>
    duration.value > 0 ? Math.min(100, (sliderValue.value / duration.value) * 100) : 0
  );

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
    // 预取进度条 rect，供 touch 兜底链使用（延迟到布局完成后再查）
    setTimeout(() => queryBarRect(), 300);
  });

  onUnmounted(() => {
    ctx = null;
    if (typeof window !== 'undefined') {
      window.removeEventListener('mousemove', onWindowMouseMove);
      window.removeEventListener('mouseup', onWindowMouseUp);
    }
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
    const target = Math.max(0, Math.min(seconds, duration.value || seconds));
    ctx?.seek(target);
    currentTime.value = target;
    sliderValue.value = Math.floor(target);
  }
  function fastForward() {
    seek((currentTime.value || 0) + props.speedStep);
  }
  function fastRewind() {
    seek((currentTime.value || 0) - props.speedStep);
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

  // ========== 自绘进度条拖动（touch + H5 鼠标双通路） ==========
  // 实测约束：uni-h5 会把 view 上的 @touchxxx 包装成 uni 事件——detail 为空对象、
  // currentTarget 为无 getBoundingClientRect 的普通对象；桌面鼠标也不会触发 touch 链。
  // 因此：
  // - H5 桌面端走 @mousedown（原生 MouseEvent，target 为真实 DOM，可实时取 rect）
  //   + window 上的 mousemove/mouseup；
  // - 触摸端（小程序/App/移动端 H5）走 touch 链，rect 取缓存 →(若可用)detail，
  //   每次交互结束后异步刷新缓存，避免滚动/布局变化导致过期。
  const barId = 'xyytPlayerBar';
  let barRect: { left: number; width: number } | null = null;
  let barPercent = 0;

  function queryBarRect() {
    try {
      uni.createSelectorQuery()
        .in(instance?.proxy)
        .select('#' + barId)
        .boundingClientRect((r: any) => {
          if (r && r.width > 0) barRect = { left: r.left, width: r.width };
        })
        .exec();
    } catch {
      /* 查询失败时沿用旧缓存 */
    }
  }

  function percentFromX(x: number, rect: { left: number; width: number }): number {
    return Math.max(0, Math.min(100, ((x - rect.left) / rect.width) * 100));
  }

  function previewBarTime() {
    const sec = (barPercent / 100) * duration.value;
    sliderValue.value = Math.floor(sec);
    currentTime.value = sec;
  }

  // ----- touch 链（小程序 / App / 移动端 H5） -----
  function onBarTouchStart(e: any) {
    if (!durationReady.value || mouseDragging) return;
    let rect = barRect;
    const d = e?.detail;
    if (d && typeof d.width === 'number' && d.width > 0) {
      rect = { left: d.left ?? 0, width: d.width };
    }
    if (!rect) return;
    barRect = rect;
    dragging.value = true;
    const t = e?.touches?.[0];
    barPercent = percentFromX(Number(t?.clientX ?? t?.pageX ?? rect.left), rect);
    previewBarTime();
  }
  function onBarTouchMove(e: any) {
    if (!dragging.value || !barRect) return;
    const t = e?.changedTouches?.[0];
    if (!t) return;
    barPercent = percentFromX(Number(t.clientX ?? t.pageX ?? 0), barRect);
    previewBarTime();
  }
  function onBarTouchEnd() {
    if (!dragging.value || mouseDragging) return;
    dragging.value = false;
    // 未发生位移时 barPercent 即 tap 位置：点击轨道直接 seek
    seek((barPercent / 100) * duration.value);
    queryBarRect(); // 交互后刷新缓存，供滚动/布局变化后的下一次拖动使用
  }

  // ----- H5 桌面鼠标链（仅浏览器存在 window 时生效，小程序端整段跳过） -----
  let mouseDragging = false;
  function onBarMouseDown(e: any) {
    if (!durationReady.value || typeof window === 'undefined') return;
    const el = e?.target;
    const rect =
      el && typeof el.getBoundingClientRect === 'function' ? el.getBoundingClientRect() : barRect;
    if (!rect || rect.width <= 0) return;
    barRect = { left: rect.left, width: rect.width };
    mouseDragging = true;
    dragging.value = true;
    barPercent = percentFromX(Number(e?.clientX ?? 0), barRect);
    previewBarTime();
    window.addEventListener('mousemove', onWindowMouseMove);
    window.addEventListener('mouseup', onWindowMouseUp);
  }
  function onWindowMouseMove(e: MouseEvent) {
    if (!mouseDragging || !barRect) return;
    barPercent = percentFromX(e.clientX, barRect);
    previewBarTime();
  }
  function onWindowMouseUp() {
    if (!mouseDragging) return;
    mouseDragging = false;
    dragging.value = false;
    seek((barPercent / 100) * duration.value);
    window.removeEventListener('mousemove', onWindowMouseMove);
    window.removeEventListener('mouseup', onWindowMouseUp);
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
    // 拖动期间不让 timeupdate 覆盖滑块，避免与手指位置打架
    if (!dragging.value) sliderValue.value = Math.floor(ct);
    emit('timeupdate', { currentTime: ct, duration: du });
  }
  function onLoadedMetadata(e: any) {
    const du = Number(e?.detail?.duration ?? 0);
    if (du > 0) duration.value = du;
  }
  function onWaiting() {
    // 缓冲中：不改变 isPlaying，等待自动恢复
  }
  function onError(e: any) {
    emit('error', e?.detail ?? e);
    uni.showToast({ title: '视频加载失败', icon: 'none' });
  }
  function onFullScreenChange(e: any) {
    // 全屏是否成功的唯一判据是回调，而非“API 已调用”
    isFullScreen.value = !!e?.detail?.fullScreen;
    emit('fullscreenchange', { fullScreen: isFullScreen.value });
    // 布局变化后刷新进度条 rect（与 900ms 全屏过渡守卫对齐，避免取到过渡中间态）
    setTimeout(() => queryBarRect(), 950);
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
    /* 高度兜底：父级未给高度时 height:100% 会塌陷为 0，
       用 16:9 宽高比保证播放器始终可见；父级有固定高度时仍走 height:100% */
    aspect-ratio: 16 / 9;
    height: auto;
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
  .qvp-ico-pause {
    display: flex;
    flex-direction: row;
    gap: 10rpx;
  }
  .qvp-pause-bar {
    width: 8rpx;
    height: 30rpx;
    border-radius: 4rpx;
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

  /* 底部控制栏 */
  .qvp-bar {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    height: 84rpx;
    padding: 0 24rpx;
    /* #ifdef H5 */
    /* 仅 H5 全屏时控制栏贴屏幕底，需预留刘海屏/手势条安全区；
       App/小程序全屏由原生层接管、自绘栏不到屏幕底，加安全区只会造成底部空隙 */
    padding-bottom: constant(safe-area-inset-bottom);
    padding-bottom: env(safe-area-inset-bottom);
    /* #endif */
    display: flex;
    flex-direction: row;
    align-items: center;
    background: linear-gradient(180deg, rgba(0, 0, 0, 0) 0%, rgba(0, 0, 0, 0.62) 88%);
    pointer-events: auto;
    box-sizing: content-box;
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
    transition: background-color 0.15s ease, opacity 0.15s ease;
  }
  .qvp-btn:active {
    background: rgba(255, 255, 255, 0.18);
  }
  /* H5 桌面端悬停反馈；触摸设备 hover 恒 false，不会误触发 */
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
  .qvp-step-arrow {
    font-size: 26rpx;
    line-height: 1;
  }

  /* 全屏：四角框图标 + 文字 */
  .qvp-btn--fs {
    margin-left: 12rpx;
    gap: 8rpx;
  }
  .qvp-ico-fs {
    position: relative;
    width: 26rpx;
    height: 26rpx;
  }
  .qvp-fs-corner {
    position: absolute;
    width: 10rpx;
    height: 10rpx;
  }
  .qvp-fs-tl {
    left: 0;
    top: 0;
    border-left: 3rpx solid #fff;
    border-top: 3rpx solid #fff;
  }
  .qvp-fs-tr {
    right: 0;
    top: 0;
    border-right: 3rpx solid #fff;
    border-top: 3rpx solid #fff;
  }
  .qvp-fs-bl {
    left: 0;
    bottom: 0;
    border-left: 3rpx solid #fff;
    border-bottom: 3rpx solid #fff;
  }
  .qvp-fs-br {
    right: 0;
    bottom: 0;
    border-right: 3rpx solid #fff;
    border-bottom: 3rpx solid #fff;
  }
  .qvp-ico--fs-label {
    font-size: 22rpx;
    line-height: 1;
    color: #fff;
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
    font-size: 26rpx;
  }

  /* 自绘进度条：触摸热区 44rpx，视觉细轨道居中 */
  .qvp-progress {
    flex: 1;
    height: 64rpx;
    display: flex;
    align-items: center;
    pointer-events: auto;
  }
  .qvp-track {
    position: relative;
    width: 100%;
    height: 6rpx;
    border-radius: 3rpx;
    background: rgba(255, 255, 255, 0.28);
  }
  .qvp-filled {
    position: absolute;
    left: 0;
    top: 0;
    bottom: 0;
    border-radius: 3rpx;
    /* 主题色来自 uni.scss 全局变量 */
    background: $xyyt-primary;
  }
  .qvp-knob {
    position: absolute;
    top: 50%;
    width: 24rpx;
    height: 24rpx;
    margin-top: -12rpx;
    margin-left: -12rpx;
    border-radius: 12rpx;
    background: #fff;
    box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.4);
  }
  .qvp-progress--fs .qvp-track {
    height: 8rpx;
  }
  .qvp-progress--fs .qvp-knob {
    width: 30rpx;
    height: 30rpx;
    margin-top: -15rpx;
    margin-left: -15rpx;
    border-radius: 15rpx;
  }
</style>
