<template>
  <div class="employee-schedule">
    <!-- Прогресс-бар загрузки -->
    <v-overlay v-model="isLoading" class="loading-overlay">
      <v-card class="loading-card pa-6 text-center">
        <v-progress-circular
          :size="80"
          :width="8"
          color="primary"
          indeterminate
          class="mb-4"
        ></v-progress-circular>
        <h3 class="mb-2">Загрузка данных</h3>
        <p class="text-subtitle-1 mb-4">Пожалуйста, подождите...</p>
        <v-progress-linear
          v-model="loadingProgress"
          color="primary"
          height="8"
          rounded
          class="loading-progress"
        ></v-progress-linear>
        <p class="text-caption mt-2">{{ loadingProgress }}% завершено</p>
      </v-card>
    </v-overlay>

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

            <v-autocomplete
              v-model="filters.selected.absenceTypes"
              :items="absenceTypeOptions"
              item-title="text"
              item-value="value"
              label="Тип отсутствия"
              single-line
              hide-details
              variant="outlined"
              multiple
              chips
              clearable
              prepend-inner-icon="mdi-calendar-blank"
            >
              <template v-slot:prepend-item>
                <v-list-item>
                  <v-list-item-content>
                    <v-list-item-title>
                      <v-checkbox 
                        label="Выбрать все типы" 
                        v-model="filters.selectAll.absenceTypes" 
                        @change="() => toggleSelectAll('absenceTypes')"
                        :disabled="absenceTypeOptions.length === 0"
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
        color="grey" 
        :prepend-icon="isManager ? 'mdi-eye' : 'mdi-clock-plus'"
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
        color="orange"
        :prepend-icon="isManager ? 'mdi-account-multiple-plus' : 'mdi-calendar-clock'"
        @click="openDayoffDialog"
        :class="{ 'manager-btn': isManager }"
      >
        Добавить отгул
      </v-btn>
      <v-btn
        color="success"
        :prepend-icon="isManager ? 'mdi-account-multiple-plus' : 'mdi-beach'"
        @click="openVacationDialog"
        :class="{ 'manager-btn': isManager }"
      >
        Добавить отпуск
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
    <div class="table-container">
      <v-overlay v-model="isTableLoading" class="table-overlay">
        <v-progress-circular
          size="40"
          color="primary"
          indeterminate
        ></v-progress-circular>
      </v-overlay>
      <v-table class="schedule-table mb-6" :class="{ 'year-view': viewMode === 'year' }">
      <thead>
        <tr>
          <th class="employee-column">Сотрудник</th>
          <th class="department-column">Отдел</th>
          <template v-if="viewMode === 'year'">
            <!-- Для года - двойная шапка -->
            <th 
              v-for="month in months"
              :key="'month-' + month.number"
              :colspan="daysInYearMonth(month.number)"
              class="month-header"
              :class="{ 'current-month': isCurrentMonth(month.number) }"
              :ref="el => setMonthRef(month.number, el)"
            >
              {{ month.name }}
            </th>
          </template>
          <template v-else>
            <!-- Для других режимов - обычная шапка -->
            <th 
              v-for="period in periods" 
              :key="period.key"
              class="day-column"
              :style="{ minWidth: periodWidth }"
            >
              {{ period.label }}
            </th>
          </template>
        </tr>
        <tr v-if="viewMode === 'year'">
          <th class="employee-column"></th>
          <th class="department-column"></th>
          <th 
            v-for="day in yearDays"
            :key="'day-' + day.key"
            class="day-column year-day"
            :class="{ 
              'weekend-day': isWeekend(day.date),
              'current-day': isCurrentDay(day.date)
            }"
          >
            {{ day.label }}
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
            {{ getDepartmentName(employee.departmentIds) }}
          </td>
          
          <template v-if="viewMode === 'year'">
            <!-- Год: объединение подряд идущих дней с тем же статусом и часами (как в месяце) -->
            <td 
              v-for="(merged, yidx) in getMergedYearDays(employee)"
              :key="'ycell-' + employee.id + '-' + yidx"
              :colspan="merged.span"
              class="schedule-cell year-cell"
              :class="{ 
                'clickable': merged.status,
                'weekend-cell': merged.span === 1 && isWeekend(merged.yearDay.date),
                'current-day-cell': merged.span === 1 && isCurrentDay(merged.yearDay.date)
              }"
              @click="merged.status && openYearMergedDialog(employee, merged)"
            >
              <v-chip
                v-if="merged.status"
                :color="getStatusColor(merged.status)"
                size="x-small"
                class="status-chip year-chip"
              >
                {{ getYearStatusText(merged.status) }}
              </v-chip>
              <span v-else class="empty-cell">—</span>
            </td>
          </template>
          <!-- Ячейки в зависимости от режима -->
          <template v-else-if="viewMode === 'month'">
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
          
          <template v-else-if="viewMode === 'week'">
            <!-- Неделя: объединение слотов как в месяце -->
            <td 
              v-for="(merged, widx) in getMergedWeekPeriods(employee)" 
              :key="'w-' + employee.id + '-' + widx"
              :colspan="merged.span"
              class="schedule-cell"
              :class="{ 'clickable': merged.status }"
              @click="merged.status && openDateDialog(employee, merged)"
            >
              <v-chip
                v-if="merged.status"
                :color="getStatusColor(merged.status)"
                size="small"
                class="status-chip"
              >
                {{ getStatusText(merged.status) }}
                <span v-if="merged.hours" class="ml-1">{{ merged.hours }}</span>
              </v-chip>
              <span v-else class="empty-cell">—</span>
            </td>
          </template>
          <template v-else>
            <!-- День -->
            <td 
              v-for="period in periods" 
              :key="period.key"
              class="schedule-cell"
              :class="{ 'clickable': getStatusForPeriod(employee, period) }"
              @click="getStatusForPeriod(employee, period) && openPeriodDialog(employee, period)"
            >
              <v-chip
                v-if="getStatusForPeriod(employee, period)"
                :color="getStatusColor(getStatusForPeriod(employee, period))"
                size="small"
                class="status-chip"
              >
                {{ getStatusText(getStatusForPeriod(employee, period)) }}
                <span v-if="getHoursForPeriod(employee, period)" class="ml-1 font-weight-bold">
                  {{ getHoursForPeriod(employee, period) }}
                </span>
              </v-chip>
              <span v-else class="empty-cell">—</span>
            </td>
          </template>
        </tr>
        <tr v-if="filteredEmployees.length === 0">
          <td :colspan="totalColumns" class="text-center pa-4">
            <v-icon icon="mdi-alert" size="large" color="warning" class="mb-2"></v-icon>
            <p>Нет сотрудников, соответствующих выбранным фильтрам</p>
          </td>
        </tr>
      </tbody>
    </v-table>
    </div>

    <div class="stats-container" v-if="selectedEmployeeForStats">
    <!-- Блок статистики для выбранного пользователя -->
    <v-card class="mb-4 stats-card">
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
            </v-sheet>
          </v-col>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ totalOvertimeHours }}</div>
              <div class="text-subtitle-2">Переработки всего</div>
            </v-sheet>
          </v-col>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ weekendOvertimeHours }}</div>
              <div class="text-subtitle-2">Переработки в выходные</div>
            </v-sheet>
          </v-col>
          <v-col cols="12" md="2.4">
            <v-sheet class="pa-3 text-center stat-item">
              <div class="text-h6">{{ weekdayOvertimeHours }}</div>
              <div class="text-subtitle-2">Переработки в будние</div>
            </v-sheet>
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>

    <v-card class="mb-4">
  <v-card-title class="bg-primary text-white">
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
          <td>{{ formatTime(item.details.startTime) }}</td>
          <td>{{ formatTime(item.details.endTime) }}</td>
          <td>{{ item.hours }}</td>
          <td>{{ getEventName(item.details.event) }}</td>
          <td>{{ getWorkTypeName(item.details.workType) }}</td>
          <td>{{ item.details.justification }}</td>
        </tr>
        <tr v-if="overtimeStats.length === 0">
          <td colspan="8" class="text-center pa-4">
            <v-icon icon="mdi-information" size="large" color="info" class="mb-2"></v-icon>
            <p>Нет данных о переработках</p>
          </td>
        </tr>
      </tbody>
    </v-table>
  </v-card-text>
