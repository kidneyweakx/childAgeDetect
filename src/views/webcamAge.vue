<template>
  <div class="webrtc_face_recognition">
    <div class="see">
      <video
        id="inputVideo"
        poster="1280x720.png"
        muted
        loop
        playsinline
        @loadedmetadata="FaceAgeAndGender"
      ></video>
      <canvas id="overlay"></canvas>
      <div id="time_and_fps" >
        <span id="right_bottom_time" style="color: white;"></span>
        <span id="right_bottom_fps" style="color: white;"></span>
      </div>
    </div>
    <div class="option">
      <div>
        <label>Control Panel：</label>
        <button @click="fnOpen">Open Webcam</button>
        <button @click="fnClose">End WebCam</button>
      </div>
      <div>
        <span style="margin-right: 20px;"
          >Switch ：</span
        >
        <label>
          Front Camera
          <input
            type="radio"
            v-model="constraints.video.facingMode"
            value="user"
          />
        </label>
        <label>
          Back Camera
          <input
            type="radio"
            v-model="constraints.video.facingMode"
            value="environment"
          />
        </label>
      </div>
      <div>
        <label>Detect Single or All Faces：</label>
        <label>
          Single Faces
          <input type="radio" v-model="detectFace" value="detectSingleFace" />
        </label>
        <label>
          All Faces
          <input type="radio" v-model="detectFace" value="detectAllFaces" />
        </label>
      </div>
      <div>
        <label>Choose diff model</label>
        <label>
          ssdMobilenetv1
          <input type="radio" v-model="nets" value="ssdMobilenetv1" />
        </label>
        <label>
          tinyFaceDetector
          <input type="radio" v-model="nets" value="tinyFaceDetector" />
        </label>
      </div>
    </div>
  </div>
</template>

<script>
import * as faceapi from "face-api.js";
export default {
  name: "WebRTCFaceRecognition",
  data() {
    return {
      nets: "ssdMobilenetv1",
      options: null,
      withBoxes: true,
      detectFace: "detectSingleFace",
      detection: "age_gender",
      videoEl: null,
      canvasEl: null,
      timeout: 0,
      // Stream Settings
      constraints: {
        audio: false,
        video: {
          width: {
            min: 320,
            ideal: 1280,
            max: 1920,
          },
          height: {
            min: 240,
            ideal: 720,
            max: 1080,
          },
          frameRate: {
            min: 15,
            ideal: 30,
            max: 60,
          },
          // front cam | back cam
          facingMode: "environment",
        },
      },
    };
  },
  watch: {
    nets(val) {
      this.nets = val;
      this.fnInit();
    },
    detection(val) {
      this.detection = val;
      this.videoEl.pause();
      setTimeout(() => {
        this.videoEl.play();
        setTimeout(() => this.FaceAgeAndGender(), 300);
      }, 300);
    },
  },
  mounted() {
    this.$nextTick(() => {
      this.fnInit();
    });
  },
  methods: {
    // load models
    async fnInit() {
      await faceapi.nets[this.nets].loadFromUri("./models"); // algo
      await faceapi.loadFaceLandmarkModel("./models"); // landmark
      await faceapi.loadFaceExpressionModel("./models"); // expression
      await faceapi.loadAgeGenderModel("./models"); // age
      // change nets
      switch (this.nets) {
        case "ssdMobilenetv1":
          this.options = new faceapi.SsdMobilenetv1Options({
            minConfidence: 0.5, // 0.1 ~ 0.9
          });
          break;
        case "tinyFaceDetector":
          this.options = new faceapi.TinyFaceDetectorOptions({
            inputSize: 512, // 160 224 320 416 512 608
            scoreThreshold: 0.5, // 0.1 ~ 0.9
          });
          break;
      }

      this.videoEl = document.getElementById("inputVideo");
      this.canvasEl = document.getElementById("overlay");
    },
    // Draw Result
    async FaceAgeAndGender() {
      console.log("RunFaceAgeAndGender");
      if (this.videoEl.paused) return clearTimeout(this.timeout);
      const result = await faceapi[this.detectFace](this.videoEl, this.options)
        .withFaceLandmarks()
        .withAgeAndGender();
      if (result && !this.videoEl.paused) {
        const dims = faceapi.matchDimensions(this.canvasEl, this.videoEl, true);
        const resizeResults = faceapi.resizeResults(result, dims);
        this.withBoxes
          ? faceapi.draw.drawDetections(this.canvasEl, resizeResults)
          : faceapi.draw.drawFaceLandmarks(this.canvasEl, resizeResults);
        if (Array.isArray(resizeResults)) {
          resizeResults.forEach((result) => {
            const { age, gender, genderProbability } = result;
            new faceapi.draw.DrawTextField(
              [
                `${Math.round(age, 0)} years`,
                `${gender} (${Math.round(genderProbability)})`,
              ],
              result.detection.box.bottomLeft
            ).draw(this.canvasEl);
          });
        } else {
          const { age, gender, genderProbability } = resizeResults;
          new faceapi.draw.DrawTextField(
            [
              `${Math.round(age, 0)} years`,
              `${gender} (${Math.round(genderProbability)})`,
            ],
            resizeResults.detection.box.bottomLeft
          ).draw(this.canvasEl);
        }
      } 
      this.timeout = setTimeout(() => this.FaceAgeAndGender());
    },

    // open webcam
    fnOpen() {
      if (typeof window.stream === "object") return;
      clearTimeout(this.timeout);
      this.timeout = setTimeout(() => {
        clearTimeout(this.timeout);
        navigator.mediaDevices
          .getUserMedia(this.constraints)
          .then(this.fnSuccess)
          .catch(this.fnError);
      }, 300);
    },
    // webcam Success!
    fnSuccess(stream) {
      window.stream = stream;
      this.videoEl.srcObject = stream;
      this.videoEl.play();
    },
    // webcam Error!
    fnError(error) {
      console.log(error);
      alert("Error :" + error);
    },
    // End Webcam
    fnClose() {
      this.canvasEl
        .getContext("2d")
        .clearRect(0, 0, this.canvasEl.width, this.canvasEl.height);
      this.videoEl.pause();
      clearTimeout(this.timeout);
      if (typeof window.stream === "object") {
        window.stream.getTracks().forEach((track) => track.stop());
        window.stream = "";
        this.videoEl.srcObject = null;
      }
    },
  },
  beforeDestroy() {
    this.fnClose();
  },
};
</script>

<style scoped>
button {
  height: 30px;
  border: 2px #42b983 solid;
  border-radius: 4px;
  background: #42b983;
  color: white;
  margin: 10px;
}
.see {
  border: 4px solid #37474F;  
  box-sizing:content-box;
  position: relative;
  overflow:auto;
}
.see canvas {
  position: absolute;
  top: 0;
  left: 0;
}
.option {
  padding-bottom: 20px;
}
.option div {
  padding: 10px;
  border-bottom: 2px #42b983 solid;
}
.option div label {
  margin-right: 20px;
}
</style>
