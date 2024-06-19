<template>
  <div class="container">
    <div class="calendar-wrapper">
      <div class="calendar-button">
        <q-btn no-caps class="button" color="red" @click="calendar.prev()">
          &lt; Prev
        </q-btn>
        <q-btn no-caps class="button" @click="calendar.moveToToday()">
          Today
        </q-btn>
        <q-btn no-caps class="button" color="primary" @click="calendar.next()">
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
          <div v-if="getDayOff(timestamp.date) === 1" class="blue-dot"></div>
          <div
            v-else-if="getDayOff(timestamp.date) === 0.5"
            class="red-dot"
          ></div>
          <div
            v-else-if="getDayOff(timestamp.date) === 0"
            class="grey-dot"
          ></div>
          <div class="record-time bg-red">
            {{ clockinHistory[timestamp.date]?.timein }}
          </div>
          <div class="record-time bg-blue">
            {{ clockinHistory[timestamp.date]?.timeout }}
          </div>
        </template>
      </q-calendar-month>
    </div>
    <div class="input-wrapper">
      <div class="time-wrapper">
        <q-time color="red" v-model="timein" />
        <q-time v-model="timeout" />
      </div>
      <div class="button-wrapper">
        <q-btn class="button" color="red" label="清除时间" @click="clearTime" />
        <q-btn class="button" @click="initializeDayOff">管理休假</q-btn>
        <q-btn
          class="button"
          color="primary"
          label="保存时间"
          @click="saveTime"
        />
      </div>
    </div>
    <div class="notify">{{ monthClockinInfo }}</div>
  </div>
  <q-dialog v-model="dayOffDialog">
    <q-card style="min-width: 350px">
      <q-card-section align="center">
        <div class="text-h6">{{ selectedDate }}</div>
      </q-card-section>
      <q-card-section align="center" class="q-pt-none">
        <q-option-group
          inline
          :options="dayOffOptions"
          type="radio"
          v-model="dayOffDay"
        />
        <q-input label="备注" v-model="dayoffReason" dense autofocus />
      </q-card-section>
      <q-card-actions align="around" class="text-primary">
        <q-btn
          @click="clearDayOff"
          color="red"
          flat
          label="清除休假"
          v-close-popup
        />
        <q-btn @click="addDayOff" flat label="确认" v-close-popup />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup lang="ts">
import { today } from '@quasar/quasar-ui-qcalendar/src/index.js';
import { computed, ref } from 'vue';
type ClockinRecord = {
  timein?: string;
  timeout?: string;
  dayOff?: number;
  dayoffReason?: string;
};

type ClockinHistory = {
  [date: string]: ClockinRecord;
};

