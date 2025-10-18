<template>
  <div class="weather-card" role="region" aria-label="当前天气">
    <template v-if="loading">
      <div class="msg">加载中…</div>
    </template>

    <template v-else-if="disabled">
      <div class="msg">天气服务已禁用或无法获取地理位置</div>
      <div class="actions">
        <button class="retry" @click="retry">重试</button>
      </div>
    </template>

    <template v-else-if="error">
      <div class="msg">{{ error }}</div>
      <div class="actions">
        <button class="retry" @click="retry">重试</button>
      </div>
    </template>

    <template v-else>
      <div class="top">
        <div class="location">{{ cityDisplay }}</div>
        <div class="cond">{{ condition }}</div>
      </div>
      <div class="bottom">
        <img v-if="iconUrl" :src="iconUrl" alt="天气图标" class="icon" />
        <div class="temp">{{ Math.round(temp) }}<span>°C</span></div>
      </div>
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";

const loading = ref(true);
const disabled = ref(false);
const error = ref<string | null>(null);

const city = ref("");
const temp = ref(28);
const condition = ref("");
const icon = ref<string | null>(null);

const iconUrl = ref<string | null>(null);

let abortCtrl: AbortController | null = null;

// API key is read at runtime from chrome.storage.local or localStorage.
const apiKey = ref<string>(import.meta.env.VITE_WEATHER_API_KEY || "");

// remember last successful coords so we don't re-request geolocation on every key change
const coords = ref<{ lat: number; lon: number } | null>(null);

// storage change listener reference so we can remove it on unmount
let storageListener: ((changes: any, areaName: string) => void) | null = null;

async function getGeolocationMethod() {
  try {
    const win: any = window as any;
    if (win.chrome && win.chrome.storage && win.chrome.storage.local) {
      const result = await win.chrome.storage.local.get(['geolocation_method']);
      return result.geolocation_method || 'ip';
    }
  } catch (e) {
    // ignore
  }
  return localStorage.getItem('geolocation_method') || 'ip';
}


// helper to read stored key
async function readApiKeyFromStorage() {
  try {
    const win: any = window as any;
    if (win.chrome && win.chrome.storage) {
      if (win.chrome.storage.sync) {
        return await new Promise<string>((resolve) => {
          win.chrome.storage.sync.get(["VITE_WEATHER_API_KEY"], (items: any) => {
            resolve(items?.VITE_WEATHER_API_KEY || localStorage.getItem("VITE_WEATHER_API_KEY") || "");
          });
        });
      }
      if (win.chrome.storage.local) {
        return await new Promise<string>((resolve) => {
          win.chrome.storage.local.get(["VITE_WEATHER_API_KEY"], (items: any) => {
            resolve(items?.VITE_WEATHER_API_KEY || localStorage.getItem("VITE_WEATHER_API_KEY") || "");
          });
        });
      }
    }
  } catch (e) {
    // ignore
  }
  return localStorage.getItem("VITE_WEATHER_API_KEY") || "";
}

const CACHE_KEY = 'weather_cache';
const CACHE_TTL_MS = 15 * 60 * 1000; // 15 mins

type WeatherCache = {
  lat: number;
  lon: number;
  ts: number;
  payload: {
    temp: number;
    condition: string;
    icon: string | null;
    iconUrl: string | null;
    city: string;
  };
};

async function readCache(): Promise<WeatherCache | null> {
  try {
    const win: any = window as any;
    if (win.chrome && win.chrome.storage && win.chrome.storage.local) {
      const res = await new Promise<any>((resolve) => {
        win.chrome.storage.local.get([CACHE_KEY], (r: any) => resolve(r?.[CACHE_KEY] || null));
      });
      return res;
    }
  } catch (e) {
    // ignore
  }
  try {
    const raw = localStorage.getItem(CACHE_KEY);
    return raw ? JSON.parse(raw) : null;
  } catch (e) {
    return null;
  }
}

async function writeCache(c: WeatherCache) {
  try {
    const win: any = window as any;
    if (win.chrome && win.chrome.storage && win.chrome.storage.local) {
      await new Promise<void>((resolve) => {
        win.chrome.storage.local.set({ [CACHE_KEY]: c }, () => resolve());
      });
      return;
    }
  } catch (e) {
    // ignore
  }
  try {
    localStorage.setItem(CACHE_KEY, JSON.stringify(c));
  } catch (e) {
    // ignore
  }
}

function coordsMatch(aLat: number, aLon: number, bLat: number, bLon: number) {
  const THRESH = 0.02; // ~2km tolerance
  return Math.abs(aLat - bLat) <= THRESH && Math.abs(aLon - bLon) <= THRESH;
}

