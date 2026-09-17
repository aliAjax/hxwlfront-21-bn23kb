<script setup lang="ts">
import { computed, reactive, ref } from "vue";

type Field = {
  key: string;
  label: string;
  type?: "number" | "date" | "select";
  options?: readonly string[];
};

type RecordItem = {
  id: string;
  status: string;
  notes: string;
  createdAt: string;
  [key: string]: string | number;
};

const project = {
  "number": 21,
  "folder": "hxwl/frontend/hxwlfront-21",
  "framework": "vue",
  "title": "油站网点地图管理",
  "subtitle": "维护油站位置、营业状态和库存摘要。",
  "industry": "石油",
  "stack": [
    "Vue3",
    "Vite",
    "TypeScript",
    "Element Plus",
    "Leaflet"
  ],
  "storageKey": "hxwlfront-21-station-map",
  "formTitle": "新增油站",
  "primaryAction": "保存油站",
  "entityLabel": "油站",
  "statuses": [
    "营业中",
    "暂停营业",
    "库存紧张"
  ],
  "filters": [
    "全部区域",
    "东区",
    "西区",
    "机场线"
  ],
  "fields": [
    {
      "key": "station",
      "label": "油站名称"
    },
    {
      "key": "area",
      "label": "区域",
      "type": "select",
      "options": [
        "东区",
        "西区",
        "机场线"
      ]
    },
    {
      "key": "stock",
      "label": "库存摘要L",
      "type": "number"
    },
    {
      "key": "manager",
      "label": "负责人"
    }
  ],
  "records": [
    {
      "station": "东区一站",
      "area": "东区",
      "stock": 36000,
      "manager": "刘站长",
      "status": "营业中",
      "notes": "库存正常"
    },
    {
      "station": "机场快线站",
      "area": "机场线",
      "stock": 9000,
      "manager": "王站长",
      "status": "库存紧张",
      "notes": "柴油待补"
    }
  ],
  "metricLabels": [
    "油站数",
    "营业中",
    "库存紧张"
  ]
} as const;

const fields = project.fields as readonly Field[];
const statuses = [...project.statuses];

const STATUS_OPEN = "营业中";
const STATUS_TIGHT = "库存紧张";
const STOCK_LOW_THRESHOLD = 10000;

function stockOf(value: unknown): number {
  const stock = Number(value);
  return Number.isFinite(stock) ? stock : NaN;
}

// 库存低于阈值（或库存无效）时按低库存处理，只允许“库存紧张”
function isLowStock(value: unknown): boolean {
  const stock = stockOf(value);
  return Number.isNaN(stock) || stock < STOCK_LOW_THRESHOLD;
}

// 库存与状态的闭环规则：低库存只能是“库存紧张”；高库存不得保留“库存紧张”，自动转“营业中”
function normalizeStatus(stock: unknown, status: string): string {
  if (isLowStock(stock)) return STATUS_TIGHT;
  if (status === STATUS_TIGHT) return STATUS_OPEN;
  return status;
}

function createBlank() {
  return Object.fromEntries(fields.map((field) => [field.key, field.type === "number" ? 0 : ""]));
}

function loadRecords(): RecordItem[] {
  const raw = localStorage.getItem(project.storageKey);
  if (!raw) {
    return project.records.map((record, index) => ({
      ...record,
      status: normalizeStatus(record.stock, record.status),
      id: `seed-${index + 1}`,
      createdAt: new Date(Date.now() - index * 86400000).toISOString()
    })) as RecordItem[];
  }
  try {
    const parsed = JSON.parse(raw) as RecordItem[];
    if (!Array.isArray(parsed)) return [];
    // 读取旧数据时按闭环规则就地修正，并把修正结果回写 localStorage
    let changed = false;
    const fixed = parsed.map((record) => {
      const status = normalizeStatus(record.stock, record.status);
      if (status === record.status) return record;
      changed = true;
      return { ...record, status };
    });
    if (changed) localStorage.setItem(project.storageKey, JSON.stringify(fixed));
    return fixed;
  } catch {
    return [];
  }
}

const records = ref<RecordItem[]>(loadRecords());
const form = reactive<Record<string, string | number>>(createBlank());
const formStatus = ref<string>(STATUS_OPEN);
const formError = ref("");
const cardErrors = reactive<Record<string, string>>({});
const note = ref("");
const filter = ref(project.filters[0]);

const filteredRecords = computed(() => {
  if (filter.value.startsWith("全部")) return records.value;
  return records.value.filter((record) => Object.values(record).includes(filter.value));
});

const metrics = computed(() => {
  const total = records.value.length;
  const open = records.value.filter((record) => record.status === STATUS_OPEN).length;
  const tight = records.value.filter((record) => record.status === STATUS_TIGHT).length;
  return [total, open, tight];
});

const chartRows = computed(() => statuses.map((status) => ({
  status,
  value: records.value.filter((record) => record.status === status).length
})));

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

function persist() {
  localStorage.setItem(project.storageKey, JSON.stringify(records.value));
}

