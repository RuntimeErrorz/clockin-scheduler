<template>
  <div class="container">
    <div class="calendar-wrapper">
      <div class="calendar-button">
        <q-btn no-caps class="button" color="red" @click="calendar?.prev()">
          &lt; Prev
        </q-btn>
        <q-btn no-caps class="button" @click="calendar?.moveToToday()">
          Today
        </q-btn>
        <q-btn no-caps class="button" color="primary" @click="calendar?.next()">
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
import { computed, ref } from 'vue';

// 日期和时间工具函数
const dateUtils = {
  // 获取今天的日期，格式为YYYY-MM-DD
  today(): string {
    const now = new Date();
    const year = now.getFullYear();
    const month = (now.getMonth() + 1).toString().padStart(2, '0');
    const day = now.getDate().toString().padStart(2, '0');
    return `${year}-${month}-${day}`;
  },

  // 格式化月份，格式为YYYY-MM
  formatMonth(date: Date): string {
    const year = date.getFullYear();
    const month = (date.getMonth() + 1).toString().padStart(2, '0');
    return `${year}-${month}`;
  },

  // 获取当前时间，格式为HH:MM
  getCurrentTime(): string {
    const now = new Date();
    const hours = now.getHours().toString().padStart(2, '0');
    const minutes = now.getMinutes().toString().padStart(2, '0');
    return `${hours}:${minutes}`;
  },

  // 计算两个时间之间的分钟差
  calculateTimeDiff(timein: string, timeout: string): number {
    const [inHour, inMin] = timein.split(':').map(Number);
    const [outHour, outMin] = timeout.split(':').map(Number);
    let minutesDiff = outHour * 60 + outMin - (inHour * 60 + inMin);
    return minutesDiff;
  },

  isWorkDay(date: Date): boolean {
    const day = date.getDay();
    return day !== 0 && day !== 6;
  },
};

// 类型定义
type ClockinRecord = {
  timein?: string;
  timeout?: string;
  dayOff?: number;
  dayoffReason?: string;
};

type ClockinHistory = {
  [date: string]: ClockinRecord;
};

interface CalendarRef {
  prev: () => void;
  next: () => void;
  moveToToday: () => void;
}

// 状态管理
const calendar = ref<CalendarRef | null>(null);
const selectedDate = ref(dateUtils.today());
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
  const nowHHMM = dateUtils.getCurrentTime();

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
  const currentRecord = clockinHistory.value[selectedDate.value] || {};
  clockinHistory.value[selectedDate.value] = {
    ...currentRecord, // 保留原有的所有字段
    timein: timein.value, // 更新打卡时间
    timeout: timeout.value, // 更新打卡时间
  };

  localStorage.setItem('clockinHistory', JSON.stringify(clockinHistory.value));
};

const clearDayOff = () => {
  const record = clockinHistory.value[selectedDate.value];
  if (record) {
    delete record['dayOff'];
    delete record['dayoffReason'];
    localStorage.setItem(
      'clockinHistory',
      JSON.stringify(clockinHistory.value)
    );
  }
};

const clearTime = () => {
  const record = clockinHistory.value[selectedDate.value];
  if (record) {
    delete record['timein'];
    delete record['timeout'];
    localStorage.setItem(
      'clockinHistory',
      JSON.stringify(clockinHistory.value)
    );
  }
  initializeTime();
};

// 计算剩余工作日数量
const getRemainingDaysInMonth = () => {
  const currentDate = new Date();
  const today = dateUtils.today();
  const currentYear = currentDate.getFullYear();
  const currentMonth = currentDate.getMonth();
  const lastDayOfMonth = new Date(currentYear, currentMonth + 1, 0).getDate();
  let remainingDays = 0;

  // 检查当天是否应该计入剩余天数
  const todayRecord = clockinHistory.value[today];
  const isTodayCompleted = todayRecord?.timein && todayRecord?.timeout;

  if (!isTodayCompleted) {
    remainingDays++;
  }

  for (let day = currentDate.getDate() + 1; day <= lastDayOfMonth; day++) {
    const date = new Date(currentYear, currentMonth, day);
    if (!dateUtils.isWorkDay(date)) {
      continue;
    }
    remainingDays++;
  }
  return remainingDays;
};

// 计算指定月份的打卡和休假信息
const calculateMonthInfo = (monthStr: string) => {
  let clockinHours = 0;
  let monthDayOff = 0;

  for (const [date, record] of Object.entries(clockinHistory.value)) {
    if (date.startsWith(monthStr)) {
      if (record.timein && record.timeout) {
        clockinHours += dateUtils.calculateTimeDiff(
          record.timein,
          record.timeout
        );
      }
      if (record.dayOff) {
        monthDayOff += record.dayOff;
      }
    }
  }

  return {
    clockinHours,
    monthDayOff,
    totalHours: clockinHours + monthDayOff * 8 * 60,
  };
};

// 计算属性
const monthClockinInfo = computed(() => {
  const selectedDateObj = new Date(selectedDate.value);
  const selectedMonth = dateUtils.formatMonth(selectedDateObj);

  const { monthDayOff, totalHours } = calculateMonthInfo(selectedMonth);

  const remainingDays = getRemainingDaysInMonth();
  const remainingHours = (240 * 60 - totalHours) / remainingDays / 60.0;

  const currentDate = new Date();
  const currentMonth = dateUtils.formatMonth(currentDate);

  console.log(
    `当前月份：${currentMonth}，选中月份：${selectedMonth}，剩余天数：${remainingDays}，剩余小时：${remainingHours}`
  );

  return (
    `${selectedMonth}：已请假 ${monthDayOff} 天，总打卡 ${(
      totalHours / 60
    ).toFixed(2)} 小时` +
    (currentMonth === selectedMonth
      ? `，接下来的 ${remainingDays} 天每天需打卡 ${Math.max(
          0,
          remainingHours
        ).toFixed(2)} 小时`
      : '')
  );
});

// 初始化
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
