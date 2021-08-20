<template>
  <q-page class>
    <div class="controls row flex q-pt-sm shadow-2">
      <div class="q-ml-md items-center" style="width: 70px">
        <q-btn
          v-if="showLoadButton && !isLoading"
          color="primary"
          no-caps
          dense
        >
          Load File
          <input
            type="file"
            class="q-uploader__input overflow-hidden absolute-full"
            v-on:change="fileChosen"
            ref="fileInput"
            accept="audio/wav"
          />
        </q-btn>
        <q-circular-progress
          v-else-if="isLoading"
          size="24px"
          indeterminate
          color="primary"
        />
      </div>

      <div class="q-ml-lg">
        <q-btn
          color="primary"
          flat
          round
          icon="fast_rewind"
          size="md"
          @click="wavesurfer.skipBackward(1)"
        />
        <q-btn
          v-if="isPlaying"
          color="primary"
          round
          icon="pause"
          size="sm"
          @click="wavesurfer.pause()"
        />
        <q-btn
          v-if="!isPlaying"
          color="primary"
          round
          icon="play_arrow"
          size="sm"
          @click="wavesurfer.play()"
        />
        <q-btn
          color="primary"
          flat
          round
          icon="fast_forward"
          size="md"
          @click="wavesurfer.skipForward(1)"
        />
      </div>

      <div class="q-ml-xl row items-center">
        <div class="col q-mr-sm">Speed:</div>
        <q-input
          dense
          v-model.number="playbackRate"
          :min="1"
          :max="8"
          style="width: 50px"
          type="number"
          filled
        />
      </div>

      <!--zoom not properly handled yet
-->
      <div class="q-ml-xl row items-center">
        <div class="q-mr-sm">Zoom:</div>
        <q-slider
          dense
          style="width: 120px"
          :min="1"
          :max="5000"
          :model-value="zoom"
          @update:modelValue="zoomChanged"
          label
        />
      </div>

      <div v-if="doSpectrogram" class="q-ml-xl items-center">
        <q-btn
          v-if="!isLoading && !updatingSpectrogram"
          dense
          label="Spectrogram"
          no-caps
          color="primary"
          @click="updateSpectrogram"
        />
        <q-circular-progress
          v-else-if="updatingSpectrogram"
          size="23px"
          indeterminate
          color="primary"
        />
      </div>
    </div>

    <div class="audio-container" ref="audio-container">
      <div class="row q-ma-md">
        <div class="col-12">
          <div class="col-12" id="wave-timeline"></div>
          <div class="col-12" id="waveform"></div>
          <div class="col-12" id="wave-spectrogram"></div>
        </div>
      </div>
    </div>
  </q-page>
</template>

<script>
import WaveSurfer from "wavesurfer.js"
import RegionsPlugin from "wavesurfer.js/dist/plugin/wavesurfer.regions"
import SpectrogramPlugin from "wavesurfer.js/dist/plugin/wavesurfer.spectrogram"
import TimelinePlugin from "wavesurfer.js/dist/plugin/wavesurfer.timeline"

function createWaveSurfer() {
  const wavesurfer = WaveSurfer.create({
    container: "#waveform",
    // barWidth: 3,
    // height: 256,
    // autoCenter: false,

    // "Use the PeakCache to improve rendering speed of large waveforms"
    partialRender: true,

    responsive: true,

    normalize: true,
    backend: 'WebAudio',
    // renderer: 'MultiCanvas',
    pixelRatio: 1,
    plugins: [
      RegionsPlugin.create({
        regions: [],
      }),
      TimelinePlugin.create({
        container: "#wave-timeline",
      }),
    ],

    // minPxPerSec: 100,
    scrollParent: true,
  })

  const regionColor = "hsla(400, 100%, 30%, 0.2)"
  wavesurfer.enableDragSelection({
    color: regionColor,
  })

  wavesurfer.on("region-created", region => {
    console.debug(
        `region-created: id=${region.id} start=${region.start} end=${region.end}`
    )
  })

  wavesurfer.on("region-click", (region, e) => {
    console.debug('region=', region)
    const loop = e.shiftKey
    e.stopPropagation()
    const isPlaying = wavesurfer.isPlaying()
    console.debug(
        `region-click: id=${region.id} start=${region.start} end=${region.end} loop=${loop} isPlaying=${isPlaying}`
    )
    if (wavesurfer.isPlaying()) {
      region.setLoop(false)
      wavesurfer.pause()
    }
    else {
      if (loop) {
        region.playLoop()
        // the following still needed after pausing region
        region.setLoop(true)
      }
      else region.play()
    }
  })

  return wavesurfer
}

