<template>
  <div>
    <form>
      <div class="id">
        <input type="text" value="" v-model="name" placeholder="请输入用户名" />
        <input type="text" value="" v-model="age" placeholder="请输入年龄" />
        <input
          type="file"
          accept=".jpg, .jpeg, .png"
          @change="getFile($event, '1')"
        />
        <img :src="imgUrl1" alt="身份证" />
        <input
          type="file"
          accept=".jpg, .jpeg, .png"
          @change="getFile($event, '2')"
        />
        <img :src="imgUrl2" alt="学生卡" />
      </div>
      <button @click="submitForm($event)">提交</button>
      <button @click="showCamera()">开启摄像头</button>
      <div id="bg-content" style="display: none">
        <div class="show-photo">
          <canvas id="myCanvas" width="600px" height="600px" />
          <video
            id="video"
            width="600px"
            height="600px"
            autoplay="autoplay"
          ></video>
          <canvas
            id="canvas"
            width="600px"
            height="600px"
            style="display: none"
          ></canvas>
          <a id="downloadA"></a>
        </div>
        <div class="take-photo">
          <button type="button" @click="takePhoto()">拍照</button>
        </div>
        <div class="close-photo">
          <button type="button" @click="closePhoto()">关闭</button>
        </div>
      </div>
    </form>
  </div>
</template>

<script>
import axios from "axios";
import html2canvas from "html2canvas";
// import nodejs bindings to native tensorflow,
// not required, but will speed up things drastically (python required)
// import '@tensorflow/tfjs-node';

// // implements nodejs wrappers for HTMLCanvasElement, HTMLImageElement, ImageData
// import * as canvas from 'canvas';

