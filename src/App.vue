<template>
  <input type="file" accept="video/*" @change="handleFile" />
  <div id="wasm-playground">
    <div style="width: 100%">
      <video id="video" controls ref="testVideo" src="" width="600" height="550"></video>
      <canvas id="canvas" ref="testCanvas" src="" width="  598" height="550" style="border: 1px solid #000"></canvas>
      <canvas id="offscreen" ref="offscreenDom" src="" width="  598" height="550" style="border: 1px solid #000; display: none"></canvas>
    </div>

    <h2>帧率: {{ fpsNum }}</h2>

    <div style="margin-left: 100px">
      <h3>灰度滤镜运算</h3>
      <select v-model="modeChoose" name="" id="" @change="modeChange">
        <option value="0">未设置</option>
        <option value="1">JavaScript</option>
        <option value="2">WASM</option>
      </select>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref } from "vue";

export default defineComponent({
  name: "App",
  setup() {
    const testVideo = ref<HTMLVideoElement>();
    const testCanvas = ref<HTMLCanvasElement>();
    const offscreenDom = ref<HTMLCanvasElement>();
    const offscreen = ref<OffscreenCanvas>();
    const offscreenCtx = ref<any>();
    const canvasSize = ref<{ width: number; height: number }>({ width: 0, height: 0 });
    const canvasCtx = ref<CanvasRenderingContext2D>();
    const videoSize = ref<{ width: number; height: number }>({ width: 0, height: 0 });
    const fpsNum = ref<number>(0);
    const modeChoose = ref<number>(0);
    let animeTimer = 0;
    let frameCount = 0;
    let startTime = 0;
    let lastTime = 0;


    const videoPlayCallBack = () => {
      startTime = performance.now();
      lastTime = startTime;
      canvasAnime();
    };

    const videoPauseCallBack = () => {
      cancelAnimationFrame(animeTimer);
    };

    const drawVideoToCanvas = () => {
      let scale = videoSize.value.width / canvasSize.value.width;
      let width = canvasSize.value.width;
      let height = videoSize.value.height / scale;
      let top = (canvasSize.value.height - height) / 2;

      if (offscreenCtx.value) {
        offscreenCtx.value.drawImage(testVideo.value!, 0, top, width, height);
        const imageData = offscreenCtx.value.getImageData(0, 0, canvasSize.value.width, canvasSize.value.height);

        //灰度图‘
        const data = imageData.data;
        for (let i = 0; i < data.length; i += 4) {
          const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
          data[i] = avg;
          data[i + 1] = avg;
          data[i + 2] = avg;
        }

        //渲染
        canvasCtx.value!.putImageData(imageData, 0, 0);
      }
    };

    const canvasAnime = (currentTime?: number) => {
      drawVideoToCanvas();
      animeTimer = requestAnimationFrame(canvasAnime);
      //计算帧率
      frameCount++;
      if (currentTime! - lastTime >= 1000) {
        fpsNum.value = frameCount;
        frameCount = 0;
        lastTime = currentTime!;
      }
    };

    onMounted(() => {
      if (testVideo.value && testCanvas.value) {
        canvasCtx.value = testCanvas.value.getContext("2d", { willReadFrequently: false })!;
        canvasSize.value.width = testCanvas.value.width;
        canvasSize.value.height = testCanvas.value.height;
        offscreen.value = new OffscreenCanvas(canvasSize.value.width, canvasSize.value.height);
        offscreenCtx.value = offscreen.value.getContext("2d", { willReadFrequently: false })!;
      }

      //@ts-ignore
      window.processVideoData = (imageData: ImageData, fps: number) => {
        // console.log(imageData,fpsNum)
        const data = imageData.data;
        //灰度图‘
        for (let i = 0; i < data.length; i += 4) {
          const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
          data[i] = avg;
          data[i + 1] = avg;
          data[i + 2] = avg;
        }
        canvasCtx.value!.putImageData(imageData, 0, 0);
        fpsNum.value = fps;
      };

      //@ts-ignore
      const go = new Go(); // Defined in wasm_exec.js

      WebAssembly.instantiateStreaming(fetch("/wasm/main.wasm"), go.importObject).then((result) => {
        go.run(result.instance);
        //@ts-ignore
        wasm_handleFile = window.Wasm_handleFile as Function;
        //@ts-ignore
        wasm_videoPlayCallBack = window.Wasm_videoPlayCallBack as Function;
        //@ts-ignore
        wasm_videoPauseCallBack = window.Wasm_videoPauseCallBack as Function;
      });

      console.log(window);
    });

    const modeChange = () => {
      console.log(modeChoose.value);
      if (modeChoose.value == 1) {

        testVideo.value!.onplay = videoPlayCallBack;
        testVideo.value!.onpause = videoPauseCallBack;

      } else {
        testVideo.value!.onplay = wasm_videoPlayCallBack;
        testVideo.value!.onpause = wasm_videoPauseCallBack;
      }
      console.log(testVideo.value!.onplay);
      console.log(testVideo.value!.onpause);
    };

    let wasm_handleFile: any;
    let wasm_videoPlayCallBack: any;
    let wasm_videoPauseCallBack: any;

    //@ts-ignore
    window.processVideoData = (imageData: ImageData, fps: number) => {
      // console.log(imageData,fpsNum)
      const data = imageData.data;
      //灰度图‘
      for (let i = 0; i < data.length; i += 4) {
        const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
        data[i] = avg;
        data[i + 1] = avg;
        data[i + 2] = avg;
      }
      canvasCtx.value!.putImageData(imageData, 0, 0);
      fpsNum.value = fps;
    };

    const handleFile = (event: Event) => {
      const file = (event.target as HTMLInputElement).files![0];

      if (modeChoose.value == 1) {
        const url = URL.createObjectURL(file);

        testVideo.value!.src = url;
        //获取视频的宽高
        testVideo.value!.onloadedmetadata = () => {
          console.log(testVideo.value!.videoWidth);
          console.log(testVideo.value!.videoHeight);
          videoSize.value.width = testVideo.value!.videoWidth;
          videoSize.value.height = testVideo.value!.videoHeight;
        };
      }

      if (modeChoose.value == 2) {
        //在testVideo上播放 javascript
        let videoSize_wasm = {
          width: videoSize.value.width,
          height: videoSize.value.height,
        };
        let canvasSize_wasm = {
          width: canvasSize.value.width,
          height: canvasSize.value.height,
        };
        wasm_handleFile(file, testVideo, videoSize_wasm, canvasCtx.value, canvasSize_wasm);
      }
    };

    return {
      handleFile: handleFile,
      testVideo: testVideo,
      testCanvas: testCanvas,
      fpsNum: fpsNum,
      modeChoose: modeChoose,
      modeChange: modeChange,
      offscreenDom: offscreenDom,
    };
  },
});
</script>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
