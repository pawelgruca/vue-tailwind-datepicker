<template>
  <div ref="calendarRef" class="relative">
    <div
      class="bg-white bg-opacity-5 rounded-md h-12 overflow-hidden border border-gray-300 focus-within:border-blue-500 flex items-center px-3"
      :class="{
        'border-red': props.error,
      }"
    >
      <CalendarIcon
        class="h-6 w-6 text-gray-400 cursor-pointer hover:text-gray-500 transition"
        @click="toggleShowCalendar"
      />
      <input
        placeholder="dd/mm/yyyy"
        class="bg-transparent border-none w-full h-full typo-t1 outline-none px-3"
        :value="displayDate"
        ref="inputRef"
        @input="term = $event.target?.value"
        @click="showCalendar = true"
      />
    </div>

    <div
      v-if="showCalendar"
      class="w-72 rounded-md shadow-md px-4 py-2 absolute z-10 bg-white"
      :class="[fromTop < 420 ? 'top-14' : 'bottom-12', fromRight < 640 ? 'right-0' : '']"
    >
      <div class="flex py-2 items-center justify-between">
        <div class="font-bold text-gray-500 cursor-pointer hover:bg-gray-100 rounded-md p-1" @click="decrease">
          <svg class="h-5 w-5" fill="currentColor" viewBox="0 0 24 24">
            <path fill="currentColor" d="M10.05 16.94V12.94H18.97L19 10.93H10.05V6.94L5.05 11.94Z" />
          </svg>
        </div>

        <div class="text-gray-900 flex flex-col items-center justify-start">
          <span>{{ monthName }} {{ date.year }}</span>
        </div>
        <div class="font-bold text-gray-500 text-end cursor-pointer hover:bg-gray-100 rounded-md p-1" @click="increase">
          <svg class="h-5 w-5" fill="currentColor" viewBox="0 0 24 24">
            <path fill="currentColor" d="M14 16.94V12.94H5.08L5.05 10.93H14V6.94L19 11.94Z" />
          </svg>
        </div>
      </div>

      <div class="grid grid-cols-7 gap-1">
        <span class="w-9 h-8 flex items-center justify-center" v-for="day in days" :key="day">{{ day }}</span>
        <span
          v-for="day in previous"
          :key="day"
          class="text-gray-400 flex items-center justify-center rounded-md hover:bg-gray-100 w-9 h-8 cursor-pointer"
          @click="onPreviousClick(day)"
          >{{ day }}
        </span>
        <span
          v-for="day in current"
          :key="day.day"
          class="flex items-center justify-center rounded-md hover:bg-gray-100 w-9 h-8 cursor-pointer"
          :class="[day.selected && 'bg-blue-500 text-white hover:bg-blue-500 hover:bg-opacity-95']"
          @click="onCurrentClick(day.day)"
          >{{ day.day }}
        </span>
        <span
          v-for="day in next"
          :key="day"
          class="text-gray-400 flex items-center justify-center rounded-md hover:bg-gray-100 w-9 h-8 cursor-pointer"
          @click="onNextClick(day)"
          >{{ day }}
        </span>
      </div>
      <div className="flex w-full justify-between mt-1.5">
        <span class="!font-bold text-gray-500 hover:text-black transition cursor-pointer" @click="onTodayClick">
          Today
        </span>
        <span class="font-bold text-gray-500 hover:text-black transition cursor-pointer" @click="clearSelected">
          Clear
        </span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, onUpdated, ref, watch } from 'vue';
import CalendarIcon from '../assets/CalendarIcon.vue';

interface Props {
  error?: string;
  modelValue?: string;
}

const days = ['Mo', 'Tu', 'We', 'Th', 'Fr', 'Sa', 'Su'];

const props = defineProps<Props>();
const emit = defineEmits(['update:modelValue']);

const inputRef = ref();
const calendarRef = ref();
const term = ref('');
const fromTop = ref();
const fromRight = ref();

const showCalendar = ref(false);

const date = ref({
  year: props.modelValue ? new Date(props.modelValue).getFullYear() : new Date().getFullYear(),
  month: props.modelValue ? new Date(props.modelValue).getMonth() : new Date().getMonth(),
});

const selected = ref({
  year: date.value.year,
  month: date.value.month,
  day: props.modelValue ? new Date(props.modelValue).getDate() : '',
});

