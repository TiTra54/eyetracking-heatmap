<script setup>
import Startseite from './components/Startseite.vue'
import EyeTrackingApp from './components/EyeTrackingApp.vue'

</script>

<template>
    <!--<Startseite />-->

  <EyeTrackingApp />
</template>

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