</v-card>
  </div>
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
                variant="outlined"
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
                variant="outlined"
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
          <v-btn 
            v-if="canDeletePeriod"
            color="error" 
            variant="text" 
            @click="deletePeriod"
          >
            Удалить
          </v-btn>
          <v-btn color="secondary" variant="text" @click="dateDialog = false">
            Отмена
          </v-btn>
          <v-btn 
            color="primary" 
            @click="updatePeriod" 
            :disabled="!isValidSelection"
          >
            Обновить
          </v-btn>
        </v-card-actions>

      </v-card>
    </v-dialog>

    <!-- Диалог добавления переработки -->
    <v-dialog v-model="overtimeDialog" max-width="600px">
      <v-card>
        <v-card-title class="bg-warning text-white d-flex align-center">
          {{ overtimeForm.id ? 'Редактирование переработки' : 'Добавить переработку' }}
          <v-btn icon @click="overtimeDialog = false" class="ml-auto" color="white" variant="text">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        
        <v-card-text class="pt-4">

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
              <v-textarea
                v-model="overtimeDuration"
                label="5) Длительность"
                variant="outlined"
                required
                readonly
                rows="1"
              ></v-textarea>
              <v-textarea
                v-model="overtimeForm.event"
                label="6) Мероприятия"
                variant="outlined"
                required
                rows="1"
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeForm.workType"
                :items="workTypes"
                label="7) Тип работ"
                variant="outlined"
                rows="1"
                required
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeForm.justification"
                label="8) Обоснование работы во вне рабочее время"
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
            v-if="overtimeForm.id && overtimeForm.employeeId === currentUser?.ID"
            color="error" 
            variant="text" 
            @click="deleteOvertimeItem"
            :disabled="!overtimeForm.id"
          >
            Удалить
          </v-btn>
          <v-btn color="secondary" variant="text" @click="overtimeDialog = false">
            Отмена
          </v-btn>
          <v-btn 
            color="warning" 
            @click="saveOvertime"
            :disabled="!isOvertimeValid"
          >
            Сохранить
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
          <v-btn color="secondary" variant="text" @click="remoteDialog = false">
            Отмена
          </v-btn>
          <v-btn 
            color="info" 
            @click="saveRemote"
            :disabled="!isRemoteValid"
          >
            Сохранить
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Диалог добавления отпуска -->
    <v-dialog v-model="vacationDialog" max-width="500px">
      <v-card>
        <v-card-title class="bg-success text-white d-flex align-center">
          Добавить отпуск
          <v-btn icon @click="vacationDialog = false" class="ml-auto" color="white" variant="text">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>

        <v-card-text class="pt-4">
          <!-- Employee selector for manager -->
          <v-row v-if="isManager">
            <v-col cols="12">
              <v-autocomplete
                v-model="vacationForm.selectedEmployees"
                :items="dialogEmployeeOptions"
                item-title="name"
                item-value="id"
                label="Сотрудник отдела"
                chips
                clearable
                variant="outlined"
                prepend-inner-icon="mdi-account-multiple"
                hint="Оставьте пустым для себя"
                persistent-hint
              >
              </v-autocomplete>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="vacationForm.startDate"
                label="Дата начала"
                type="date"
                :min="minDate"
                :max="maxDate"
                variant="outlined"
                required
              ></v-text-field>
            </v-col>

            <v-col cols="12" sm="6">
              <v-text-field
                v-model="vacationForm.endDate"
                label="Дата окончания"
                type="date"
                :min="vacationForm.startDate || minDate"
                :max="maxDate"
                variant="outlined"
                required
              ></v-text-field>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="vacationForm.additionalInfo"
                label="Дополнительная информация"
                variant="outlined"
                rows="3"
                placeholder="Комментарий, причина и т.д."
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row v-if="vacationError" class="mt-2">
            <v-col cols="12">
              <v-alert type="error" density="compact">
                {{ vacationError }}
              </v-alert>
            </v-col>
          </v-row>
        </v-card-text>

        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="secondary" variant="text" @click="vacationDialog = false">
            Отмена
          </v-btn>
          <v-btn
            color="success"
            @click="saveVacation"
            :disabled="!isVacationValid"
          >
            Сохранить
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Диалог просмотра переработки (для руководителя) -->
    <v-dialog v-model="overtimeViewDialog" max-width="700px" persistent>
      <v-card>
        <v-card-title class="bg-info text-white d-flex align-center">
          Просмотр переработки
          <v-btn icon @click="overtimeViewDialog = false" class="ml-auto" color="white" variant="text">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        
        <v-card-text class="pt-4">
          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeViewForm.whatDone"
                label="1) Что сделано"
                variant="outlined"
                rows="3"
                readonly
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-text-field
                v-model="overtimeViewForm.date"
                label="2) День"
                type="date"
                readonly
                variant="outlined"
              ></v-text-field>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="overtimeViewForm.startTime"
                label="3) Время начала работ"
                readonly
                variant="outlined"
              ></v-text-field>
            </v-col>
            
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="overtimeViewForm.endTime"
                label="4) Время завершения работ"
                readonly
                variant="outlined"
              ></v-text-field>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeViewDuration"
                label="5) Длительность"
                variant="outlined"
                readonly
                rows="1"
              ></v-textarea>
              <v-textarea
                v-model="overtimeViewForm.event"
                label="6) Мероприятия"
                variant="outlined"
                readonly
                rows="1"
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeViewForm.workType"
                label="7) Тип работ"
                variant="outlined"
                readonly
                rows="1"
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="overtimeViewForm.justification"
                label="8) Обоснование работы во вне рабочее время"
                variant="outlined"
                readonly
                rows="2"
              ></v-textarea>
            </v-col>
          </v-row>
        </v-card-text>
        
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="secondary" @click="overtimeViewDialog = false">
            Закрыть
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Диалог добавления отгула -->
    <v-dialog v-model="dayoffDialog" max-width="500px">
      <v-card>
        <v-card-title class="bg-orange text-white d-flex align-center">
          Добавить отгул
          <v-btn icon @click="dayoffDialog = false" class="ml-auto" color="white" variant="text">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>

        <v-card-text class="pt-4">
          <!-- Employee selector for manager -->
          <v-row v-if="isManager">
            <v-col cols="12">
              <v-autocomplete
                v-model="dayoffForm.selectedEmployees"
                :items="dialogEmployeeOptions"
                item-title="name"
                item-value="id"
                label="Сотрудники отдела (можно выбрать несколько)"
                multiple
                chips
                clearable
                variant="outlined"
                prepend-inner-icon="mdi-account-multiple"
                hint="Оставьте пустым для себя"
                persistent-hint
              >
                <template v-slot:prepend-item>
                  <v-list-item>
                    <v-list-item-title>
                      <v-checkbox
                        label="Все в отделе"
                        :model-value="dayoffForm.selectedEmployees.length === dialogEmployeeOptions.length"
                        @click="toggleAllDialogEmployees('dayoff')"
                      />
                    </v-list-item-title>
                  </v-list-item>
                </template>
              </v-autocomplete>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12" sm="6">
              <v-text-field
                v-model="dayoffForm.startDate"
                label="Дата начала"
                type="date"
                :min="minDate"
                :max="maxDate"
                variant="outlined"
                required
              ></v-text-field>
            </v-col>

            <v-col cols="12" sm="6">
              <v-text-field
                v-model="dayoffForm.endDate"
                label="Дата окончания"
                type="date"
                :min="dayoffForm.startDate || minDate"
                :max="maxDate"
                variant="outlined"
                required
              ></v-text-field>
            </v-col>
          </v-row>

          <v-row>
            <v-col cols="12">
              <v-textarea
                v-model="dayoffForm.additionalInfo"
                label="Дополнительная информация"
                variant="outlined"
                rows="3"
                placeholder="Комментарий, причина и т.д."
              ></v-textarea>
            </v-col>
          </v-row>

          <v-row v-if="dayoffError" class="mt-2">
            <v-col cols="12">
              <v-alert type="error" density="compact">
                {{ dayoffError }}
              </v-alert>
            </v-col>
          </v-row>
        </v-card-text>

        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="secondary" variant="text" @click="dayoffDialog = false">
            Отмена
          </v-btn>
          <v-btn
            color="orange"
            @click="saveDayoff"
            :disabled="!isDayoffValid"
          >
            Сохранить
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>
<script setup>
import { ref, computed, watch, nextTick } from 'vue'
import moment from 'moment'
import 'moment/dist/locale/ru'
moment.locale('ru')
import * as XLSX from 'xlsx'

import { callApi } from '../functions/callApi'

const monthRefs = ref({})
const yearViewContainer = ref(null)
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
const overtimeViewDialog = ref(false)
const remoteDialog = ref(false)
const vacationDialog = ref(false)
const dayoffDialog = ref(false)
const overtimeError = ref('')
const overtimeViewForm = ref({
  id: null,
  whatDone: '',
  date: '',
  startTime: '',
  endTime: '',
  event: '',
  workType: '',
  justification: ''
})
const remoteError = ref('')
const vacationError = ref('')
const dayoffError = ref('')

// Форма для переработки (readonly copy for view)
const overtimeViewDuration = computed(() => {
  if (!overtimeViewForm.value.startTime || !overtimeViewForm.value.endTime) {
    return 'Не указано'
  }
  const start = moment(overtimeViewForm.value.startTime, 'HH:mm')
  const end = moment(overtimeViewForm.value.endTime, 'HH:mm')
  if (!start.isValid() || !end.isValid()) return 'Некорректное время'
  const duration = moment.duration(end.diff(start))
  const hours = Math.floor(duration.asHours())
  const minutes = duration.minutes()
  let result = ''
  if (hours > 0) result += `${hours} ч`
  if (minutes > 0) { if (result) result += ' '; result += `${minutes} мин` }
  return result || '0 минут'
})

// Форма для редактирования переработки
const overtimeForm = ref({
  id: null, // ID элемента для редактирования
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
  dates: []
})

// Форма для отпуска
const vacationForm = ref({
  startDate: null,
  endDate: null,
  additionalInfo: '',
  selectedEmployees: []
})

// Форма для отгула
const dayoffForm = ref({
  startDate: null,
  endDate: null,
  additionalInfo: '',
  selectedEmployees: []
})

function stringifyDeptId(id) {
  if (id == null || id === '') return ''
  return String(id)
}

/** Отделы из `rootIds` и все дочерние по цепочке PARENT (department.get). */
function expandDepartmentsWithChildren(rootIds, deptList) {
  const managed = new Set(
    rootIds.map(stringifyDeptId).filter((id) => id && id !== '0')
  )
  if (managed.size === 0) return []
  let changed = true
  while (changed) {
    changed = false
    for (const dept of deptList) {
      const id = stringifyDeptId(dept.ID)
      if (!id || managed.has(id)) continue
      const parent = stringifyDeptId(dept.PARENT)
      if (!parent || parent === '0') continue
      if (managed.has(parent)) {
        managed.add(id)
        changed = true
      }
    }
  }
  return Array.from(managed)
}

/** Отделы, где текущий пользователь указан как UF_HEAD (руководитель отдела). */
const headedDepartmentIds = computed(() => {
  if (!currentUser.value || !departments.value.length) return []
  const uid = stringifyDeptId(currentUser.value.ID)
  return departments.value
    .filter((d) => stringifyDeptId(d.UF_HEAD) === uid)
    .map((d) => stringifyDeptId(d.ID))
    .filter(Boolean)
})

// Manager department IDs: профиль + отделы руководства + все дочерние (PARENT)
const managerDepartments = computed(() => {
  if (!currentUser.value) return []
  const managerEmp = employees.value.find((emp) => emp.id === currentUser.value.ID)
  const fromProfile = managerEmp ? managerEmp.departmentIds.map(stringifyDeptId).filter(Boolean) : []
  const seeds = [...new Set([...fromProfile, ...headedDepartmentIds.value])]
  return expandDepartmentsWithChildren(seeds, departments.value)
})

// Computed: dialog employee options (manager's depts only)
const dialogEmployeeOptions = computed(() => {
  if (!isManager.value) return []
  return employees.value
    .filter(emp => managerDepartments.value.some(dept => emp.departmentIds.includes(dept)))
    .map(emp => ({ id: emp.id, name: emp.name }))
})

// Данные мероприятий из CRM
const events = ref([])

const workTypes = [
  { title: 'Разработка', value: 'development' },
  { title: 'Тестирование', value: 'testing' },
  { title: 'Аналитика', value: 'analytics' },
  { title: 'Дизайн', value: 'design' },
  { title: 'Документация', value: 'documentation' },
]

// Моковые данные для отделов (оставляем пока мок, но можно будет заменить на реальные)
const departments = ref([])

// Триггер для обновления таблицы
const updateTrigger = ref(0)

// Опции статусов
const statusOptions = [
  { text: 'Отпуск', value: 'vacation' },
  { text: 'Отгул', value: 'dayoff' },
  { text: 'Удаленка', value: 'remote' },
  { text: 'Переработка', value: 'overtime' }
]

const absenceTypeOptions = [
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
    employees: [],
    absenceTypes: []
  },
  selectAll: {
    departments: false,
    employees: false,
    absenceTypes: false
  }
})

