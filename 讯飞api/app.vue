<template>
  <div class="sound">
    <textarea
      v-model="text"
      name=""
      id=""
      cols="30"
      rows="10"
      placeholder="点击按钮开始说话，您的语音将会被转为文本，请允许浏览器获取麦克风权限"
    ></textarea>
    <div class="btn">
      <button @click="startSpeechRecognition()">开始识别</button>
      <button @click="closeWebsocket()">停止识别</button>
    </div>
  </div>
  <div class="voice">
    <textarea
      v-model="vioce_text"
      name=""
      id=""
      cols="30"
      rows="10"
      placeholder="点击按钮将文本转为语音"
    ></textarea>
    <div class="btn">
      <button @click="play">开始合成</button>
      <button @click="pause">停止播放</button>
    </div>
  </div>
</template>

<script>
import {
  getWebSocketUrl,
  renderResult,
  resetResultText,
  toBase64,
} from "assets/speech";
import { jsRecorder } from "assets/speech/record.js"; //引入录音文件
import TtsRecorder from "assets/vioce";
const ttsRecorder = new TtsRecorder();
export default {
  name: "home",
  data() {
    return {
      text: "", //文本框文本
      vioce_text: "", //文本转语音
      isRecording: false,
      websocket: null, //websocket定义
      recStart: null, //开始录音
      recStop: null, //结束录音
    };
  },
  methods: {
    play() {
      //要合成的文本
      ttsRecorder.setParams({
        // 文本内容
        text: this.vioce_text,
        // 角色
        //  voiceName: '',
        // 语速
        speed: 50,
        // 音量
        voice: 50,
      });
      ttsRecorder.start();
    },
    pause() {
      ttsRecorder.stop();
    },
    startSpeechRecognition() {
      resetResultText();
      // 判断麦克风是否打开
      navigator.mediaDevices
        .getUserMedia({ audio: true })
        .then((stream) => {
          //调用websocket方法
          // 如果需要加一些图片验证或者其他验证在这里加一些逻辑，成功之后调用websocket
          this.isRecording = true;
          // 调用录音方法
          this.recStart();
          this.asrWebSocket();
        })
        .catch((error) => {
          this.$message.error("麦克风未打开！");
          // 麦克风未打开
          switch (error.message || error.name) {
            case "PERMISSION_DENIED":
            case "PermissionDeniedError":
              console.info("用户拒绝提供信息。");
              break;
            case "NOT_SUPPORTED_ERROR":
            case "NotSupportedError":
              console.info("浏览器不支持硬件设备。");
              break;
            case "MANDATORY_UNSATISFIED_ERROR":
            case "MandatoryUnsatisfiedError":
              console.info("无法发现指定的硬件设备。");
              break;
            default:
              console.info(
                "无法打开麦克风。异常信息:" + (error.code || error.name)
              );
              break;
          }
        });
    },
    asrWebSocket() {
      //使用后端提供的地址
      let websocketUrl = getWebSocketUrl();
      //new 一个websocket
      this.websocket = new WebSocket(websocketUrl);
      console.log(this.websocket, "new websocket");
      this.websocket.onopen = () => {
        console.log("连接成功");
        //链接成功之后发送websocket请求参数
        this.sendStart();
      };
      // 客户端接收服务端返回的数据
      this.websocket.onmessage = (evt) => {
        this.seach = renderResult(evt.data);
        this.text = this.seach;
        console.log("返回识别结果吗", this.seach);
      };
      this.websocket.onerror = (evt) => {
        console.error("websocket-asr错误：", evt);
      };
      // 关闭连接
      this.websocket.onclose = (evt) => {
        console.log("websocket-asr关闭：", evt);
      };
    },
    // 请求参数 =====看后端提供的参数信息========
    sendStart() {
      var params = {
        common: {
          app_id: "92df8dfe",
        },
        business: {
          language: "zh_cn",
          domain: "iat",
          accent: "mandarin",
          vad_eos: 5000,
          dwa: "wpgs",
        },
        data: {
          status: 0,
          format: "audio/L16;rate=16000",
          encoding: "raw",
        },
      };
      this.websocket.send(JSON.stringify(params));
    },
    // 关闭websocket
    closeWebsocket() {
      console.log("关闭socket连接");
      //关闭录音
      this.recStop();
      // websocket关闭参数
      if (this.websocket && this.websocket.readyState == 1) {
        this.websocket.send(
          JSON.stringify({
            data: {
              status: 2,
              format: "audio/L16;rate=16000",
              encoding: "raw",
              audio: "",
            },
          })
        );
      }
      if (!this.isRecording) return;
      this.isRecording = false;
    },
  },
  mounted() {
    // 使用 jsRecorder 创建一个对象并进行解构操作
    const callback = (blob, encTime) => {
      // 发送音频ArrayBuffer
      if (
        this.websocket &&
        this.websocket.readyState == 1 &&
        this.isRecording
      ) {
        // 创建一个FileReader实例
        const reader = new FileReader();
        // 定义onload事件处理器
        reader.onload = function (e) {
          // 这里的e.target.result就是一个ArrayBuffer
          const arrayBuffer = e.target.result;
          // 发送ArrayBuffer
          this.websocket.send(
            JSON.stringify({
              data: {
                status: 1,
                format: "audio/L16;rate=16000",
                encoding: "raw",
                audio: toBase64(new Uint8Array(arrayBuffer)),
              },
            })
          );
        }.bind(this); // 使用bind确保this指向正确
        // 使用FileReader的readAsArrayBuffer方法读取Blob
        reader.readAsArrayBuffer(blob);
      }
    };
    const recorder = jsRecorder(callback);
    // 开始录音、结束录音方法
    this.recStart = recorder.recStart;
    this.recStop = recorder.recStop;
  },
};
</script>
