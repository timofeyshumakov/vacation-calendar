<template>
  <div class="employee-schedule">
    <!-- Шапка с навигацией по месяцам -->
    <v-card class="mb-4">
      <v-card-title class="d-flex align-center justify-space-between">
        <v-btn icon @click="prevPeriod">
          <v-icon>mdi-chevron-left</v-icon>
        </v-btn>
        
        <span class="text-h5">{{ periodTitle }}</span>
        
        <v-btn icon @click="nextPeriod">
          <v-icon>mdi-chevron-right</v-icon>
        </v-btn>
      </v-card-title>
      
      <v-card-actions class="d-flex align-center justify-space-between">
        <div class="d-flex align-center">
          <v-btn-toggle
            v-model="viewMode"
            mandatory
            class="mr-4"
          >
            <v-btn value="day" size="small">День</v-btn>
            <v-btn value="week" size="small">Неделя</v-btn>
            <v-btn value="month" size="small">Месяц</v-btn>
            <v-btn value="year" size="small">Год</v-btn>
          </v-btn-toggle>
          
          <v-btn
            v-if="viewMode === 'week'"
            color="primary"
            variant="text"
            size="small"
            @click="goToCurrentWeek"
          >
            Текущая неделя
          </v-btn>
        </div>
        
        <v-btn
          color="success"
          prepend-icon="mdi-download"
          @click="exportToExcel"
        >
          Выгрузить в Excel
        </v-btn>
      </v-card-actions>
    </v-card>

    <!-- Панель фильтров -->
    <v-expansion-panels class="panel mb-4" v-model="panel">
      <v-expansion-panel>
        <v-expansion-panel-title>
          <v-icon start>mdi-filter</v-icon>
          Фильтры
        </v-expansion-panel-title>
        <v-expansion-panel-text>
          <div class="filters">
            <!-- Фильтр по отделам -->
            <v-autocomplete
              v-model="filters.selected.departments"
              :items="filters.value.departments"
              item-title="NAME"
              item-value="ID"
              label="Отдел"
              single-line
              hide-details
              variant="outlined"
              multiple
              chips
              clearable
              prepend-inner-icon="mdi-office-building"
            >
              <template v-slot:prepend-item>
                <v-list-item>
                  <v-list-item-content>
                    <v-list-item-title>
                      <v-checkbox 
                        label="Выбрать все отделы" 
                        v-model="filters.selectAll.departments" 
                        @change="() => toggleSelectAll('departments')"
                        :disabled="filters.value.departments.length === 0"
                      />
                    </v-list-item-title>
                  </v-list-item-content>
                </v-list-item>
              </template>
            </v-autocomplete>

            <!-- Фильтр по ответственным -->
            <v-autocomplete
              v-model="filters.selected.employees"
              :items="filteredEmployeeOptions"
              item-title="name"
              item-value="id"
              label="Ответственный"
              single-line
              hide-details
              variant="outlined"
              multiple
              chips
              clearable
              prepend-inner-icon="mdi-account"
            >
              <template v-slot:prepend-item>
                <v-list-item>
                  <v-list-item-content>
                    <v-list-item-title>
                      <v-checkbox 
                        label="Выбрать всех ответственных" 
                        v-model="filters.selectAll.employees" 
                        @change="() => toggleSelectAll('employees')"
                        :disabled="filteredEmployeeOptions.length === 0"
                      />
                    </v-list-item-title>
                  </v-list-item-content>
                </v-list-item>
              </template>
            </v-autocomplete>
          </div>
        </v-expansion-panel-text>
      </v-expansion-panel>
    </v-expansion-panels>

    <!-- Кнопки действий -->
    <div class="action-buttons mb-4">
      <v-btn 
        color="warning" 
        prepend-icon="mdi-clock-plus"
        @click="openOvertimeDialog"
      >
        Добавить переработку
      </v-btn>
      <v-btn 
        color="info" 
        prepend-icon="mdi-home"
        @click="openRemoteDialog"
      >
        Добавить удаленку
      </v-btn>
      <v-btn 
        color="secondary" 
        prepend-icon="mdi-filter-off"
        @click="disableFilters"
        variant="outlined"
      >
        Сбросить фильтры
      </v-btn>
    </div>

    <!-- Таблица сотрудников -->
    <v-table class="schedule-table mb-6">
      <thead>
        <tr>
          <th class="employee-column">Сотрудник</th>
          <th class="department-column">Отдел</th>
          <th 
            v-for="period in periods" 
            :key="period.key"
            class="day-column"
            :style="{ minWidth: periodWidth }"
          >
            {{ period.label }}
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="employee in filteredEmployees" :key="employee.id">
          <!-- ФИО сотрудника -->
          <td 
            class="employee-name"
            @click="selectEmployeeForStats(employee)"
          >
            {{ employee.name }}
          </td>
          
          <!-- Отдел сотрудника -->
          <td class="department-name">
            {{ getDepartmentName(employee.departmentId) }}
          </td>
          
          <!-- Ячейки в зависимости от режима -->
          <template v-if="viewMode === 'month'">
            <!-- Существующее отображение для месяца -->
            <td 
              v-for="(mergedDay, index) in getMergedDays(employee)" 
              :key="index"
              :colspan="mergedDay.span"
              class="schedule-cell"
              :class="{ 'clickable': mergedDay.status }"
              @click="mergedDay.status && openDateDialog(employee, mergedDay)"
            >
              <v-chip
                v-if="mergedDay.status"
                :color="getStatusColor(mergedDay.status)"
                size="small"
                class="status-chip"
              >
                {{ getStatusText(mergedDay.status) }}
                <span v-if="mergedDay.hours" class="ml-1">{{ mergedDay.hours }}</span>
              </v-chip>
              <span v-else class="empty-cell">—</span>
            </td>
          </template>
          
          <template v-else>
            <!-- Отображение для дня/недели/года -->
            <td 
              v-for="period in periods" 
              :key="period.key"
              class="schedule-cell"
              :class="{ 'clickable': getStatusForPeriod(employee, period) }"
              @click="openPeriodDialog(employee, period)"
            >
              <v-chip
                v-if="getStatusForPeriod(employee, period)"
                :color="getStatusColor(getStatusForPeriod(employee, period))"
                size="small"
                class="status-chip"
              >
                {{ getStatusText(getStatusForPeriod(employee, period)) }}
                <span v-if="getHoursForPeriod(employee, period)" class="ml-1">
                  {{ getHoursForPeriod(employee, period) }}
                </span>
              </v-chip>
              <span v-else class="empty-cell">—</span>
            </td>
          </template>
        </tr>
        <tr v-if="filteredEmployees.length === 0">
          <td :colspan="periods.length + 2" class="text-center pa-4">
            <v-icon icon="mdi-alert" size="large" color="warning" class="mb-2"></v-icon>
            <p>Нет сотрудников, соответствующих выбранным фильтрам</p>
          </td>
        </tr>
      </tbody>
    </v-table>

    <!-- Блок статистики для выбранного пользователя -->
    <v-card v-if="selectedEmployeeForStats" class="mb-4 stats-card">
      <v-card-title class="bg-primary text-white">
        Статистика для {{ selectedEmployeeForStats.name }}
        <v-btn 
          icon 
          @click="selectedEmployeeForStats = null" 
          class="ml-auto" 
          color="white" 
          variant="text"
          size="small"
        >
          <v-icon>mdi-close</v-icon>
        </v-btn>
      </v-card-title>
      <v-card-text>
        <v-row>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ vacationDaysInSchedule }}</div>
              <div class="text-subtitle-2">Дней отпуска в графике</div>
            </v-sheet>
          </v-col>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ vacationDaysLeft }}</div>
              <div class="text-subtitle-2">Осталось отпускных дней</div>
              <div class="text-caption">(28 - использовано)</div>
            </v-sheet>
          </v-col>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ totalOvertimeHours }}</div>
              <div class="text-subtitle-2">Переработанные часы</div>
              <div class="text-caption">всего за месяц</div>
            </v-sheet>
          </v-col>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ weekendOvertimeHours }}</div>
              <div class="text-subtitle-2">Переработки в выходные</div>
              <div class="text-caption">часы</div>
            </v-sheet>
          </v-col>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ weekdayOvertimeHours }}</div>
              <div class="text-subtitle-2">Переработки в будние</div>
              <div class="text-caption">часы</div>
            </v-sheet>
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>

    <v-card class="mb-4">
      <v-card-title class="bg-warning text-white">
        Статистика по переработкам
      </v-card-title>
      <v-card-text>
        <v-table class="overtime-stats-table">
          <thead>
            <tr>
              <th>Сотрудник</th>
              <th>Дата</th>
              <th>Время начала</th>
              <th>Время окончания</th>
              <th>Всего часов</th>
              <th>Мероприятие</th>
              <th>Тип работ</th>
              <th>Обоснование</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in overtimeStats" :key="index">
              <td>{{ getEmployeeName(item.employeeId) }}</td>
              <td>{{ formatDate(item.details.date) }}</td>
              <td>{{ item.details.startTime }}</td>
              <td>{{ item.details.endTime }}</td>
              <td>{{ item.hours }}</td>
              <td>{{ getEventName(item.details.event) }}</td>
              <td>{{ getWorkTypeName(item.details.workType) }}</td>
              <td>{{ item.details.justification }}</td>
            </tr>
            <tr v-if="overtimeStats.length === 0">
              <td colspan="8" class="text-center pa-4">
                <v-icon icon="mdi-information" size="large" color="info" class="mb-2"></v-icon>
                <p>Нет данных о переработках за текущий месяц</p>
              </td>
            </tr>
          </tbody>
        </v-table>
      </v-card-text>
    </v-card>
    <!-- Диалог выбора диапазона дат -->
    <v-dialog v-model="dateDialog" max-width="500px">
      <v-card>
        <v-card-title class="date-title d-flex align-center">
          Изменить период для {{ selectedEmployee?.name }}
          <v-btn icon @click="dateDialog = false" class="ml-auto">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        
        <v-card-text>
          <v-row>
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="startDate"
                label="Начальная дата"
                type="date"
                :min="minDate"
                :max="endDate || maxDate"
                @update:model-value="validateDates"
              ></v-text-field>
            </v-col>
            
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="endDate"
                label="Конечная дата"
                type="date"
                :min="startDate || minDate"
                :max="maxDate"
                @update:model-value="validateDates"
              ></v-text-field>
            </v-col>
          </v-row>
          
          <v-row v-if="dateError" class="mt-2">
            <v-col cols="12">
              <v-alert type="error" density="compact">
                {{ dateError }}
              </v-alert>
            </v-col>
          </v-row>

          <v-row class="mt-4">
            <v-col cols="12">
              <v-select
                v-model="selectedStatus"
                :items="statusOptions"
                label="Выберите новый статус"
                item-title="text"
                item-value="value"
                :disabled="!canChangeStatus"
              ></v-select>
            </v-col>
          </v-row>

          <!-- Поле для ввода времени переработки -->
          <v-row v-if="selectedStatus === 'overtime'" class="mt-2">
            <v-col cols="12">
              <v-text-field
                v-model="overtimeHours"
                label="Время переработки (например: 2ч 15м или 2:15)"
                placeholder="2ч 15м"
                hint="Формат: часы и минуты (2ч 15м, 1:30, 45м)"
                persistent-hint
              ></v-text-field>
            </v-col>
          </v-row>
        </v-card-text>
        
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn 
            color="error" 
            @click="deletePeriod" 
            :disabled="!canDelete"
          >
            Удалить
          </v-btn>
          <v-btn 
            color="primary" 
            @click="updatePeriod" 
            :disabled="!isValidSelection"
          >
            Обновить
          </v-btn>
          <v-btn color="secondary" variant="text" @click="dateDialog = false">
            Отмена
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Диалог добавления переработки -->
    <v-dialog v-model="overtimeDialog" max-width="600px">
      <v-card>
        <v-card-title class="bg-warning text-white d-flex align-center">
          Добавить переработку
          <v-btn icon @click="overtimeDialog = false" class="ml-auto" color="white" variant="text">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        
        <v-card-text class="pt-4">
          <v-row>
            <v-col cols="12">
              <v-select
                v-model="overtimeForm.employeeId"
                :items="employees"
                item-title="name"
                item-value="id"
                label="Сотрудник"
                variant="outlined"
                required
              ></v-select>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeForm.whatDone"
                label="1) Что сделано"
                variant="outlined"
                rows="3"
                required
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-text-field
                v-model="overtimeForm.date"
                label="2) День"
                type="date"
                :min="minDate"
                :max="maxDate"
                variant="outlined"
                required
              ></v-text-field>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="overtimeForm.startTime"
                label="3) Время начала работ"
                type="time"
                variant="outlined"
                required
              ></v-text-field>
            </v-col>
            
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="overtimeForm.endTime"
                label="4) Время завершения работ"
                type="time"
                variant="outlined"
                required
              ></v-text-field>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-select
                v-model="overtimeForm.event"
                :items="mockEvents"
                label="5) Мероприятия"
                variant="outlined"
                required
              ></v-select>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-select
                v-model="overtimeForm.workType"
                :items="workTypes"
                label="6) Тип работ"
                variant="outlined"
                required
              ></v-select>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeForm.justification"
                label="7) Обоснование работы во вне рабочее время"
                variant="outlined"
                rows="2"
                required
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row v-if="overtimeError" class="mt-2">
            <v-col cols="12">
              <v-alert type="error" density="compact">
                {{ overtimeError }}
              </v-alert>
            </v-col>
          </v-row>
        </v-card-text>
        
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn 
            color="warning" 
            @click="saveOvertime"
            :disabled="!isOvertimeValid"
          >
            Сохранить
          </v-btn>
          <v-btn color="secondary" variant="text" @click="overtimeDialog = false">
            Отмена
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Диалог добавления удаленки -->
    <v-dialog v-model="remoteDialog" max-width="500px">
      <v-card>
        <v-card-title class="bg-info text-white d-flex align-center">
          Добавить удаленку
          <v-btn icon @click="remoteDialog = false" class="ml-auto" color="white" variant="text">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        
        <v-card-text class="pt-4">
          <v-row>
            <v-col cols="12">
              <v-select
                v-model="remoteForm.employeeId"
                :items="employees"
                item-title="name"
                item-value="id"
                label="Сотрудник"
                variant="outlined"
                required
              ></v-select>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-label class="mb-2">Выберите дни удаленной работы</v-label>
              <v-date-picker
                v-model="remoteForm.dates"
                multiple
                :min="minDate"
                :max="maxDate"
                title="Календарь"
              ></v-date-picker>
            </v-col>
          </v-row>

          <v-row v-if="remoteError" class="mt-2">
            <v-col cols="12">
              <v-alert type="error" density="compact">
                {{ remoteError }}
              </v-alert>
            </v-col>
          </v-row>

          <v-row v-if="remoteForm.dates && remoteForm.dates.length > 0" class="mt-2">
            <v-col cols="12">
              <v-chip-group>
                <v-chip
                  v-for="date in remoteForm.dates"
                  :key="date"
                  color="info"
                  size="small"
                  class="mr-1"
                >
                  {{ moment(date).format('DD.MM.YYYY') }}
                </v-chip>
              </v-chip-group>
            </v-col>
          </v-row>
        </v-card-text>
        
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn 
            color="info" 
            @click="saveRemote"
            :disabled="!isRemoteValid"
          >
            Сохранить
          </v-btn>
          <v-btn color="secondary" variant="text" @click="remoteDialog = false">
            Отмена
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import moment from 'moment'
import * as XLSX from 'xlsx'

