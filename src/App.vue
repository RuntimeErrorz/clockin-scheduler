<template>
  <div class="container">
    <div class="calendar-wrapper">
      <div class="calendar-button">
        <q-btn
          no-caps
          class="button"
          style="margin: 2px"
          color="primary"
          @click="calendar.prev()"
        >
          &lt; Prev
        </q-btn>
        <q-btn
          no-caps
          class="button"
          style="margin: 2px"
          @click="calendar.moveToToday()"
        >
          Today
        </q-btn>
        <q-btn
          no-caps
          class="button"
          style="margin: 2px"
          color="red"
          @click="calendar.next()"
        >
          Next &gt;
        </q-btn>
      </div>
      <q-calendar-month
        ref="calendar"
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
    <div class="input-wrapper">
      <div class="time-wrapper">
        <q-time v-model="time" />
        <q-time color="red" v-model="time2" />
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
        <q-slider
          v-model="dayOffModel"
          color="blue"
          :min="0"
          :max="15"
          :step="0.5"
          @change="changeDay"
        />
      </div>
    </div>
    <div class="notify">{{ accmulatedHours }}</div>
  </div>
</template>

<script setup lang="ts">
import { today } from '@quasar/quasar-ui-qcalendar/src/index.js';
import { computed, ref, watch } from 'vue';
let calendar = ref(null);
let selectedDate = ref(today());
let time = ref('');
let time2 = ref('');
let clockinHistory, dayoffHistory;
const history = localStorage.getItem('clockinHistory');
if (history) {
  clockinHistory = ref(JSON.parse(history));
} else {
  clockinHistory = ref([]);
}
dayoffHistory = localStorage.getItem('dayoffHistory');
if (dayoffHistory) {
  dayoffHistory = ref(JSON.parse(dayoffHistory));
} else {
  dayoffHistory = ref([]);
}
let dayOffModel = ref(0);
const selectedMonth = computed(() => selectedDate.value.substring(0, 7));
const dayOff = computed(() => {
  return (
    dayoffHistory.value.find(
      (record: { month: string }) =>
        record.month === selectedDate.value.substring(0, 7)
    )?.dayOff || 0
  );
});
dayOffModel.value = dayOff.value;
watch(dayOff, () => {
  dayOffModel.value = dayOff.value;
});
function changeDay() {
  const index = dayoffHistory.value.findIndex(
    (record: { month: string }) => record.month === selectedMonth.value
  );
  if (index > -1) {
    dayoffHistory.value[index].dayOff = dayOffModel.value;
  } else {
    dayoffHistory.value.push({
      month: selectedMonth,
      dayOff: dayOffModel.value,
    });
  }
  localStorage.setItem('dayoffHistory', JSON.stringify(dayoffHistory.value));
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
  let total = 0;
  clockinHistory.value.forEach((record) => {
    if (
      record.date.substring(0, 7) === selectedMonth.value &&
      record.time &&
      record.time2
    ) {
      const time = record.time.split(':');
      const time2 = record.time2.split(':');
      total += (parseInt(time2[0]) - parseInt(time[0])) * 60;
      total += parseInt(time2[1]) - parseInt(time[1]);
    }
  });
  const remainingDays = getRemainingDaysInMonth();
  total += dayOff.value * 8 * 60;
  let remainingHours = (240 * 60 - total) / remainingDays / 60.0;
  remainingHours = Math.round(remainingHours * 10) / 10;
  const currentMonth = new Date().toISOString().substring(0, 7);
  return currentMonth != selectedMonth.value
    ? `${selectedMonth.value}：已请假 ${dayOffModel.value} 天，已打卡 ${
        Math.round((total / 60) * 10) / 10
      } 小时`
    : `${selectedMonth.value}：已请假 ${dayOffModel.value} 天，已打卡 ${
        Math.round((total / 60) * 10) / 10
      } 小时，之后每天需打卡 ${Math.max(0, remainingHours)} 小时`;

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
defineOptions({
  name: 'App',
});
</script>

<style lang="scss" scoped>
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  justify-content: space-between;
  .calendar-wrapper {
    .calendar-button {
      justify-content: center;
      display: flex;
      margin-bottom: 10px;
    }
    .record-time {
      display: flex;
      justify-content: center;
      font-size: 12px;
      color: white;
    }
    .bg-green {
      background: #1976d2 !important;
    }

    .bg-orange {
      background: #f44336 !important;
    }
  }
  .input-wrapper {
    .time-wrapper {
      > *:not(:first-child) {
        margin-left: 20px;
      }
      display: flex;
      justify-content: center;
    }
    .button-wrapper {
      > *:not(:first-child) {
        margin-left: 20px;
      }
      margin-top: 20px;
      display: flex;
      justify-content: center;
    }
    .slider-wrapper {
      margin-left: 20px;
      margin-right: 20px;
      margin-top: 8px;
      margin-bottom: -20px;
    }
  }
  .notify {
    display: flex;
    justify-content: center;
    font-size: 24px;
    text-align: center;
  }
}
.q-time {
  min-width: 192px;
  max-width: 192px;
}
html,
body,
#app {
  margin: 0;
  padding: 0;
  height: 100%;
}
</style>