// Состояние загрузки
const isLoading = ref(true)
const loadingProgress = ref(0)
const isTableLoading = ref(false)

// Текущий пользователь
const currentUser = ref(null)

// Manager detection - check position OR is department head (UF_HEAD)
const isManager = computed(() => {
  if (!currentUser.value) return false
  
  const hasManagerPosition = currentUser.value.workPosition?.toLowerCase().includes('руководитель')
  if (hasManagerPosition) return true
  
  // Check if heads any department
  const currentUserId = String(currentUser.value.ID)
  console.log(departments.value.some(dept => String(dept.UF_HEAD) == currentUserId));
  return departments.value.some(dept => String(dept.UF_HEAD) == currentUserId)
})


// Переменная для поиска по сотруднику
const searchEmployee = ref('')

// Реальные данные
const employees = ref([])
const schedule = ref([])

const absenceMapping = {
  10372: 'vacation',     // Отпуск
  10374: 'dayoff',       // Отгул
  11602: 'remote',       // Удаленка
  11604: 'overtime'      // Переработка
}

const absenceTypeToId = {
  'vacation': 10372,
  'dayoff': 10374,
  'remote': 11602,
  'overtime': 11604
}

// Функция загрузки данных
async function loadData() {
  try {
    // Если это не первая загрузка, показываем загрузку таблицы
    if (!isLoading.value) {
      isTableLoading.value = true;
    } else {
      loadingProgress.value = 0;
    }

    // Этап 1: Загрузка отделов (20%)
    loadingProgress.value = 20;
    departments.value = await callApi("department.get", {}, []);
    console.log(departments.value);

    // Обновляем отделы в фильтрах
    filters.value.value.departments = departments.value;

    // Загружаем мероприятия
    events.value = await callApi("crm.item.list", {}, null, 1052, null, null);
    console.log('Загружены мероприятия:', events.value);

    // Этап 2: Загрузка текущего пользователя и сотрудников (40%)
    loadingProgress.value = 40;
    // Загружаем текущего пользователя
    try {
      currentUser.value = await callApi("user.current", {}, [], null, null, null);
      console.log('Текущий пользователь:', currentUser.value);
    } catch (error) {
      console.warn('Не удалось загрузить текущего пользователя:', error);
    }

    const usersResponse = await callApi("user.get", {}, [], null, null, null);
    employees.value = usersResponse.map(user => ({
      id: user.ID,
      name: `${user.NAME || ''} ${user.LAST_NAME || ''} ${user.SECOND_NAME || ''}`.trim(),
      departmentIds: Array.isArray(user.UF_DEPARTMENT) ? user.UF_DEPARTMENT.map(id => String(id)) : (user.UF_DEPARTMENT ? [String(user.UF_DEPARTMENT)] : []),
      workPosition: user.WORK_POSITION || '',
      schedule: []
    }));
    
    console.log('Загружены сотрудники:', employees.value);

    // Этап 3: Загрузка графика (60%)
    loadingProgress.value = 60;
    const scheduleResponse = await callApi("crm.item.list", {}, null, 1048, null, null);
    console.log('Загружен график:', scheduleResponse);

    // Фильтруем график по стадиям: отгул и отпуск должны иметь определенные стадии
    const filteredScheduleResponse = scheduleResponse.filter(item => {
      const absenceTypeId = item.ufCrm36_1737068344170;
      const status = absenceMapping[absenceTypeId] || null;

      // Если статус не определен, пропускаем
      if (!status) return false;

      // Стадии, которые ДОЛЖНЫ быть у отпуска и отгула
      const allowedStages = ["DT1048_108:SUCCESS"];

      // Если это отпуск или отгул
      if (status === 'vacation' || status === 'dayoff') {
        // Оставляем только те, которые находятся в разрешенных стадиях
        return allowedStages.includes(item.stageId);
      }

      // Для остальных статусов (удаленка, переработка) оставляем все
      return true;
    });

    // Преобразуем данные графика
    schedule.value = filteredScheduleResponse.map(item => {
      const absenceTypeId = item.ufCrm36_1737068344170;
      const status = absenceMapping[absenceTypeId] || null;
      
      // Проверяем новый формат удаленки с массивом дат
      const remoteDates = item.ufCrm36_1773132483885;
      
      if (status === 'remote' && remoteDates && Array.isArray(remoteDates) && remoteDates.length > 0) {
        // Новый формат: массив дат в поле ufCrm36_1773132483885
        // Делим даты на слоты (подряд идущие даты)
        const sortedDates = [...remoteDates].sort();
        const slots = [];
        let currentSlot = [sortedDates[0]];
        
        for (let i = 1; i < sortedDates.length; i++) {
          const prevDate = moment(sortedDates[i - 1]);
          const currDate = moment(sortedDates[i]);
          const diff = currDate.diff(prevDate, 'days');
          
          if (diff === 1) {
            // Даты идут подряд - добавляем к текущему слоту
            currentSlot.push(sortedDates[i]);
          } else {
            // Начался новый слот
            slots.push(currentSlot);
            currentSlot = [sortedDates[i]];
          }
        }
        // Добавляем последний слот
        slots.push(currentSlot);
        
        // Создаем отдельные записи для каждого слота
        return slots.map((slotDates) => {
          const startDate = moment(slotDates[0]);
          const endDate = moment(slotDates[slotDates.length - 1]);

          return {
            employeeId: String(item.assignedById),
            status: status,
            startDate: startDate.format('YYYY-MM-DD'),
            endDate: endDate.format('YYYY-MM-DD'),
            days: getDaysArray(startDate, endDate),
            details: {
              id: item.id,
              remoteDates: slotDates, // даты только этого слота (подряд)
              allRemoteDates: sortedDates // полный массив CRM — для частичного удаления слота
            }
          };
        });
      } else {
        // Старый формат или другие статусы: даты в полях ufCrm36_1737068421810 и ufCrm36_1737068451414
        const startDate = moment(item.ufCrm36_1737068421810);
        let endDate = moment(item.ufCrm36_1737068451414);
        // Конец периода часто не заполняют (переработка за один день) — иначе moment(null) невалиден и days = []
        if (!item.ufCrm36_1737068451414 || !endDate.isValid()) {
          endDate = startDate.clone();
        }

        return {
          employeeId: String(item.assignedById),
          status: status,
          startDate: startDate.format('YYYY-MM-DD'),
          endDate: endDate.format('YYYY-MM-DD'),
          days: getDaysArray(startDate, endDate),
          details: {
            id: item.id,
            whatDone: item.ufCrm36_1772712553,
            event: item.ufCrm36_1773312024, //ufCrm36_1772712787
            workType: item.ufCrm36_1772712618,
            justification: item.ufCrm36_1772712662,
            startTime: item.ufCrm36_1772712459,
            endTime: item.ufCrm36_1772712490,
            date: item.ufCrm36_1737068421810,
            additionalInfo: item.ufCrm36_1737068503448 || ''
          }
        };
      }
    }).flat().filter(item => item !== null);

    // Этап 4: Загрузка графика (80%)
    loadingProgress.value = 80;
    // Весь год выбранной даты: и месячный/недельный вид берут нужный период из schedule, годовой — все месяцы
    const rangeStart = moment(currentDate.value).startOf('year');
    const rangeEnd = moment(currentDate.value).endOf('year');

    const filteredSchedule = schedule.value.filter(scheduleItem => {
      const scheduleStart = moment(scheduleItem.startDate);
      const scheduleEnd = moment(scheduleItem.endDate);
      return scheduleStart.isSameOrBefore(rangeEnd) && scheduleEnd.isSameOrAfter(rangeStart);
    });

    // Объединяем отфильтрованный график с сотрудниками
    employees.value = employees.value.map(employee => {
      const employeeSchedule = [];

      filteredSchedule.forEach(scheduleItem => {
        if (scheduleItem.employeeId === employee.id) {
          scheduleItem.days.forEach(day => {
            const dayOfMonth = day.date();
            const monthOfYear = day.month() + 1; // месяцы в moment начинаются с 0
            const yearOfSchedule = day.year();
            // Проверяем, что день относится к выбранному году в календаре
            if (Number(yearOfSchedule) === Number(currentYear.value)) {
              employeeSchedule.push({
                day: Number(dayOfMonth),
                month: Number(monthOfYear),
                year: Number(yearOfSchedule),
                status: scheduleItem.status,
                hours: scheduleItem.status === 'overtime' ? calculateHours(scheduleItem.details.startTime, scheduleItem.details.endTime) : null,
                details: scheduleItem.details
              });
            }
          });
        }
      });

      return {
        ...employee,
        schedule: employeeSchedule.sort((a, b) => a.month - b.month || a.day - b.day)
      };
    });

    console.log('Итоговые данные сотрудников:', employees.value);
    
    // Этап 3: Обработка данных (80%)
    loadingProgress.value = 80;

    // Обновляем список сотрудников в фильтрах
    filters.value.value.employees = employees.value.map(emp => ({
      id: emp.id,
      name: emp.name,
      departmentIds: emp.departmentIds
    }));

    // Этап 4: Завершение (100%)
    loadingProgress.value = 100;

    // Небольшая задержка для показа 100%
    setTimeout(() => {
      isLoading.value = false;
      isTableLoading.value = false;
    }, 100);

  } catch (error) {
    console.error('Ошибка при загрузке данных:', error);
    isLoading.value = false;
    isTableLoading.value = false;
    loadingProgress.value = 0;
  }
}

// Функции для проверки прав на редактирование
function canEditVacationOrDayoff(currentUserId, targetEmployeeId) {
  // Self or manager editing department employee
  if (String(currentUserId) === String(targetEmployeeId)) return true;

  const currentUserEmp = employees.value.find(
    (emp) => String(emp.id) === String(currentUserId)
  );
  if (!currentUserEmp) return false;

  const targetEmp = employees.value.find(
    (emp) => String(emp.id) === String(targetEmployeeId)
  );
  if (!targetEmp) return false;
  
  // Manager can edit if target in manager's departments
  const managerDepts = managerDepartments.value;
  return managerDepts.length > 0 && managerDepts.some(dept => targetEmp.departmentIds.includes(dept));
}

function canEditRemoteOrOvertime(currentUserId, targetEmployeeId, status) {
  const currentUserEmp = employees.value.find(
    (emp) => String(emp.id) === String(currentUserId)
  );
  if (!currentUserEmp) return false;

  // Self always (в т.ч. руководитель добавляет переработку себе)
  if (String(currentUserId) === String(targetEmployeeId)) return true;
  
  // Managers: allow remote but BLOCK overtime
  if (status === 'overtime') return false;
  
  const targetEmp = employees.value.find(
    (emp) => String(emp.id) === String(targetEmployeeId)
  );
  if (!targetEmp) return false;

  const managerDepts = managerDepartments.value;
  return managerDepts.length > 0 && managerDepts.some(dept => targetEmp.departmentIds.includes(dept));
}

