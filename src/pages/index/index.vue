<template>
  <view class="demo-page">
    <view class="demo-title">video-player-xyyt 跨端视频播放示例</view>

    <!-- 通过 easycom 自动引入 uni_modules 组件，无需手动 import -->
    <video-player-xyyt
      ref="playerRef"
      :src="videoUrl"
      :poster="posterUrl"
      :autoplay="false"
      :speed-step="15"
      @play="onEvent('play')"
      @pause="onEvent('pause')"
      @ended="onEvent('ended')"
      @fullscreenchange="onFullScreen"
      @timeupdate="onTime"
      @error="onEvent('error')"
    />

    <view class="demo-panel">
      <view class="demo-row">最近事件：{{ lastEvent }}</view>
      <view class="demo-row">播放进度：{{ fmt(cur) }} / {{ fmt(dur) }}</view>
      <view class="demo-btns">
        <button size="mini" @click="callPlay">外部调用：播放</button>
        <button size="mini" @click="callPause">外部调用：暂停</button>
        <button size="mini" @click="callSeek">外部调用：跳到 30s</button>
      </view>
    </view>

    <view class="demo-tip">
      可用功能：全屏（Android/iOS/鸿蒙/小程序/H5）、播放/暂停、+15s / -15s 快进快退、拖动进度条。
      videoUrl 请替换为你自己的可访问视频地址。
    </view>
  </view>
</template>

<script setup lang="ts">
  import { ref } from 'vue';

  // 示例公网视频，实际使用请替换为业务视频地址
  const videoUrl = ref('https://www.w3schools.com/html/mov_bbb.mp4');
  const posterUrl = ref('');

  const playerRef = ref<any>(null);
  const lastEvent = ref('-');
  const cur = ref(0);
  const dur = ref(0);

  function onEvent(name: string) {
    lastEvent.value = name;
  }
  function onFullScreen(e: { fullScreen: boolean }) {
    lastEvent.value = e.fullScreen ? '进入全屏' : '退出全屏';
  }
  function onTime(e: { currentTime: number; duration: number }) {
    cur.value = e.currentTime;
    dur.value = e.duration;
  }

  function callPlay() {
    playerRef.value?.play();
  }
  function callPause() {
    playerRef.value?.pause();
  }
  function callSeek() {
    playerRef.value?.seek(30);
  }

  function fmt(sec: number) {
    const s = Math.max(0, Math.floor(sec || 0));
    const m = Math.floor(s / 60);
    const r = s % 60;
    return (m < 10 ? '0' + m : '' + m) + ':' + (r < 10 ? '0' + r : '' + r);
  }
</script>

<style lang="scss" scoped>
  .demo-page {
    padding: 24rpx;
  }
  .demo-title {
    font-size: 32rpx;
    font-weight: 600;
    color: #333;
    margin-bottom: 20rpx;
  }
  .demo-panel {
    margin-top: 24rpx;
    padding: 20rpx;
    background: #fff;
    border-radius: 12rpx;
  }
  .demo-row {
    font-size: 26rpx;
    color: #555;
    margin-bottom: 12rpx;
  }
  .demo-btns {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    gap: 12rpx;
    margin-top: 8rpx;
  }
  .demo-tip {
    margin-top: 24rpx;
    font-size: 24rpx;
    color: #999;
    line-height: 36rpx;
  }
</style>