// Состояние для навигации по месяцам
const currentYear = computed(() => currentDate.value.year())
const currentMonth = computed(() => currentDate.value.month())
const currentMonthName = computed(() => currentDate.value.format('MMMM'))

// Количество дней в текущем месяце
const daysInMonth = computed(() => {
  return currentDate.value.daysInMonth()
})

// Минимальная и максимальная даты для выбора
const minDate = computed(() => {
  return moment(currentDate.value).startOf('month').format('YYYY-MM-DD')
})

const maxDate = computed(() => {
  return moment(currentDate.value).endOf('month').format('YYYY-MM-DD')
})

// Навигация по месяцам
const prevMonth = () => {
  currentDate.value = moment(currentDate.value).subtract(1, 'month')
}

const nextMonth = () => {
  currentDate.value = moment(currentDate.value).add(1, 'month')
}

// Состояние для панели фильтров
const panel = ref(true)

// Состояние для диалога выбора дат
const dateDialog = ref(false)
const startDate = ref(null)
const endDate = ref(null)
const selectedStatus = ref(null)
const selectedEmployee = ref(null)
const selectedPeriod = ref(null)
const overtimeHours = ref('')
const dateError = ref('')

// Состояние для диалогов
const overtimeDialog = ref(false)
const remoteDialog = ref(false)
const overtimeError = ref('')
const remoteError = ref('')

