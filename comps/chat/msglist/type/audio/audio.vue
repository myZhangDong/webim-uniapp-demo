<template>
<view class="audio-player" @tap="audioPlay" :style="'opacity: ' + opcity">
	<text class="time">语音消息 {{ time }}</text>
	<view class="controls play-btn" @tap="audioPlay">
		<image :src="style == 'self'? '../../../../../static/images/voicemsgmy.png' : '../../../../../static/images/voicemsg.png'"></image>
	</view>
</view>
</template>

<script>
let audioCtxFc = require("./audioCtxFactory");
let playStatus = require("./playStatus");

export default {
  data() {
    return {
      playStatus: playStatus,
      curStatus: playStatus.STOP,
      time: "0'",
      opcity: 1,
      __comps__: {
        audioCtx: null
      },
      style: ""
    };
  },

  components: {},
  props: {
    msg: {
      type: Object,
      val: {}
    }
  },
  obeyMuteSwitch: false,
  autoplay: true,

  // lifetimes
  created() {},

  beforeMount() {
    this.time = this.msg.msg.length + "''"
    this.style = this.msg.style
  },

  moved() {},

  destroyed() {
    let audioCtx = this.$data.__comps__.audioCtx = audioCtxFc.getCtx(this.msg.mid);
    this.audioPause(audioCtx);
    this.delEvent(); //audioCtx.destroy();
  },

  mounted() {
    let self = this;
    let curl = '';
    let audioCtx = this.$data.__comps__.audioCtx = audioCtxFc.getCtx(this.msg.mid);
    audioCtx.autoplay = false;
    audioCtx.loop = false; //

    this.onPlaying = () => {
      //console.log("onPlaying", JSON.stringify(this.data));
      this.curStatus = playStatus.PLAYING
      uni.inter && clearInterval(uni.inter);
      uni.inter = setInterval(() => {
        let opcity = this.opcity;
        this.opcity = opcity == 1 ? 0.4 : 1
      }, 500);
    };

    this.onPause = () => {
      // console.log("onPause", JSON.stringify(this.data));
      // 第二次播放会立即抛出一个异常的 onPause
      if (parseInt(this.time, 10) < 1) {
        return;
      }

      this.curStatus = playStatus.PAUSE
      this.opcity = 1
    };

    this.onDone = () => {
      // console.log("onDone", JSON.stringify(this.data));
      this.curStatus = playStatus.STOP
      this.opcity = 1
      clearInterval(uni.inter);
    }; // 多次播放会丢失这个回调


    this.onTimeUpdate = () => {
      this.time = (audioCtx.currentTime >> 0) + "'"
    };

    this.onWait = () => {
      uni.showToast({
        title: "下载中...",
        duration: 1000
      });
    };

    this.addEvent();
  },

  methods: {
    audioPlay() {
      uni.inter && clearInterval(uni.inter);
      let audioCtx = this.$data.__comps__.audioCtx;
      var curl = '';
      uni.downloadFile({
        url: this.msg.msg.data,
        header: {
          "X-Requested-With": "XMLHttpRequest",
          Accept: "audio/mp3",
          Authorization: "Bearer " + this.msg.msg.token
        },

        success(res) {
          curl = res.tempFilePath;
          console.log('音频本地', audioCtx); //renderableMsg.msg.url = res.tempFilePath;

          audioCtx.src = curl;
          audioCtx.play();
        },

        fail(e) {
          console.log("downloadFile failed", e);
          uni.showToast({
            title: "下载失败",
            duration: 1000
          });
        }

      });
    },

    audioPause(auCtx) {
      //let audioCtx = this.data.__comps__.audioCtx;
      let audioCtx = this.$data.__comps__.audioCtx = audioCtxFc.getCtx(this.msg.mid) || auCtx;
      audioCtx && audioCtx.pause();
    },

    addEvent() {
      let audioCtx = this.$data.__comps__.audioCtx;
      audioCtx.onPlay(this.onPlaying);
      audioCtx.onPause(this.onPause);
      audioCtx.onWaiting(this.onPause);
      audioCtx.onStop(this.onDone);
      audioCtx.onEnded(this.onDone);
      audioCtx.onError(this.onDone);
      audioCtx.onWaiting(this.onWait); //audioCtx.onTimeUpdate(this.onTimeUpdate);
    },

    delEvent() {
      let audioCtx = this.$data.__comps__.audioCtx;
      audioCtx.offPlay(this.onPlaying);
      audioCtx.offPause(this.onPause);
      audioCtx.offWaiting(this.onPause);
      audioCtx.offStop(this.onDone);
      audioCtx.offEnded(this.onDone);
      audioCtx.offError(this.onDone);
      audioCtx.offWaiting(this.onWait); // 多次播放会丢失这个回调，所以不用卸载
      // audioCtx.offTimeUpdate(this.onTimeUpdate);
    }

  }
};
</script>
<style>
@import "./audio.css";
</style>