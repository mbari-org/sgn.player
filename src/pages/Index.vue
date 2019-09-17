<template>
  <q-page class>
    <div>
      <q-circular-progress
        v-if="isLoading"
        size="32px" indeterminate color="primary"
      />
    </div>

    <div class="controls row flex q-pb-md q-pt-md shadow-10">
      <q-btn
        label="Spectrogram" no-caps color="primary" class="q-ma-sm"
        @click="updateSpectrogram"
        :disable="isLoading || updatingSpectrogram"
        :loading="updatingSpectrogram"
      />

      <div class="q-ml-xl">
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
          size="md"
          @click="wavesurfer.pause()"
        />
        <q-btn
          v-if="!isPlaying"
          color="primary"
          round
          icon="play_arrow"
          size="md"
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

<!--zoom not properly handled yet
      <div style="width:200px" class="q-ml-xl">
        <q-slider
          :min="1" :max="1000"
          :value="zoom"
          @change="zoomChanged"
          label
        />
        <q-tooltip>Zoom</q-tooltip>
      </div>
-->
    </div>

    <div class="audio-container">
      <div class="row q-ma-md">
        <div class="col-12">
          <div class="col-12" id="wave-timeline"></div>
          <div class="col-12" id="waveform"></div>
          <div class="col-12" style="height:128px" id="wave-spectrogram"></div>
        </div>
      </div>
    </div>
  </q-page>
</template>

<script>
  import { EventBus } from "../services/event-bus.js"
  import WaveSurfer from 'wavesurfer.js'
  import RegionsPlugin from 'wavesurfer.js/dist/plugin/wavesurfer.regions'
  import SpectrogramPlugin from 'wavesurfer.js/dist/plugin/wavesurfer.spectrogram'
  import TimelinePlugin from 'wavesurfer.js/dist/plugin/wavesurfer.timeline'

  export default {
    name: 'PageIndex',

    data: () => ({
      wavesurfer: null,
      isLoading: false,
      zoom: 1,
      updatingSpectrogram: false,
    }),

    computed: {
      isPlaying() {
        return this.wavesurfer && this.wavesurfer.isPlaying()
      }
    },

    async created() {
      EventBus.$on("fileChosen", this.loadFile)
      document.addEventListener('keyup', this.onKeyUp)
    },

    async destroyed() {
      EventBus.$off("fileChosen", this.loadFile)
      document.removeEventListener('keyup', this.onKeyUp)
    },

    async mounted() {
      if (!this.wavesurfer) {
        this.createWaveSurfer()
      }
    },

    methods: {
      createWaveSurfer() {
        const regions = RegionsPlugin.create({
          regions: [],
        })

        const timeline = TimelinePlugin.create({
          container: '#wave-timeline',
        })

        const options = {
          container: "#waveform",
          // barWidth: 3,
          height: 256,
          plugins: [
            regions,
            timeline,
          ],
        }

        // "Use the PeakCache to improve rendering speed of large waveforms"
        partialRender: true,

        // options.minPxPerSec = 100
        options.scrollParent = true

        this.wavesurfer = WaveSurfer.create(options)

        this.wavesurfer.load("statics/from_HBSe_20151207T070326__124.45328_126.52461.wav")
        //this.wavesurfer.load("https://ia902606.us.archive.org/35/items/shortpoetry_047_librivox/song_cjrg_teasdale_64kb.mp3")

        const regionColor = 'hsla(400, 100%, 30%, 0.2)'
        this.wavesurfer.enableDragSelection({
          color: regionColor,
        })
        this.wavesurfer.on("region-created", region => {
          console.log(`region-created: id=${region.id} start=${region.start} end=${region.end}`)
        })
        this.wavesurfer.on("region-click", (region, e) => {
          const loop = e.shiftKey
          e.stopPropagation()
          console.log(`region-click: id=${region.id} start=${region.start} end=${region.end} loop=${loop} isPlaying=${this.isPlaying}`)
          if (!this.isPlaying) {
            if (loop)
              region.playLoop()
            else
              region.play()
          }
        })

        this.wavesurfer.on("error", err => {
          console.error(err)
          this.isLoading = false
          this.$q.notify({ message: err })
        })

        this.wavesurfer.on("loading", () => {
          this.isLoading = true
        })

        this.wavesurfer.on("ready", () => {
          this.isLoading = false
        })
      },

      createSpectrogramPlugin() {
        return SpectrogramPlugin.create({
          container: '#wave-spectrogram',
          deferInit: true,
          // fftSamples: 1024,
          // noverlap: 512,
          // windowFunc: 'hamming',
          labels: true,
          // pixelRatio: 2,
        })
      },

      destroySpectrogramPlugin() {
        const active = this.wavesurfer.getActivePlugins()
        console.log('active=', active)
        if (active.spectrogram) {
          this.wavesurfer.destroyPlugin('spectrogram')
          console.log('spectrogram destroyed')
        }
      },

      loadFile(file) {
        if (file.target.files.length == 0) return

        this.destroySpectrogramPlugin()

        const blob = file.target.files[0]
        console.log('Loading', blob)
        this.wavesurfer.loadBlob(blob)
      },

      zoomChanged(zoom) {
        this.zoom = zoom
        console.log('zoomChanged', this.zoom)
        this.wavesurfer.zoom(this.zoom)
      },

      updateSpectrogram() {
        console.log('updateSpectrogram: updatingSpectrogram=', this.updatingSpectrogram)
        if (this.updatingSpectrogram) {
          return
        }

        this.updatingSpectrogram = true

        this.destroySpectrogramPlugin()

        setTimeout(() => {
          console.log('creating spectrogram')
          const spectrogram = this.createSpectrogramPlugin()
          this.wavesurfer.addPlugin(spectrogram)
          this.wavesurfer.initPlugin('spectrogram')
          this.$nextTick(() => {
            this.updatingSpectrogram = false
            console.log('spectrogram inited')
          })
        }, 200)
      },

      onKeyUp(event) {
        // console.log('onKeyUp', event)
        switch (event.code) {
          case 'Space':
            this.wavesurfer.playPause()
            break;
          case 'ArrowLeft':
            this.wavesurfer.skipBackward(1)
            break;
          case 'ArrowRight':
            this.wavesurfer.skipBackward(-1)
            break;
        }
      },
    },
  }
</script>