// Форма для переработки
const overtimeForm = ref({
  employeeId: null,
  whatDone: '',
  date: null,
  startTime: '',
  endTime: '',
  event: null,
  workType: null,
  justification: ''
})

// Форма для удаленки
const remoteForm = ref({
  employeeId: null,
  dates: []
})

// Моковые данные для выпадающих списков
const mockEvents = [
  { title: 'Конференция "IT Today"', value: 1 },
  { title: 'Семинар по Vue.js', value: 2 },
  { title: 'Встреча с партнерами', value: 3 },
  { title: 'Обучение сотрудников', value: 4 },
]

const workTypes = [
  { title: 'Разработка', value: 'development' },
  { title: 'Тестирование', value: 'testing' },
  { title: 'Аналитика', value: 'analytics' },
  { title: 'Дизайн', value: 'design' },
  { title: 'Документация', value: 'documentation' },
]

// Моковые данные для отделов
const departments = ref([
  { ID: 1, NAME: 'Отдел разработки' },
  { ID: 2, NAME: 'Отдел тестирования' },
  { ID: 3, NAME: 'Отдел аналитики' },
  { ID: 4, NAME: 'Отдел дизайна' },
  { ID: 5, NAME: 'Отдел продаж' },
])

// Триггер для обновления таблицы
const updateTrigger = ref(0)

// Опции статусов
const statusOptions = [
  { text: 'Отпуск', value: 'vacation' },
  { text: 'Отгул', value: 'dayoff' },
  { text: 'Удаленка', value: 'remote' },
  { text: 'Переработка', value: 'overtime' }
]

// Объект фильтров
const filters = ref({
  value: {
    departments: departments.value,
    employees: []
  },
  selected: {
    departments: [],
    employees: []
  },
  selectAll: {
    departments: false,
    employees: false
  }
})

// Моковые данные сотрудников с отделами
const employees = ref([
  {
    id: 1,
    name: 'Иванов Иван Иванович',
    departmentId: 1,
    schedule: [
      { day: 1, status: 'vacation' },
      { day: 2, status: 'vacation' },
      { day: 3, status: 'vacation' },
      { day: 4, status: null },
      { day: 5, status: null },
      { day: 6, status: 'remote' },
      { day: 7, status: 'remote' },
      { day: 8, status: 'dayoff' },
      { day: 9, status: null },
      { 
        day: 10, 
        status: 'overtime', 
        hours: '2ч 30м',
        details: {
          whatDone: 'Разработка нового функционала',
          date: '2024-01-10',
          startTime: '18:00',
          endTime: '20:30',
          event: 1,
          workType: 'development',
          justification: 'Срочный релиз'
        }
      },
      { 
        day: 11, 
        status: 'overtime', 
        hours: '1ч 45м',
        details: {
          whatDone: 'Исправление критических багов',
          date: '2024-01-11',
          startTime: '19:00',
          endTime: '20:45',
          event: 2,
          workType: 'testing',
          justification: 'Инцидент на проде'
        }
      },
      { 
        day: 12, 
        status: 'overtime', 
        hours: '3ч',
        details: {
          whatDone: 'Подготовка документации',
          date: '2024-01-12',
          startTime: '17:30',
          endTime: '20:30',
          event: 3,
          workType: 'documentation',
          justification: 'Срочный отчет'
        }
      },
    ]
  },
  {
    id: 2,
    name: 'Петров Петр Петрович',
    departmentId: 1,
    schedule: [
      { day: 1, status: null },
      { day: 2, status: null },
      { day: 3, status: 'vacation' },
      { day: 4, status: 'vacation' },
      { day: 5, status: 'vacation' },
      { day: 6, status: 'vacation' },
      { day: 7, status: null },
      { day: 8, status: 'dayoff' },
      { day: 9, status: 'remote' },
      { day: 10, status: 'remote' },
    ]
  },
  {
    id: 3,
    name: 'Сидорова Анна Сергеевна',
    departmentId: 2,
    schedule: [
      { day: 1, status: 'remote' },
      { day: 2, status: 'remote' },
      { day: 3, status: 'dayoff' },
      { day: 4, status: null },
      { day: 5, status: null },
      { day: 6, status: 'vacation' },
      { day: 7, status: 'vacation' },
    ]
  },
  {
    id: 4,
    name: 'Козлов Дмитрий Николаевич',
    departmentId: 3,
    schedule: [
      { day: 1, status: null },
      { day: 2, status: null },
      { day: 3, status: null },
      { 
        day: 4, 
        status: 'overtime', 
        hours: '1ч 30м',
        details: {
          whatDone: 'Анализ данных',
          date: '2024-01-04',
          startTime: '18:00',
          endTime: '19:30',
          event: 2,
          workType: 'analytics',
          justification: 'Срочный запрос'
        }
      },
      { day: 5, status: 'remote' },
      { day: 6, status: 'remote' },
      { day: 7, status: 'dayoff' },
    ]
  },
  {
    id: 5,
    name: 'Смирнова Елена Александровна',
    departmentId: 4,
    schedule: [
      { day: 1, status: 'remote' },
      { day: 2, status: 'remote' },
      { day: 3, status: 'remote' },
      { day: 4, status: 'vacation' },
      { day: 5, status: 'vacation' },
      { day: 6, status: null },
      { day: 7, status: null },
    ]
  },
  {
    id: 6,
    name: 'Морозов Алексей Викторович',
    departmentId: 5,
    schedule: [
      { day: 1, status: 'dayoff' },
      { day: 2, status: null },
      { day: 3, status: null },
      { day: 4, status: 'remote' },
      { 
        day: 5, 
        status: 'overtime', 
        hours: '3ч',
        details: {
          whatDone: 'Встреча с клиентом',
          date: '2024-01-05',
          startTime: '17:00',
          endTime: '20:00',
          event: 3,
          workType: 'analytics',
          justification: 'Срочные переговоры'
        }
      },
      { 
        day: 6, 
        status: 'overtime', 
        hours: '2ч',
        details: {
          whatDone: 'Разработка презентации',
          date: '2024-01-06',
          startTime: '16:00',
          endTime: '18:00',
          event: 4,
          workType: 'design',
          justification: 'Презентация для заказчика'
        }
      },
      { day: 7, status: null },
    ]
  }
])
const viewMode = ref('month') // day, week, month, year