function nextStatus(status: string) {
  const index = statuses.indexOf(status);
  return statuses[(index + 1) % statuses.length];
}

function primaryText(record: RecordItem) {
  const first = fields[0];
  const second = fields[1];
  return [record[first.key], record[second.key]].filter(Boolean).join(" / ") || project.entityLabel;
}

function clearFormError() {
  formError.value = "";
}

function submit() {
  // 低库存只能新增为“库存紧张”：就地报错，并保持表单当前选择不变
  if (isLowStock(form.stock) && formStatus.value !== STATUS_TIGHT) {
    formError.value = `库存低于${STOCK_LOW_THRESHOLD}L时只允许“库存紧张”，请调整库存或状态后再保存`;
    return;
  }
  // 高库存不得保留“库存紧张”：自动转为“营业中”
  const status = normalizeStatus(form.stock, formStatus.value);
  records.value = [
    {
      ...form,
      id: crypto.randomUUID(),
      status,
      notes: note.value || "暂无备注",
      createdAt: new Date().toISOString()
    } as RecordItem,
    ...records.value
  ];
  Object.assign(form, createBlank());
  formStatus.value = STATUS_OPEN;
  formError.value = "";
  note.value = "";
  persist();
}

function flow(record: RecordItem) {
  const next = nextStatus(record.status);
  // 低库存流转到“库存紧张”以外的状态：就地报错，并保持原状态
  if (isLowStock(record.stock) && next !== STATUS_TIGHT) {
    cardErrors[record.id] = `库存低于${STOCK_LOW_THRESHOLD}L时只允许“库存紧张”，无法流转为“${next}”`;
    return;
  }
  // 高库存流转到“库存紧张”时自动转为“营业中”
  record.status = normalizeStatus(record.stock, next);
  delete cardErrors[record.id];
  persist();
}

function remove(id: string) {
  records.value = records.value.filter((record) => record.id !== id);
  delete cardErrors[id];
  persist();
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">{{ project.industry }}行业前端最小闭环</p>
          <h1>{{ project.title }}</h1>
          <p class="subtitle">{{ project.subtitle }}</p>
        </div>
        <div class="stack">
          <span v-for="item in project.stack" :key="item" class="tag">{{ item }}</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="(label, index) in project.metricLabels" :key="label" class="metric">
          <span>{{ label }}</span>
          <strong>{{ metrics[index] }}</strong>
        </article>
      </section>

      <section class="workspace">
        <form class="panel" @submit.prevent="submit">
          <h2>{{ project.formTitle }}</h2>
          <div class="form-grid">
            <label v-for="field in fields" :key="field.key">
              {{ field.label }}
              <select v-if="field.type === 'select'" v-model="form[field.key]" required>
                <option value="">请选择</option>
                <option v-for="option in field.options" :key="option">{{ option }}</option>
              </select>
              <input
                v-else
                v-model="form[field.key]"
                :type="field.type || 'text'"
                :class="{ invalid: formError && field.key === 'stock' }"
                @input="clearFormError"
                required
              />
            </label>
            <label>
              营业状态
              <select v-model="formStatus" :class="{ invalid: formError }" @change="clearFormError">
                <option v-for="status in statuses" :key="status" :value="status">{{ status }}</option>
              </select>
            </label>
            <label>
              备注
              <textarea v-model="note" placeholder="填写处理说明或现场备注" />
            </label>
            <p v-if="formError" class="error-text" role="alert">{{ formError }}</p>
            <button type="submit">{{ project.primaryAction }}</button>
          </div>
        </form>

        <section class="list-panel">
          <div class="toolbar">
            <h2>{{ project.entityLabel }}列表</h2>
            <select v-model="filter">
              <option v-for="item in project.filters" :key="item">{{ item }}</option>
            </select>
          </div>

          <div class="record-grid">
            <div v-if="filteredRecords.length === 0" class="empty">暂无匹配数据</div>
            <article v-for="record in filteredRecords" :key="record.id" class="record">
              <div class="record-head">
                <p class="record-title">{{ primaryText(record) }}</p>
                <span class="status">{{ record.status }}</span>
              </div>
              <div class="details">
                <span v-for="field in fields" :key="field.key">{{ field.label }}: {{ record[field.key] }}</span>
              </div>
              <p class="note">{{ record.notes }}</p>
              <div class="actions">
                <button type="button" @click="flow(record)">流转状态</button>
                <button class="secondary" type="button" @click="navigator.clipboard?.writeText(primaryText(record))">复制摘要</button>
                <button class="danger" type="button" @click="remove(record.id)">删除</button>
              </div>
              <p v-if="cardErrors[record.id]" class="error-text" role="alert">{{ cardErrors[record.id] }}</p>
            </article>
          </div>

          <div class="mini-chart">
            <div v-for="row in chartRows" :key="row.status" class="bar">
              <span>{{ row.status }}</span>
              <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
              <strong>{{ row.value }}</strong>
            </div>
          </div>
        </section>
      </section>
    </div>
  </main>
</template>
