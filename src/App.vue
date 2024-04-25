<template>
  <div class="container">
    <div>
      <q-calendar-month
        v-model="selectedDate"
        :now="selectedDate"
        @click-day="onClickDay"
      >
        <template #day="{ scope: { timestamp } }">
          <template
            v-for="record in clockinHistoryMap[timestamp.date]"
            :key="record.id"
          >
            <div class="record">
              <div class="record-time bg-green">{{ record.time }}</div>
              <div class="record-time bg-orange">{{ record.time2 }}</div>
            </div>
          </template>
        </template>
      </q-calendar-month>
    </div>

    <div class="wrapper">
      <div class="time-wrapper">
        <q-time class="time" v-model="time" />
        <q-time color="red" class="time" v-model="time2" />
      </div>
      <div class="button-wrapper">
        <q-btn
          class="button"
          color="primary"
          label="保存时间"
          @click="saveTime"
        />
        <q-btn class="button" color="red" label="清除该天" @click="clearTime" />
      </div>
      <div class="slider-wrapper">
        <div class="notify">请假天数：{{ dayOff }}</div>
        <q-slider
          style="margin-top: 10px"
          v-model="dayOff"
          color="blue"
          :min="0"
          :max="15"
          :step="0.5"
          @change="changeDay"
        />
      </div>
    </div>
    <div style="display: flex; justify-content: center">
      <div class="notify">
        本月总打卡时长{{ accmulatedHours[0] }}小时，接下来每天至少需要打卡{{
          accmulatedHours[1]
        }}小时
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { today } from '@quasar/quasar-ui-qcalendar/src/index.js';
import { computed, ref } from 'vue';
let dayOff = ref(0);
let selectedDate = ref(today());
let time = ref('');
let time2 = ref('');
let clockinHistory;
const history = localStorage.getItem('clockinHistory');
if (history) {
  clockinHistory = ref(JSON.parse(history));
} else {
  clockinHistory = ref([]);
}
function initializeDay() {
  dayOff.value = localStorage.getItem('dayOff')
    ? parseFloat(localStorage.getItem('dayOff'))
    : 0;
}
function changeDay() {
  localStorage.setItem('dayOff', dayOff.value.toString());
}
function initializeTime() {
  const index = clockinHistory.value.findIndex(
    (record: { date: string }) => record.date === selectedDate.value
  );
  let nowHHMM = new Date().toLocaleTimeString([], {
    hour: '2-digit',
    minute: '2-digit',
  });
  time.value = clockinHistory.value[index]?.time
    ? clockinHistory.value[index].time
    : nowHHMM;

  time2.value = clockinHistory.value[index]?.time2
    ? clockinHistory.value[index].time2
    : clockinHistory.value[index]?.time
    ? nowHHMM
    : '';
}
function onClickDay(data: { scope: { timestamp: { date: string } } }) {
  selectedDate.value = data.scope.timestamp.date;
  initializeTime();
}
const clockinHistoryMap = computed(() => {
  const map = {};
  clockinHistory.value.forEach((record: { date: string | number }) => {
    (map[record.date] = map[record.date] || []).push(record);
  });
  console.log(map);
  return map;
});
function saveTime() {
  const index = clockinHistory.value.findIndex(
    (record: { date: string }) => record.date === selectedDate.value
  );
  if (index > -1) {
    clockinHistory.value.splice(index, 1);
  }
  clockinHistory.value.push({
    date: selectedDate.value,
    time: time.value,
    time2: time2.value,
  });
  localStorage.setItem('clockinHistory', JSON.stringify(clockinHistory.value));
}
function clearTime() {
  const index = clockinHistory.value.findIndex(
    (record: { date: string }) => record.date === selectedDate.value
  );
  if (index > -1) {
    clockinHistory.value.splice(index, 1);
  }
  localStorage.setItem('clockinHistory', JSON.stringify(clockinHistory.value));
  initializeTime();
}
const accmulatedHours = computed(() => {
  const currentMonth = new Date().getMonth() + 1;
  let total = 0;
  clockinHistory.value.forEach((record) => {
    const recordDate = new Date(record.date);
    const recordMonth = recordDate.getMonth() + 1;
    if (recordMonth === currentMonth && record.time && record.time2) {
      const time = record.time.split(':');
      const time2 = record.time2.split(':');
      total += (parseInt(time2[0]) - parseInt(time[0])) * 60;
      total += parseInt(time2[1]) - parseInt(time[1]);
    }
  });
  const remainingDays = getRemainingDaysInMonth();
  total += dayOff.value * 8 * 60;
  let remainingHours = (240 * 60 - total) / remainingDays / 60.0;
  remainingHours = Math.round(remainingHours * 100) / 100;
  return [Math.round((total / 60) * 100) / 100, Math.max(0, remainingHours)];

  function getRemainingDaysInMonth() {
    const currentDate = new Date();
    const currentYear = currentDate.getFullYear();
    const currentMonth = currentDate.getMonth();
    const lastDayOfMonth = new Date(currentYear, currentMonth + 1, 0).getDate();
    let remainingDays = 0;

    for (let day = currentDate.getDate() + 1; day <= lastDayOfMonth; day++) {
      const date = new Date(currentYear, currentMonth, day);
      if (isWeekend(date)) {
        continue;
      }
      remainingDays++;
    }
    return remainingDays;
  }

  function isWeekend(date) {
    const day = date.getDay();
    return day === 0 || day === 6;
  }
});
initializeTime();
initializeDay();
defineOptions({
  name: 'App',
});
</script>

<style lang="scss" scoped>
.slider-wrapper {
  padding-left: 10px;
  padding-right: 10px;
  justify-content: center;
  align-items: center;
  display: flex;
  flex-direction: column;
  margin-top: 40px;
}

.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  justify-content: space-between;
}

.record-time {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  font-size: 12px;
}
.bg-green {
  color: white;
  background: #1976d2 !important;
}

.bg-orange {
  color: white;
  background: #f44336 !important;
}

.record {
  justify-content: center;
  cursor: pointer;
}
.time-wrapper {
  > *:not(:first-child) {
    margin-left: 20px;
  }
  display: flex;
  justify-content: center;
}
.time {
  align-self: center;
}
.button-wrapper {
  > *:not(:first-child) {
    margin-left: 20px;
  }
  margin-top: 20px;
  display: flex;
  justify-content: center;
}
.button {
  align-self: center;
}
.notify {
  bottom: 20px;
  font-size: 24px;
  text-align: center;
}
.q-time {
  min-width: 192px;
  max-width: 192px;
}
</style>
