<template>
  <div>
    <div v-if="loading" class="d-flex justify-center pa-4">
      <v-progress-circular indeterminate size="24" width="2" />
    </div>
    <div v-else-if="!isConfigured" class="text-center pa-3">
      <v-icon size="32" color="grey" class="mb-2">mdi-cog-outline</v-icon>
      <div class="text-body-2 text-medium-emphasis mb-2">{{ $t('plugin_nvidia_driver.not_configured') }}</div>
      <v-btn size="small" variant="tonal" color="primary" :to="`/plugins/nvidia-driver`">
        <v-icon start size="16">mdi-open-in-app</v-icon>
        {{ $t('plugin_nvidia_driver.configure') }}
      </v-btn>
    </div>
    <div v-else>
      <div v-for="(gpu, index) in settings.gpus" :key="index" :class="{ 'mt-3': index > 0 }">
        <div class="d-flex align-center mb-2" style="min-width: 0">
          <span class="text-body-2 font-weight-bold" :title="getGpuName(gpu.uuid)" style="white-space: nowrap; overflow: hidden; text-overflow: ellipsis; min-width: 0; flex: 1">{{ getGpuName(gpu.uuid) }}</span>
          <v-chip size="x-small" variant="outlined" class="ml-2" style="flex-shrink: 0">{{ gpu.pci }}</v-chip>
        </div>
        <div v-if="gpuData[gpu.uuid]">
          <div v-if="gpuData[gpu.uuid].temperature_gpu != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>{{ $t('plugin_nvidia_driver.temperature') }}</b></span>
            <span class="text-caption" style="flex-shrink: 0">{{ gpuData[gpu.uuid].temperature_gpu }} °C</span>
          </div>
          <div v-if="gpuData[gpu.uuid].power_draw != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>{{ $t('plugin_nvidia_driver.power') }}</b></span>
            <span class="text-caption" style="flex-shrink: 0">{{ parseFloat(gpuData[gpu.uuid].power_draw).toFixed(1) }} W</span>
          </div>
          <div v-if="gpuData[gpu.uuid].clocks_graphics != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>{{ $t('plugin_nvidia_driver.gpu_clock') }}</b></span>
            <span class="text-caption" style="flex-shrink: 0">{{ gpuData[gpu.uuid].clocks_graphics }} MHz</span>
          </div>
          <div v-if="gpuData[gpu.uuid].fan_speed != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>{{ $t('plugin_nvidia_driver.fan') }}</b></span>
            <span class="text-caption" style="flex-shrink: 0">{{ gpuData[gpu.uuid].fan_speed }} %</span>
          </div>
          <div v-if="gpuData[gpu.uuid].utilization_gpu != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>GPU</b></span>
            <v-progress-linear
              :model-value="gpuData[gpu.uuid].utilization_gpu"
              height="10"
              :color="gpuData[gpu.uuid].utilization_gpu >= 90 ? 'red' : gpuData[gpu.uuid].utilization_gpu >= 75 ? 'orange' : 'green'"
              style="border-radius: 5px; overflow: hidden; flex: 1"
            >
              <template #default>
                <span style="font-size: 9px">{{ gpuData[gpu.uuid].utilization_gpu }}%</span>
              </template>
            </v-progress-linear>
          </div>
          <div v-if="gpuData[gpu.uuid].utilization_memory != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>MEM</b></span>
            <v-progress-linear
              :model-value="gpuData[gpu.uuid].utilization_memory"
              height="10"
              :color="gpuData[gpu.uuid].utilization_memory >= 90 ? 'red' : gpuData[gpu.uuid].utilization_memory >= 75 ? 'orange' : 'green'"
              style="border-radius: 5px; overflow: hidden; flex: 1"
            >
              <template #default>
                <span style="font-size: 9px">{{ gpuData[gpu.uuid].utilization_memory }}%</span>
              </template>
            </v-progress-linear>
          </div>
          <div v-if="gpuData[gpu.uuid].memory_used != null && gpuData[gpu.uuid].memory_total != null" class="d-flex align-center mb-1" style="gap: 6px">
            <span class="text-caption" style="width: 80px; flex-shrink: 0"><b>VRAM</b></span>
            <v-progress-linear
              :model-value="getVramPercent(gpu.uuid)"
              height="10"
              :color="getVramPercent(gpu.uuid) >= 90 ? 'red' : getVramPercent(gpu.uuid) >= 75 ? 'orange' : 'green'"
              style="border-radius: 5px; overflow: hidden; flex: 1"
            >
              <template #default>
                <span style="font-size: 9px">{{ getVramPercent(gpu.uuid).toFixed(0) }}%</span>
              </template>
            </v-progress-linear>
          </div>
          <div v-if="getProcessCount(gpu.uuid) > 0" class="text-caption text-medium-emphasis mt-1">
            <v-icon size="12">mdi-application-outline</v-icon>
            {{ getProcessCount(gpu.uuid) }} {{ getProcessCount(gpu.uuid) === 1 ? $t('plugin_nvidia_driver.process') : $t('plugin_nvidia_driver.processes') }}
          </div>
        </div>
        <div v-else class="text-caption text-medium-emphasis text-center pa-2">
          {{ $t('plugin_nvidia_driver.loading') }}
        </div>
        <v-divider v-if="index < settings.gpus.length - 1" class="mt-2" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

const loading = ref(true);
const settings = ref({ gpus: [], interval: 2 });
const gpuData = ref({});
const processData = ref({});
const availableGpus = ref([]);
let pollInterval = null;

const isConfigured = computed(() => {
  return settings.value.gpus && settings.value.gpus.length > 0 && settings.value.gpus.some((g) => g.uuid);
});

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const getGpuName = (uuid) => {
  const gpu = availableGpus.value.find((g) => g.uuid === uuid);
  return gpu ? gpu.name : `GPU ${uuid}`;
};