// 国家映射
import { COUNTRY_MAP, PROVINCE_MAP } from "../utils/regionMaps";


function makeIconUrl(code: string) {
  return `https://openweathermap.org/img/wn/${code}@2x.png`;
}

async function fetchWeather(lat: number, lon: number) {
  // remember coords for subsequent refetches
  coords.value = { lat, lon };
  // Check cache first
  try {
    const cached = await readCache();
    if (cached && Date.now() - cached.ts < CACHE_TTL_MS && coordsMatch(lat, lon, cached.lat, cached.lon)) {
      temp.value = cached.payload.temp;
      condition.value = cached.payload.condition;
      icon.value = cached.payload.icon;
      iconUrl.value = cached.payload.iconUrl;
      city.value = cached.payload.city;
      loading.value = false;
      return;
    }
  } catch (e) {

  }
  if (!apiKey.value) {
    error.value = "未配置天气 API Key (请在扩展弹出窗口中设置)";
    loading.value = false;
    return;
  }

  abortCtrl = new AbortController();
  const signal = abortCtrl.signal;

  try {
  const onecallUrl = `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${apiKey.value}&lang=zh_cn`;
    const res = await fetch(onecallUrl, { signal });
    if (!res.ok) throw new Error(`天气 API 返回 ${res.status}`);
    const data = await res.json();

    const toCelsius = (v: number) => {
      // 如果温度看起来像开尔文（>100），则转换为摄氏度；否则视为已是摄氏度
      return typeof v === "number" ? (v > 100 ? v - 273.15 : v) : v;
    };

    if (data && data.current) {
      const cur = data.current;
      temp.value = toCelsius(cur.temp);
      const w = Array.isArray(cur.weather) && cur.weather[0];
      condition.value = (w && (w.description || w.main)) || "暂无信息";
      icon.value = w && w.icon ? w.icon : null;
      iconUrl.value = icon.value ? makeIconUrl(icon.value) : null;
    } else if (data && data.main) {
      // 当前天气接口返回结构
      temp.value = toCelsius(data.main.temp);
      const w = Array.isArray(data.weather) && data.weather[0];
      condition.value = (w && (w.description || w.main)) || "暂无信息";
      icon.value = w && w.icon ? w.icon : null;
      iconUrl.value = icon.value ? makeIconUrl(icon.value) : null;
    } else {
      throw new Error("天气数据格式异常");
    }

    // reverse geocoding to get city name
    try {
  const revUrl = `https://api.openweathermap.org/geo/1.0/reverse?lat=${lat}&lon=${lon}&limit=1&appid=${apiKey.value}`;
      const r2 = await fetch(revUrl, { signal });
      if (r2.ok) {
        const arr = await r2.json();
        if (Array.isArray(arr) && arr[0]) {
          const item = arr[0];
          // 优先选择 local_names.zh
          const localNames = item.local_names || {};
          const cityName = localNames.zh || localNames["zh-CN"] || item.name || "";
          const countryCode = item.country || "";
          const countryMapped = COUNTRY_MAP[countryCode];
          const stateRaw = item.state || "";
          const provinceMapped = PROVINCE_MAP[stateRaw];

          const countryPart = countryMapped || countryCode || "";
          const provincePart = provinceMapped || stateRaw || "";

          if (countryCode === 'CN') {
            const parts = [] as string[];
            if (countryPart) parts.push(countryPart);
            if (provincePart) parts.push(provincePart);
            if (cityName) parts.push(cityName);
            city.value = parts.join(' · ');
          } else if (countryPart) {
            if (cityName) {
              city.value = `${countryPart} · ${cityName}`;
            } else {
              city.value = countryPart;
            }
          } else {
            city.value = cityName || "";
          }
        }
      }
    } catch (e) {
      // silently ignore reverse geocoding failure
    }

    loading.value = false;

    // write cache
    try {
      const payload = {
        temp: temp.value,
        condition: condition.value,
        icon: icon.value,
        iconUrl: iconUrl.value,
        city: city.value,
      };
      await writeCache({ lat, lon, ts: Date.now(), payload });
    } catch (e) {
      // ignore cache write errors
    }
  } catch (e: any) {
    if (e.name === "AbortError") return;
    error.value = e.message || String(e);
    loading.value = false;
  }
}

async function fetchCoordsByIp() {
  try {
    const res = await fetch('https://ipapi.co/latlong/');
    if (!res.ok) throw new Error('IP-based geolocation failed');
    const text = await res.text();
    const [lat, lon] = text.split(',').map(Number);
    if (lat && lon) {
      return { lat, lon };
    }
    throw new Error('Invalid coordinates from IP API');
  } catch (e) {
    error.value = '无法通过 IP 获取地理位置';
    loading.value = false;
    return null;
  }
}