import * as faceapi from "face-api.js";
export default {
  data() {
    return {
      name: "",
      age: "",
      file1: "",
      file2: "",
      imgUrl1: "",
      imgUrl2: "",
      video: null,
      canvas: null,
      dir: "https://static.insta360.com/face/model",
      displaySize: {},
      timeout:null,
      faceapiReady: false
    };
  },
  mounted() {
    this.$nextTick(() => {
      this.init();
    });
  },
  methods: {
    async init() {
      this.video = document.getElementById("video");
      this.canvas = document.getElementById("myCanvas");
      this.displaySize = { width: video.width, height: video.height };
      video.addEventListener("play", () => {
        console.log("play lisetner");
        console.log(this.video)
        // this.canvas = faceapi.createCanvasFromMedia(this.video);
        // document.body.append(this.canvas);
        this.getResult();
        this.timeout = setInterval(this.getResult,1000);
      });
      // 加载模型
      await faceapi.nets.ssdMobilenetv1.loadFromUri(this.dir);
      await faceapi.nets.faceLandmark68Net.loadFromUri(this.dir);
      console.log("模型加载成功");
      this.faceapiReady = true
    },
    async detect() {
      // const options = new faceapi.TinyFaceDetectorOptions({ scoreThreshold: 0.2, inputSize: 608 });
      const options = new faceapi.SsdMobilenetv1Options({
        minConfidence: 0.5,
        maxResults: 3,
      });
      // 检测人脸
      const detections = await faceapi
        .detectAllFaces(this.video, options)
        .withFaceLandmarks();
      return detections;
    },
    draw(detections) {
      const dims = faceapi.matchDimensions(this.canvas, this.video, true);
      const resizedDetections = faceapi.resizeResults(detections, dims);
      this.canvas.getContext("2d").clearRect(0, 0, this.canvas.width, this.canvas.height);
      faceapi.draw.drawDetections(this.canvas, resizedDetections);
      faceapi.draw.drawFaceLandmarks(this.canvas, resizedDetections);
    },
    async getResult() {
      if (!this.faceapiReady) {
                return;
            }
      console.log("aaaaaa")
      let detections = await this.detect();
      this.draw(detections);
    },
    getFile(event, img) {
      if (img === "1") {
        this.file1 = event.target.files[0];
        this.imgUrl1 = this.getImg(this.file1);
      } else {
        this.file2 = event.target.files[0];
        this.imgUrl2 = this.getImg(this.file2);
      }
    },
    getImg(file) {
      let url = "";
      if (window.createObjectURL !== undefined) {
        // basic
        url = window.createObjectURL(file);
      } else if (window.URL !== undefined) {
        // mozilla(firefox)
        url = window.URL.createObjectURL(file);
      } else if (window.webkitURL !== undefined) {
        // webkit or chrome
        url = window.webkitURL.createObjectURL(file);
      }
      return url;
    },
    showCamera() {
      document.getElementById("bg-content").style = "display:block";
      // console.log(document.getElementsByClassName("bg-content")[0].style)
      let constraints = {
        video: { width: 600, height: 600 },
        audio: true,
      };
      //获得video摄像头区域
      let video = document.getElementById("video");
      //这里介绍新的方法，返回一个 Promise对象
      // 这个Promise对象返回成功后的回调函数带一个 MediaStream 对象作为其参数
      // then()是Promise对象里的方法
      // then()方法是异步执行，当then()前的方法执行完后再执行then()内部的程序
      // 避免数据没有获取到
      let promise = navigator.mediaDevices.getUserMedia(constraints);
      let that = this;
      promise.then(function (MediaStream) {
        that.mediaStreamTrack =
          typeof MediaStream.stop === "function"
            ? MediaStream
            : MediaStream.getTracks()[1];
        video.srcObject = MediaStream;
        video.muted = true;
        video.play();
      });
    },
    /**
     * 拍摄照片
     */
    takePhoto() {
      let video = document.getElementById("video");
      let canvas = document.getElementById("canvas");
      let ctx = canvas.getContext("2d");
      ctx.drawImage(video, 0, 0);
      ctx.translate(canvas.width, 0);
      ctx.scale(-1, 1);
      let img = document.getElementById("canvas").toDataURL("image/png");
      var formData = new FormData();
      var blob = this.dataURItoBlob(img);
      formData.append("file", blob, Date.now() + ".jpg");
      let config = {
        headers: {
          "Content-Type": "multipart/form-data",
        },
      };
      axios
        .post("http://127.0.0.1:5000/upload", formData, config)
        .then(function (response) {
          console.log(response.data);
        });
      // var triggerDownload = $("#downloadA")
      //   .attr("href", img)
      //   .attr("download", "micro-blog.png");
      // triggerDownload[0].click();
    },
    dataURItoBlob(dataURI) {
      // base64转buffer
      var byteString = atob(dataURI.split(",")[1]);
      var mimeString = dataURI.split(",")[0].split(":")[1].split(";")[0];
      var ab = new ArrayBuffer(byteString.length);
      var ia = new Uint8Array(ab);
      for (var i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
      }
      return new Blob([ab], { type: mimeString });
    },

    /**
     * 关闭摄像头
     */
    closePhoto() {
      this.mediaStreamTrack.stop();
      document.getElementById("bg-content").style = "display:none";
      this.mediaStreamTrack = null;
      clearTimeout(this.timeout);
    },
    submitForm(event) {
      event.preventDefault();
      let formData = new FormData();
      // console.log(this.file);
      formData.append("name", this.name);
      formData.append("age", this.age);
      formData.append("file1", this.file1);
      formData.append("file2", this.file2);

      let config = {
        headers: {
          "Content-Type": "multipart/form-data",
        },
      };
      axios
        .post("http://127.0.0.1:3000/uploadmany/upload", formData, config)
        .then(function (response) {
          console.log(response.data);
        });
    },
  },
};
</script>
<style>
img {
  width: 100px;
  height: 100px;
}
#bg-content{
  position: relative;
}
#myCanvas {
  position: absolute;
  top: 0;
  /* left: 0; */
}

</style>