<script setup lang="ts">
import { ref, computed } from 'vue'
import { call } from 'frappe-ui'
import Form from '../components/Form.vue'
import useDataSourceStore from './data_source'

const show = defineModel({ default: false })
const sources = useDataSourceStore()

const database = ref<any>({
  database_type: 'Google Sheet',
  title: '',
  sheet_url: '',
  connection_string: '',
})

const sheetOptions = ref<string[]>([])
const selectedSheets = ref<string[]>([])

function buildConnectionString() {
  database.value.connection_string = JSON.stringify(
    {
      sheet_url: database.value.sheet_url,
      // ✅ Lưu danh sách sheet user đã chọn
      sheets: selectedSheets.value,
    },
    null,
    2,
  )
}

async function fetchSheets() {
  if (!database.value.sheet_url) return
  try {
    const result = await call(
      'insight_gsheet_connector.insight_gsheet_connector.gsheet_source.get_sheet_list',
      { sheet_url: database.value.sheet_url },
    )
    if (Array.isArray(result)) {
      sheetOptions.value = result
      // reset selection nếu URL đổi
      selectedSheets.value = []
      console.log('Sheets:', result)
    }
  } catch (e) {
    console.error('Failed to fetch sheet list', e)
  }
}

const form = ref()

const fields = [
  {
    name: 'title',
    label: 'Title',
    type: 'text',
    placeholder: 'My Google Sheet',
    required: true,
  },
  {
    name: 'sheet_url',
    label: 'Google Sheet URL',
    type: 'text',
    placeholder: 'https://docs.google.com/spreadsheets/d/…',
    required: true,
  },
]

const connected = ref<boolean | null>(null)

const fetchButton = computed(() => ({
  label: 'Lấy danh sách Sheet',
  disabled: !database.value.sheet_url,
  loading: false,
  variant: 'subtle',
  theme: 'gray',
  onClick() {
    fetchSheets()
  },
}))

const connectButton = computed(() => {
  const _button: any = {
    label: 'Test Connection',
    disabled:
      form.value?.hasRequiredFields === false ||
      sources.testing ||
      sources.creating ||
      selectedSheets.value.length === 0,
    loading: sources.testing,
    variant: 'subtle',
    theme: 'gray',
    onClick() {
      buildConnectionString()
      sources.testConnection(database.value).then((result: boolean) => {
        connected.value = Boolean(result)
      })
    },
  }

  if (sources.testing) {
    _button.label = 'Connecting...'
  } else if (connected.value) {
    _button.label = 'Connected'
    _button.variant = 'outline'
    _button.theme = 'green'
  } else if (connected.value === false) {
    _button.label = 'Failed, Retry?'
    _button.variant = 'outline'
    _button.theme = 'red'
  }

  return _button
})

const submitButton = computed(() => ({
  label: 'Add Google Sheet Source',
  disabled:
    form.value?.hasRequiredFields === false ||
    !connected.value ||
    sources.creating ||
    selectedSheets.value.length === 0,
  loading: sources.creating,
  variant: connected.value ? 'solid' : 'subtle',
  onClick() {
    buildConnectionString()
    sources.createDataSource(database.value).then(() => {
      show.value = false
    })
  },
}))
</script>

<template>
  <Dialog v-model="show" :options="{ title: 'Connect to Google Sheet' }">
    <template #body-content>
      <Form
        ref="form"
        class="flex-1"
        v-model="database"
        :fields="fields"
        :actions="[fetchButton, connectButton, submitButton]"
      />

      <!-- ✅ Danh sách sheet để chọn nhiều -->
      <div v-if="sheetOptions.length" class="mt-3 space-y-1">
        <div class="text-sm font-medium">Select Sheets to sync</div>
        <div class="max-h-40 overflow-auto border rounded p-2 space-y-1">
          <label
            v-for="name in sheetOptions"
            :key="name"
            class="flex items-center gap-2 text-sm cursor-pointer"
          >
            <input
              type="checkbox"
              :value="name"
              v-model="selectedSheets"
            />
            <span>{{ name }}</span>
          </label>
        </div>
        <div class="text-xs text-gray-500 pt-1">
          Chỉ các sheet được chọn ở đây mới được tạo thành bảng trong Insights.
        </div>
      </div>
    </template>
  </Dialog>
</template>
