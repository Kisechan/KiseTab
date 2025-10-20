<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'

const STORAGE_KEY = 'kisetab_update_check'
const localVersion = chrome.runtime.getManifest().version
const updateCheckEnabled = ref(true)
const latestVersion = ref<string | null>(null)
const isUpdateAvailable = computed(() => !!latestVersion.value && latestVersion.value !== localVersion)

async function readFromSync(key: string) {
  try {
    const win: any = window as any
    if (win.chrome && win.chrome.storage && win.chrome.storage.sync) {
      return await new Promise<any>((resolve) => win.chrome.storage.sync.get([key], (res: any) => resolve(res?.[key] || null)))
    }
  } catch {}
  return localStorage.getItem(key)
}

async function saveToSync(key: string, value: string) {
  try {
    const win: any = window as any
    if (win.chrome && win.chrome.storage && win.chrome.storage.sync) {
      await new Promise<void>((resolve) => win.chrome.storage.sync.set({ [key]: value }, () => resolve()))
      return
    }
  } catch {}
  localStorage.setItem(key, value)
}

async function loadPref() {
  const v = await readFromSync(STORAGE_KEY)
  updateCheckEnabled.value = v === null ? true : (v === 'true')
}

async function savePref() {
  await saveToSync(STORAGE_KEY, updateCheckEnabled.value ? 'true' : 'false')
}

async function checkUpdate() {
  try {
    const resp = await fetch('https://api.github.com/repos/KiseLab/KiseTab/releases/latest')
    if (!resp.ok) return
    const data = await resp.json()
    const tag = data.tag_name.replace(/^v/, '')
    latestVersion.value = tag
  } catch {}
}

onMounted(async () => {
  await loadPref()
  if (updateCheckEnabled.value) {
    await checkUpdate()
  }
})
</script>

<template>
  <div class="update-check-root">
    <div class="setting-row">
      <label for="update-toggle" class="label">更新检查</label>
      <div class="toggle-switch-container">
        <span class="value">{{ updateCheckEnabled ? '已启用' : '已禁用' }}</span>
        <div class="toggle-switch">
          <input type="checkbox" id="update-toggle" v-model="updateCheckEnabled" @change="savePref" />
          <label for="update-toggle" class="slider"></label>
        </div>
      </div>
    </div>

    <div v-if="isUpdateAvailable" class="update-card">
      <div class="update-card-content">
        <div class="badge">New</div>
        <div class="meta">
          <div class="title">发现新版本</div>
          <div class="version">{{ latestVersion }} → 当前 {{ localVersion }}</div>
        </div>
        <div class="actions">
          <a class="btn" href="https://github.com/KiseLab/KiseTab/releases/latest" target="_blank">查看 Release</a>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.update-card {
  margin-top: 10px;
  background: linear-gradient(180deg, rgba(79,70,229,0.06), rgba(79,70,229,0.02));
  border: 1px solid rgba(79,70,229,0.12);
  border-radius: 10px;
  padding: 10px;
}
.update-card-content { display:flex; align-items:center; gap:12px }
.badge { background:#4f46e5; color:white; padding:6px 8px; border-radius:8px; font-weight:600 }
.meta .title { font-weight:700 }
.meta .version { font-size:12px; color:rgba(0,0,0,0.6) }
.btn { background:#4f46e5; color:#fff; padding:6px 10px; border-radius:8px; text-decoration:none }

/* reuse toggle styles from App.vue */
.setting-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 6px 0;
}
.label {
  font-size: 13px;
  color: var(--text-dark);
  flex: 0 0 auto;
}
.toggle-switch-container {
  display: flex;
  align-items: center;
  gap: 10px;
}
.value {
  font-size: 13px;
  color: rgba(34,34,34,0.7);
}
.toggle-switch {
  position: relative;
  width: 44px;
  height: 26px;
}
.toggle-switch input { display: none; }
.toggle-switch .slider {
  position: absolute;
  cursor: pointer;
  top: 0; left: 0; right: 0; bottom: 0;
  background-color: #ccc;
  border-radius: 999px;
  transition: background-color 0.2s;
}
.toggle-switch .slider:before {
  content: '';
  position: absolute;
  height: 20px;
  width: 20px;
  left: 3px;
  top: 3px;
  background: #fff;
  border-radius: 50%;
  box-shadow: 0 1px 2px rgba(0,0,0,0.12);
  transition: transform 0.22s ease;
}
.toggle-switch input:checked + .slider {
  background-color: #4f46e5;
}
.toggle-switch input:checked + .slider:before {
  transform: translateX(18px);
}
</style>