function getLocationAndFetch() {
  getGeolocationMethod().then(method => {
    if (method === 'ip') {
      fetchCoordsByIp().then(coords => {
        if (coords) {
          fetchWeather(coords.lat, coords.lon);
        }
      });
    } else {
      // Permission-based
      if (!("geolocation" in navigator)) {
        disabled.value = true;
        loading.value = false;
        return;
      }

      navigator.geolocation.getCurrentPosition(
        (pos) => {
          const lat = pos.coords.latitude;
          const lon = pos.coords.longitude;
          coords.value = { lat, lon };
          fetchWeather(lat, lon);
        },
        (err) => {
          // permission denied or other errors
          disabled.value = true;
          loading.value = false;
        },
        { enableHighAccuracy: false, timeout: 8000, maximumAge: 1000 * 60 * 10 }
      );
    }
  });
}

onMounted(() => {
  // start with defaults visible while loading
  loading.value = true;
  disabled.value = false;
  error.value = null;

  // load API key from storage first
  readApiKeyFromStorage().then((k) => {
    apiKey.value = k || apiKey.value;
    if (!apiKey.value) {
      // no key yet – show hint but keep listening for changes
      loading.value = false;
      error.value = "未配置天气 API Key (请在扩展弹出窗口中设置)";
      return;
    }

    // if we already have coords (unlikely on first mount) use them, else request location
    getLocationAndFetch();
  });

  // listen for storage changes so popup can update the key and trigger a refetch
  const win: any = window as any;
  storageListener = (changes: any, areaName: string) => {
    if (areaName !== "local") return;
    if (changes.VITE_WEATHER_API_KEY || changes.geolocation_method) {
      // reset states and refetch
      loading.value = true;
      error.value = null;
      disabled.value = false;
      
      readApiKeyFromStorage().then(key => {
        apiKey.value = key || '';
        getLocationAndFetch();
      });
    }
  };

  if (win.chrome && win.chrome.storage && win.chrome.storage.onChanged && storageListener) {
    win.chrome.storage.onChanged.addListener(storageListener);
  }
});

onUnmounted(() => {
  if (abortCtrl) abortCtrl.abort();
  const win: any = window as any;
  if (win.chrome && win.chrome.storage && win.chrome.storage.onChanged && storageListener) {
    try {
      win.chrome.storage.onChanged.removeListener(storageListener);
    } catch (e) {
      // ignore
    }
  }
});

const cityDisplay = computed(() => city.value);

function retry() {
  // reset flags and try again
  loading.value = true;
  disabled.value = false;
  error.value = null;

  if (coords.value) {
    fetchWeather(coords.value.lat, coords.value.lon);
  } else {
    getLocationAndFetch();
  }
}
</script>

<style scoped>
.weather-card {
  width: 210px;
  padding: 12px 14px;
  border-radius: 12px;
  background: linear-gradient(
    180deg,
    rgba(255, 255, 255, 0.92),
    rgba(255, 255, 255, 0.86)
  );
  box-shadow: 0 10px 30px rgba(139, 92, 246, 0.08),
    0 2px 8px rgba(0, 0, 0, 0.04);
  color: var(--text-dark);
  border: 1px solid rgba(139, 92, 246, 0.06);
  backdrop-filter: blur(6px);
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.weather-card .top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}
.location {
  font-size: 12px;
  color: rgba(34, 34, 34, 0.7);
}
.cond {
  font-size: 13px;
  font-weight: 600;
  color: var(--accent);
}
.weather-card .bottom {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  gap: 8px;
}
.temp {
  font-size: 26px;
  font-weight: 700;
  color: var(--text-dark);
}
.temp span {
  font-size: 12px;
  margin-left: 2px;
}
.icon {
  width: 44px;
  height: 44px;
}
.msg {
  font-size: 13px;
  color: rgba(34, 34, 34, 0.75);
  text-align: center;
}

.actions {
  display: flex;
  justify-content: center;
  margin-top: 8px;
}
.retry {
  background: linear-gradient(90deg, var(--accent), #7b61ff);
  color: white;
  border: none;
  padding: 6px 10px;
  border-radius: 8px;
  font-size: 13px;
  cursor: pointer;
}
.retry:active { transform: translateY(1px); }

/* small screen: hide */
@media (max-width: 720px) {
  .weather-card {
    display: none;
  }
}
</style>
