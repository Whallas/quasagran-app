<template>
  <q-page class="constrain-more q-pa-md">
    <div class="camera-frame q-pa-md">
      <video
        v-show="!imageCaptured"
        ref="video"
        class="full-width"
        autoplay
        playsinline
      />
      <canvas
        v-show="imageCaptured"
        ref="canvas"
        class="full-width"
        height="240"
      />
    </div>

    <div class="text-center q-pa-md">
      <q-btn
        v-if="hasCameraSupport"
        color="grey-10"
        icon="eva-camera"
        size="lg"
        round
        @click="captureImage"
      />
      <q-file
        v-else
        v-model="imageUploaded"
        label="Choose a image"
        accept="image/*"
        outlined
        @input="captureImageFallback"
      >
        <template v-slot:prepend>
          <q-icon name="eva-attach-outline" />
        </template>
      </q-file>
    </div>

    <div class="row justify-center q-ma-md">
      <q-input
        v-model="post.caption"
        class="col col-sm-6"
        label="Caption"
        dense
      />
    </div>

    <div class="row justify-center q-ma-md">
      <q-input
        v-model="post.location"
        class="col col-sm-6"
        :loading="locationLoading"
        label="Location"
        dense
      >
        <template v-slot:append>
          <q-btn
            v-if="locationSupported && !locationLoading"
            icon="eva-navigation-2-outline"
            dense
            flat
            round
            @click="getLocation"
          />
        </template>
      </q-input>
    </div>

    <div class="row justify-center q-ma-md">
      <q-btn color="primary" label="Post Image" rounded unelevated />
    </div>
  </q-page>
</template>

<script>
import { uid } from "quasar";
require("md-gum-polyfill");

export default {
  name: "PageCamera",
  data() {
    return {
      post: {
        id: uid(),
        caption: "",
        location: "",
        photo: null,
        date: Date.now(),
      },
      imageCaptured: false,
      imageUploaded: [],
      hasCameraSupport: true,
      locationLoading: false,
    };
  },
  computed: {
    locationSupported() {
      return "geolocation" in navigator;
    },
  },
  methods: {
    initCamera() {
      navigator.mediaDevices
        .getUserMedia({ video: true })
        .then((stream) => {
          this.$refs.video.srcObject = stream;
        })
        .catch((err) => {
          this.hasCameraSupport = false;
        });
    },
    captureImage() {
      const { video, canvas } = this.$refs;
      const context = canvas.getContext("2d");

      canvas.width = video.getBoundingClientRect().width;
      canvas.height = video.getBoundingClientRect().height;

      context.drawImage(video, 0, 0, canvas.width, canvas.height);

      this.imageCaptured = true;
      this.post.photo = this.dataURItoBlob(canvas.toDataURL());

      this.disableCamera();
    },
    dataURItoBlob(dataURI) {
      // convert base64 to raw binary data held in a string
      // doesn't handle URLEncoded DataURIs - see SO answer #6850276 for code that does this
      var byteString = atob(dataURI.split(",")[1]);

      // separate out the mime component
      var mimeString = dataURI.split(",")[0].split(":")[1].split(";")[0];

      // write the bytes of the string to an ArrayBuffer
      var ab = new ArrayBuffer(byteString.length);

      // create a view into the buffer
      var ia = new Uint8Array(ab);

      // set the bytes of the buffer to the correct values
      for (var i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
      }

      // write the ArrayBuffer to a blob, and you're done
      var blob = new Blob([ab], { type: mimeString });
      return blob;
    },
    captureImageFallback(file) {
      this.post.photo = file;
      const { canvas } = this.$refs;
      const context = canvas.getContext("2d");
      const reader = new FileReader();

      reader.onload = (event) => {
        const img = new Image();
        img.onload = () => {
          canvas.width = img.width;
          canvas.height = img.height;
          context.drawImage(img, 0, 0);
          this.imageCaptured = true;
        };
        img.src = event.target.result;
      };
      reader.readAsDataURL(file);
    },
    disableCamera() {
      this.$refs.video.srcObject.getVideoTracks().forEach((track) => {
        track.stop();
      });
    },
    getLocation() {
      this.locationLoading = true;
      navigator.geolocation.getCurrentPosition(
        this.getCityAndCountry,
        (e) => this.locationError(e),
        { timeout: 7000 }
      );
    },
    getCityAndCountry(posisition) {
      const { latitude, longitude } = posisition.coords;
      const apiUrl = `https://geocode.xyz/${latitude},${longitude}?json=1`;
      this.$axios
        .get(apiUrl)
        .then(this.locationSuccess)
        .catch((e) => this.locationError(e));
    },
    locationSuccess(result) {
      const { city, country } = result.data;
      this.post.location = `${city}${country ? ", " + country : ""}`;
      this.locationLoading = false;
    },
    locationError(e) {
      this.$q.dialog({
        title: "Error",
        message: e.message,
      });
      this.locationLoading = false;
    },
  },
  mounted() {
    this.initCamera();
  },
  beforeDestroy() {
    if (this.hasCameraSupport) {
      this.disableCamera();
    }
  },
};
</script>

<style lang="sass">
.camera-frame
  border: 2px solid $grey-10
  border-radius: 10px
</style>
