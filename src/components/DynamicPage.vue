<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue';
import { useRoute } from 'vue-router';
import { useStore, type Config } from '../store'; // 路径按实际调整
import axios from 'axios';

const store = useStore();
const route = useRoute();

/* ----------------- state ----------------- */
const config = ref<Config>({} as Config);
const pageMap = ref<Record<string, string>>({});
const html = ref('');

/* -------------- helpers ------------------ */
const hasCache = (ns: string, key: string) =>
  ns in store.state.cache && key in store.state.cache[ns];

const loadPage = async (page: string, cb?: (h: string) => void) => {
  if (!pageMap.value[page]) return;

  const url = pageMap.value[page];
  const ns = 'page';

  if (hasCache(ns, url)) {
    cb?.(store.state.cache[ns][url]);
    return;
  }

  try {
    const { data } = await axios.get<string>(url);
    store.commit({ type: 'put', namespace: ns, key: url, value: data });
    cb?.(data);
  } catch (err: any) {
    if (err?.response?.status === 404) cb?.(`请求的页面 ${url} 不存在`);
  }
};

const updatePageCallback = (h: string) => {
  if (html.value !== h) html.value = h;
};

/* ----------------- lifecycle ----------------- */
onMounted(() => {
  config.value = store.state.config;
  // 建立 url → path 的映射
  config.value.header.forEach((p) => {
    if (!p.path.startsWith('redirect:')) pageMap.value[p.url] = p.path;
  });

  // 首屏
  loadPage(`/${route.params.page as string}`, updatePageCallback);

  // 预缓存所有静态页
  Object.values(pageMap.value).forEach((p) => loadPage(p));
});

/* --------------- watchers ------------------ */
watch(
  () => route.params.page,
  (val) => loadPage(`/${val as string}`, updatePageCallback)
);
</script>

<template>
  <div v-html="html"></div>
</template>

<style scoped>
/* 保持原样 */
</style>