// Вспомогательная функция для получения массива дней между датами
function getDaysArray(startDate, endDate) {
  const days = [];
  let currentDate = moment(startDate);
  const lastDate = moment(endDate);
  
  while (currentDate <= lastDate) {
    days.push(moment(currentDate));
    currentDate.add(1, 'day');
  }
  
  return days;
}

/** Число из полей CRM/JSON (строка и число сравниваем одинаково). */
function num(v) {
  const n = Number(v)
  return Number.isFinite(n) ? n : NaN
}

/** Подряд идущие дни с одинаковым статусом склеиваются, кроме переработки (1 запись CRM = 1 ячейка). */
function canMergeAdjacentStatus(status, hours, prevStatus, prevHours) {
  if (status !== prevStatus || hours !== prevHours) return false
  if (status === 'overtime') return false
  return true
}

// Вспомогательная функция для расчета часов переработки
function calculateHours(startDate, endDate) {
  if (!startDate || !endDate) {
    return '0ч';
  }

  const start = moment(startDate);
  const end = moment(endDate);

  if (!start.isValid() || !end.isValid()) {
    return '0ч';
  }

  const duration = moment.duration(end.diff(start));
  const hours = Math.floor(duration.asHours());
  const minutes = duration.minutes();

  const hoursText = hours > 0 ? `${hours}ч` : '';
  const minutesText = minutes > 0 ? `${minutes}м` : '';
  return [hoursText, minutesText].filter(Boolean).join(' ') || '0ч';
}

// Загружаем данные при инициализации
loadData();

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
      return `${start.format('DD MMM')} — ${end.format('DD MMM YYYY')}`
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
      return months.value.map(month => ({
        key: month.number,
        label: month.shortName
      }))
    
    default:
      return []
  }
})

