<template>
  <div class="container" data-aos="fade-left">
    <div
      class="d-flex justify-content-between align-items-center"
      style="height: 100px"
    >
      <div class="b">
        <br />
        <h3>{{ thisMonth }} 월 총 지출: {{ totalExpense }}원</h3>
        <h3>{{ thisMonth }} 월 총 수입: {{ totalIncome }}원<br /></h3>
        <h3>{{ thisMonth }} 월 수입 및 지출 통계 : {{ asset }}원<br /></h3>
      </div>
    </div>
  </div>
  <div id="calendar" data-aos="fade-left">
    <FullCalendar
      ref="calendar"
      :options="calendarOptions"
      :events="events"
      :dayCal="dayCal"
    />
  </div>
  <CalendarPopup
    v-if="showPopup"
    @close="closePopup"
    @save="saveTransaction"
    :selectedDate="selectedDate"
    :events="events"
    :dayCal="dayCal"
    :totalIncome="selectedDayData.dailyIncome"
    :totalExpense="selectedDayData.dailyExpense"
    :incomeDetails="selectedDayData.dailyIncomeList"
    :expenseDetails="selectedDayData.dailyExpenseList"
  />
</template>

<script setup>
import { reactive, ref, onMounted, watch } from "vue";
import FullCalendar from "@fullcalendar/vue3";
import dayGridPlugin from "@fullcalendar/daygrid";
import interactionPlugin from "@fullcalendar/interaction";
import CalendarPopup from "@/components/CalendarPopup.vue";
import axios from "axios";
import { useprofileStore } from "@/stores/profileStore";
import { useRouter } from "vue-router";

const profileStore = useprofileStore();
const router = useRouter();
const profile = reactive({
  id: 1,
  name: "",
  phone: "",
  date: "",
  nickname: "",
});

const profileItem = {
  name: profile.name,
  phone: profile.phone,
  date: profile.date,
  nickname: profile.nickname,
};

async function updateProfile() {
  try {
    await profileStore.addProfile(profile.id, profileItem);
    router.push("/home");
  } catch (error) {
    console.error("프로필 업데이트 실패:", error);
  }
}

updateProfile();

const asset = ref(0);
const totalIncome = ref(0);
const totalExpense = ref(0);
const showPopup = ref(false);
const selectedDate = ref(null);
const selectedDayData = ref({
  dailyIncome: 0,
  dailyExpense: 0,
  dailyIncomeList: "",
  dailyExpenseList: "",
});
const calendar = ref(null);
const events = ref([]);
const dayCal = ref([]);
const thisMonth = ref(new Date().getMonth() + 1);

const calculateMonthlyTotals = (month, dayCal) => {
  let income = 0;
  let expense = 0;
  for (const date in dayCal) {
    const eventMonth = new Date(date).getMonth() + 1;
    if (eventMonth === month) {
      income += dayCal[date].totalIncome;
      expense += dayCal[date].totalExpense;
    }
  }
  return { income, expense };
};

const calendarOptions = ref({
  plugins: [dayGridPlugin, interactionPlugin],
  initialView: "dayGridMonth",
  dateClick: function (info) {
    showPopup.value = true;
    selectedDate.value = info.dateStr;
    const dayData = dayCal.value[0][info.dateStr] || {
      dailyIncome: 0,
      dailyExpense: 0,
      dailyIncomeList: "",
      dailyExpenseList: "",
    };
    selectedDayData.value = {
      dailyIncome: dayData.totalIncome,
      dailyExpense: dayData.totalExpense,
      dailyIncomeList: dayData.incomes.join(", "),
      dailyExpenseList: dayData.expenses.join(", "),
    };
  },
  events: [],
  datesSet: function (info) {
    const newMonth = new Date(info.start).getMonth() + 1;
    thisMonth.value = newMonth;
  },
});

watch(thisMonth, (newMonth) => {
  const totals = calculateMonthlyTotals(newMonth, dayCal.value[0]);
  totalIncome.value = totals.income;
  totalExpense.value = totals.expense;
  asset.value = totals.income - totals.expense;
});

async function fetchProfile() {
  try {
    await profileStore.getProfile(profile.id);
    Object.assign(profile, profileStore.profile);
  } catch (error) {
    console.error("프로필 가져오기 실패:", error);
  }
}

const axiosEvents = async () => {
  try {
    const response = await axios.get("http://localhost:3000/budget");
    const data = response.data;
    const eventsArray = [];
    const dailyCalc = {};

    data.forEach((dailyData) => {
      const date = dailyData.date ? dailyData.date.substring(0, 10) : null;
      const type = dailyData.type;
      const amount = dailyData.amount;
      const category = dailyData.category;

      if (!dailyCalc[date]) {
        dailyCalc[date] = {
          incomes: [],
          expenses: [],
          totalIncome: 0,
          totalExpense: 0,
          category: [],
        };
      }

      if (type === "수입") {
        dailyCalc[date].incomes.push({ amount, category });
        dailyCalc[date].totalIncome += amount;
      } else if (type === "지출") {
        dailyCalc[date].expenses.push({ amount, category });
        dailyCalc[date].totalExpense += amount;
      }
    });

    const currentMonth = new Date().getMonth() + 1;
    thisMonth.value = currentMonth;

    const totals = calculateMonthlyTotals(currentMonth, dailyCalc);
    totalIncome.value = totals.income;
    totalExpense.value = totals.expense;
    asset.value = totals.income - totals.expense;

    for (const date in dailyCalc) {
      const incomeDetails = dailyCalc[date].incomes
        .map((income) => JSON.stringify(income))
        .join("\n");
      const expenseDetails = dailyCalc[date].expenses
        .map((expense) => JSON.stringify(expense))
        .join("\n");
      eventsArray.push({
        dateAmount: `${date}: 총 수입 - ${dailyCalc[date].totalIncome}, 총 지출 - ${dailyCalc[date].totalExpense} (수입 내역: ${incomeDetails}, 지출 내역: ${expenseDetails})`,
        date: date,
        dailyIncome: dailyCalc[date].totalIncome,
        dailyExpense: dailyCalc[date].totalExpense,
        dailyIncomeList: incomeDetails,
        dailyExpenseList: expenseDetails,
      });
    }

    dayCal.value.push(dailyCalc);
    events.value = eventsArray;
  } catch (error) {
    console.error("Error fetching events:", error);
  }

  const d = dayCal.value[0];

  for (const date in d) {
    calendarOptions.value.events.push({
      title: `수입: ${dayCal.value[0][date].totalIncome} 지출: ${dayCal.value[0][date].totalExpense}`,
      start: date,
      end: date,
    });
  }
};

onMounted(async () => {
  await fetchProfile();
  axiosEvents();
});

function closePopup() {
  showPopup.value = false;
}

function saveTransaction(transactionData) {
  console.log("Saving transaction:", transactionData);
}
</script>

<style scoped>
.container {
  margin: 50px;
}
div.b {
  background-color: #46aaeb;
  text-align: left;
  padding: 20px;
}
#calendar {
  padding: 50px;
}
</style>
