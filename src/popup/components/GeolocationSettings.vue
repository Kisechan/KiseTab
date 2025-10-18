<template>
  <div class="setting-row">
    <label for="geo-toggle" class="label">定位方式</label>
    <div class="toggle-switch-container">
      <span class="value">{{ isIpBased ? 'IP 定位' : '浏览器定位' }}</span>
      <div class="toggle-switch">
        <input type="checkbox" id="geo-toggle" v-model="isIpBased" @change="saveSetting" />
        <label for="geo-toggle" class="slider"></label>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';

const isIpBased = ref(true); // Default to IP-based
const STORAGE_KEY = 'geolocation_method';

async function saveSetting() {
  const method = isIpBased.value ? 'ip' : 'permission';
  try {
    const win: any = window as any;
    if (win.chrome?.storage?.sync) {
      await win.chrome.storage.sync.set({ [STORAGE_KEY]: method });
      return
    }
    if (win.chrome?.storage?.local) {
      await win.chrome.storage.local.set({ [STORAGE_KEY]: method });
      return
    }
    localStorage.setItem(STORAGE_KEY, method);
  } catch (e) {
    console.error('Failed to save geolocation setting:', e);
  }
}

onMounted(async () => {
  let method = 'ip'; // Default
  try {
    const win: any = window as any;
    if (win.chrome?.storage?.sync) {
      const result = await win.chrome.storage.sync.get([STORAGE_KEY]);
      method = result[STORAGE_KEY] || 'ip';
    } else if (win.chrome?.storage?.local) {
      const result = await win.chrome.storage.local.get([STORAGE_KEY]);
      method = result[STORAGE_KEY] || 'ip';
    } else {
      method = localStorage.getItem(STORAGE_KEY) || 'ip';
    }
  } catch (e) {
    method = localStorage.getItem(STORAGE_KEY) || 'ip';
  }
  isIpBased.value = method === 'ip';
});
</script>

<style scoped>
.setting-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  font-size: 13px;
}
.label {
  font-weight: 500;
}
.toggle-switch-container {
  display: flex;
  align-items: center;
  gap: 10px;
}
.value {
  color: rgba(34, 34, 34, 0.7);
}
.toggle-switch {
  position: relative;
  display: inline-block;
  width: 40px;
  height: 22px;
}
.toggle-switch input {
  opacity: 0;
  width: 0;
  height: 0;
}
.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #ccc;
  transition: .4s;
  border-radius: 22px;
}
.slider:before {
  position: absolute;
  content: "";
  height: 16px;
  width: 16px;
  left: 3px;
  bottom: 3px;
  background-color: white;
  transition: .4s;
  border-radius: 50%;
}
input:checked + .slider {
  background-color: #4f46e5;
}
input:checked + .slider:before {
  transform: translateX(18px);
}
</style>
