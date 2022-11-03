<!--
 * @Descripttion: 
 * @Author: junshi junshi@ssc-hn.com
 * @Date: 2022-10-18
 * @LastEditors: junshi junshi@ssc-hn.com
 * @LastEditTime: 2022-11-03
-->
<script setup lang="ts">
import { useRouter, useRoute } from "vue-router";
import { ref, reactive, watch } from "vue";

import { storeToRefs } from "pinia";
import SongInfo from "@/components/SongInfo.vue";
import { useSongStore } from "@/stores/song";
import { useLyricStore } from "@/stores/lyric";

const router = useRouter();

const state = reactive({
  isPlay: false,
  menuList: ["正在播放", "推荐", "搜索", "播放历史"],
  mentRouter: ["playlist", "toplist", "search", "historylist"],
  /**当前分类*/
  activeIndex: 0,
  songList: [],
  audioProgress: 0, // 进度
  playType: 1, // 播放类型，1:列表循环  2：随机播放 3：单曲循环
  playIndex: 0, // 当前播放哪一首
  thumbTranslateX: 0,
  lyricIndex: 0, // 歌词到哪一行了
  currentTime: 0, // 当前播放进度
  progressL: 0, // 进度条总长度
  lyricInfo: [],
  endTime: 0,
  volume: 80,
  playStatus: false, // 搜索或者历史播放完成后关闭播放状态按钮
});
const audio = ref();
const { song, isPlay } = storeToRefs(useSongStore());
const { onChangeSong, onChangePlay } = useSongStore();
const { lyricInfo } = useLyricStore();

const ChangeActive = async (index: number) => {
  state.activeIndex = index;
  router.push({
    path: `/${state.mentRouter[state.activeIndex]}`,
  });
};

const TimeToString = (seconds: string) => {
  let param = parseInt(seconds);
  let hh = "",
    mm = "",
    ss = "";
  if (param >= 0 && param < 60) {
    param < 10 ? (ss = "0" + param) : (ss = param);
    return "00:" + ss;
  } else if (param >= 60 && param < 3600) {
    mm = parseInt(param / 60);
    mm < 10 ? (mm = "0" + mm) : mm;
    param - parseInt(mm * 60) < 10 ? (ss = "0" + String(param - parseInt(mm * 60))) : (ss = param - parseInt(mm * 60));
    return mm + ":" + ss;
  }
};

/**
 * 获取音乐时长
 */
function getDuration() {
  state.endTime = audio.value.duration;
}

/**
 *  当前播放时间
 */
const timeupdate = (e: any) => {
  let compareTime = e.target.currentTime;
  for (let i = 0; i < lyricInfo.length; i++) {
    if (compareTime > parseInt(lyricInfo[i].time)) {
      const index = lyricInfo[i].index;
      if (i === parseInt(index)) {
        state.lyricIndex = i;
      }
    }
  }
  state.currentTime = compareTime;
  // state.audioProgress = compareTime / audio.value.duration;
  // state.thumbTranslateX = (state.audioProgress * state.progressL).toFixed(3);
};

/**
 * 通过进度条改变当前音频进度
 * @param value 当前的值
 */
function changeCurrentTime(value: number) {
  audio.value.currentTime = parseInt(value);
  state.currentTime = value;
}

/**
 * 播放到媒体的结束位置
 */
const palyEnded = () => {
  console.log("is play end", state.playType);
  onChangeSong("next");
};

/**
 * 控制播放按钮
 */
function controlPlay() {
  onChangePlay(!isPlay.value);
  if (!audio.value.paused) {
    audio.value.pause();
  } else {
    audio.value.play();
  }
}

function play() {
  onChangePlay(true);
}
function pause() {
  onChangePlay(false);
}

/**
 * 监听歌曲源变化
 */
watch(
  () => song.value.url,
  (newValue) => {
    audio.value.load();
    var playPromise = audio.value.play();
    if (playPromise !== undefined) {
      playPromise
        .then((_: any) => {
          play();
        })
        .catch((error: any) => {
          audio.value.load();
          // play();
          console.log(error);
        });
    }
  }
);
/**
 * 监听播放开启变化
 */
