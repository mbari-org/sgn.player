<template>
  <q-page>
    <div class="controls row flex q-pl-md q-py-sm q-gutter-lg shadow-2">
      <div class="row items-center">
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
            :loading="isLoading"
            v-on:change="fileChosen"
            ref="fileInput"
            accept="audio/wav"
          />
        </q-btn>
      </div>

      <div class="row items-center bg-blue-1">
        <q-btn
          color="primary"
          flat
          round
          icon="fast_rewind"
          size="md"
          @click="wavesurfer.skipBackward(1)"
        />
        <div>
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
        </div>
        <q-btn
          color="primary"
          flat
          round
          icon="fast_forward"
          size="md"
          @click="wavesurfer.skipForward(1)"
        />
      </div>

      <div class="row items-center">
        <div class="col q-mr-sm">Speed:</div>
        <q-input
          dense
          v-model.number="playbackRate"
          :min="1"
          :max="12"
          style="width:4em"
          type="number"
          filled
        />
      </div>

      <div class="row items-center q-gutter-x-md">
        <div>Zoom:</div>
        <div>{{ zoom }}</div>
        <q-slider
          dense
          style="width: 120px"
          :min="1"
          :max="2000"
          v-model.number="zoom"
        />
      </div>

      <div
          class="q-px-xs row items-center q-gutter-x-sm shadow-5"
      >
        <div>
          <q-btn
            :disable="isLoading || spectrogram.updating"
            dense
            label="Spectrogram"
            no-caps
            color="primary"
            @click="updateSpectrogram"
            :loading="spectrogram.updating"
          />
        </div>
        <div class="bg-blue-1 q-pa-xs rounded-borders">
          <q-checkbox
            label="Labels"
            v-model="spectrogram.opts.labels"
            dense
          />
        </div>
        <div class="row items-center q-gutter-x-sm">
          <div class="row items-center q-gutter-x-xs bg-blue-1 q-pr-xs rounded-borders">
            <div>FFT:</div>
            <q-input
                dense
                outlined
                prefix="2^"
                v-model.number="spectrogram.opts.fftSamplesExp"
                @change="spectrogramFftParamsAdjusted"
                :min="9"
                :max="12"
                style="width:6em"
                type="number"
            />
            <div>= {{ spectrogram.opts.fftSamples }}</div>
          </div>
          <div class="row items-center q-gutter-x-xs bg-blue-1 q-pr-xs rounded-borders">
            <div>Overlap:</div>
            <q-input
                dense
                outlined
                v-model.number="spectrogram.opts.noverlapPercent"
                @change="spectrogramFftParamsAdjusted"
                :min="25"
                :max="80"
                suffix="%"
                style="width:6em"
                type="number"
            >
            </q-input>
            <div>= {{ spectrogram.opts.noverlap }}</div>
          </div>
        </div>
        <div>
          <q-btn
              dense round size="sm"
              icon="close"
              color="primary"
              @click="removeSpectrogram"
          />
        </div>
      </div>
    </div>

    <div class="audio-container" ref="audio-container">
      <div class="col-12 q-ma-md">
        <div class="col-12" id="wave-timeline"></div>
        <div class="col-12" id="waveform"></div>
        <div class="col-12" id="wave-spectrogram"></div>
      </div>
    </div>
  </q-page>
</template>

<script>
import WaveSurfer from "wavesurfer.js"
import RegionsPlugin from "wavesurfer.js/dist/plugin/wavesurfer.regions"
import SpectrogramPlugin from "wavesurfer.js/dist/plugin/wavesurfer.spectrogram"
import TimelinePlugin from "wavesurfer.js/dist/plugin/wavesurfer.timeline"

import { debounce } from 'quasar'

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
    // backend: 'WebAudio',
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

function createSpectrogramPlugin(moreOpts = {}) {
  const baseOpts = {
    container: "#wave-spectrogram",
    deferInit: true,
  }
  const opts = {...baseOpts, ...moreOpts}
  return SpectrogramPlugin.create(opts)
}