// Текущая дата для навигации
const currentDate = ref(moment())

// Заголовок периода
const periodTitle = computed(() => {
  switch (viewMode.value) {
    case 'day':
      return currentDate.value.format('DD MMMM YYYY')
    case 'week':
      const start = moment(currentDate.value).startOf('week')
      const end = moment(currentDate.value).endOf('week')
      return `${start.format('DD MMM')} - ${end.format('DD MMM YYYY')}`
    case 'month':
      return currentDate.value.format('MMMM YYYY')
    case 'year':
      return currentDate.value.format('YYYY')
    default:
      return ''
  }
})

// Периоды для отображения
const periods = computed(() => {
  switch (viewMode.value) {
    case 'day':
      return [{ key: currentDate.value.format('YYYY-MM-DD'), label: currentDate.value.format('DD.MM') }]
    
    case 'week':
      const weekDays = []
      const start = moment(currentDate.value).startOf('week')
      for (let i = 0; i < 7; i++) {
        const day = moment(start).add(i, 'days')
        weekDays.push({
          key: day.format('YYYY-MM-DD'),
          label: day.format('dd DD.MM')
        })
      }
      return weekDays
    
    case 'month':
      const monthDays = []
      for (let i = 1; i <= daysInMonth.value; i++) {
        monthDays.push({
          key: i,
          label: i.toString()
        })
      }
      return monthDays
    
    case 'year':
      const months = []
      for (let i = 0; i < 12; i++) {
        const month = moment(currentDate.value).month(i)
        months.push({
          key: i + 1,
          label: month.format('MMM')
        })
      }
      return months
    
    default:
      return []
  }
})

// Ширина колонки в зависимости от режима
const periodWidth = computed(() => {
  switch (viewMode.value) {
    case 'day': return '150px'
    case 'week': return '100px'
    case 'month': return '50px'
    case 'year': return '80px'
    default: return '50px'
  }
})

// Навигация
const prevPeriod = () => {
  switch (viewMode.value) {
    case 'day':
      currentDate.value = moment(currentDate.value).subtract(1, 'day')
      break
    case 'week':
      currentDate.value = moment(currentDate.value).subtract(1, 'week')
      break
    case 'month':
      currentDate.value = moment(currentDate.value).subtract(1, 'month')
      break
    case 'year':
      currentDate.value = moment(currentDate.value).subtract(1, 'year')
      break
  }
}

const nextPeriod = () => {
  switch (viewMode.value) {
    case 'day':
      currentDate.value = moment(currentDate.value).add(1, 'day')
      break
    case 'week':
      currentDate.value = moment(currentDate.value).add(1, 'week')
      break
    case 'month':
      currentDate.value = moment(currentDate.value).add(1, 'month')
      break
    case 'year':
      currentDate.value = moment(currentDate.value).add(1, 'year')
      break
  }
}

const goToCurrentWeek = () => {
  currentDate.value = moment()
}

// Получение статуса для периода
const getStatusForPeriod = (employee, period) => {
  if (viewMode.value === 'month') return null
  
  let date
  if (viewMode.value === 'day' || viewMode.value === 'week') {
    date = moment(period.key).date()
  } else if (viewMode.value === 'year') {
    // Для года нужно агрегировать данные по месяцам
    const monthSchedule = employee.schedule.filter(item => {
      const itemDate = moment(item.details?.date || `${currentDate.value.year()}-${period.key}-01`)
      return itemDate.month() + 1 === period.key
    })
    
    if (monthSchedule.length > 0) {
      // Если есть переработки в этом месяце
      const overtimeCount = monthSchedule.filter(s => s.status === 'overtime').length
      if (overtimeCount > 0) return 'overtime'
      
      // Другие статусы
      const statuses = monthSchedule.map(s => s.status)
      if (statuses.includes('vacation')) return 'vacation'
      if (statuses.includes('remote')) return 'remote'
      if (statuses.includes('dayoff')) return 'dayoff'
    }
    return null
  }
  
  const scheduleItem = employee.schedule.find(s => s.day === date)
  return scheduleItem?.status || null
}

const getHoursForPeriod = (employee, period) => {
  if (viewMode.value === 'year') {
    const monthSchedule = employee.schedule.filter(item => {
      const itemDate = moment(item.details?.date || `${currentDate.value.year()}-${period.key}-01`)
      return itemDate.month() + 1 === period.key && item.status === 'overtime'
    })
    
    if (monthSchedule.length > 0) {
      const totalMinutes = monthSchedule.reduce((total, item) => {
        if (item.hours) {
          const hoursMatch = item.hours.match(/(\d+)ч/)
          const minutesMatch = item.hours.match(/(\d+)м/)
          if (hoursMatch) total += parseInt(hoursMatch[1]) * 60
          if (minutesMatch) total += parseInt(minutesMatch[1])
        }
        return total
      }, 0)
      
      const hours = Math.floor(totalMinutes / 60)
      const minutes = totalMinutes % 60
      return minutes > 0 ? `${hours}ч ${minutes}м` : `${hours}ч`
    }
    return null
  }
  
  const date = viewMode.value === 'week' ? moment(period.key).date() : period.key
  const scheduleItem = employee.schedule.find(s => s.day === date)
  return scheduleItem?.hours || null
}