watch(
  () => isPlay.value,
  (newValue) => {
    if (!audio.value.paused && !newValue) {
      audio.value.pause();
    } else {
      audio.value.play();
    }
  }
);
</script>

<template>
  <div class="net-easy-player">
    <div class="background-flitter" :style="`background-image: url(${song?.image});`"></div>
    <div class="music-mask"></div>

    <audio
      ref="audio"
      hidden="true"
      @canplay="getDuration"
      @timeupdate="timeupdate"
      @ended="palyEnded"
      @play="play"
      @pause="pause"
      :volume="state.volume / 100"
    >
      <source :src="song?.url" type="audio/mpeg" />
    </audio>
    <div class="music-header">
      <div></div>
      <div>vue3开发播放器</div>
      <div>登录</div>
    </div>
    <div class="music-container">
      <div class="list-main">
        <div class="menu-box">
          <div
            :class="`menu-button${state.activeIndex === index ? ' active' : ''}`"
            v-for="(item, index) in state.menuList"
            :key="index"
            @click="ChangeActive(index)"
          >
            <span class="bnt"> {{ item }}</span>
          </div>
        </div>
        <div class="main-container">
          <!-- <router-view v-slot="{ Component }">
            <keep-alive>
              <component :is="Component" :key="$route.name" v-if="$route.meta.keepAlive" />
            </keep-alive>
            <component :is="Component" :key="$route.name" v-if="!$route.meta.keepAlive" />
          </router-view> -->
          <RouterView />
        </div>
      </div>
      <div class="song-info-warp">
        <SongInfo :lyricIndex="state.lyricIndex"></SongInfo>
      </div>
    </div>
    <div class="music-footer">
      <div class="song-cover">
        <img class="audioCover" :src="song.image" :title="`${song?.name}`" />
      </div>
      <div class="play-icon-container">
        <img class="play-icon" src="../assets/arrow_01.png" @click="onChangeSong('last')" alt="上一曲" />
        <img v-show="!isPlay" @click="controlPlay" class="play-icon" src="../assets/play_01.png" alt="播放" />
        <img v-show="isPlay" @click="controlPlay" class="play-icon" src="../assets/play_02.png" alt="暂停" />
        <img class="play-icon" src="../assets/arrow_02.png" @click="onChangeSong('next')" alt="下一曲" />
      </div>
      <div class="music-speed">
        <div class="name-time">
          <div>{{ song.name }}</div>
          <div>{{ TimeToString(state.currentTime) }}/{{ TimeToString(state.endTime) }}</div>
        </div>
        <el-slider
          v-model="state.currentTime"
          :min="0"
          :step="0.1"
          :max="state.endTime"
          :show-tooltip="false"
          :show-input-controls="false"
          @change="changeCurrentTime"
        />
      </div>
      <div class="action-container">
        <el-dropdown>
          <span class="el-dropdown-link"> 模式 </span>
          <template #dropdown>
            <el-dropdown-menu>
              <el-dropdown-item>列表循环</el-dropdown-item>
              <el-dropdown-item>随机播放</el-dropdown-item>
              <el-dropdown-item>单曲循环</el-dropdown-item>
            </el-dropdown-menu>
          </template>
        </el-dropdown>
        <div class="volume">
          <el-slider v-model="state.volume"></el-slider>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="less" scoped>
