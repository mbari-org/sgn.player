<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>
        <q-toolbar-title> Sgn.Player </q-toolbar-title>

        <a href="https://github.com/mbari-org/sgn.player" >
          <img src="GitHub-Mark-Light-32px.png" >
        </a>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script>
export default {
  name: "MyLayout",

  data: () => ({}),

  methods: {},
};
</script>