const getVramPercent = (uuid) => {
  const data = gpuData.value[uuid];
  if (!data) return 0;
  const total = parseFloat(data.memory_total) || 1;
  const used = parseFloat(data.memory_used) || 0;
  return (used / total) * 100;
};

const getProcessCount = (uuid) => {
  const procs = processData.value[uuid];
  if (!procs) return 0;
  return procs.length;
};

const parseNvidiaSmiXml = (xmlString) => {
  if (!xmlString || typeof xmlString !== 'string') return null;
  try {
    const parser = new DOMParser();
    const xmlDoc = parser.parseFromString(xmlString, 'text/xml');
    const gpus = xmlDoc.querySelectorAll('nvidia_smi_log > gpu');
    if (gpus.length === 0) return null;

    const parseValue = (element) => {
      if (!element) return null;
      const text = element.textContent?.trim();
      if (!text || text === '[N/A]' || text === '[Not Supported]') return null;
      return text;
    };

    const parseNumber = (element) => {
      const val = parseValue(element);
      if (val === null) return null;
      const num = parseFloat(val);
      return isNaN(num) ? null : num;
    };

    return Array.from(gpus).map((gpu) => {
      const utilization = gpu.querySelector('utilization');
      const memory = gpu.querySelector('fb_memory_usage');
      const temperature = gpu.querySelector('temperature');
      const power = gpu.querySelector('power_readings');
      const clocks = gpu.querySelector('clocks');

      return {
        uuid: parseValue(gpu.querySelector('uuid')) || '',
        utilization_gpu: parseNumber(utilization?.querySelector('gpu_util')) || 0,
        utilization_memory: parseNumber(utilization?.querySelector('memory_util')) || 0,
        memory_total: parseNumber(memory?.querySelector('total')) || 0,
        memory_used: parseNumber(memory?.querySelector('used')) || 0,
        temperature_gpu: parseNumber(temperature?.querySelector('gpu_temp')) || null,
        power_draw: parseNumber(power?.querySelector('power_draw')) || null,
        clocks_graphics: parseNumber(clocks?.querySelector('graphics_clock')) || null,
        fan_speed: parseNumber(gpu.querySelector('fan_speed')) || null,
      };
    });
  } catch {
    return null;
  }
};

const parseProcessXml = (xmlString) => {
  if (!xmlString || typeof xmlString !== 'string') return [];
  try {
    const parser = new DOMParser();
    const xmlDoc = parser.parseFromString(xmlString, 'text/xml');
    const gpus = xmlDoc.querySelectorAll('nvidia_smi_log > gpu');
    const processes = [];

    for (const gpu of gpus) {
      const gpuUuid = gpu.querySelector('uuid')?.textContent?.trim() || '';
      const processList = gpu.querySelector('processes');
      if (!processList) continue;

      const procElements = processList.querySelectorAll('process_info');
      for (const proc of procElements) {
        const pid = proc.querySelector('pid')?.textContent?.trim();
        if (pid) {
          processes.push({ gpu_uuid: gpuUuid, pid });
        }
      }
    }

    return processes;
  } catch {
    return [];
  }
};

const fetchSettings = async () => {
  try {
    const res = await fetch('/api/v1/mos/plugins/settings/nvidia-driver', {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      settings.value = {
        gpus: Array.isArray(data.gpus) ? data.gpus : [],
        interval: data.interval || 2,
      };
    }
  } catch { }
};

const fetchAvailableGpus = async () => {
  try {
    const res = await fetch('/api/v1/system/gpus', {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      const nvidiaGpus = data.NVIDIA || data.Nvidia || data.nvidia || [];
      if (Array.isArray(nvidiaGpus)) {
        availableGpus.value = nvidiaGpus.map((gpu) => ({
          pci: gpu.pci,
          name: gpu.name,
          uuid: gpu.uuid || gpu.pci,
        }));
      }
    }
  } catch { }
};

const fetchGpuData = async () => {
  if (!isConfigured.value) return;

  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'nvidia-smi',
        args: ['-q', '-x'],
        timeout: 5,
        parse_json: false,
      }),
    });

    if (res.ok) {
      const data = await res.json();
      if (data.success === false) return;

      const output = data.output;
      if (!output || typeof output !== 'string') return;

      const gpus = parseNvidiaSmiXml(output);
      if (gpus && gpus.length > 0) {
        const configuredUuids = new Set(settings.value.gpus.map((g) => g.uuid));
        for (const gpuDataItem of gpus) {
          if (configuredUuids.has(gpuDataItem.uuid)) {
            gpuData.value[gpuDataItem.uuid] = gpuDataItem;
          }
        }
      }

      const processes = parseProcessXml(output);
      const byGpu = {};
      for (const proc of processes) {
        if (!byGpu[proc.gpu_uuid]) byGpu[proc.gpu_uuid] = [];
        byGpu[proc.gpu_uuid].push(proc);
      }
      processData.value = byGpu;
    }
  } catch { }
};

const startPolling = () => {
  stopPolling();
  if (!isConfigured.value) return;
  fetchGpuData();
  const interval = Math.max(2, settings.value.interval) * 1000;
  pollInterval = setInterval(fetchGpuData, interval);
};

const stopPolling = () => {
  if (pollInterval) {
    clearInterval(pollInterval);
    pollInterval = null;
  }
};

onMounted(async () => {
  try {
    await Promise.all([fetchSettings(), fetchAvailableGpus()]);
    if (isConfigured.value) {
      startPolling();
    }
  } catch { }
  finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  stopPolling();
});
</script>
