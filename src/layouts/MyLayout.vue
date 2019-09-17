<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>

        <q-btn
          v-if="showLoadButton"
          color="white" text-color="primary">
          Load File
          <input
            type="file"
            class="q-uploader__input overflow-hidden absolute-full"
            v-on:change="fileChosen"
            ref="fileInput"
            accept="audio/wav"
          />
        </q-btn>

        <q-toolbar-title />

        <span>very preliminary</span>

      </q-toolbar>
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script>
  import { EventBus } from '../services/event-bus.js'

  export default {
    name: 'MyLayout',

    data: () => ({
      showLoadButton: true, // trick to have the file input work not only the very 1st time is used.
    }),

    methods: {
      fileChosen(file) {
        this.showLoadButton = false
        EventBus.$emit('fileChosen', file)
        setTimeout(() => {
          this.showLoadButton = true
        }, 1000)
      },
    },
  }
</script>