function createSpectrogramPlugin() {
  return SpectrogramPlugin.create({
    container: "#wave-spectrogram",
    deferInit: true,
    // fftSamples: 512,
    // noverlap: 256,
    // windowFunc: 'hamming',
    labels: true,
    // pixelRatio: 2,
  })
}

function  destroySpectrogramPlugin(wavesurfer) {
  const active = wavesurfer.getActivePlugins()
  console.debug("active plugins=", active)
  if (active.spectrogram) {
    wavesurfer.destroyPlugin("spectrogram")
    console.debug("spectrogram destroyed")
  }
}

export default {
  name: "PageIndex",

  data: () => ({
    showLoadButton: true, // trick to have the file input work not only the very 1st time is used.
    wavesurfer: null,
    regionsPlugin: null,
    isLoading: false,
    playbackRate: 1,
    zoom: 1,
    doSpectrogram: true,
    updatingSpectrogram: false,
  }),

  computed: {
    isPlaying() {
      return this.wavesurfer && this.wavesurfer.isPlaying()
    },
  },

  created() {
    document.addEventListener("keyup", this.onKeyUp)
  },

  destroyed() {
    document.removeEventListener("keyup", this.onKeyUp)
  },

  mounted() {
    if (!this.wavesurfer) {
      this.createWaveSurfer()
    }
  },

  methods: {
    createWaveSurfer() {
      this.wavesurfer = createWaveSurfer()

      this.wavesurfer.load(
        "from_HBSe_20151207T070326__124.45328_126.52461.wav"
      )
      //this.wavesurfer.load("https://ia902606.us.archive.org/35/items/shortpoetry_047_librivox/song_cjrg_teasdale_64kb.mp3")

      this.wavesurfer.on("error", (err) => {
        console.error("on error", err)
        this.isLoading = false
        this.showLoadButton = true
        this.$q.notify({ message: err })
      })

      this.wavesurfer.on("loading", () => {
        console.debug("on loading")
        this.isLoading = true
        this.showLoadButton = false
      })

      this.wavesurfer.on("ready", () => {
        console.debug("on ready")
        this.isLoading = false
        this.showLoadButton = true
        this.wavesurfer.setHeight(120)
      })
    },

    fileChosen(file) {
      this.showLoadButton = false
      this.loadFile(file)
    },

    loadFile(file) {
      if (file.target.files.length === 0) return

      this.wavesurfer.empty()
      destroySpectrogramPlugin(this.wavesurfer)

      const blob = file.target.files[0]
      console.debug("Loading", blob)
      this.wavesurfer.loadBlob(blob)
    },

    zoomChanged(zoom) {
      this.zoom = zoom
      console.debug("zoomChanged", this.zoom)
      if (this.zoom === 1) {
        this.doSpectrogram = true
      }
      else {
        destroySpectrogramPlugin(this.wavesurfer)
        this.doSpectrogram = false
      }
      // zoom is pxPerSec
      this.wavesurfer.zoom(Number(this.zoom))
    },

    updateSpectrogram() {
      if (!this.doSpectrogram) return

      console.debug(
        "updateSpectrogram: updatingSpectrogram=",
        this.updatingSpectrogram
      )
      if (this.updatingSpectrogram) {
        return
      }

      destroySpectrogramPlugin(this.wavesurfer)

      this.updatingSpectrogram = true

      setTimeout(() => {
        console.debug("creating spectrogram")
        const spectrogram = createSpectrogramPlugin()
        this.wavesurfer.addPlugin(spectrogram)
        console.debug("initing spectrogram")
        try {
          this.wavesurfer.initPlugin("spectrogram")
        }
        catch (e) {
          this.updatingSpectrogram = false
          console.error("error initing spectrogram", e)
        }
        this.$nextTick(() => {
          if (this.updatingSpectrogram) {
            this.updatingSpectrogram = false
            console.debug("spectrogram inited")
          }
        })
      }, 200)
    },

    onKeyUp(event) {
      // console.debug('onKeyUp', event)
      switch (event.code) {
        case "Space":
          this.wavesurfer.playPause()
          break
        case "ArrowLeft":
          // this.wavesurfer.skipBackward(1)
          this.$refs["audio-container"].scrollLeft -= 20
          break
        case "ArrowRight":
          // this.wavesurfer.skipBackward(-1)
          this.$refs["audio-container"].scrollLeft += 20
          break
      }
    },
  },

  watch: {
    playbackRate() {
      console.debug("setPlaybackRate", this.playbackRate)
      this.wavesurfer.setPlaybackRate(this.playbackRate)
    },
  },
}
</script>