.net-easy-player {
  width: 100%;
  height: 100%;
  color: #fff;
  overflow: hidden;
  font-size: 18px;

  /*遮罩*/
  .music-mask,
  .background-flitter {
    position: absolute;
    top: 0;
    right: 0;
    left: 0;
    bottom: 0;
    width: 100%;
    height: 100%;
  }

  .music-mask {
    z-index: -1;
    background: rgba(126, 126, 126, 0.4);
  }

  .background-flitter {
    z-index: -2;
    background-repeat: no-repeat;
    background-size: cover;
    background-position: 50%;
    filter: blur(12px);
    opacity: 0.7;
    transition: all 0.8s;
    transform: scale(1.1);
  }

  .music-header {
    width: 100%;
    line-height: 60px;
    font-size: 18px;

    display: flex;
    justify-content: space-between;
    padding: 0 24px;
    box-sizing: border-box;
  }

  .music-container {
    position: relative;
    width: 100%;
    height: calc(~"100% - 144px");
    display: flex;

    .list-main {
      width: calc(~"100% - 320px");
      height: 100%;

      .menu-box {
        width: 100%;
        display: flex;
        flex-direction: row;
        height: 60px;
        align-items: center;
        padding-left: 24px;
        box-sizing: border-box;
        overflow: hidden;

        .menu-button {
          width: 96px;
          border: 1px solid #dddcdc;
          height: 36px;
          text-align: center;
          color: #dddcdc;
          line-height: 35px;
          font-size: 15px;
          margin-right: 20px;
          cursor: pointer;
          border-radius: 3px;
          box-sizing: border-box;
          position: relative;
          overflow: hidden;

          &:hover {
            border-color: @vue_color;
            color: @vue_color;
            box-shadow: 1px 1px 8px 2px @vue_color_option;
          }

          &:before {
            content: "";
            position: absolute;
            top: 0;
            left: -100%;
            width: 100%;
            height: 100%;
            background: linear-gradient(120deg, transparent, @vue_color_option, transparent);
            transition: all 650ms;
          }

          &:hover:before {
            left: 100%;
          }

          &.active {
            --border-width: 2px;
            position: relative;
            display: flex;
            justify-content: center;
            align-items: center;
            line-height: 36px;
            // width: 300px;
            // height: 200px;
            font-family: Lato, sans-serif;
            // font-size: 2.5rem;
            text-transform: uppercase;
            color: white;
            border-color: white;
            background: rgba(235, 227, 227, 0.6);
            border-radius: var(--border-width);
            box-shadow: none;

            &::before {
              display: none;
            }

            &::after {
              position: absolute;
              content: "";
              top: calc(-1 * var(--border-width));
              left: calc(-1 * var(--border-width));
              z-index: -1;
              width: calc(100% + var(--border-width) * 2);
              height: calc(100% + var(--border-width) * 2);
              background: linear-gradient(
                60deg,
                hsl(224, 85%, 66%),
                hsl(269, 85%, 66%),
                hsl(314, 85%, 66%),
                hsl(359, 85%, 66%),
                hsl(44, 85%, 66%),
                hsl(89, 85%, 66%),
                hsl(134, 85%, 66%),
                hsl(179, 85%, 66%)
              );
              background-size: 300% 300%;
              background-position: 0 50%;
              border-radius: calc(2 * var(--border-width));
              animation: moveGradient 4s alternate infinite;
            }
          }

          @keyframes moveGradient {
            50% {
              background-position: 100% 50%;
            }
          }
        }
      }

      .main-container {
        width: 100%;
        box-sizing: border-box;
        padding-left: 24px;
        height: calc(~"100% - 60px");
        position: relative;
        overflow: auto;
      }
    }

    .song-info-warp {
      width: 320px;
      height: 100%;
      overflow: hidden;
    }
  }

  .music-footer {
    width: 100%;
    height: 72px;
    padding: 0 24px;
    box-sizing: border-box;
    display: flex;
    box-shadow: -5px 0 20px rgb(83, 80, 80);

    .play-icon-container {
      width: 240px;
      height: 100%;
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 0 20px;
      box-sizing: border-box;

      .play-icon {
        width: 40px;
        height: 40px;
        display: block;
        cursor: pointer;
      }
    }

    .song-cover {
      width: 80px;
      padding: 9px 16px 9px 10px;
      box-sizing: border-box;

      .audioCover {
        display: block;
        width: 54px;
        height: 54px;
        border-radius: 8px;
      }
    }

    .music-speed {
      width: calc(~"100% - 520px");
      height: 100%;
      position: relative;

      .name-time {
        width: 100%;
        height: 36px;
        line-height: 36px;
        color: #fff;
        display: flex;
        justify-content: space-between;
        font-size: 14px;
      }
    }

    .action-container {
      width: 200px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 0 20px;
      box-sizing: border-box;

      .el-dropdown-link {
        color: #fff;
      }

      .volume {
        width: 100px;
      }
    }
  }
}
</style>
<style>
.el-slider .el-slider__bar {
  background-color: #40ce8f !important;
}

.el-slider__button {
  border: #40ce8f;
}
</style>