function destroySpectrogramPlugin(wavesurfer) {
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
    spectrogram: {
      opts: {
        labels: true,
        fftSamplesExp: 10,
        fftSamples: 1024, // === 2 ^ fftSamplesExp
        noverlap: 512,
        noverlapPercent: 50,
        windowFunc: 'hann',
        pixelRatio: 1,
      },
      updating: false,
    },
  }),

  computed: {
    isPlaying() {
      return this.wavesurfer && this.wavesurfer.isPlaying()
    },
  },

  created() {
    this.zoomChanged = debounce(this.zoomChanged, 500)
    document.addEventListener("keyup", this.onKeyUp)
  },

  destroyed() {
    document.removeEventListener("keyup", this.onKeyUp)
  },

  mounted() {
    this.createWaveSurferWith({
      url: "from_HBSe_20151207T070326__124.45328_126.52461.wav"
      // url: "https://ia902606.us.archive.org/35/items/shortpoetry_047_librivox/song_cjrg_teasdale_64kb.mp3"
    })
  },

  methods: {
    destroyWaveSurfer() {
      if (this.wavesurfer) {
        // maybe the destroy down below takes care of these:
        // this.wavesurfer.empty()
        // destroySpectrogramPlugin(this.wavesurfer)

        this.wavesurfer.destroy()
        this.wavesurfer = null
      }
    },

    createWaveSurferWith({ url, blob }) {
      this.destroyWaveSurfer()

      this.wavesurfer = createWaveSurfer()

      if (url) {
        console.debug("loading url=", url)
        this.wavesurfer.load(url)
      }
      else {
        console.debug("loading blob=", blob)
        this.wavesurfer.loadBlob(blob)
      }

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
        this.wavesurfer.setPlaybackRate(this.playbackRate)
        this.wavesurfer.zoom(Number(this.zoom))
      })
    },

    fileChosen(file) {
      this.showLoadButton = false
      this.loadFile(file)
    },

    loadFile(file) {
      if (file.target.files.length) {
        const blob = file.target.files[0]
        this.createWaveSurferWith({ blob })
      }
    },

    removeSpectrogram() {
      destroySpectrogramPlugin(this.wavesurfer)
    },

    zoomChanged(zoom) {
      this.zoom = zoom
      console.debug("zoomChanged", this.zoom)

      // zoom change doesn't seem to propagate to the spectrogram,
      // so, let's destroy the spectrogram and let the user recreate it
      destroySpectrogramPlugin(this.wavesurfer)

      // zoom is pxPerSec
      this.wavesurfer.zoom(Number(this.zoom))
    },

    spectrogramFftParamsAdjusted() {
      const exp = this.spectrogram.opts.fftSamplesExp
      const fftSamples = 2 ** exp
      this.spectrogram.opts.fftSamples = fftSamples

      const perc = this.spectrogram.opts.noverlapPercent
      this.spectrogram.opts.noverlap = Math.round(fftSamples * perc / 100)
    },

    updateSpectrogram() {
      console.debug(
        "updateSpectrogram: spectrogram.updating=",
        this.spectrogram.updating
      )
      if (this.spectrogram.updating) {
        return
      }

      destroySpectrogramPlugin(this.wavesurfer)

      this.spectrogram.updating = true

      setTimeout(() => {
        console.debug("creating spectrogram")
        const spectrogram = createSpectrogramPlugin(this.spectrogram.opts)

        this.wavesurfer.addPlugin(spectrogram)
        console.debug("initing spectrogram")
        try {
          this.wavesurfer.initPlugin("spectrogram")
        }
        catch (e) {
          this.spectrogram.updating = false
          console.error("error initing spectrogram", e)
        }
        this.$nextTick(() => {
          if (this.spectrogram.updating) {
            this.spectrogram.updating = false
            console.debug("spectrogram inited")
          }
        })
      }, 200)
    },

    onKeyUp(event) {
      // console.debug('onKeyUp', event)
      switch (event.code) {
        case "Space":
          this.wavesurfer && this.wavesurfer.playPause()
          break
        case "ArrowLeft":
          this.$refs["audio-container"].scrollLeft -= 20
          break
        case "ArrowRight":
          this.$refs["audio-container"].scrollLeft += 20
          break
      }
    },
  },

  watch: {
    playbackRate(playbackRate) {
      this.wavesurfer.setPlaybackRate(playbackRate)
    },

    zoom(zoom) {
      this.zoomChanged(zoom)
    },
  },
}
</script>