// Открытие диалога для периода
const openPeriodDialog = (employee, period) => {
  if (viewMode.value === 'year') {
    // Для года показываем сводку по месяцу
    alert(`Статистика за ${period.label} для ${employee.name}\n\n` +
          `Отпуск: ${getMonthStats(employee, period.key, 'vacation')} дней\n` +
          `Удаленка: ${getMonthStats(employee, period.key, 'remote')} дней\n` +
          `Отгулы: ${getMonthStats(employee, period.key, 'dayoff')} дней\n` +
          `Переработки: ${getMonthStats(employee, period.key, 'overtime')} часов`)
  } else {
    // Для дня/недели показываем существующий диалог
    let day
    if (viewMode.value === 'week') {
      day = moment(period.key).date()
    } else {
      day = period.key
    }
    
    const mergedDay = {
      startDay: day,
      span: 1,
      status: getStatusForPeriod(employee, period),
      hours: getHoursForPeriod(employee, period),
      details: employee.schedule.find(s => s.day === day)?.details
    }
    
    openDateDialog(employee, mergedDay)
  }
}

// Вспомогательная функция для статистики по месяцу
const getMonthStats = (employee, month, type) => {
  const monthSchedule = employee.schedule.filter(item => {
    const itemDate = moment(item.details?.date || `${currentDate.value.year()}-${month}-01`)
    return itemDate.month() + 1 === month && 
           (type === 'overtime' ? item.status === 'overtime' : item.status === type)
  })
  
  if (type === 'overtime') {
    const totalMinutes = monthSchedule.reduce((total, item) => {
      if (item.hours) {
        const hoursMatch = item.hours.match(/(\d+)ч/)
        const minutesMatch = item.hours.match(/(\d+)м/)
        if (hoursMatch) total += parseInt(hoursMatch[1]) * 60
        if (minutesMatch) total += parseInt(minutesMatch[1])
      }
      return total
    }, 0)
    
    const hours = Math.floor(totalMinutes / 60)
    const minutes = totalMinutes % 60
    return minutes > 0 ? `${hours}ч ${minutes}м` : `${hours}ч`
  }
  
  return monthSchedule.length
}

// Функция выгрузки в Excel
const exportToExcel = () => {
  try {
    // Создаем массив данных для Excel
    const excelData = []
    
    // Заголовки
    const headers = ['Сотрудник', 'Отдел']
    periods.value.forEach(p => headers.push(p.label))
    excelData.push(headers)
    
    // Данные по сотрудникам
    filteredEmployees.value.forEach(employee => {
      const row = [
        employee.name,
        getDepartmentName(employee.departmentId)
      ]
      
      if (viewMode.value === 'month') {
        // Для месяца выводим каждый день
        for (let day = 1; day <= daysInMonth.value; day++) {
          const scheduleItem = employee.schedule.find(s => s.day === day)
          if (scheduleItem) {
            const status = getStatusText(scheduleItem.status)
            const hours = scheduleItem.hours ? ` (${scheduleItem.hours})` : ''
            row.push(status + hours)
          } else {
            row.push('—')
          }
        }
      } else {
        // Для дня/недели/года
        periods.value.forEach(period => {
          const status = getStatusForPeriod(employee, period)
          const hours = getHoursForPeriod(employee, period)
          if (status) {
            row.push(getStatusText(status) + (hours ? ` (${hours})` : ''))
          } else {
            row.push('—')
          }
        })
      }
      
      excelData.push(row)
    })
    
    // Создаем рабочий лист
    const ws = XLSX.utils.aoa_to_sheet(excelData)
    
    // Настраиваем ширину колонок
    const colWidths = []
    colWidths.push({ wch: 30 }) // Сотрудник
    colWidths.push({ wch: 20 }) // Отдел
    periods.value.forEach(() => colWidths.push({ wch: 15 }))
    ws['!cols'] = colWidths
    
    // Создаем рабочую книгу
    const wb = XLSX.utils.book_new()
    XLSX.utils.book_append_sheet(wb, ws, 'График работы')
    
    // Сохраняем файл
    const fileName = `график_работы_${periodTitle.value.replace(/\s+/g, '_')}.xlsx`
    XLSX.writeFile(wb, fileName)
    
    console.log('Файл успешно выгружен:', fileName)
  } catch (error) {
    console.error('Ошибка при выгрузке в Excel:', error)
    alert('Произошла ошибка при выгрузке в Excel')
  }
}

// Обновляем список сотрудников в фильтрах
filters.value.value.employees = employees.value.map(emp => ({
  id: emp.id,
  name: emp.name,
  departmentId: emp.departmentId
}))

// Отфильтрованные сотрудники
const filteredEmployees = computed(() => {
  return employees.value.filter(employee => {
    // Фильтр по отделам
    if (filters.value.selected.departments.length > 0) {
      if (!filters.value.selected.departments.includes(employee.departmentId)) {
        return false
      }
    }
    
    // Фильтр по ответственным
    if (filters.value.selected.employees.length > 0) {
      if (!filters.value.selected.employees.includes(employee.id)) {
        return false
      }
    }
    
    return true
  })
})

// Отфильтрованные опции для выбора ответственных
const filteredEmployeeOptions = computed(() => {
  // Если нет сотрудников, возвращаем пустой массив
  if (!employees.value.length) return []
  
  // Если выбран отдел, показываем только сотрудников из этого отдела
  if (filters.value.selected.departments.length > 0) {
    return employees.value
      .filter(emp => filters.value.selected.departments.includes(emp.departmentId))
      .map(emp => ({ id: emp.id, name: emp.name }))
  }
  
  // Иначе показываем всех сотрудников
  return employees.value.map(emp => ({ id: emp.id, name: emp.name }))
})

// Функция получения названия отдела
const getDepartmentName = (departmentId) => {
  const dept = departments.value.find(d => d.ID === departmentId)
  return dept ? dept.NAME : 'Не указан'
}

// Функция переключения выбора всех элементов
const toggleSelectAll = (type) => {
  try {
    if (type === 'employees') {
      // Для сотрудников используем отфильтрованный список
      if (filters.value.selectAll[type]) {
        filters.value.selected[type] = filteredEmployeeOptions.value.map(item => item.id)
      } else {
        filters.value.selected[type] = []
      }
    } else {
      // Для отделов
      if (filters.value.selectAll[type]) {
        filters.value.selected[type] = filters.value.value[type].map(item => item.ID)
      } else {
        filters.value.selected[type] = []
      }
    }
  } catch (error) {
    console.error('Ошибка при выборе элементов:', error)
  }
}

// Функция сброса фильтров
const disableFilters = () => {
  panel.value = false
  for (let i = 0; i < Object.keys(filters.value.selectAll).length; i++) {
    filters.value.selectAll[Object.keys(filters.value.selectAll)[i]] = false
    filters.value.selected[Object.keys(filters.value.selected)[i]] = []
  }
}
const vacationDaysInSchedule = computed(() => {
  if (!selectedEmployeeForStats.value) return 0
  
  return selectedEmployeeForStats.value.schedule.filter(
    item => item.status === 'vacation'
  ).length
})