const calendar = ref(null);
const selectedDate = ref(today());
const timein = ref('');
const timeout = ref('');
const storedHistory = localStorage.getItem('clockinHistory');
const clockinHistory = ref<ClockinHistory>(
  storedHistory ? JSON.parse(storedHistory) : {}
);
const dayOffDialog = ref(false);
const dayoffReason = ref('');
const dayOffDay = ref(NaN);
const dayOffOptions = [
  { label: '0 天', value: 0, color: 'grey' },
  { label: '0.5 天', value: 0.5, color: 'red' },
  { label: '1 天', value: 1, color: 'blue' },
];
const addDayOff = () => {
  const record = clockinHistory.value[selectedDate.value];
  if (record) {
    record.dayOff = dayOffDay.value;
    record.dayoffReason = dayoffReason.value;
  } else {
    clockinHistory.value[selectedDate.value] = {
      dayOff: dayOffDay.value,
      dayoffReason: dayoffReason.value,
    };
  }
  localStorage.setItem('clockinHistory', JSON.stringify(clockinHistory.value));
  dayOffDialog.value = false;
};
const getDayOff = (date: string) => {
  const record = clockinHistory.value[date];
  return record?.dayOff;
};
const initializeTime = () => {
  const record = clockinHistory.value[selectedDate.value];
  const nowHHMM = new Date().toLocaleTimeString([], {
    hour: '2-digit',
    minute: '2-digit',
  });
  timein.value = record?.timein ? record.timein : nowHHMM;
  timeout.value = record?.timeout
    ? record.timeout
    : record?.timein
    ? nowHHMM
    : '';
};
const initializeDayOff = () => {
  dayOffDialog.value = true;
  const record = clockinHistory.value[selectedDate.value];
  dayOffDay.value =
    typeof record?.dayOff == 'number' && !isNaN(record.dayOff)
      ? record.dayOff
      : NaN;
  dayoffReason.value = record?.dayoffReason ? record.dayoffReason : '';
};
const onClickDay = (data: { scope: { timestamp: { date: string } } }) => {
  selectedDate.value = data.scope.timestamp.date;
  initializeTime();
};
const saveTime = () => {
  clockinHistory.value[selectedDate.value] = {
    timein: timein.value,
    timeout: timeout.value,
  };
  localStorage.setItem('clockinHistory', JSON.stringify(clockinHistory.value));
};
const clearDayOff = () => {
  const record = clockinHistory.value[selectedDate.value];
  delete record['dayOff'];
  delete record['dayoffReason'];
  localStorage.setItem('clockinHistory', JSON.stringify(clockinHistory.value));
};
const clearTime = () => {
  const record = clockinHistory.value[selectedDate.value];
  delete record['timein'];
  delete record['timeout'];
  localStorage.setItem('clockinHistory', JSON.stringify(clockinHistory.value));
  initializeTime();
};
const monthClockinInfo = computed(() => {
  const selectedMonth = selectedDate.value.substring(0, 7);
  let clockinHours = 0;
  let monthDayOff = 0;
  for (const [date, record] of Object.entries(clockinHistory.value)) {
    if (date.startsWith(selectedMonth)) {
      if (record.timein && record.timeout) {
        const timein = record.timein.split(':');
        const timeout = record.timeout.split(':');
        clockinHours += (parseInt(timeout[0]) - parseInt(timein[0])) * 60;
        clockinHours += parseInt(timeout[1]) - parseInt(timein[1]);
      }
      if (record.dayOff) {
        monthDayOff += record.dayOff;
      }
    }
  }
  const totalHours = clockinHours + monthDayOff * 8 * 60;
  const remainingDays = getRemainingDaysInMonth();
  const remainingHours = (240 * 60 - totalHours) / remainingDays / 60.0;
  const currentMonth = new Date().toISOString().substring(0, 7);
  return (
    `${selectedMonth}：已请假 ${monthDayOff} 天，总打卡 ${(
      totalHours / 60
    ).toPrecision(4)} 小时` +
    (currentMonth === selectedMonth
      ? `，接下来的 ${remainingDays} 天每天需打卡 ${Math.max(
          0,
          remainingHours
        ).toPrecision(3)} 小时`
      : '')
  );

  function getRemainingDaysInMonth() {
    const currentDate = new Date();
    const today = currentDate.toISOString().substring(0, 10);
    const currentYear = currentDate.getFullYear();
    const currentMonth = currentDate.getMonth();
    const lastDayOfMonth = new Date(currentYear, currentMonth + 1, 0).getDate();
    let remainingDays = 0;
    for (let day = currentDate.getDate() + 1; day <= lastDayOfMonth; day++) {
      const date = new Date(currentYear, currentMonth, day);
      if (date.getDay() === 0 || date.getDay() === 6) {
        continue;
      }
      remainingDays++;
    }
    return clockinHistory.value[today]?.timeout
      ? remainingDays
      : remainingDays + 1;
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
      margin: 5px 0 10px 0;
      justify-content: center;
      display: flex;
      > *:not(:first-child) {
        margin-left: 10px;
      }
    }
    .dot-dayoff {
      position: absolute;
      right: 2px;
      top: -20px;
      width: 8px;
      height: 8px;
      border-radius: 50%;
    }
    .grey-dot {
      @extend.dot-dayoff;
      background-color: $grey;
    }
    .blue-dot {
      @extend.dot-dayoff;
      background-color: $blue-8;
    }
    .red-dot {
      @extend.dot-dayoff;
      background-color: $red;
    }
    .record-time {
      display: flex;
      justify-content: center;
      font-size: 12px;
      color: white;
    }
    .bg-blue {
      background: $blue-8 !important;
    }

    .bg-red {
      background: $red !important;
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
  }
  .notify {
    margin: 0 10px 6px 10px;
    display: flex;
    justify-content: center;
    font-size: 22px;
    text-align: center;
  }
}
.q-time {
  min-width: 192px;
  max-width: 192px;
}
.q-dialog__inner.flex > .q-card {
  margin-bottom: 20vh;
}
</style>
