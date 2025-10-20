<script setup lang="ts">
import ApiKeyForm from './components/ApiKeyForm.vue'
import GeolocationSettings from './components/GeolocationSettings.vue'
import UpdateCheck from './components/UpdateCheck.vue'
import { ref, onMounted } from 'vue'

const repo = "https://github.com/KiseLab/KiseTab";

const STORAGE_KEYS = {
  DEFAULT_CITY: 'kisetab_default_city',
  THEME: 'kisetab_theme',
  UPDATE_CHECK: 'kisetab_update_check'
}

const defaultCity = ref('')
const theme = ref('system')
// update logic moved to UpdateCheck.vue

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

    <div class="section">
      <UpdateCheck />
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
.update-banner {
  background: #e6f7ff;
  border: 1px solid #91d5ff;
  color: #0050b3;
  padding: 8px;
  border-radius: 6px;
  margin-top: 8px;
  font-size: 13px;
  text-align: center;
}
.toggle input {
  margin-right: 6px;
}

/* setting row / toggle styles (match GeolocationSettings) */
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

