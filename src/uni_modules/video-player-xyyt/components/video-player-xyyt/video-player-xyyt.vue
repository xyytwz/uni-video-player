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
          <text class="qvp-ico">▶</text>
        </view>
      </view>

      <!-- 底部控制栏：播放/暂停 + 时间 + 拖动进度 + 快退/快进 + 全屏 -->
      <view class="qvp-bar" @click.stop>
        <view class="qvp-btn" @click.stop="togglePlay">
          <text class="qvp-ico">{{ isPlaying ? '❚❚' : '▶' }}</text>
        </view>

        <text class="qvp-time" :class="{ 'qvp-time--fs': isFullScreen }">
          {{ fmt(currentTime) }} / {{ fmt(duration) }}
        </text>

        <slider
          class="qvp-slider"
          :value="sliderValue"
          :min="0"
          :max="durationMax"
          :step="1"
          :disabled="!durationReady"
          active-color="#ff7d00"
          background-color="rgba(255,255,255,0.30)"
          block-color="#ffffff"
          block-size="16"
          @changing="onSliderChanging"
          @change="onSliderChange"
        />

        <template v-if="showSpeedButtons">
          <view class="qvp-btn" @click.stop="fastRewind">
            <text class="qvp-ico qvp-ico--step">-{{ speedStep }}</text>
          </view>
          <view class="qvp-btn" @click.stop="fastForward">
            <text class="qvp-ico qvp-ico--step">+{{ speedStep }}</text>
          </view>
        </template>

        <view v-if="showFullScreen" class="qvp-btn" @click.stop="toggleFullScreen">
          <text class="qvp-ico qvp-ico--fs-label">{{ isFullScreen ? '退出全屏' : '全屏' }}</text>
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

  const durationMax = computed(() => (duration.value > 1 ? Math.floor(duration.value) : 100));
  const durationReady = computed(() => duration.value > 0);

  /** 平台判定：iOS 全屏用竖屏(0)，其余用横屏(-90)。鸿蒙不属于 ios，走横屏。 */
  function isIOSPlatform(): boolean {
    try {
      return uni.getSystemInfoSync().platform?.toLowerCase() === 'ios';
    } catch {
      return false;
    }
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
    const dir = props.fullScreenDirection ?? (isIOSPlatform() ? 0 : -90);
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

  // ========== 进度拖动（slider 跨端一致，避免在各端手写触摸坐标换算） ==========
  function onSliderChanging(e: any) {
    dragging.value = true;
    const v = Number(e?.detail?.value ?? 0);
    sliderValue.value = v;
    currentTime.value = v;
  }
  function onSliderChange(e: any) {
    dragging.value = false;
    const v = Number(e?.detail?.value ?? 0);
    seek(v);
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
  }
  .qvp-video {
    width: 100%;
    height: 100%;
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
    width: 110rpx;
    height: 110rpx;
    border-radius: 55rpx;
    background: rgba(0, 0, 0, 0.45);
    display: flex;
    align-items: center;
    justify-content: center;
  }

  /* 底部控制栏 */
  .qvp-bar {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    height: 88rpx;
    padding: 0 20rpx;
    display: flex;
    flex-direction: row;
    align-items: center;
    background: linear-gradient(180deg, rgba(0, 0, 0, 0) 0%, rgba(0, 0, 0, 0.55) 100%);
    pointer-events: auto;
  }
  .qvp-btn {
    min-width: 60rpx;
    height: 60rpx;
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: auto;
  }
  .qvp-ico {
    color: #fff;
    font-size: 30rpx;
    line-height: 1;
  }
  .qvp-ico--step {
    font-size: 26rpx;
    padding: 6rpx 10rpx;
    border: 1rpx solid rgba(255, 255, 255, 0.6);
    border-radius: 24rpx;
  }
  .qvp-ico--fs-label {
    font-size: 26rpx;
  }
  .qvp-time {
    color: #fff;
    font-size: 24rpx;
    margin: 0 12rpx;
    white-space: nowrap;
  }
  .qvp-time--fs {
    font-size: 28rpx;
  }
  .qvp-slider {
    flex: 1;
    margin: 0 8rpx;
  }
</style>
