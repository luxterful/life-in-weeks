<script setup lang="ts">
import { computed, ref, onMounted } from 'vue'

const TOTAL_YEARS = 90
const WEEKS_PER_YEAR = 52

const birthInput = ref<string>('')
const hasDobParam = ref(false)

// Life expectancy references (source-provided):
// Women: 81 years and 15 weeks; Men: 76 years and 10 weeks
const LIFE_EXPECTANCY = {
  women: { years: 81, weeks: 15, totalWeeks: 81 * WEEKS_PER_YEAR + 15 },
  men: { years: 76, weeks: 10, totalWeeks: 76 * WEEKS_PER_YEAR + 10 },
} as const

const birthDate = computed<Date | null>(() => {
  if (!birthInput.value) return null
  const parts = birthInput.value.split('-').map(Number)
  if (parts.length !== 3) return null
  const [year, month, day] = parts
  if (!year || !month || !day) return null
  // Construct using local time to avoid timezone shifts
  return new Date(year, month - 1, day)
})

const today = ref(new Date())

function startOfDay(d: Date): Date {
  return new Date(d.getFullYear(), d.getMonth(), d.getDate())
}

function getAgeYears(birth: Date, onDate: Date): number {
  let age = onDate.getFullYear() - birth.getFullYear()
  const birthMonth = birth.getMonth()
  const birthDay = birth.getDate()
  const month = onDate.getMonth()
  const day = onDate.getDate()
  if (month < birthMonth || (month === birthMonth && day < birthDay)) {
    age -= 1
  }
  return Math.max(0, age)
}

function lastBirthday(birth: Date, onDate: Date): Date {
  const age = getAgeYears(birth, onDate)
  // Construct last birthday in the current year context
  return new Date(birth.getFullYear() + age, birth.getMonth(), birth.getDate())
}

function weeksSinceLastBirthday(birth: Date, onDate: Date): number {
  const msPerWeek = 1000 * 60 * 60 * 24 * 7
  const start = startOfDay(lastBirthday(birth, onDate))
  const end = startOfDay(onDate)
  const diff = end.getTime() - start.getTime()
  if (diff <= 0) return 0
  return Math.floor(diff / msPerWeek)
}

function isWeekPassed(yearIndex: number, weekIndex: number): boolean {
  if (!birthDate.value) return false
  const now = today.value
  const birth = birthDate.value
  const ageYears = getAgeYears(birth, now)

  if (yearIndex < ageYears) return true
  if (yearIndex > ageYears) return false

  const weeksThisYear = Math.min(WEEKS_PER_YEAR, weeksSinceLastBirthday(birth, now))
  return weekIndex < weeksThisYear
}

function isValidDateInput(value: string): boolean {
  // Accept YYYY-MM-DD only
  if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) return false
  const [y, m, d] = value.split('-').map(Number)
  if (!y || !m || !d) return false
  const dt = new Date(y, m - 1, d)
  return dt.getFullYear() === y && dt.getMonth() === m - 1 && dt.getDate() === d
}

onMounted(() => {
  const params = new URLSearchParams(window.location.search)
  const dob = params.get('dob')
  if (dob && isValidDateInput(dob)) {
    birthInput.value = dob
    hasDobParam.value = true
  }
})
</script>

<template>
  <div class="p-4 min-h-screen w-full bg-neutral-950 text-neutral-100">
    <div class="mx-auto max-w-[900px]">
      <div v-if="!hasDobParam" class="flex items-center gap-2 mb-4 justify-center">
        <label class="font-semibold" for="birthday">Birthday</label>
        <input
          id="birthday"
          type="date"
          v-model="birthInput"
          class="px-2 py-1 rounded border border-neutral-700 bg-neutral-900 text-neutral-100"
        />
      </div>

      <table class="border-separate border-spacing-1 mx-auto" aria-label="Life in weeks">
        <tbody>
          <tr v-for="year in TOTAL_YEARS" :key="year - 1">
            <td
              class="pr-2 w-8 text-right align-middle text-[10px] text-white select-none relative"
            >
              <span class="absolute -top-0.5 right-0">
                {{ year % 5 === 0 ? year : '' }}
              </span>
            </td>
            <td
              v-for="week in WEEKS_PER_YEAR"
              :key="week - 1"
              class="size-2 rounded-xs"
              :class="[
                isWeekPassed(year - 1, week - 1) ? 'bg-rose-600' : 'bg-neutral-800',
                week === LIFE_EXPECTANCY.women.weeks && year === LIFE_EXPECTANCY.women.years
                  ? 'border-1 border-emerald-500'
                  : '',
                week === LIFE_EXPECTANCY.men.weeks && year === LIFE_EXPECTANCY.men.years
                  ? 'border-1 border-blue-500'
                  : '',
              ]"
              aria-hidden="true"
            />
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>