// Переработанные часы в выходные
const weekendOvertimeHours = computed(() => {
  if (!selectedEmployeeForStats.value) return '0ч'
  
  const year = currentYear.value
  const month = currentMonth.value
  let totalMinutes = 0
  
  selectedEmployeeForStats.value.schedule
    .filter(item => item.status === 'overtime' && item.hours)
    .forEach(item => {
      // Проверяем, является ли день выходным (суббота или воскресенье)
      const date = new Date(year, month, item.day - 1) // item.day начинается с 1
      const dayOfWeek = date.getDay()
      
      if (dayOfWeek === 0 || dayOfWeek === 6) {
        // Парсим строку типа "2ч 30м" или "1ч 45м"
        const hoursMatch = item.hours.match(/(\d+)ч/)
        const minutesMatch = item.hours.match(/(\d+)м/)
        
        if (hoursMatch) totalMinutes += parseInt(hoursMatch[1]) * 60
        if (minutesMatch) totalMinutes += parseInt(minutesMatch[1])
      }
    })
  
  const hours = Math.floor(totalMinutes / 60)
  const minutes = totalMinutes % 60
  
  return minutes > 0 ? `${hours}ч ${minutes}м` : `${hours}ч`
})

// Переработанные часы в будние
const weekdayOvertimeHours = computed(() => {
  if (!selectedEmployeeForStats.value) return '0ч'
  
  const year = currentYear.value
  const month = currentMonth.value
  let totalMinutes = 0
  
  selectedEmployeeForStats.value.schedule
    .filter(item => item.status === 'overtime' && item.hours)
    .forEach(item => {
      // Проверяем, является ли день будним (понедельник-пятница)
      const date = new Date(year, month, item.day - 1) // item.day начинается с 1
      const dayOfWeek = date.getDay()
      
      if (dayOfWeek >= 1 && dayOfWeek <= 5) {
        // Парсим строку типа "2ч 30м" или "1ч 45м"
        const hoursMatch = item.hours.match(/(\d+)ч/)
        const minutesMatch = item.hours.match(/(\d+)м/)
        
        if (hoursMatch) totalMinutes += parseInt(hoursMatch[1]) * 60
        if (minutesMatch) totalMinutes += parseInt(minutesMatch[1])
      }
    })
  
  const hours = Math.floor(totalMinutes / 60)
  const minutes = totalMinutes % 60
  
  return minutes > 0 ? `${hours}ч ${minutes}м` : `${hours}ч`
})
// Проверка валидности формы переработки
const isOvertimeValid = computed(() => {
  return overtimeForm.value.employeeId &&
         overtimeForm.value.whatDone &&
         overtimeForm.value.date &&
         overtimeForm.value.startTime &&
         overtimeForm.value.endTime &&
         overtimeForm.value.event &&
         overtimeForm.value.workType &&
         overtimeForm.value.justification
})

// Проверка валидности формы удаленки
const isRemoteValid = computed(() => {
  return remoteForm.value.employeeId && remoteForm.value.dates && remoteForm.value.dates.length > 0
})

// Проверка возможности изменения статуса
const canChangeStatus = computed(() => {
  return selectedPeriod.value && selectedPeriod.value.status
})

// Проверка возможности удаления
const canDelete = computed(() => {
  return selectedPeriod.value && selectedPeriod.value.status
})

// Проверка на валидность выбора
const isValidSelection = computed(() => {
  if (!startDate.value || !endDate.value || !selectedStatus.value || dateError.value) {
    return false
  }
  
  // Для переработки проверяем наличие времени
  if (selectedStatus.value === 'overtime' && !overtimeHours.value) {
    return false
  }
  
  return true
})

// Валидация дат
const validateDates = () => {
  if (startDate.value && endDate.value) {
    if (new Date(startDate.value) > new Date(endDate.value)) {
      dateError.value = 'Начальная дата не может быть позже конечной'
      return false
    }
    
    // Проверка пересечения с другими периодами
    if (selectedEmployee.value && selectedPeriod.value) {
      const start = moment(startDate.value).date()
      const end = moment(endDate.value).date()
      
      // Получаем все периоды кроме текущего
      const otherPeriods = getMergedDays(selectedEmployee.value)
        .filter(p => p.status && p.startDay !== selectedPeriod.value.startDay)
      
      // Проверяем пересечение
      const hasOverlap = otherPeriods.some(period => {
        const periodStart = period.startDay
        const periodEnd = periodStart + period.span - 1
        return !(end < periodStart || start > periodEnd)
      })
      
      if (hasOverlap) {
        dateError.value = 'Новый период пересекается с существующим'
        return false
      }
    }
  }
  dateError.value = ''
  return true
}

// Функция для объединения дней с одинаковыми статусами
const getMergedDays = (employee) => {
  // Используем updateTrigger для принудительного пересчета
  updateTrigger.value
  
  const merged = []
  let currentSpan = 0
  let currentStatus = null
  let currentHours = null
  let currentDetails = null
  let startDay = 1

  // Сортируем расписание по дням
  const sortedSchedule = [...employee.schedule].sort((a, b) => a.day - b.day)
  
  for (let day = 1; day <= daysInMonth.value; day++) {
    const scheduleItem = sortedSchedule.find(s => s.day === day)
    const status = scheduleItem?.status || null
    const hours = scheduleItem?.hours || null
    const details = scheduleItem?.details || null

    if (day === 1) {
      currentStatus = status
      currentHours = hours
      currentDetails = details
      startDay = day
      currentSpan = 1
      continue
    }

    if (status === currentStatus && hours === currentHours) {
      currentSpan++
    } else {
      // Сохраняем предыдущую группу
      if (currentStatus !== null || currentSpan > 0) {
        merged.push({
          startDay,
          status: currentStatus,
          hours: currentHours,
          details: currentDetails, // Сохраняем детали
          span: currentSpan
        })
      }
      
      // Начинаем новую группу
      currentStatus = status
      currentHours = hours
      currentDetails = details
      startDay = day
      currentSpan = 1
    }

    // Если последний день месяца
    if (day === daysInMonth.value) {
      merged.push({
        startDay,
        status: currentStatus,
        hours: currentHours,
        details: currentDetails, // Сохраняем детали
        span: currentSpan
      })
    }
  }

  return merged
}

// Функция для получения цвета статуса
const getStatusColor = (status) => {
  const colors = {
    'vacation': 'primary',
    'dayoff': 'success',
    'remote': 'info',
    'overtime': 'warning'
  }
  return colors[status] || 'grey'
}

// Функция для получения текста статуса
const getStatusText = (status) => {
  const texts = {
    'vacation': 'Отпуск',
    'dayoff': 'Отгул',
    'remote': 'Удаленка',
    'overtime': ' '
  }
  return texts[status] || status
}

// Функция открытия диалога с выбором периода
const openDateDialog = (employee, period) => {
  selectedEmployee.value = employee
  selectedPeriod.value = period
  
  // Устанавливаем даты на основе выбранного периода
  startDate.value = moment(currentDate.value).date(period.startDay).format('YYYY-MM-DD')
  endDate.value = moment(currentDate.value).date(period.startDay + period.span - 1).format('YYYY-MM-DD')
  selectedStatus.value = period.status
  overtimeHours.value = period.hours || ''
  
  // Если это переработка и есть детали, заполняем форму для редактирования
  if (period.status === 'overtime' && period.details) {
    // Здесь можно заполнить дополнительные поля, если нужно
    console.log('Детали переработки:', period.details)
  }
  
  dateError.value = ''
  dateDialog.value = true
}