// Ширина колонки в зависимости от режима
const periodWidth = computed(() => {
  switch (viewMode.value) {
    case 'day': return '120px'
    case 'week': return '80px'
    case 'month': return '40px'
    case 'year': return '64px'
    default: return '40px'
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
  // Перезагружаем данные при смене месяца или года (годовой вид — полный год в schedule)
  if (viewMode.value === 'month' || viewMode.value === 'year') {
    loadData();
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
  // Перезагружаем данные при смене месяца или года (годовой вид — полный год в schedule)
  if (viewMode.value === 'month' || viewMode.value === 'year') {
    loadData();
  }
}

const goToCurrentWeek = () => {
  currentDate.value = moment()
}

// Получение статуса для периода
const getStatusForPeriod = (employee, period) => {
  if (viewMode.value === 'year') return null
  if (viewMode.value === 'month') return null

  if (viewMode.value === 'day') {
    const scheduleItem = employee.schedule.find(
      (s) => s.details?.date && moment(s.details.date).isSame(moment(period.key), 'day')
    )
    return scheduleItem?.status || null
  }
  if (viewMode.value === 'week') {
    return scheduleItemForCalendarDate(employee, moment(period.key))?.status || null
  }
  return null
}

const getHoursForPeriod = (employee, period) => {
  // Year: sum overtime hours per month
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
  
  // Day/Week: find by calendar date, extract/calc hours for overtime
  const scheduleItem =
    viewMode.value === 'day'
      ? employee.schedule.find(
          (s) => s.details?.date && moment(s.details.date).isSame(moment(period.key), 'day')
        )
      : scheduleItemForCalendarDate(employee, moment(period.key))
  
  if (scheduleItem?.status !== 'overtime') return null
  
  // Use pre-calculated or calc from details
  if (scheduleItem.hours) return scheduleItem.hours
  
  // Fallback calc from start/end times
  return calculateHours(scheduleItem.details?.startTime, scheduleItem.details?.endTime)
}

// Открытие диалога для периода
const openPeriodDialog = (employee, period) => {
  if (viewMode.value === 'year') return
  
  const pm = moment(period.key)
  const mergedDay = {
    startDay: pm.date(),
    span: 1,
    status: getStatusForPeriod(employee, period),
    hours: getHoursForPeriod(employee, period),
    details:
      viewMode.value === 'day'
        ? employee.schedule.find(
            (s) => s.details?.date && moment(s.details.date).isSame(pm, 'day')
          )?.details
        : scheduleItemForCalendarDate(employee, pm)?.details,
    dateStart: pm.format('YYYY-MM-DD'),
    dateEnd: pm.format('YYYY-MM-DD')
  }
  
  openDateDialog(employee, mergedDay)
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
    const excelData = []

    if (viewMode.value === 'year') {
      // ГОД: Каждый день отдельно (как месяц)
      const headers = ['Сотрудник', 'Отдел']
      yearDays.value.forEach(day => {
        const monthStr = day.month.toString().padStart(2, '0')
        const dayStr = day.day.toString().padStart(2, '0')
        headers.push(`${monthStr}.${dayStr}`)
      })
      excelData.push(headers)

      filteredEmployees.value.forEach(employee => {
        const row = [employee.name, getDepartmentName(employee.departmentIds)]
        
        yearDays.value.forEach(day => {
          const status = getStatusForYearDay(employee, day)
          if (status) {
            const hours = employee.schedule.find(s => s.day === day.day && s.month === day.month)?.hours
            row.push(getStatusText(status) + (hours ? ` (${hours})` : ''))
          } else {
            row.push('—')
          }
        })
        
        excelData.push(row)
      })
      
      // Узкие колонки для года
      const colWidths = [{wch:30}, {wch:20}, ...Array(yearDays.value.length).fill({wch:8})]
      
      const ws = XLSX.utils.aoa_to_sheet(excelData)
      ws['!cols'] = colWidths
      
      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, ws, 'График_год_дни')
      
      const fileName = `график_работы_${currentDate.value.format('YYYY')}_все_дни.xlsx`
      XLSX.writeFile(wb, fileName)
      
    } else {
      // СТАРЫЙ КОД для остальных режимов
      const headers = ['Сотрудник', 'Отдел']
      periods.value.forEach(p => headers.push(p.label))
      excelData.push(headers)
      
      filteredEmployees.value.forEach(employee => {
        const row = [employee.name, getDepartmentName(employee.departmentIds)]
        
        if (viewMode.value === 'month') {
          const y = num(currentYear.value)
          const mo = num(currentMonth.value + 1)
          for (let day = 1; day <= daysInMonth.value; day++) {
            const scheduleItem = employee.schedule.find((s) => {
              if (num(s.day) !== day || num(s.month) !== mo) return false
              const iy =
                s.year === undefined || s.year === null || s.year === '' ? y : num(s.year)
              return iy === y
            })
            if (scheduleItem && scheduleItem.status) {
              const status = getStatusText(scheduleItem.status)
              const hours = scheduleItem.hours ? ` (${scheduleItem.hours})` : ''
              row.push(status + hours)
            } else {
              row.push('—')
            }
          }
        } else {
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
      
      const colWidths = [{wch:30}, {wch:20}, ...Array(periods.value.length).fill({wch:15})]
      
      const ws = XLSX.utils.aoa_to_sheet(excelData)
      ws['!cols'] = colWidths
      
      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, ws, 'График работы')
      
      const fileName = `график_работы_${periodTitle.value.replace(/\s+/g, '_')}.xlsx`
      XLSX.writeFile(wb, fileName)
    }
    
    console.log('Файл успешно выгружен')
  } catch (error) {
    console.error('Ошибка при выгрузке в Excel:', error)
    alert('Произошла ошибка при выгрузке в Excel')
  }
}

// Отфильтрованные сотрудники
const filteredEmployees = computed(() => {
  return employees.value.filter(employee => {
    // Фильтр по отделам
    if (filters.value.selected.departments.length > 0) {
      const hasMatchingDepartment = filters.value.selected.departments.some(selectedDept =>
        employee.departmentIds.includes(selectedDept)
      )
      if (!hasMatchingDepartment) {
        return false
      }
    }

    // Фильтр по ответственным
    if (filters.value.selected.employees.length > 0) {
      if (!filters.value.selected.employees.includes(employee.id)) {
        return false
      }
    }

    // Фильтр по поисковому запросу
    if (searchEmployee.value.trim()) {
      const searchTerm = searchEmployee.value.toLowerCase().trim()
      if (!employee.name.toLowerCase().includes(searchTerm)) {
        return false
      }
    }

    // Скрываем сотрудников без графика
    if (employee.schedule.length === 0) {
      return false
    }

    return true
  }).map(employee => {
    // Если фильтр по типам отсутствия не активен, возвращаем сотрудника без изменений
    if (filters.value.selected.absenceTypes.length === 0) {
      return employee
    }

    // Создаем копию сотрудника с отфильтрованным расписанием
    const filteredSchedule = employee.schedule.filter(item =>
      filters.value.selected.absenceTypes.includes(item.status)
    )

    return {
      ...employee,
      schedule: filteredSchedule
    }
  }).filter(employee => {
    // Если фильтр по типам отсутствия активен, оставляем только сотрудников
    // у которых есть записи после фильтрации
    if (filters.value.selected.absenceTypes.length > 0) {
      return employee.schedule.length > 0
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
      .filter(emp => {
        return filters.value.selected.departments.some(selectedDept =>
          emp.departmentIds.includes(selectedDept)
        )
      })
      .map(emp => ({ id: emp.id, name: emp.name }))
  }

  // Иначе показываем всех сотрудников
  return employees.value.map(emp => ({ id: emp.id, name: emp.name }))
})
const formatTime = (dateTime) => {
  if (!dateTime) return ''
  
  // Если это уже время в формате HH:mm
  if (typeof dateTime === 'string' && dateTime.match(/^\d{2}:\d{2}$/)) {
    return dateTime
  }
  
  // Обрабатываем ISO даты с timezone
  let momentTime
  if (typeof dateTime === 'string') {
    if (dateTime.includes('T') && dateTime.includes('+')) {
      // ISO формат с timezone, например "2026-03-05T12:12:00+03:00"
      momentTime = moment(dateTime)
    } else if (dateTime.includes('T')) {
      // ISO формат без timezone
      momentTime = moment(dateTime.split('T')[1]?.split('+')[0], 'HH:mm:ss')
    } else {
      // Обычное время
      momentTime = moment(dateTime, 'HH:mm')
    }
  } else {
    momentTime = moment(dateTime)
  }
  
  return momentTime.isValid() ? momentTime.format('HH:mm') : String(dateTime)
}
// Функция получения названия отдела (или отделов)
const getDepartmentName = (departmentIds) => {
  if (!departmentIds || departmentIds.length === 0) return 'Не указан'

  const deptNames = departmentIds.map(id => {
    const dept = departments.value.find(d => d.ID === id)
    return dept ? dept.NAME : 'Неизвестный отдел'
  })

  return deptNames.join(', ')
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
    } else if (type === 'absenceTypes') {
      // Для типов отсутствия
      if (filters.value.selectAll[type]) {
        filters.value.selected[type] = absenceTypeOptions.map(item => item.value)
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

// Функция сброса фильтров (обновленная)
const disableFilters = () => {
  panel.value = false
  searchEmployee.value = ''
  for (let i = 0; i < Object.keys(filters.value.selectAll).length; i++) {
    filters.value.selectAll[Object.keys(filters.value.selectAll)[i]] = false
    filters.value.selected[Object.keys(filters.value.selected)[i]] = []
  }
}

const vacationDaysInSchedule = computed(() => {
  if (!selectedEmployeeForStats.value) return 0
  
  return selectedEmployeeForStats.value.schedule.filter(
    item => item.status === 'vacation' && item.year === currentYear.value
  ).length
})

// Переработанные часы в выходные
const weekendOvertimeHours = computed(() => {
  if (!selectedEmployeeForStats.value) return '0ч'
  
  const year = currentYear.value
  const month = currentMonth.value
  let totalMinutes = 0
  
  selectedEmployeeForStats.value.schedule
    .filter(item => item.status === 'overtime' && item.hours && item.month === month + 1 && item.year === year)
    .forEach(item => {
      // Проверяем, является ли день выходным (суббота или воскресенье)
      const date = new Date(year, month, item.day - 1)
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

// Новые computed свойства для года
const months = computed(() => {
  const monthsList = []
  for (let i = 0; i < 12; i++) {
    const monthDate = moment(currentDate.value).month(i)
    monthsList.push({
      number: i + 1,
      name: monthDate.format('MMMM YYYY'),
      shortName: monthDate.format('MMM'),
      days: monthDate.daysInMonth()
    })
  }
  return monthsList
})

const yearDays = computed(() => {
  const days = []
  for (let month = 0; month < 12; month++) {
    const monthDate = moment(currentDate.value).month(month)
    const daysInMonth = monthDate.daysInMonth()
    
    for (let day = 1; day <= daysInMonth; day++) {
      const currentDay = moment(currentDate.value).month(month).date(day)
      days.push({
        key: `${month + 1}-${day}`,
        date: currentDay,
        month: month + 1,
        day: day,
        label: day.toString(),
        isWeekend: currentDay.day() === 0 || currentDay.day() === 6
      })
    }
  }
  return days
})

const totalColumns = computed(() => {
  if (viewMode.value === 'year') {
    return yearDays.value.length + 2
  }
  return periods.value.length + 2
})

// Функция для получения количества дней в месяце года
const daysInYearMonth = (month) => {
  return moment(currentDate.value).month(month - 1).daysInMonth()
}

// Проверка на текущий месяц
const isCurrentMonth = (month) => {
  const now = moment()
  return now.year() === currentYear.value && now.month() + 1 === month
}

// Проверка на текущий день
const isCurrentDay = (date) => {
  const now = moment()
  return date.isSame(now, 'day')
}

// Проверка на выходной
const isWeekend = (date) => {
  const day = date.day()
  return day === 0 || day === 6
}

// Функция для установки ref месяца
const setMonthRef = (month, el) => {
  if (el) {
    monthRefs.value[month] = el
  }
}

// Функция скролла к текущему месяцу
const scrollToCurrentMonth = async () => {
  await nextTick()
  const currentMonth = moment().month() + 1
  const monthElement = monthRefs.value[currentMonth]
  
  if (monthElement) {
    // Находим родительский контейнер с прокруткой
    const tableContainer = document.querySelector('.v-table__wrapper')
    if (tableContainer) {
      const monthOffset = monthElement.offsetLeft
      tableContainer.scrollTo({
        left: monthOffset - 100,
        behavior: 'smooth'
      })
    }
  }
}

// Получение статуса для дня в году
const getStatusForYearDay = (employee, day) => {
  const scheduleItem = employee.schedule.find(
    (s) => num(s.day) === day.day && num(s.month) === day.month
  )
  return scheduleItem?.status || null
}

// Год: объединение подряд идущих дней с тем же статусом и часами (как в месяце)
const getMergedYearDays = (employee) => {
  updateTrigger.value
  if (viewMode.value !== 'year') return []
  const days = yearDays.value
  if (!days.length) return []

  const merged = []
  let i = 0
  while (i < days.length) {
    const d = days[i]
    const item = employee.schedule.find(
      (s) => num(s.day) === d.day && num(s.month) === d.month
    )
    const status = item?.status ?? null
    const hours = item?.hours ?? null
    const details = item?.details ?? null
    let span = 1
    let j = i + 1
    while (j < days.length) {
      const d2 = days[j]
      const item2 = employee.schedule.find(
        (s) => num(s.day) === d2.day && num(s.month) === d2.month
      )
      const st2 = item2?.status ?? null
      const h2 = item2?.hours ?? null
      if (canMergeAdjacentStatus(st2, h2, status, hours)) {
        span++
        j++
      } else {
        break
      }
    }
    merged.push({
      yearDay: d,
      span,
      status,
      hours,
      details
    })
    i += span
  }
  return merged
}

const openYearMergedDialog = (employee, merged) => {
  if (!merged.status) return
  const startM = merged.yearDay.date.clone()
  const endM = startM.clone().add(merged.span - 1, 'days')
  openDateDialog(employee, {
    startDay: merged.yearDay.day,
    span: merged.span,
    status: merged.status,
    hours: merged.hours,
    details: merged.details,
    dateStart: startM.format('YYYY-MM-DD'),
    dateEnd: endM.format('YYYY-MM-DD')
  })
}

// Получение текста статуса для года (сокращенный)
const getYearStatusText = (status) => {
  const texts = {
    'vacation': 'О',
    'dayoff': 'Г',
    'remote': 'У',
    'overtime': 'П'
  }
  return texts[status] || status
}

// Переработанные часы в будние
const weekdayOvertimeHours = computed(() => {
  if (!selectedEmployeeForStats.value) return '0ч'
  
  const year = currentYear.value
  const month = currentMonth.value
  let totalMinutes = 0
  
  selectedEmployeeForStats.value.schedule
    .filter(item => item.status === 'overtime' && item.hours && item.month === month + 1 && item.year === year)
    .forEach(item => {
      // Проверяем, является ли день будним (понедельник-пятница)
      const date = new Date(year, month, item.day - 1)
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
  return true;
  return overtimeForm.value.currentUserId &&
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
  return remoteForm.value.dates && remoteForm.value.dates.length > 0
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
      const mergedList =
        viewMode.value === 'week'
          ? getMergedWeekPeriods(selectedEmployee.value)
          : getMergedDays(selectedEmployee.value)

      const otherPeriods = mergedList.filter((p) => {
        if (!p.status) return false
        if (selectedPeriod.value.dateStart && p.dateStart) {
          return !(
            p.dateStart === selectedPeriod.value.dateStart &&
            p.dateEnd === selectedPeriod.value.dateEnd
          )
        }
        return p.startDay !== selectedPeriod.value.startDay
      })

      const ns = moment(startDate.value).startOf('day')
      const ne = moment(endDate.value).startOf('day')

      const hasOverlap = otherPeriods.some((period) => {
        let ps
        let pe
        if (period.dateStart && period.dateEnd) {
          ps = moment(period.dateStart).startOf('day')
          pe = moment(period.dateEnd).startOf('day')
        } else {
          ps = moment(currentDate.value).date(period.startDay).startOf('day')
          pe = moment(currentDate.value)
            .date(period.startDay + period.span - 1)
            .startOf('day')
        }
        return ns.isSameOrBefore(pe, 'day') && ne.isSameOrAfter(ps, 'day')
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

function scheduleItemForCalendarDate(employee, dayMoment) {
  const y = dayMoment.year()
  const m = dayMoment.month() + 1
  const d = dayMoment.date()
  return employee.schedule.find((s) => {
    if (num(s.day) !== d || num(s.month) !== m) return false
    if (s.year === undefined || s.year === null || s.year === '') return true
    return num(s.year) === y
  })
}

// Функция для объединения дней с одинаковыми статусами (текущий месяц)
const getMergedDays = (employee) => {
  // Используем updateTrigger для принудительного пересчета
  updateTrigger.value
  
  const merged = []
  let currentSpan = 0
  let currentStatus = null
  let currentHours = null
  let currentDetails = null
  let startDay = 1

  const y = num(currentYear.value)
  const m = num(currentMonth.value + 1)

  // Только дни выбранного месяца (в schedule может быть весь год).
  // day/month/year из CRM могут быть строками — сравниваем через Number.
  const scheduleMap = new Map()
  employee.schedule.forEach((item) => {
    const iy = item.year === undefined || item.year === null || item.year === '' ? y : num(item.year)
    if (iy !== y || num(item.month) !== m) return
    const id = num(item.day)
    if (!Number.isNaN(id)) scheduleMap.set(id, item)
  })

  for (let day = 1; day <= daysInMonth.value; day++) {
    const scheduleItem = scheduleMap.get(day)
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

    if (canMergeAdjacentStatus(status, hours, currentStatus, currentHours)) {
      currentSpan++
    } else {
      // Сохраняем предыдущую группу
      merged.push({
        startDay,
        status: currentStatus,
        hours: currentHours,
        details: currentDetails,
        span: currentSpan
      })
      
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
        details: currentDetails,
        span: currentSpan
      })
    }
  }

  return merged
}

// Неделя: объединение подряд идущих дней с тем же статусом и часами (как в месяце)
const getMergedWeekPeriods = (employee) => {
  updateTrigger.value
  if (viewMode.value !== 'week') return []
  const weekPeriods = periods.value
  if (!weekPeriods.length) return []

  const merged = []
  let i = 0
  while (i < weekPeriods.length) {
    const m = moment(weekPeriods[i].key)
    const item = scheduleItemForCalendarDate(employee, m)
    const status = item?.status ?? null
    const hours = item?.hours ?? null
    const details = item?.details ?? null
    let span = 1
    let j = i + 1
    while (j < weekPeriods.length) {
      const m2 = moment(weekPeriods[j].key)
      const item2 = scheduleItemForCalendarDate(employee, m2)
      const st2 = item2?.status ?? null
      const h2 = item2?.hours ?? null
      if (canMergeAdjacentStatus(st2, h2, status, hours)) {
        span++
        j++
      } else {
        break
      }
    }
    const startM = moment(weekPeriods[i].key)
    const endM = moment(weekPeriods[i + span - 1].key)
    merged.push({
      startDay: startM.date(),
      span,
      status,
      hours,
      details,
      dateStart: startM.format('YYYY-MM-DD'),
      dateEnd: endM.format('YYYY-MM-DD')
    })
    i += span
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
  // Проверяем права на редактирование
  const currentUserId = currentUser.value?.ID;
  if (!currentUserId) {
    alert('Не удалось определить текущего пользователя');
    return;
  }

  // Проверяем права в зависимости от типа статуса
    if (period.status === 'vacation' || period.status === 'dayoff') {
      if (!canEditVacationOrDayoff(currentUserId, employee.id)) {
        alert('Только руководитель отдела может редактировать отпуск и отгул подчиненных');
        return;
      }
    } else if (period.status === 'remote') {
      if (!canEditRemoteOrOvertime(currentUserId, employee.id, period.status)) {
        alert('Нет прав на редактирование чужой удаленки');
        return;
      }
    } else if (period.status === 'overtime') {
      if (!canEditRemoteOrOvertime(currentUserId, employee.id, period.status)) {
        openOvertimeViewDialog(period.details)
        return;
      }
    }

  // Если это переработка
  if (period.status === 'overtime') {
    openOvertimeEditDialog(employee, period); // Handles manager block
    return;
  }

  selectedEmployee.value = employee
  selectedPeriod.value = period

  // Устанавливаем даты на основе выбранного периода
  if (period.dateStart && period.dateEnd) {
    startDate.value = period.dateStart
    endDate.value = period.dateEnd
  } else {
    startDate.value = moment(currentDate.value).date(period.startDay).format('YYYY-MM-DD')
    endDate.value = moment(currentDate.value).date(period.startDay + period.span - 1).format('YYYY-MM-DD')
  }
  selectedStatus.value = period.status
  overtimeHours.value = period.hours || ''

  dateError.value = ''
  dateDialog.value = true
}

// Функция обновления периода
const updatePeriod = async () => {
  if (isValidSelection.value && selectedEmployee.value && selectedPeriod.value) {
    const currentUserId = currentUser.value?.ID;
    if (!currentUserId) {
      alert('Не удалось определить текущего пользователя');
      return;
    }

    // Check permissions
    if (selectedPeriod.value.status === 'vacation' || selectedPeriod.value.status === 'dayoff') {
      if (!canEditVacationOrDayoff(currentUserId, selectedEmployee.value.id)) {
        alert('Только руководитель отдела может редактировать отпуск и отгул подчиненных');
        return;
      }
    } else if (selectedPeriod.value.status === 'remote' || selectedPeriod.value.status === 'overtime') {
      if (!canEditRemoteOrOvertime(currentUserId, selectedEmployee.value.id)) {
        alert('Нет прав на редактирование чужой удаленки/переработки (только себя или руководитель отдела)');
        return;
      }
    }

    try {
      const st = selectedPeriod.value.status
      const itemId = selectedPeriod.value.details?.id

      // Отпуск / отгул: те же UF, что при создании — затем перезагрузка графика
      if ((st === 'vacation' || st === 'dayoff') && itemId) {
        const startDateTime = moment(startDate.value + ' 00:00', 'YYYY-MM-DD HH:mm').format()
        const endDateTime = moment(endDate.value + ' 23:59', 'YYYY-MM-DD HH:mm').format()
        const typeId = st === 'vacation' ? 10372 : 10374
        await new Promise((resolve, reject) => {
          BX24.callMethod(
            'crm.item.update',
            {
              entityTypeId: 1048,
              id: itemId,
              fields: {
                ufCrm36_1737068344170: typeId,
                ufCrm36_1737068421810: startDateTime,
                ufCrm36_1737068451414: endDateTime,
                ufCrm36_1737068503448: selectedPeriod.value.details?.additionalInfo ?? '',
                categoryId: 108
              }
            },
            (result) => {
              if (result.error()) reject(new Error(String(result.error())))
              else resolve(result.data())
            }
          )
        })
        await loadData()
        dateDialog.value = false
        return
      }

      // Удалёнка: массив дат + границы периода как в форме создания
      if (st === 'remote' && itemId) {
        const newRemoteDates = []
        const start = moment(startDate.value)
        const end = moment(endDate.value)
        let current = start.clone()
        while (current <= end) {
          newRemoteDates.push(current.format('YYYY-MM-DD'))
          current.add(1, 'day')
        }
        await new Promise((resolve, reject) => {
          BX24.callMethod(
            'crm.item.update',
            {
              entityTypeId: 1048,
              id: itemId,
              fields: {
                ufCrm36_1773132483885: newRemoteDates,
                ufCrm36_1737068421810: newRemoteDates[0],
                ufCrm36_1737068451414: newRemoteDates[newRemoteDates.length - 1],
                categoryId: 168
              }
            },
            (result) => {
              if (result.error()) reject(new Error(String(result.error())))
              else resolve(result.data())
            }
          )
        })
        await loadData()
        dateDialog.value = false
        return
      }

      // Переработка и прочее без отдельного CRM-пути — локальное обновление (устар.)
      const oldDetails = selectedPeriod.value.details
      for (let day = selectedPeriod.value.startDay; day < selectedPeriod.value.startDay + selectedPeriod.value.span; day++) {
        const index = selectedEmployee.value.schedule.findIndex((s) => s.day === day)
        if (index !== -1) selectedEmployee.value.schedule.splice(index, 1)
      }

      const startDay = moment(startDate.value).date()
      const endDay = moment(endDate.value).date()
      for (let day = startDay; day <= endDay; day++) {
        const scheduleItem = { day, status: selectedStatus.value }
        if (selectedStatus.value === 'overtime' && overtimeHours.value) {
          scheduleItem.hours = overtimeHours.value
        }
        if (oldDetails) scheduleItem.details = oldDetails

        const existingIndex = selectedEmployee.value.schedule.findIndex((s) => s.day === day)
        if (existingIndex !== -1) {
          selectedEmployee.value.schedule[existingIndex] = scheduleItem
        } else {
          selectedEmployee.value.schedule.push(scheduleItem)
        }
      }

      selectedEmployee.value.schedule.sort((a, b) => a.day - b.day)
      if (selectedEmployeeForStats.value?.id === selectedEmployee.value.id) {
        selectedEmployeeForStats.value = { ...selectedEmployee.value }
      }

      updateTrigger.value++
      console.log('Период обновлен:', selectedStatus.value)
      dateDialog.value = false
    } catch (error) {
      console.error('Ошибка обновления:', error)
      alert(error?.message || 'Ошибка обновления периода')
    }
  }
}

// Enhanced deletePeriod with CRM support
const deletePeriod = (crmDelete = true) => {
  if (selectedEmployee.value && selectedPeriod.value) {
    const currentUserId = currentUser.value?.ID
    if (!currentUserId) {
      alert('Не удалось определить текущего пользователя')
      return
    }
    if (selectedPeriod.value.status === 'overtime' && !canEditRemoteOrOvertime(currentUserId, selectedEmployee.value.id, 'overtime')) {
      alert('Руководитель не может удалять переработки')
      return
    }
    if (selectedPeriod.value.status === 'remote' && !canEditRemoteOrOvertime(currentUserId, selectedEmployee.value.id, 'remote')) {
      alert('Нет прав на удаление этой удалёнки')
      return
    }
    if (crmDelete && selectedPeriod.value?.details?.id) {
      const det = selectedPeriod.value.details
      const normD = (d) => moment(d).format('YYYY-MM-DD')

      // Удалёнка с массивом: несколько слотов в одном элементе — убираем только даты слота из UF
      if (
        selectedPeriod.value.status === 'remote' &&
        Array.isArray(det.remoteDates) &&
        det.remoteDates.length > 0 &&
        Array.isArray(det.allRemoteDates) &&
        det.allRemoteDates.length > 0
      ) {
        const removeSet = new Set(det.remoteDates.map(normD))
        const remaining = det.allRemoteDates
          .map(normD)
          .filter((d) => !removeSet.has(d))
          .sort()

        if (remaining.length > 0) {
          BX24.callMethod(
            'crm.item.update',
            {
              entityTypeId: 1048,
              id: det.id,
              fields: {
                ufCrm36_1773132483885: remaining,
                ufCrm36_1737068421810: remaining[0],
                ufCrm36_1737068451414: remaining[remaining.length - 1]
              }
            },
            (result) => {
              if (result.error()) {
                alert('Ошибка обновления удалёнки в CRM: ' + result.error())
                console.error(result.error())
              } else {
                console.log('CRM remote dates updated, remaining:', remaining)
              }
            }
          )
          loadData()
          dateDialog.value = false
          return
        }
        // remaining пуст — удаляем элемент целиком (ниже)
      }

      // Один слот / не массив / после удаления последнего слота — полное удаление элемента
      BX24.callMethod('crm.item.delete', {
        entityTypeId: 1048,
        id: selectedPeriod.value.details.id
      }, (result) => {
        if (result.error()) {
          alert('Ошибка удаления из CRM: ' + result.error())
          console.error(result.error())
        } else {
          console.log('CRM deleted:', selectedPeriod.value.details.id)
        }
      })
      loadData()
      dateDialog.value = false
      return
    }
    
    // Local-only delete (fallback)

    const status = selectedPeriod.value.status
    if (status === 'vacation' || status === 'dayoff') {
      if (!canEditVacationOrDayoff(currentUserId, selectedEmployee.value.id)) {
        alert('Только руководитель отдела может удалять отпуск/отгул')
        return
      }
    } else if (status === 'overtime') {
      if (!canEditRemoteOrOvertime(currentUserId, selectedEmployee.value.id)) {
        alert('Нет прав на удаление переработки')
        return
      }
    }

    // Remove local schedule entries
    for (let day = selectedPeriod.value.startDay; 
         day < selectedPeriod.value.startDay + selectedPeriod.value.span; 
         day++) {
      const index = selectedEmployee.value.schedule.findIndex(s => s.day === day)
      if (index !== -1) selectedEmployee.value.schedule.splice(index, 1)
    }
    
    updateTrigger.value++
    console.log('Local delete:', selectedEmployee.value.name)
  }
  
  dateDialog.value = false
}

// Функции для диалога переработки
const openOvertimeDialog = () => {
  if (isManager.value) {
    // Managers: open full overtime stats view
    selectedEmployeeForStats.value = null // Clear to show all
  }
  // Employees: normal add flow
  overtimeForm.value = {
    id: null,
    whatDone: '',
    date: moment().format('YYYY-MM-DD'),
    startTime: '09:00',
    endTime: '18:00',
    event: null,
    workType: null,
    justification: ''
  }
  overtimeError.value = ''
  overtimeDialog.value = true
}

// Функция открытия диалога редактирования переработки
const openOvertimeViewDialog = (details) => {
  // Load data into readonly form
  const extractTime = (dateTime) => {
    if (!dateTime) return ''
    const momentTime = moment(dateTime)
    return momentTime.isValid() ? momentTime.format('HH:mm') : dateTime
  }

  const extractDate = (dateTime) => {
    if (!dateTime) return ''
    const momentTime = moment(dateTime)
    return momentTime.isValid() ? momentTime.format('YYYY-MM-DD') : dateTime
  }

  const findEventById = (eventId) => {
    if (!eventId) return ''
    const event = events.value.find(e => String(e.id) === String(eventId))
    return event ? (event.title || event.name || event.NAME || `ID: ${eventId}`) : eventId
  }
  
  overtimeViewForm.value = {
    id: details?.id || null,
    whatDone: details?.whatDone || '',
    date: extractDate(details?.date || details?.startTime),
    startTime: extractTime(details?.startTime),
    endTime: extractTime(details?.endTime),
    event: findEventById(details?.event),
    workType: details?.workType || '',
    justification: details?.justification || ''
  }
  overtimeViewDialog.value = true
}

const openOvertimeEditDialog = (employee, period) => {
  const currentUserId = currentUser.value?.ID
  // Своя переработка — можно редактировать и руководителю; чужая у руководителя — только просмотр
  if (!canEditRemoteOrOvertime(currentUserId, employee.id, 'overtime')) {
    openOvertimeViewDialog(period.details)
    return
  }
  // Редактирование (себе или сотруднику — для переработки только себе)
  console.log('Редактирование переработки:', period);
  const extractTime = (dateTime) => {
    if (!dateTime) return ''
    const momentTime = moment(dateTime)
    return momentTime.isValid() ? momentTime.format('HH:mm') : dateTime
  }

  const extractDate = (dateTime) => {
    if (!dateTime) return moment(currentDate.value).date(period.startDay).format('YYYY-MM-DD')
    
    const momentTime = moment(dateTime)
    if (momentTime.isValid()) {
      return momentTime.format('YYYY-MM-DD')
    }
    
    if (typeof dateTime === 'number' || !isNaN(parseInt(dateTime))) {
      return moment(currentDate.value).date(parseInt(dateTime)).format('YYYY-MM-DD')
    }
    
    return dateTime
  }

  const findEventById = (eventId) => {
    if (!eventId) return null
    return events.value.find(event => event.id == eventId) || null
  }
  
  overtimeForm.value = {
    id: period.details?.id || null,
    employeeId: employee.id, // Added for delete validation
    whatDone: period.details?.whatDone || '',
    date: extractDate(period.details?.startTime || period.details?.date),
    startTime: extractTime(period.details?.startTime),
    endTime: extractTime(period.details?.endTime),
    event: findEventById(period.details?.event) || null,
    workType: period.details?.workType || null,
    justification: period.details?.justification || ''
  }
  
  overtimeError.value = ''
  overtimeDialog.value = true
}

const saveOvertime = async () => {
  const currentUserId = currentUser.value?.ID;
  if (!currentUserId) {
    overtimeError.value = 'Не удалось определить текущего пользователя'
    return
  }
  // Добавление/редактирование только своей переработки (assignedById = текущий пользователь)
  if (!canEditRemoteOrOvertime(currentUserId, currentUserId, 'overtime')) {
    overtimeError.value = 'Нет прав на сохранение переработки'
    return
  }
  if (!isOvertimeValid.value) {
    overtimeError.value = 'Заполните все обязательные поля'
    return
  }

  try {
    // Создаем полные datetime значения из даты и времени
    const startDateTime = moment(overtimeForm.value.date + ' ' + overtimeForm.value.startTime, 'YYYY-MM-DD HH:mm').format();
    const endDateTime = moment(overtimeForm.value.date + ' ' + overtimeForm.value.endTime, 'YYYY-MM-DD HH:mm').format();

    const overtimeData = {
      ufCrm36_1737068344170: 11604,
      ufCrm36_1772712553: overtimeForm.value.whatDone, // Что сделано
      ufCrm36_1737068421810: overtimeForm.value.date, // День
      ufCrm36_1772712459: startDateTime, // Время начала работ (полный datetime)
      ufCrm36_1772712490: endDateTime, // Время завершения работ (полный datetime)
      //ufCrm36_1772712787: overtimeForm.value.event?.id || overtimeForm.value.event, // Мероприятие (ID)
      ufCrm36_1773312024: overtimeForm.value.event || '',
      ufCrm36_1772712618: overtimeForm.value.workType, // Тип работ
      ufCrm36_1772712662: overtimeForm.value.justification // Обоснование
    };

    if (overtimeForm.value.id) {
      // Обновляем существующий элемент
      BX24.callMethod(
          'crm.item.update',
          {
              id: overtimeForm.value.id,
              fields: overtimeData
          },
          (result) => {
              result.error()
                  ? console.error(result.error())
                  : console.info(result.data())
              ;
          }
      );
      console.log('Обновлена переработка в CRM:', overtimeForm.value);
    } else {
      // Создаем новый элемент
      overtimeData.assignedById = currentUserId;
      overtimeData.categoryId = 168,
      BX24.callMethod(
          'crm.item.add',
          {
              entityTypeId: 1048,
              fields: overtimeData
          },
          (result) => {
              result.error()
                  ? console.error(result.error())
                  : console.info(result.data())
              ;
          }
      );
      console.log('Добавлена переработка в CRM:', overtimeForm.value);
    }

    // Перезагружаем данные после успешного сохранения
    await loadData();

    overtimeDialog.value = false
  } catch (error) {
    console.error('Ошибка при сохранении переработки:', error);
    overtimeError.value = 'Ошибка при сохранении переработки'
  }
}

const deleteOvertimeItem = async () => {
  if (!overtimeForm.value.id || !overtimeForm.value.employeeId) {
    overtimeError.value = 'Невозможно удалить: отсутствует ID записи'
    return
  }

  if (overtimeForm.value.employeeId !== currentUser.value?.ID) {
    overtimeError.value = 'Можно удалять только свои переработки'
    return
  }

  try {
    // CRM delete
    await new Promise((resolve, reject) => {
      BX24.callMethod('crm.item.delete', {
        entityTypeId: 1048,
        id: overtimeForm.value.id
      }, (result) => {
        if (result.error()) {
          reject(new Error(result.error()))
        } else {
          resolve(result)
        }
      })
    })

    // Refresh data
    await loadData()

    // Close dialog
    overtimeDialog.value = false
    alert('Переработка удалена успешно')
    console.log('Deleted overtime:', overtimeForm.value.id)
  } catch (error) {
    console.error('Delete error:', error)
    overtimeError.value = 'Ошибка удаления: ' + error.message
  }
}

// Функции для диалога удаленки
const openRemoteDialog = () => {
  // Сброс формы
  remoteForm.value = {
    dates: []
  }
  remoteError.value = ''
  remoteDialog.value = true
}

// Функции для диалога отпуска
const toggleAllDialogEmployees = (formType) => {
  const form = formType === 'vacation' ? vacationForm.value : dayoffForm.value
  const options = dialogEmployeeOptions.value
  if (form.selectedEmployees.length === options.length) {
    form.selectedEmployees = []
  } else {
    form.selectedEmployees = options.map(opt => opt.id)
  }
}

const openVacationDialog = () => {
  // Сброс формы, prefill from filters if manager
  const currentUserId = currentUser.value?.ID
  vacationForm.value = {
    startDate: null,
    endDate: null,
    additionalInfo: '',
    selectedEmployees: isManager.value && filters.value.selected.employees.length > 0 
      ? filters.value.selected.employees.filter(id => dialogEmployeeOptions.value.some(opt => opt.id === id))
      : []
  }
  vacationError.value = ''
  vacationDialog.value = true
}

// Функции для диалога отгула
const openDayoffDialog = () => {
  // Сброс формы, prefill from filters if manager
  const currentUserId = currentUser.value?.ID
  dayoffForm.value = {
    startDate: null,
    endDate: null,
    additionalInfo: '',
    selectedEmployees: isManager.value && filters.value.selected.employees.length > 0 
      ? filters.value.selected.employees.filter(id => dialogEmployeeOptions.value.some(opt => opt.id === id))
      : []
  }
  dayoffError.value = ''
  dayoffDialog.value = true
}

const saveRemote = async () => {
  if (!isRemoteValid.value) {
    remoteError.value = 'Выберите сотрудника и хотя бы один день'
    return
  }

  // Получаем ID текущего пользователя
  const currentUserId = currentUser.value?.ID;
  if (!currentUserId) {
    remoteError.value = 'Не удалось определить текущего пользователя'
    return
  }

  try {
    // Создаем 1 элемент удаленки в CRM с массивом выбранных дат
    const remoteData = {
      assignedById: currentUserId,
      ufCrm36_1737068344170: 11602,
      ufCrm36_1737068421810: remoteForm.value.dates[0], // Самая ранняя дата
      ufCrm36_1737068451414: remoteForm.value.dates[remoteForm.value.dates.length - 1], // Самая поздняя дата
      ufCrm36_1773132483885: remoteForm.value.dates // Массив выбранных дат
    };
    remoteData.categoryId = 168;
    
    BX24.callMethod(
      'crm.item.add',
      {
        entityTypeId: 1048,
        fields: remoteData
      },
      (result) => 
      {
        result.error() 
          ? console.error(result.error()) 
          : console.info(result.data())
        ;
      }
    );

    // Перезагружаем данные после успешного сохранения
    await loadData();

    remoteDialog.value = false
    console.log('Добавлена удаленка в CRM:', remoteForm.value)
  } catch (error) {
    console.error('Ошибка при сохранении удаленки:', error);
    remoteError.value = 'Ошибка при сохранении удаленки'
  }
}

// Функции для диалога отпуска
const saveVacation = async () => {
  if (!isVacationValid.value) {
    vacationError.value = 'Заполните все обязательные поля'
    return
  }

  const currentUserId = currentUser.value?.ID;
  if (!currentUserId) {
    vacationError.value = 'Не удалось определить текущего пользователя'
    return
  }

  try {
    const startDateTime = moment(vacationForm.value.startDate + ' 00:00', 'YYYY-MM-DD HH:mm').format();
    const endDateTime = moment(vacationForm.value.endDate + ' 23:59', 'YYYY-MM-DD HH:mm').format();
    const commonData = {
      ufCrm36_1737068344170: 10372, // Тип: отпуск
      ufCrm36_1737068421810: startDateTime,
      ufCrm36_1737068451414: endDateTime,
      ufCrm36_1737068503448: vacationForm.value.additionalInfo || '',
      categoryId: 108,
      stageId: isManager.value ? 'DT1048_108:SUCCESS' : 'DT1048_108:UC_NQK5LN'
    }

      const empData = { ...commonData, ASSIGNED_BY_ID: vacationForm.value.selectedEmployees ?? currentUserId }
      BX24.callMethod(
        'crm.item.add',
        {
          entityTypeId: 1048,
          fields: empData
        },
        (result) => {
          if (result.error()) {
            console.error('Error creating vacation:', result.error())
          } else {
            console.info('Vacation created:', result.data())
          }
        }
      )
    await loadData()
    vacationDialog.value = false
  } catch (error) {
    console.error('Error saving vacation:', error)
    vacationError.value = 'Ошибка при сохранении отпуска'
  }
}

// Функции для диалога отгула
const saveDayoff = async () => {
  if (!isDayoffValid.value) {
    dayoffError.value = 'Заполните все обязательные поля'
    return
  }

  const currentUserId = currentUser.value?.ID;
  if (!currentUserId) {
    dayoffError.value = 'Не удалось определить текущего пользователя'
    return
  }

  try {
    const startDateTime = moment(dayoffForm.value.startDate + ' 00:00', 'YYYY-MM-DD HH:mm').format();
    const endDateTime = moment(dayoffForm.value.endDate + ' 23:59', 'YYYY-MM-DD HH:mm').format();
    const commonData = {
      ufCrm36_1737068344170: 10374, // Тип: отгул
      ufCrm36_1737068421810: startDateTime,
      ufCrm36_1737068451414: endDateTime,
      ufCrm36_1737068503448: dayoffForm.value.additionalInfo || '',
      categoryId: 108,
      stageId: isManager.value ? 'DT1048_108:SUCCESS' : 'DT1048_108:UC_NQK5LN'
    }

    // Determine target employee IDs
    let targetEmployeeIds
    if (isManager.value && dayoffForm.value.selectedEmployees.length > 0) {
      // Validate all selected are in manager's departments
      const invalidEmps = dayoffForm.value.selectedEmployees.filter(id => 
        !dialogEmployeeOptions.value.some(opt => opt.id === id)
      )
      if (invalidEmps.length > 0) {
        dayoffError.value = 'Некоторые сотрудники не из вашего отдела'
        return
      }
      targetEmployeeIds = dayoffForm.value.selectedEmployees
      console.log(`Creating dayoff for ${targetEmployeeIds.length} employees`)
    } else {
      targetEmployeeIds = [currentUserId]
    }

    // Create one CRM item per employee
    for (const empId of targetEmployeeIds) {
      const empData = { ...commonData, assignedById: empId }
      BX24.callMethod(
        'crm.item.add',
        {
          entityTypeId: 1048,
          fields: empData
        },
        (result) => {
          if (result.error()) {
            console.error('Error creating dayoff:', result.error())
          } else {
            console.info('Dayoff created:', result.data())
          }
        }
      )
    }

    await loadData()
    dayoffDialog.value = false
    console.log('Dayoff(s) saved for:', targetEmployeeIds)
  } catch (error) {
    console.error('Error saving dayoff:', error)
    dayoffError.value = 'Ошибка при сохранении отгула'
  }
}

// Валидация форм
const isVacationValid = computed(() => {
  return vacationForm.value.startDate && vacationForm.value.endDate
})

const isDayoffValid = computed(() => {
  return dayoffForm.value.startDate && dayoffForm.value.endDate
})

// Permissions for delete (vacation, dayoff, overtime)
const canDeletePeriod = computed(() => {
  if (!selectedPeriod.value?.details?.id) return false
  const status = selectedPeriod.value.status
  if (status === 'vacation' || status === 'dayoff') {
    return canEditVacationOrDayoff(currentUser.value?.ID, selectedEmployee.value?.id)
  }
  if (status === 'overtime' || status === 'remote') {
    return canEditRemoteOrOvertime(currentUser.value?.ID, selectedEmployee.value?.id)
  }
  return false
})


const selectedEmployeeForStats = ref(null)

// Функция выбора сотрудника для статистики
const selectEmployeeForStats = (employee) => {
  selectedEmployeeForStats.value = employee
}

// Вычисляемое свойство для длительности переработки
const overtimeDuration = computed(() => {
  if (!overtimeForm.value.startTime || !overtimeForm.value.endTime) {
    return 'Не указано'
  }

  const start = moment(overtimeForm.value.startTime, 'HH:mm')
  const end = moment(overtimeForm.value.endTime, 'HH:mm')

  if (!start.isValid() || !end.isValid()) {
    return 'Некорректное время'
  }

  const duration = moment.duration(end.diff(start))

  const hours = Math.floor(duration.asHours())
  const minutes = duration.minutes()

  if (hours === 0 && minutes === 0) {
    return '0 минут'
  }

  let result = ''
  if (hours > 0) {
    result += `${hours} ч`
  }
  if (minutes > 0) {
    if (result) result += ' '
    result += `${minutes} мин`
  }

  return result || '0 минут'
})

// Вычисляемые свойства для статистики
const vacationDaysLeft = computed(() => {
  if (!selectedEmployeeForStats.value) return 0
  
  // Считаем использованные отпускные дни за текущий месяц
  const usedVacationDays = selectedEmployeeForStats.value.schedule.filter(
    item => item.status === 'vacation' && item.year === currentYear.value
  ).length
  
  // Баланс отпускных дней (28 - использовано)
  return Math.max(0, 28 - usedVacationDays)
})

const totalOvertimeHours = computed(() => {
  if (!selectedEmployeeForStats.value) return '0ч'
  
  const overtimeItems = selectedEmployeeForStats.value.schedule.filter(
    item => item.status === 'overtime' && item.hours && item.year === currentYear.value
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
  
  // Собираем только переработки выбранного сотрудника за текущий месяц
  employee.schedule
    .filter(item => item.status === 'overtime' && item.details && item.year === currentYear.value)
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
  if (!eventId) return 'Не указано'
  const event = events.value.find(e => String(e.id) === String(eventId))
  // Проверяем разные возможные поля для названия
  return event ? (event.title || event.name || event.NAME || `ID: ${eventId}`) : `ID: ${eventId}`
}

const getWorkTypeName = (workType) => {
  if (!workType) return 'Не указано'
  const type = workTypes.find(t => t.value === workType)
  return type ? type.title : workType
}

const formatDate = (date) => {
  if (!date) return ''

  // Если дата уже в формате DD.MM.YYYY, возвращаем как есть
  if (typeof date === 'string' && date.match(/^\d{2}\.\d{2}\.\d{4}$/)) {
    return date
  }

  // Обрабатываем ISO даты с timezone
  let momentDate
  if (typeof date === 'string') {
    if (date.includes('T') && date.includes('+')) {
      // ISO формат с timezone, например "2026-03-05T12:12:00+03:00"
      momentDate = moment(date)
    } else if (date.includes('T')) {
      // ISO формат без timezone
      momentDate = moment(date.split('T')[0])
    } else {
      // Обычная дата
      momentDate = moment(date)
    }
  } else {
    momentDate = moment(date)
  }

  return momentDate.isValid() ? momentDate.format('DD.MM.YYYY') : String(date)
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
        return employee && filters.value.selected.departments.some(selectedDept =>
          employee.departmentIds.includes(selectedDept)
        )
      })
    }
  }
)

watch(viewMode, (newMode) => {
  if (newMode === 'year') {
    scrollToCurrentMonth()
  }
})

watch(
  () => filters.value.selected.absenceTypes,
  (newVal) => {
    filters.value.selectAll.absenceTypes = newVal.length === absenceTypeOptions.length
  },
  { deep: true }
)

watch(selectedEmployeeForStats, () => {
  // Просто триггерим обновление computed свойства
  updateTrigger.value++
}, { deep: true })

// Автоматическое отображение статистики при выборе сотрудника в фильтре
watch(
  () => filters.value.selected.employees,
  (newVal) => {
    // Если выбран ровно один сотрудник
    if (newVal.length === 1) {
      const employeeId = newVal[0]
      const employee = employees.value.find(e => e.id === employeeId)
      if (employee) {
        selectedEmployeeForStats.value = employee
      }
    } else {
      // Если выбрано 0 или больше 1 сотрудника - очищаем статистику
      selectedEmployeeForStats.value = null
    }
  },
  { deep: true }
)
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

.manager-btn
  animation: pulse 2s infinite
  box-shadow: 0 2px 8px rgba(0,0,0,0.2)

@keyframes pulse
  0%
    box-shadow: 0 2px 8px rgba(0,0,0,0.2)
  50%
    box-shadow: 0 4px 16px rgba(255,193,7,0.4)
  100%
    box-shadow: 0 2px 8px rgba(0,0,0,0.2)

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
    min-width: 160px
    position: sticky
    left: 0
    background-color: #1976D2
    z-index: 2

  .department-column
    min-width: 120px
    background-color: #1976D2
    z-index: 1
    
  .day-column
    min-width: 32px
    font-size: 0.9rem
    
  .employee-name
    font-weight: 500
    background-color: #f5f5f5
    position: sticky
    left: 0
    z-index: 1

  .department-name
    background-color: #f5f5f5
    
  &.year-view
      th.month-header
        background-color: #1565C0
        font-size: 0.9rem
        padding: 8px 4px
        border-left: 1px solid rgba(255,255,255,0.2)
        border-right: 1px solid rgba(255,255,255,0.2)
        min-width: 24px
        
        &.current-month
          background-color: #FFB74D
          color: #000
          font-weight: bold
          
      th.year-day
        font-size: 0.7rem
        padding: 4px 2px
        min-width: 20px
        
        &.weekend-day
          background-color: #BBDEFB
          
        &.current-day
          background-color: #FFB74D
          font-weight: bold
          
      td.year-cell
        padding: 1px
        text-align: center
        
        &.weekend-cell
          background-color: #F5F5F5
          
        &.current-day-cell
          background-color: #FFF9C4
          
      .year-chip
        width: 16px
        height: 16px
        font-size: 0.55rem
        font-weight: bold
        justify-content: center

  .schedule-cell
    padding: 2px
    text-align: center
    
    &.clickable
      cursor: pointer
      transition: background-color 0.2s
      
      &:hover
        background-color: #e3f2fd
      
    .status-chip
      width: 100%
      justify-content: center
      font-size: 0.7rem
      
    .empty-cell
      color: #999
      font-size: 0.7rem
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

.v-card .v-card-title
  white-space: pre-wrap
  font-size: 1rem

.stats-card
  max-width: 15rem

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

// Стили для прогресс-бара загрузки
.loading-overlay
  z-index: 9999 !important
  position: fixed
  display: flex
  align-items: center
  justify-content: center
  width: 100vw
  height: 100vh
  top: 0
  left: 0

.loading-card
  max-width: 400px
  border-radius: 16px
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1)

.loading-progress
  max-width: 300px
  margin: 0 auto

.stats-container
  display: flex
  flex-direction: row
  gap: 0.75rem
</style>
