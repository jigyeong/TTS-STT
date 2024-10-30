<script setup lang="ts">

import {onMounted, ref} from 'vue'

const startBtn = ref<HTMLButtonElement | null>(null);
const stopBtn = ref<HTMLButtonElement | null>(null);
const resultElement = ref<HTMLDivElement | null>(null);
const messageInput = ref<HTMLInputElement | null>(null);

const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
const recognition = new SpeechRecognition();

let isRecording = false;
let onFrameId = null;
let audioStream;
let drawVisual;

onMounted(()=>{
  if (recognition) {
    recognition.continuous = true;
    recognition.interimResults = true;
    recognition.lang = 'ko-KR';

    recognition.onstart = () => {
      startBtn.value.disabled = true;
      stopBtn.value.disabled = false;
      console.log('Recording started');
    };

    recognition.onresult = function (event) {
      let result = '';

      for (let i = event.resultIndex; i < event.results.length; i++) {
        if (event.results[i].isFinal) {
          result += event.results[i][0].transcript + ' ';
        } else {
          result += event.results[i][0].transcript;
        }
      }

      resultElement.value.innerText = result;
      console.log('result ',result);

    };

    recognition.onerror = function (event) {
      startBtn.value.disabled = false;
      stopBtn.value.disabled = true;
      console.error('Speech recognition error:', event.error);
    };

    recognition.onend = function () {
      startBtn.value.disabled = false;
      stopBtn.value.disabled = true;
      console.log('Speech recognition ended');
    };
  } else {
    console.error('Speech recognition not supported');
  }
})

function TTS ( text:string ){
  if (
      typeof SpeechSynthesisUtterance === "undefined" ||
      typeof window.speechSynthesis === "undefined"
  ) {
    return;
  }

  window.speechSynthesis.cancel(); // 현재 읽고있다면 초기화

  const speechMsg = new SpeechSynthesisUtterance();
  speechMsg.rate = 1; // 속도: 0.1 ~ 10
  speechMsg.pitch = 1; // 음높이: 0 ~ 2
  // speechMsg.lang = "ko-KR";
  speechMsg.lang = "ko-KR";
  speechMsg.text = text;

  window.speechSynthesis.speak(speechMsg);
}

function clickReadBtn(){
  console.log(messageInput.value.value);
  TTS(messageInput.value.value);
}

function clickStartBtn(){
  startBtn.value.innerText='녹음중...'
  startVisualization();
  recognition.start();
}

function clickStopBtn(){
  startBtn.value.innerText='말하기'
  stopVisualization();
  recognition.stop();
}

function startVisualization(){
  navigator.mediaDevices
    .getUserMedia({ audio: true })
    .then((stream) => {
      isRecording = true;
      audioStream = stream;

      const context = new AudioContext();
      const analyser = context.createAnalyser();
      analyser.minDecibels = -90;
      analyser.maxDecibels = 90;
      analyser.smoothingTimeConstant = 0.85;
      analyser.fftSize = 256;

      const mediaStreamAudioSourceNode =
          context.createMediaStreamSource(stream);
      mediaStreamAudioSourceNode.connect(analyser, 0);

      /*
      * START: First Sound Bar
      * */

      const pcmData = new Float32Array(analyser.fftSize);

      const onFrame = () => {
        analyser.getFloatTimeDomainData(pcmData);
        let sum = 0.0;
        for (const amplitude of pcmData) {
          sum += amplitude * amplitude;
        }
        const rms = Math.sqrt(sum / pcmData.length);
        const normalizedVolume = Math.min(1, rms / 0.5);
        colorVolumeMeter(normalizedVolume * 2);
        onFrameId = window.requestAnimationFrame(onFrame);
      };

      onFrameId = window.requestAnimationFrame(onFrame);

      /*
      * END: First Sound Bar
      * */

      /*
      * START: Second Sound Bar
      * */
      const canvas = document.querySelector(".visualizer");
      const canvasCtx = canvas.getContext("2d");
      const WIDTH = canvas.width;
      const HEIGHT = canvas.height;

      const bufferLengthAlt = analyser.frequencyBinCount;
      console.log(bufferLengthAlt);
      const dataArrayAlt = new Uint8Array(bufferLengthAlt);

      canvasCtx.clearRect(0, 0, WIDTH, HEIGHT);

      const drawAlt = function () {
        drawVisual = requestAnimationFrame(drawAlt);

        analyser.getByteFrequencyData(dataArrayAlt);

        canvasCtx.fillStyle = "rgb(0, 0, 0)";
        // canvasCtx.fillRect(0, 0, WIDTH, HEIGHT);
        canvasCtx.fillRect(0,0, WIDTH, HEIGHT);

        const barWidth = (WIDTH / bufferLengthAlt);
        let barHeight;
        let x = 0;

        const grad=canvasCtx.createLinearGradient(0,HEIGHT, 0,HEIGHT/2);
        grad.addColorStop(0, "black");
        grad.addColorStop(1, "white");

        for (let i = 0; i < bufferLengthAlt; i++) {
          barHeight = dataArrayAlt[i];

          canvasCtx.fillStyle = grad;
          canvasCtx.fillRect(
              x,
              HEIGHT - barHeight / 2,
              barWidth,
              barHeight / 2
          );

          x += barWidth + 1;
        }
      };

      drawAlt();

      /*
      * END: Second Sound Bar
      * */
    })
    .catch((error) => {
      console.error("마이크 권한 획득 실패", error);
    });
}

