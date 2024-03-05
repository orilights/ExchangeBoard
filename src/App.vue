<script setup lang="ts">
import { SettingType, Settings } from '@orilight/vue-settings'

const exchangeRate = ref<{ [key: string]: number }>({})
const statusText = ref('数据获取中')
const api = 'https://api.amarea.cn/exchange/CNY'
const currencyList = ['USD', 'JPY', 'HKD', 'SGD', 'EUR', 'TWD']
const currencyName: { [key: string]: string } = {
  USD: '美元',
  JPY: '日元',
  HKD: '港元',
  SGD: '新加坡元',
  EUR: '欧元',
  TWD: '新台币',
}
const boardList = ref(['USD:1 HKD:10 JPY:100', 'USD:3 HKD:25 JPY:500', 'JPY:200 JPY:400 JPY:800'])
const boardData = ref('')
const setting = new Settings('exchange-board')

function fetchData() {
  fetch(api)
    .then(res => res.json())
    .then((data) => {
      if (data.code !== 0)
        throw new Error(data.msg)
      exchangeRate.value = data.rates
      statusText.value = `数据更新日期: ${data.date}`
    })
    .catch((err) => {
      console.error(err)
      statusText.value = `数据获取失败: ${err.message}`
    })
}

function reverse(rate: number | string) {
  return Number((1 / Number(rate)).toFixed(3))
}

function updateBoard() {
  boardList.value = boardData.value.split('\n')
}

function exportFile(data: string, filename = 'export-{ts}.json') {
  const blob = new Blob([data], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename.replace('{ts}', Date.now().toString())
  a.click()
  URL.revokeObjectURL(url)
}

function importData() {
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = 'application/json'
  input.onchange = (e) => {
    const file = (e.target as HTMLInputElement).files?.[0]
    if (file) {
      const reader = new FileReader()
      reader.onload = (e) => {
        const data = JSON.parse(e.target?.result as string)
        boardList.value = data
        boardData.value = data.join('\n')
      }
      reader.readAsText(file)
    }
  }
  input.click()
}

function exportData() {
  exportFile(JSON.stringify(boardList.value), 'exchange-board-{ts}.json')
}

function resetData() {
  setting.clear()
  location.reload()
}

onMounted(() => {
  setting.register('boardList', boardList, SettingType.Json)
  fetchData()
  boardData.value = boardList.value.join('\n')
})
</script>

<template>
  <div class="p-4">
    <div class="p-4 bg-green-200 mb-4 rounded-lg">
      <h1 class="font-bold text-4xl pb-4">
        汇率看板
      </h1>
      <div>
        {{ statusText }}
      </div>
    </div>
    <div class="flex flex-wrap gap-4 mb-4">
      <div
        v-for="currency in currencyList" :key="currency"
        class="rounded-lg bg-blue-200 inline-block p-2 hover:bg-blue-300  transition-all hover:scale-105"
      >
        <div class="font-bold text-lg">
          {{ currencyName[currency] }}
        </div>
        1 CNY = {{ exchangeRate[currency] }} {{ currency }}<br>
        1 {{ currency }} = {{ reverse(exchangeRate[currency]) }} CNY
      </div>
    </div>

    <div class="flex flex-wrap gap-4 mb-4">
      <div
        v-for="board in boardList" :key="board"
        class="rounded-lg bg-orange-200 inline-block p-2 hover:bg-orange-300  transition-all hover:scale-105"
      >
        <div v-for="i in board.split(' ')" :key="i">
          {{ i.split(':')[1] }} {{ i.split(':')[0] }} =
          {{ (Number(i.split(':')[1]) / exchangeRate[i.split(':')[0]]).toFixed(3) }} CNY
        </div>
      </div>
    </div>
    <details class="bg-gray-200 rounded-lg p-4">
      <summary>编辑看板</summary>
      <div class="mt-2">
        <textarea v-model="boardData" rows="5" class="w-full p-2 rounded-lg" />
        <div class="flex flex-wrap gap-2">
          <button class="bg-white px-4 py-2 rounded-lg transition-all hover:scale-105" @click="updateBoard">
            保存
          </button>
          <button class="bg-white px-4 py-2 rounded-lg transition-all hover:scale-105" @click="exportData">
            导出
          </button>
          <button class="bg-white px-4 py-2 rounded-lg transition-all hover:scale-105" @click="importData">
            导入
          </button>
          <button class="bg-white px-4 py-2 rounded-lg transition-all hover:scale-105" @click="resetData">
            重置
          </button>
        </div>
      </div>
    </details>
  </div>
</template>