// Функция обновления периода
const updatePeriod = () => {
  if (isValidSelection.value && selectedEmployee.value && selectedPeriod.value) {
    // Сохраняем детали если это была переработка
    const oldDetails = selectedPeriod.value.details
    
    // Очищаем старый период
    for (let day = selectedPeriod.value.startDay; 
         day < selectedPeriod.value.startDay + selectedPeriod.value.span; 
         day++) {
      const index = selectedEmployee.value.schedule.findIndex(s => s.day === day)
      if (index !== -1) {
        selectedEmployee.value.schedule.splice(index, 1)
      }
    }
    
    // Добавляем новый период
    const start = moment(startDate.value).date()
    const end = moment(endDate.value).date()
    
    for (let day = start; day <= end; day++) {
      const scheduleItem = {
        day: day,
        status: selectedStatus.value
      }
      
      // Добавляем время и детали для переработки
      if (selectedStatus.value === 'overtime') {
        if (overtimeHours.value) {
          scheduleItem.hours = overtimeHours.value
        }
        // Сохраняем старые детали если они были
        if (oldDetails) {
          scheduleItem.details = oldDetails
        }
      }
      
      // Проверяем, не существует ли уже запись для этого дня
      const existingIndex = selectedEmployee.value.schedule.findIndex(s => s.day === day)
      if (existingIndex !== -1) {
        selectedEmployee.value.schedule[existingIndex] = scheduleItem
      } else {
        selectedEmployee.value.schedule.push(scheduleItem)
      }
    }
    
    // Сортируем расписание по дням
    selectedEmployee.value.schedule.sort((a, b) => a.day - b.day)
    
    // Если обновляемый сотрудник - это выбранный для статистики, обновляем его
    if (selectedEmployeeForStats.value && selectedEmployeeForStats.value.id === selectedEmployee.value.id) {
      selectedEmployeeForStats.value = { ...selectedEmployee.value }
    }
    
    // Принудительно обновляем таблицу
    updateTrigger.value++
    
    console.log('Обновлен статус:', selectedStatus.value)
  }
  
  dateDialog.value = false
}

// Функция удаления периода
const deletePeriod = () => {
  if (selectedEmployee.value && selectedPeriod.value) {
    // Удаляем все дни выбранного периода
    for (let day = selectedPeriod.value.startDay; 
         day < selectedPeriod.value.startDay + selectedPeriod.value.span; 
         day++) {
      const index = selectedEmployee.value.schedule.findIndex(s => s.day === day)
      if (index !== -1) {
        selectedEmployee.value.schedule.splice(index, 1)
      }
    }
    
    // Принудительно обновляем таблицу
    updateTrigger.value++
    
    console.log('Удален период для:', selectedEmployee.value.name)
  }
  
  dateDialog.value = false
}

// Функции для диалога переработки
const openOvertimeDialog = () => {
  // Сброс формы
  overtimeForm.value = {
    employeeId: null,
    whatDone: '',
    date: null,
    startTime: '',
    endTime: '',
    event: null,
    workType: null,
    justification: ''
  }
  overtimeError.value = ''
  overtimeDialog.value = true
}

const saveOvertime = () => {
  if (!isOvertimeValid.value) {
    overtimeError.value = 'Заполните все обязательные поля'
    return
  }

  const employee = employees.value.find(e => e.id === overtimeForm.value.employeeId)
  if (!employee) return

  const date = moment(overtimeForm.value.date).date()
  
  // Рассчитываем продолжительность в часах и минутах
  const start = moment(overtimeForm.value.startTime, 'HH:mm')
  const end = moment(overtimeForm.value.endTime, 'HH:mm')
  const duration = moment.duration(end.diff(start))
  const hours = Math.floor(duration.asHours())
  const minutes = duration.minutes()
  
  const hoursText = hours > 0 ? `${hours}ч` : ''
  const minutesText = minutes > 0 ? `${minutes}м` : ''
  const durationText = [hoursText, minutesText].filter(Boolean).join(' ')
  
  // Проверяем, не существует ли уже запись для этого дня
  const existingIndex = employee.schedule.findIndex(s => s.day === date)
  
  const scheduleItem = {
    day: date,
    status: 'overtime',
    hours: durationText || '0ч',
    details: {
      whatDone: overtimeForm.value.whatDone,
      date: overtimeForm.value.date,
      startTime: overtimeForm.value.startTime,
      endTime: overtimeForm.value.endTime,
      event: overtimeForm.value.event,
      workType: overtimeForm.value.workType,
      justification: overtimeForm.value.justification
    }
  }
  
  if (existingIndex !== -1) {
    employee.schedule[existingIndex] = scheduleItem
  } else {
    employee.schedule.push(scheduleItem)
  }
  
  // Сортируем расписание по дням
  employee.schedule.sort((a, b) => a.day - b.day)
  
  // Если выбранный для статистики сотрудник - тот, кому добавили переработку, обновляем статистику
  if (selectedEmployeeForStats.value && selectedEmployeeForStats.value.id === employee.id) {
    // Создаем новый объект, чтобы обновить ссылку и вызвать пересчет computed свойств
    selectedEmployeeForStats.value = { ...employee }
  }
  
  // Принудительно обновляем таблицу
  updateTrigger.value++
  
  overtimeDialog.value = false
  console.log('Добавлена переработка:', overtimeForm.value)
}


// Функции для диалога удаленки
const openRemoteDialog = () => {
  // Сброс формы
  remoteForm.value = {
    employeeId: null,
    dates: []
  }
  remoteError.value = ''
  remoteDialog.value = true
}

const saveRemote = () => {
  if (!isRemoteValid.value) {
    remoteError.value = 'Выберите сотрудника и хотя бы один день'
    return
  }

  const employee = employees.value.find(e => e.id === remoteForm.value.employeeId)
  if (!employee) return

  // Добавляем каждый выбранный день
  remoteForm.value.dates.forEach(date => {
    const day = moment(date).date()
    
    // Проверяем, не существует ли уже запись для этого дня
    const existingIndex = employee.schedule.findIndex(s => s.day === day)
    
    const scheduleItem = {
      day: day,
      status: 'remote'
    }
    
    if (existingIndex !== -1) {
      employee.schedule[existingIndex] = scheduleItem
    } else {
      employee.schedule.push(scheduleItem)
    }
  })
  
  // Сортируем расписание по дням
  employee.schedule.sort((a, b) => a.day - b.day)
  
  // Принудительно обновляем таблицу
  updateTrigger.value++
  
  remoteDialog.value = false
  console.log('Добавлена удаленка:', remoteForm.value)
}
const selectedEmployeeForStats = ref(null)

// Функция выбора сотрудника для статистики
const selectEmployeeForStats = (employee) => {
  selectedEmployeeForStats.value = employee
}