function draw() {
  requestAnimationFrame(draw);
  analyser.getByteTimeDomainData(dataArray);

  ctx.fillStyle = 'black';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.lineWidth = 2;
  ctx.strokeStyle = 'green';
  ctx.beginPath();

  const sliceWidth = canvas.width / dataArray.length;
  let x = 0;

  for (let i = 0; i < dataArray.length; i++) {
    const v = dataArray[i] / 128.0; // Normalize to 0-1
    const y = (v * canvas.height) / 2; // Scale to canvas height
    if (i === 0) {
      ctx.moveTo(x, y);
    } else {
      ctx.lineTo(x, y);
    }
    x += sliceWidth;
  }
  ctx.lineTo(canvas.width, canvas.height / 2);
  ctx.stroke();
}

function stopVisualization(){
  if (isRecording) {
    isRecording = false;
    if (onFrameId) window.cancelAnimationFrame(onFrameId);
    console.log('drawVisual : ',drawVisual)
    // if (drawVisual) window.cancelAnimationFrame(drawVisual);
  }
  if (audioStream) {
    audioStream.getTracks().forEach((track) => track.stop());
  }
}

const colorVolumeMeter = (vol) => {
  const VOL_METER_MAX = 10;
  const childrens = document.querySelectorAll(".volumeBar");

  childrens.forEach((child) => {
    child.style.backgroundColor = "#e6e6e6";
  });

  const numberOfChildToColor = normalizeToInteger(vol, 0, VOL_METER_MAX);
  const coloredChild = Array.from(childrens).slice(0, numberOfChildToColor);

  coloredChild.forEach((child) => {
    child.style.backgroundColor = "#4F4FFB";
  });
};

const normalizeToInteger = (volume, min, max) => {
  const scaledValue = Math.min(max, Math.max(min, volume * (max - min) + min));
  return Math.round(scaledValue);
};


</script>

<template>
  <div class="about">
    <div>
      <canvas class="visualizer" width="640" height="100">

      </canvas>
    </div>
    <div>
      <input ref="messageInput" type="text"/>
      <button @click="clickReadBtn">읽기</button>
      <button ref="startBtn" @click="clickStartBtn">말하기</button>
      <button ref="stopBtn" @click="clickStopBtn" disabled>중지</button>
    </div>
    <div id="volumeMeter">
      <div class="volumeBar"></div>
      <div class="volumeBar"></div>
      <div class="volumeBar"></div>
      <div class="volumeBar"></div>
      <div class="volumeBar"></div>

      <div class="volumeBar"></div>
      <div class="volumeBar"></div>
      <div class="volumeBar"></div>
      <div class="volumeBar"></div>
      <div class="volumeBar"></div>

    </div>
    <div>
      <div ref="resultElement"></div>
    </div>
  </div>
</template>

<style>
@media (min-width: 1024px) {
  .about {
    min-height: 100vh;
    display: flex;
    align-items: center;
    flex-direction: column;
    gap: 16px; /* 요소 간의 간격 조정 */
    justify-content: center;
  }
}
#volumeMeter {
  display: flex;
  width: 150px;
  height: 20px;
  gap: 4px;
}

.volumeBar {
  width: 8%;
  border-radius: 4px;
  background-color: darkgray;
}

.soundWave {
  width: 300px;
  height: 50px;
  background-color: darkgray;
}
</style>
