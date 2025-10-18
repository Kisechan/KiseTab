<template>
  <div class="p-3">
    <label class="block text-xs font-medium text-gray-700 mb-2">OpenWeather API 密钥</label>
    <div class="flex gap-2">
      <input v-model="key" class="flex-1 px-3 py-2 border rounded-md text-sm shadow-sm focus:outline-none focus:ring-2 focus:ring-indigo-500" placeholder="请输入你的 OpenWeather API Key" />
      <button @click="save" type="button" class="btn-save inline-flex items-center bg-indigo-600 text-white text-sm rounded-md">保存</button>
    </div>
    
    <p class="mt-2 text-xs text-gray-500">仅保存在你的浏览器，不会上传到任何服务器。</p>
    <p class="mt-2 text-xs text-gray-500">参考: <a href="https://openweathermap.org/api" target="_blank" class="ref-link">OpenWeather API</a></p>

    <p v-if="statusMsg" :class="['status', statusType]">{{ statusMsg }}</p>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

const key = ref('')
const STORAGE_KEY = 'VITE_WEATHER_API_KEY'

const statusMsg = ref('')
const statusType = ref('info')
let statusTimer: number | null = null

function setStatus(msg: string, type: string = 'success', timeout = 2000) {
  statusMsg.value = msg
  statusType.value = type
  if (statusTimer) {
    clearTimeout(statusTimer)
  }
  statusTimer = window.setTimeout(() => {
    statusMsg.value = ''
    statusTimer = null
  }, timeout)
}

function getStoredKey(): Promise<string | null> {
  return new Promise((resolve) => {
    try {
      const win: any = window as any;
      if (win.chrome?.storage?.sync) {
        win.chrome.storage.sync.get([STORAGE_KEY], (res: any) => {
          resolve(res?.[STORAGE_KEY] ?? null)
        })
        return
      }
      if (win.chrome?.storage?.local) {
        win.chrome.storage.local.get([STORAGE_KEY], (res: any) => {
          resolve(res?.[STORAGE_KEY] ?? null)
        })
        return
      }
      resolve(localStorage.getItem(STORAGE_KEY))
    } catch (e) {
      resolve(localStorage.getItem(STORAGE_KEY))
    }
  })
}

async function save() {
  const cur = await getStoredKey()
  if (cur === key.value) {
    // nothing to do
    setStatus('已保存（未发生更改）', 'info')
    return
  }

  try {
    const win: any = window as any;
    if (win.chrome?.storage?.sync) {
      win.chrome.storage.sync.set({ [STORAGE_KEY]: key.value })
      setStatus('已保存', 'success')
      return
    }
    if (win.chrome?.storage?.local) {
      win.chrome.storage.local.set({ [STORAGE_KEY]: key.value })
      setStatus('已保存', 'success')
      return
    }
    localStorage.setItem(STORAGE_KEY, key.value)
    setStatus('已保存（已使用回退存储）', 'success')
  } catch (e) {
    // fallback
    localStorage.setItem(STORAGE_KEY, key.value)
    setStatus('已保存（已使用回退存储）', 'success')
  }
}

onMounted(async () => {
  const v = await getStoredKey()
  key.value = v || ''
})
</script>

<style scoped>
.flex { display:flex }
.gap-2 { gap: 8px }
.p-3 { padding: 12px }
.text-sm { font-size: 13px }
.text-xs { font-size: 12px }
.text-gray-500 { color: rgba(34,34,34,0.6) }
.text-indigo-600 { color: #4f46e5 }
.mb-2 { margin-bottom: 8px }

input {
  width: 100%;
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid rgba(0,0,0,0.08);
  background: #fff;
  box-shadow: 0 1px 0 rgba(0,0,0,0.02);
}

button {
  cursor: pointer;
}

.bg-indigo-600 { background: #4f46e5 }
.bg-indigo-500 { background: #6366f1 }
.text-white { color: #fff }
.rounded-md { border-radius: 8px }
.px-3 { padding-left: 12px; padding-right: 12px }
.py-2 { padding-top: 8px; padding-bottom: 8px }

.ref-link { color: #4f46e5; text-decoration: underline; }

.btn-save {
  min-width: 68px; /* ensure enough width to avoid wrapping */
  padding: 8px 12px;
  white-space: nowrap;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.status { margin-top: 8px; font-size: 12px }
.status.success { color: #065f46 } /* green-800 */
.status.info { color: #334155 } /* slate-700 */
</style>