// Вычисляемые свойства для статистики
const vacationDaysLeft = computed(() => {
  if (!selectedEmployeeForStats.value) return 0
  
  // Считаем использованные отпускные дни за месяц
  const usedVacationDays = selectedEmployeeForStats.value.schedule.filter(
    item => item.status === 'vacation'
  ).length
  
  // Баланс отпускных дней (28 - использовано)
  return Math.max(0, 28 - usedVacationDays)
})

const totalOvertimeHours = computed(() => {
  if (!selectedEmployeeForStats.value) return '0ч'
  
  const overtimeItems = selectedEmployeeForStats.value.schedule.filter(
    item => item.status === 'overtime' && item.hours
  )
  
  let totalMinutes = 0
  
  overtimeItems.forEach(item => {
    if (item.hours) {
      // Парсим строку типа "2ч 30м" или "1ч 45м"
      const hoursMatch = item.hours.match(/(\d+)ч/)
      const minutesMatch = item.hours.match(/(\d+)м/)
      
      if (hoursMatch) totalMinutes += parseInt(hoursMatch[1]) * 60
      if (minutesMatch) totalMinutes += parseInt(minutesMatch[1])
    }
  })
  
  const hours = Math.floor(totalMinutes / 60)
  const minutes = totalMinutes % 60
  
  return minutes > 0 ? `${hours}ч ${minutes}м` : `${hours}ч`
})

const weekendDays = computed(() => {
  if (!selectedEmployeeForStats.value) return 0
  
  // Получаем все субботы и воскресенья в текущем месяце
  const weekends = []
  const year = currentYear.value
  const month = currentMonth.value
  
  for (let day = 1; day <= daysInMonth.value; day++) {
    const date = new Date(year, month, day)
    const dayOfWeek = date.getDay()
    // 0 - воскресенье, 6 - суббота
    if (dayOfWeek === 0 || dayOfWeek === 6) {
      weekends.push(day)
    }
  }
  
  return weekends.length
})

const weekdayDays = computed(() => {
  if (!selectedEmployeeForStats.value) return 0
  
  // Получаем все рабочие дни (пн-пт) в текущем месяце
  let weekdays = 0
  const year = currentYear.value
  const month = currentMonth.value
  
  for (let day = 1; day <= daysInMonth.value; day++) {
    const date = new Date(year, month, day)
    const dayOfWeek = date.getDay()
    // 1-5 - понедельник-пятница
    if (dayOfWeek >= 1 && dayOfWeek <= 5) {
      weekdays++
    }
  }
  
  return weekdays
})

// Статистика по переработкам для таблицы
const overtimeStats = computed(() => {
  // Если сотрудник не выбран, показываем пустой массив
  if (!selectedEmployeeForStats.value) {
    return []
  }
  
  const stats = []
  const employee = selectedEmployeeForStats.value
  
  // Собираем только переработки выбранного сотрудника
  employee.schedule
    .filter(item => item.status === 'overtime' && item.details)
    .forEach(item => {
      stats.push({
        employeeId: employee.id,
        ...item
      })
    })
  
  // Сортируем по дате (от новых к старым)
  return stats.sort((a, b) => {
    return moment(b.details.date).valueOf() - moment(a.details.date).valueOf()
  })
})

// Вспомогательные функции
const getEmployeeName = (employeeId) => {
  const employee = employees.value.find(e => e.id === employeeId)
  return employee ? employee.name : 'Неизвестно'
}

const getEventName = (eventId) => {
  const event = mockEvents.find(e => e.value === eventId)
  return event ? event.title : 'Неизвестно'
}

const getWorkTypeName = (workType) => {
  const type = workTypes.find(t => t.value === workType)
  return type ? type.title : 'Неизвестно'
}

const formatDate = (date) => {
  return date ? moment(date).format('DD.MM.YYYY') : ''
}
// Вотчеры для отслеживания выбора всех элементов
watch(
  () => filters.value.selected.departments,
  (newVal) => {
    filters.value.selectAll.departments = newVal.length === filters.value.value.departments.length
  },
  { deep: true }
)

watch(
  () => filters.value.selected.employees,
  (newVal) => {
    filters.value.selectAll.employees = newVal.length === filteredEmployeeOptions.value.length
  },
  { deep: true }
)

// Вотчер для обновления списка сотрудников при изменении фильтра по отделам
watch(
  () => filters.value.selected.departments,
  () => {
    // Если выбранные сотрудники не входят в выбранные отделы, убираем их
    if (filters.value.selected.departments.length > 0) {
      filters.value.selected.employees = filters.value.selected.employees.filter(empId => {
        const employee = employees.value.find(e => e.id === empId)
        return employee && filters.value.selected.departments.includes(employee.departmentId)
      })
    }
  }
)

watch(selectedEmployeeForStats, () => {
  // Просто триггерим обновление computed свойства
  updateTrigger.value++
}, { deep: true })
</script>

<style lang="sass" scoped>
.employee-schedule
  padding: 1rem

.panel
  margin-bottom: 1rem

.filters
  display: grid
  grid-template-columns: 1fr 1fr
  gap: 1rem
  margin-bottom: 1rem

.action-buttons
  display: flex
  gap: 1rem
  justify-content: flex-start
  flex-wrap: wrap

.schedule-table
  border: 1px solid #ddd
  border-radius: 4px
  overflow: hidden
  
  th
    background-color: #1976D2
    color: white
    font-weight: 500
    padding: 12px 8px
    text-align: center
    
  td
    padding: 8px
    border: 1px solid #ddd
    
  .employee-column
    min-width: 200px
    position: sticky
    left: 0
    background-color: #1976D2
    z-index: 2

  .department-column
    min-width: 150px
    background-color: #1976D2
    z-index: 1
    
  .day-column
    min-width: 40px
    font-size: 0.9rem
    
  .employee-name
    font-weight: 500
    background-color: #f5f5f5
    position: sticky
    left: 0
    z-index: 1

  .department-name
    background-color: #f5f5f5
    
  .schedule-cell
    padding: 4px
    text-align: center
    
    &.clickable
      cursor: pointer
      transition: background-color 0.2s
      
      &:hover
        background-color: #e3f2fd
      
    .status-chip
      width: 100%
      justify-content: center
      font-size: 0.8rem
      
    .empty-cell
      color: #999
      font-size: 0.8rem
      cursor: default

.date-title
  display: flex
  align-items: center
  width: 100%
  padding: 0.5rem 1rem

.bg-warning
  background-color: #fb8c00 !important

.bg-info
  background-color: #2196F3 !important

.text-white
  color: white !important

#app .day-column
  text-align: center
  border-left: 1px white solid
  border-right: 1px white solid

.stats-card
  .stat-item
    border-radius: 8px
    background-color: #f8f9fa
    transition: all 0.3s ease
    
    &:hover
      transform: translateY(-2px)
      box-shadow: 0 4px 8px rgba(0,0,0,0.1)

  .text-h6
    font-size: 1.5rem
    font-weight: 600
    color: #1976D2

  .text-subtitle-2
    color: #666
    font-weight: 500

  .text-caption
    color: #999
    
.employee-schedule
  padding: 1rem

// Добавляем стили для переключателя
.v-btn-toggle
  border: 1px solid #ddd
  border-radius: 4px
  
  .v-btn
    border-radius: 0
    border-right: 1px solid #ddd
    
    &:last-child
      border-right: none

</style>