const displayDate = computed(() => {
  if (selected.value) {
    const { day, year } = selected.value;
    const month = selected.value.month + 1;
    if (day && month.toString() && year) {
      return `${day.toString().padStart(2, '0')}/${month.toString().padStart(2, '0')}/${year}`;
    }
  }

  return '';
});

const monthName = computed(() =>
  new Date(date.value.year, date.value.month).toLocaleString('en', {
    month: 'long',
  })
);

const currentMonthLastDay = computed(() => new Date(date.value.year, date.value.month + 1, 0).getDate());

const currentMonth = computed(() => Array.from({ length: currentMonthLastDay.value }, (_, index) => index + 1));

const current = computed(() =>
  currentMonth.value.map((day) => {
    return {
      day,
      weekDay: new Date(date.value.year, date.value.month, day).getDay(),
      month: new Date(date.value.year, date.value.month).getMonth(),
      selected:
        day === selected.value?.day &&
        selected.value.month === date.value.month &&
        selected.value.year === date.value.year,
    };
  })
);

watch(term, () => {
  const [day, month, year] = term.value.split('/');

  if (year?.length === 4 && month.length >= 1 && day?.length >= 1) {
    const validMonth = +month < 1 ? 0 : +month > 12 ? 11 : +month - 1;
    const validDay = +day < 1 ? 1 : +day > current.value.length ? current.value.length : +day;
    date.value = { month: validMonth, year: +year };
    setSelected({
      year: +year,
      month: validMonth,
      day: validDay,
    });
  }
});

const previous = computed(() => {
  const previousMonthLastDay = new Date(
    date.value.month === 1 ? date.value.year - 1 : date.value.year,
    date.value.month === 1 ? 12 : date.value.month,
    0
  ).getDate();

  const previousMonth = Array.from({ length: previousMonthLastDay }, (_, index) => index + 1);

  if (current.value[0].weekDay === 1) {
    return previousMonth.slice(-7);
  }

  if (current.value[0].weekDay === 0) {
    return previousMonth.slice(-6);
  }

  return previousMonth.slice(-current.value[0].weekDay + 1);
});

const next = computed(() => {
  return Array.from(
    {
      length: 42 - current.value.length - previous.value.length,
    },
    (_, index) => index + 1
  );
});

const toggleShowCalendar = () => {
  showCalendar.value = !showCalendar.value;
};

const increase = () => {
  date.value = {
    year: date.value.month === 11 ? date.value.year + 1 : date.value.year,
    month: date.value.month === 11 ? 0 : date.value.month + 1,
  };
};

const decrease = () => {
  date.value = {
    year: date.value.month === 0 ? date.value.year - 1 : date.value.year,
    month: date.value.month === 0 ? 11 : date.value.month - 1,
  };
};

const onCurrentClick = (day: number) => {
  if (selected.value?.day === day) {
    clearSelected();
    return;
  }

  setSelected({ day, month: date.value.month, year: date.value.year });
};

const onPreviousClick = (day: number) => {
  decrease();
  setSelected({ day, month: date.value.month, year: date.value.year });
};

const onNextClick = (day: number) => {
  increase();
  setSelected({ day, month: date.value.month, year: date.value.year });
};

const setSelected = ({ day, month, year }) => {
  selected.value = { day, month, year };
};

const clearSelected = () => {
  selected.value = { day: 0, month: 0, year: 0 };
};

const onTodayClick = () => {
  const currentDate = new Date();
  const year = currentDate.getFullYear();
  const month = currentDate.getMonth();
  const day = currentDate.getDate();
  date.value = { year, month };
  setSelected({ year, month, day });
};

const hideCalendar = (event) => {
  if (!calendarRef.value.contains(event.target)) {
    showCalendar.value = false;
  }
};

onMounted(() => {
  document.addEventListener('click', hideCalendar);
});

onUpdated(() => {
  fromTop.value = inputRef.value?.getBoundingClientRect().top;
  fromRight.value = inputRef.value.getBoundingClientRect().right;
});

onBeforeUnmount(() => {
  document.removeEventListener('click', hideCalendar);
});

watch(selected, () => {
  emit(
    'update:modelValue',
    selected.value.day === 0 && selected.value.month === 0 && selected.value.year === 0
      ? null
      : new Date(Date.UTC(selected.value.year, selected.value.month, selected.value.day))
  );
});
</script>
