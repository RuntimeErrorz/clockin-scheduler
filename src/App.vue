<template>
  <QCalendarMonth />
  <p>{{ time }}</p>
  <p>{{ time2 }}</p>
  <div class="time-wrapper">
    <q-time class="time" v-model="time" />
    <q-time class="time" v-model="time2" />
  </div>
  <div class="button-wrapper">
    <q-btn class="button" color="primary" label="确认" />
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
let time = ref('');
let time2 = ref('');
defineOptions({
  name: 'App',
});
</script>

<style>
.time-wrapper {
  > *:not(:first-child) {
    margin-left: 20px;
  }
  display: flex;
  justify-content: center;
  margin-top: 20px;
}
.time {
  align-self: center;
}
.button-wrapper {
  margin-top: 20px;
  display: flex;
  justify-content: center;
}
.button {
  align-self: center;
}
</style>
