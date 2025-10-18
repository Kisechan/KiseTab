<script setup lang="ts">
import ApiKeyForm from './components/ApiKeyForm.vue'
import GeolocationSettings from './components/GeolocationSettings.vue'
import { ref, onMounted } from 'vue'

const repo = "https://github.com/KiseLab/KiseTab";

const STORAGE_KEYS = {
  DEFAULT_CITY: 'kisetab_default_city',
  THEME: 'kisetab_theme',
}

const defaultCity = ref('')
const theme = ref('system')

async function readFromSync(key: string) {
  try {
    const win: any = window as any
    if (win.chrome && win.chrome.storage && win.chrome.storage.sync) {
      const r = await new Promise<any>((resolve) => win.chrome.storage.sync.get([key], (res: any) => resolve(res?.[key] || null)))
      return r
    }
  } catch (e) {
    // ignore
  }
  return localStorage.getItem(key)
}

async function saveToSync(key: string, value: string) {
  try {
    const win: any = window as any
    if (win.chrome && win.chrome.storage && win.chrome.storage.sync) {
      await new Promise<void>((resolve) => win.chrome.storage.sync.set({ [key]: value }, () => resolve()))
      return
    }
  } catch (e) {
    // ignore
  }
  localStorage.setItem(key, value)
}

async function saveDefaultCity() {
  await saveToSync(STORAGE_KEYS.DEFAULT_CITY, defaultCity.value)
}

async function saveTheme() {
  await saveToSync(STORAGE_KEYS.THEME, theme.value)
}

onMounted(async () => {
  const c = await readFromSync(STORAGE_KEYS.DEFAULT_CITY)
  defaultCity.value = c || ''
  const t = await readFromSync(STORAGE_KEYS.THEME)
  theme.value = t || 'system'
})
</script>

<template>
  <div class="popup-root">
    <div class="header">
      <div class="title"><h2>KiseTab</h2></div>
      <div class="subtitle">简洁可定制的新标签页替代 Chrome 插件</div>
    </div>

    <h2>设置</h2>

    <div class="section">
      <h3>天气与 API</h3>
      <ApiKeyForm />
    </div>

    <div class="section">
      <h3>定位</h3>
      <GeolocationSettings />
    </div>

    <div class="footer">
      <span><a :href="repo" target="_blank" class="footer-link">GitHub</a> | © KiseLab</span>
    </div>
  </div>
</template>

<style scoped>
.popup-root {
  width: 320px;
  padding: 12px;
  box-sizing: border-box;
  font-family: Inter, system-ui, Arial, sans-serif;
  background: var(--bg-light);
  color: var(--text-dark);
  border-radius: 12px;
}
.header { margin-bottom: 8px }
.title { font-size: 16px; font-weight: 700; margin-bottom: 4px }
.subtitle { font-size: 12px; color: rgba(34,34,34,0.7) }
.section { background: rgba(255,255,255,0.9); padding: 8px; border-radius: 8px; margin-bottom: 8px }
.section h3 { margin: 0 0 8px 0; font-size: 13px }
.row { display:flex; gap:8px; align-items:center }
.row input, .row select { flex:1; padding:6px 8px; border-radius:8px; border:1px solid rgba(0,0,0,0.08) }
.row button { padding:6px 10px; border-radius:8px; background:#4f46e5; color:white; border:none }
.footer { margin-top:8px; font-size:12px; color: rgba(34, 34, 34, 0.55) }
</style>

