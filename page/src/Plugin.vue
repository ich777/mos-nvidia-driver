<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_nvidia_driver.title') }}</h2>
    <v-skeleton-loader v-if="loading" :loading="true" type="card, card, card" />
    <div v-else style="margin-bottom: 80px">
      <v-card class="mb-4 pa-0">
        <v-card-title>{{ $t('plugin_nvidia_driver.driver_selection') }}</v-card-title>
        <v-card-text class="pa-4">
          <div class="text-caption text-medium-emphasis mb-2"><strong>{{ $t('plugin_nvidia_driver.license') }}</strong></div>
          <v-radio-group v-model="selectedLicense" hide-details class="mt-0 mb-4" inline>
            <v-radio value="opensource">
              <template #label>
                <span>
                  {{ $t('plugin_nvidia_driver.open_source') }}
                  <v-chip
                    v-if="settings.license === 'opensource'"
                    size="x-small"
                    color="success"
                    class="ml-2"
                  >
                    {{ $t('plugin_nvidia_driver.selected') }}
                  </v-chip>
                </span>
              </template>
            </v-radio>
            <v-radio value="proprietary">
              <template #label>
                <span>
                  {{ $t('plugin_nvidia_driver.proprietary') }}
                  <v-chip
                    v-if="settings.license === 'proprietary'"
                    size="x-small"
                    color="success"
                    class="ml-2"
                  >
                    {{ $t('plugin_nvidia_driver.selected') }}
                  </v-chip>
                </span>
              </template>
            </v-radio>
          </v-radio-group>

          <!--
          <div class="text-caption text-medium-emphasis mb-2"><strong>Branch</strong></div>
          <v-radio-group v-model="selectedBranch" hide-details class="mt-0 mb-4" inline>
            <v-radio value="production">
              <template #label>
                <span>
                  Production
                  <v-chip
                    v-if="settings.branch === 'production'"
                    size="x-small"
                    color="success"
                    class="ml-2"
                  >
                    {{ $t('plugin_nvidia_driver.selected') }}
                  </v-chip>
                </span>
              </template>
            </v-radio>
            <v-radio value="newfeature">
              <template #label>
                <span>
                  New Feature
                  <v-chip
                    v-if="settings.branch === 'newfeature'"
                    size="x-small"
                    color="success"
                    class="ml-2"
                  >
                    {{ $t('plugin_nvidia_driver.selected') }}
                  </v-chip>
                </span>
              </template>
            </v-radio>
            <v-radio value="beta">
              <template #label>
                <span>
                  Beta
                  <v-chip
                    v-if="settings.branch === 'beta'"
                    size="x-small"
                    color="success"
                    class="ml-2"
                  >
                    {{ $t('plugin_nvidia_driver.selected') }}
                  </v-chip>
                </span>
              </template>
            </v-radio>
          </v-radio-group>
          -->

          <div class="text-caption text-medium-emphasis mb-2"><strong>{{ $t('plugin_nvidia_driver.driver_version') }}</strong></div>
          <v-select
            v-model="selectedDriverVersion"
            :items="availableDriverVersions"
            :loading="loadingVersions"
            :label="$t('plugin_nvidia_driver.select_driver_version')"
            density="compact"
            clearable
            class="mb-2"
          >
            <template #selection="{ item }">
              <span>{{ item.raw }}</span>
            </template>
            <template #item="{ item, props }">
              <v-list-item v-bind="props">
                <template #append>
                  <v-chip
                    v-if="stripVersionSuffix(settings.driver_version) === item.raw"
                    size="x-small"
                    color="success"
                  >
                    {{ $t('plugin_nvidia_driver.selected') }}
                  </v-chip>
                </template>
              </v-list-item>
            </template>
          </v-select>
        </v-card-text>
        <v-divider />
        <v-card-actions class="pa-4">
          <v-spacer />
          <v-btn
            color="onPrimary"
            :loading="saving"
            @click="saveDriverSettings"
          >
            {{ $t('plugin_nvidia_driver.update') }}
          </v-btn>
          <v-btn
            color="onPrimary"
            :loading="downloading"
            @click="downloadDriver"
          >
            {{ $t('plugin_nvidia_driver.download') }}
          </v-btn>
        </v-card-actions>
      </v-card>

      <v-card v-if="driverInfo && driverInfo.package" class="mb-4 pa-0">
        <v-card-title>{{ $t('plugin_nvidia_driver.local_driver_package') }}</v-card-title>
        <v-card-text class="pa-4">
          <v-row dense>
            <v-col cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.plugin') }}</strong></div>
              <div class="text-body-2">{{ driverInfo.plugin || '-' }}</div>
            </v-col>
            <v-col cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.kernel') }}</strong></div>
              <div class="text-body-2">{{ driverInfo.kernel || '-' }}</div>
            </v-col>
            <v-col cols="12" md="6">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.package') }}</strong></div>
              <div class="text-body-2" style="word-break: break-all">{{ driverInfo.package || '-' }}</div>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>

      <v-card v-if="!isConfigured" class="mb-4 pa-0">
        <v-card-text class="pa-4">
          {{ $t('plugin_nvidia_driver.not_configured_hint') }}
        </v-card-text>
      </v-card>

      <v-card v-if="gpuError" class="mb-4 pa-0">
        <v-card-text class="pa-4">
          <v-alert type="error" variant="tonal">
            <div class="text-body-2"><strong>{{ $t('plugin_nvidia_driver.error_loading_gpu') }}</strong></div>
            <div class="text-caption mt-1">{{ gpuError }}</div>
          </v-alert>
        </v-card-text>
      </v-card>

      <v-card v-for="(gpu, index) in settings.gpus" :key="index" class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <span>{{ getGpuName(gpu.uuid) }}</span>
          <v-spacer />
          <v-chip size="small">{{ gpu.pci }}</v-chip>
        </v-card-title>
        <v-card-text v-if="gpuData[gpu.uuid]" class="pa-4">
          <v-row dense>
            <v-col cols="12" md="6">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.uuid') }}</strong></div>
              <div class="text-body-2" style="word-break: break-all; font-family: monospace; font-size: 0.75rem">{{ gpu.uuid }}</div>
            </v-col>
            <v-col v-if="gpuData[gpu.uuid].driver_version" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.driver_version') }}</strong></div>
              <div class="text-body-2">{{ gpuData[gpu.uuid].driver_version }}</div>
            </v-col>
            <v-col v-if="gpuData[gpu.uuid].pstate" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.p_state') }}</strong></div>
              <div class="text-body-2">{{ gpuData[gpu.uuid].pstate }}</div>
            </v-col>
          </v-row>
          <v-divider class="mt-2 mb-2" />
          <v-row dense>
            <v-col v-if="gpuData[gpu.uuid].temperature_gpu != null" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.temperature') }}</strong></div>
              <div class="text-body-2">{{ gpuData[gpu.uuid].temperature_gpu }} °C</div>
            </v-col>
            <v-col v-if="gpuData[gpu.uuid].power_draw != null" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.power_draw') }}</strong></div>
              <div class="text-body-2">{{ parseFloat(gpuData[gpu.uuid].power_draw).toFixed(1) }} W</div>
            </v-col>
            <v-col v-if="gpuData[gpu.uuid].power_limit != null" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.power_limit') }}</strong></div>
              <div class="text-body-2">{{ parseFloat(gpuData[gpu.uuid].power_limit).toFixed(1) }} W</div>
            </v-col>
            <v-col v-if="gpuData[gpu.uuid].fan_speed != null" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.fan_speed') }}</strong></div>
              <div class="text-body-2">{{ gpuData[gpu.uuid].fan_speed }} %</div>
            </v-col>
            <v-col v-if="gpuData[gpu.uuid].clocks_graphics != null" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.gpu_clock') }}</strong></div>
              <div class="text-body-2">{{ gpuData[gpu.uuid].clocks_graphics }} MHz</div>
            </v-col>
            <v-col v-if="gpuData[gpu.uuid].clocks_memory != null" cols="6" md="3">
              <div class="text-caption text-medium-emphasis"><strong>{{ $t('plugin_nvidia_driver.memory_clock') }}</strong></div>
              <div class="text-body-2">{{ gpuData[gpu.uuid].clocks_memory }} MHz</div>
            </v-col>
          </v-row>
          <v-divider class="mt-2 mb-2" />
          <v-row dense>
            <v-col cols="12" md="6">
              <div class="d-flex align-center mb-1">
                <span class="text-caption" style="width: 80px"><strong>{{ $t('plugin_nvidia_driver.gpu') }}:</strong></span>
                <v-progress-linear
                  :model-value="gpuData[gpu.uuid].utilization_gpu || 0"
                  height="16"
                  :color="gpuData[gpu.uuid].utilization_gpu >= 90 ? 'red' : gpuData[gpu.uuid].utilization_gpu >= 75 ? 'orange' : 'green'"
                  style="border-radius: 7px; overflow: hidden; flex: 1"
                >
                  <template #default>
                    <span><small>{{ gpuData[gpu.uuid].utilization_gpu || 0 }}%</small></span>
                  </template>
                </v-progress-linear>
              </div>
            </v-col>
            <v-col cols="12" md="6">
              <div class="d-flex align-center mb-1">
                <span class="text-caption" style="width: 80px"><strong>{{ $t('plugin_nvidia_driver.memory') }}:</strong></span>
                <v-progress-linear
                  :model-value="gpuData[gpu.uuid].utilization_memory || 0"
                  height="16"
                  :color="gpuData[gpu.uuid].utilization_memory >= 90 ? 'red' : gpuData[gpu.uuid].utilization_memory >= 75 ? 'orange' : 'green'"
                  style="border-radius: 7px; overflow: hidden; flex: 1"
                >
                  <template #default>
                    <span><small>{{ gpuData[gpu.uuid].utilization_memory || 0 }}%</small></span>
                  </template>
                </v-progress-linear>
              </div>
            </v-col>
          </v-row>
          <v-row dense class="mt-1">
            <v-col cols="12">
              <div class="d-flex align-center mb-1">
                <span class="text-caption" style="width: 80px"><strong>{{ $t('plugin_nvidia_driver.vram') }}:</strong></span>
                <v-progress-linear
                  :model-value="getVramPercent(gpu.uuid)"
                  height="16"
                  :color="getVramPercent(gpu.uuid) >= 90 ? 'red' : getVramPercent(gpu.uuid) >= 75 ? 'orange' : 'green'"
                  style="border-radius: 7px; overflow: hidden; flex: 1"
                >
                  <template #default>
                    <span><small>{{ gpuData[gpu.uuid].memory_used || 0 }} / {{ gpuData[gpu.uuid].memory_total || 0 }} MiB</small></span>
                  </template>
                </v-progress-linear>
              </div>
            </v-col>
          </v-row>
          <v-row v-if="gpuData[gpu.uuid].encoder_util != null || gpuData[gpu.uuid].decoder_util != null" dense class="mt-2">
            <v-col cols="12">
              <div class="text-caption text-medium-emphasis mb-1"><strong>{{ $t('plugin_nvidia_driver.encoder_decoder') }}</strong></div>
              <v-row dense>
                <v-col v-if="gpuData[gpu.uuid].encoder_util != null" cols="12" md="6">
                  <div style="display: flex; align-items: center; gap: 6px">
                    <span class="text-body-2" style="width: 100px"><small><b>{{ $t('plugin_nvidia_driver.encoder') }}</b></small></span>
                    <v-progress-linear
                      :model-value="gpuData[gpu.uuid].encoder_util || 0"
                      height="12"
                      :color="gpuData[gpu.uuid].encoder_util >= 90 ? 'red' : gpuData[gpu.uuid].encoder_util >= 75 ? 'orange' : 'green'"
                      style="border-radius: 6px; overflow: hidden; flex: 1"
                    >
                      <template #default>
                        <span><small>{{ gpuData[gpu.uuid].encoder_util || 0 }}%</small></span>
                      </template>
                    </v-progress-linear>
                  </div>
                </v-col>
                <v-col v-if="gpuData[gpu.uuid].decoder_util != null" cols="12" md="6">
                  <div style="display: flex; align-items: center; gap: 6px">
                    <span class="text-body-2" style="width: 100px"><small><b>{{ $t('plugin_nvidia_driver.decoder') }}</b></small></span>
                    <v-progress-linear
                      :model-value="gpuData[gpu.uuid].decoder_util || 0"
                      height="12"
                      :color="gpuData[gpu.uuid].decoder_util >= 90 ? 'red' : gpuData[gpu.uuid].decoder_util >= 75 ? 'orange' : 'green'"
                      style="border-radius: 6px; overflow: hidden; flex: 1"
                    >
                      <template #default>
                        <span><small>{{ gpuData[gpu.uuid].decoder_util || 0 }}%</small></span>
                      </template>
                    </v-progress-linear>
                  </div>
                </v-col>
              </v-row>
            </v-col>
          </v-row>
          <v-row v-if="processData[gpu.uuid] && processData[gpu.uuid].length > 0" dense class="mt-2">
            <v-col cols="12">
              <details>
                <summary style="cursor: pointer; color: var(--v-theme-primary); text-decoration: underline" class="text-body-2 mb-1">{{ $t('plugin_nvidia_driver.processes') }}</summary>
                <div v-for="proc in processData[gpu.uuid]" :key="proc.pid" class="mb-2">
                  <div class="d-flex align-center">
                    <span class="text-body-2"><b>{{ proc.name || 'Unknown' }}</b></span>
                    <span class="text-caption text-medium-emphasis ml-2">PID: {{ proc.pid }}</span>
                    <v-spacer />
                    <span class="text-caption">{{ proc.used_memory }} MiB</span>
                  </div>
                </div>
              </details>
            </v-col>
          </v-row>
        </v-card-text>
        <v-card-text v-else class="text-center pa-8 text-grey">
          {{ $t('plugin_nvidia_driver.loading') }}
        </v-card-text>
      </v-card>
    </div>
    <v-dialog v-model="settingsDialog.value" max-width="600">
      <v-card class="pa-0">
        <v-card-title>{{ $t('plugin_nvidia_driver.settings') }}</v-card-title>
        <v-card-text>
          <v-form>
            <v-text-field
              v-model.number="settingsDialog.interval"
              :label="$t('plugin_nvidia_driver.update_interval')"
              type="number"
              min="1"
              @blur="validateInterval"
            />
            <v-select
              v-model="settingsDialog.selectedGpus"
              :items="availableGpusItems"
              item-title="title"
              item-value="value"
              :label="$t('plugin_nvidia_driver.gpus')"
              multiple
              chips
              clearable
            />
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn color="onPrimary" @click="settingsDialog.value = false">{{ $t('plugin_nvidia_driver.cancel') }}</v-btn>
          <v-btn color="onPrimary" @click="saveGpuSettings" :loading="settingsDialog.saving">{{ $t('plugin_nvidia_driver.save') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-fab color="primary" style="position: fixed; bottom: 32px; right: 32px; z-index: 1000" size="large" icon @click="openSettingsDialog">
      <v-icon>mdi-cog</v-icon>
    </v-fab>
  </div>
</template>

<script setup>
import { ref, computed, reactive, onMounted, onUnmounted, watch } from 'vue';

const PLUGIN_NAME = 'nvidia-driver';

const loading = ref(true);
const saving = ref(false);
const downloading = ref(false);
const loadingVersions = ref(false);

const settings = ref({
  license: 'opensource',
  // branch: null, // For now deactivated
  driver_version: null,
  gpus: [],
  interval: 2,
});

const selectedLicense = ref('opensource');
// const selectedBranch = ref(null); // For now deactivated
const selectedDriverVersion = ref(null);

const allDriverVersions = ref({
  opensource: [],
  proprietary: [],
});

const availableDriverVersions = computed(() => {
  return allDriverVersions.value[selectedLicense.value] || [];
});

const driverInfo = ref(null);
const availableGpus = ref([]);
const gpuData = ref({});
const processData = ref({});
const gpuError = ref(null);

let pollInterval = null;

const settingsDialog = reactive({
  value: false,
  interval: 2,
  selectedGpus: [],
  saving: false,
});

const isConfigured = computed(() => {
  return settings.value.gpus && settings.value.gpus.length > 0 && settings.value.gpus.some((g) => g.uuid);
});

const availableGpusItems = computed(() => {
  return availableGpus.value.map((gpu) => ({
    title: `${gpu.name} (${gpu.pci})`,
    value: gpu.uuid,
  }));
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

const validateInterval = () => {
  if (settingsDialog.interval < 1) {
    settingsDialog.interval = 1;
  }
};

const stripVersionSuffix = (version) => {
  if (typeof version !== 'string') return version;
  return version.replace(/-\d+$/, '');
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
      const pci = gpu.querySelector('pci');

      return {
        name: parseValue(gpu.querySelector('product_name')) || '',
        pci_bus_id: parseValue(pci?.querySelector('pci_bus')) || '',
        uuid: parseValue(gpu.querySelector('uuid')) || '',
        utilization_gpu: parseNumber(utilization?.querySelector('gpu_util')) || 0,
        utilization_memory: parseNumber(utilization?.querySelector('memory_util')) || 0,
        memory_total: parseNumber(memory?.querySelector('total')) || 0,
        memory_used: parseNumber(memory?.querySelector('used')) || 0,
        memory_free: parseNumber(memory?.querySelector('free')) || 0,
        temperature_gpu: parseNumber(temperature?.querySelector('gpu_temp')) || null,
        power_draw: parseNumber(power?.querySelector('power_draw')) || null,
        power_limit: parseNumber(power?.querySelector('power_limit')) || null,
        clocks_graphics: parseNumber(clocks?.querySelector('graphics_clock')) || null,
        clocks_memory: parseNumber(clocks?.querySelector('mem_clock')) || null,
        fan_speed: parseNumber(gpu.querySelector('fan_speed')) || null,
        driver_version: parseValue(gpu.querySelector('driver_version')) || '',
        pstate: parseValue(gpu.querySelector('performance_state')) || null,
        encoder_util: null,
        decoder_util: null,
      };
    });
  } catch (e) {
    console.error('Failed to parse nvidia-smi XML:', e);
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
        const name = proc.querySelector('process_name')?.textContent?.trim();
        const memory = proc.querySelector('used_memory')?.textContent?.trim();

        if (pid && name) {
          processes.push({
            gpu_uuid: gpuUuid,
            pid: pid,
            name: name || 'Unknown',
            used_memory: parseInt(memory) || 0,
          });
        }
      }
    }

    return processes;
  } catch (e) {
    console.error('Failed to parse nvidia-smi process XML:', e);
    return [];
  }
};

const normalizeSelectedDriverVersion = () => {
  const current = selectedDriverVersion.value;
  if (!current) return;

  const normalized = stripVersionSuffix(current);
  const match = availableDriverVersions.value.find((v) => v === normalized);
  if (match) {
    selectedDriverVersion.value = match;
  }
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });

    if (res.ok) {
      const data = await res.json();

      settings.value = {
        license: data.license || 'opensource',
        // branch: data.branch || null, // For now deactivated
        driver_version: data.driver_version || null,
        gpus: Array.isArray(data.gpus) ? data.gpus : [],
        interval: data.interval || 2,
      };

      selectedLicense.value = settings.value.license;
      // selectedBranch.value = settings.value.branch; // For now deactivated

      selectedDriverVersion.value = stripVersionSuffix(settings.value.driver_version);
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const fetchDriverInfo = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/driver/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      driverInfo.value = data;
    }
  } catch (e) {
    console.error('Failed to fetch driver info:', e);
    driverInfo.value = null;
  }
};

const fetchDriverVersions = async () => {
  loadingVersions.value = true;
  try {
    const res = await fetch('/api/v1/mos/getdrivers', {
      headers: getAuthHeaders(),
    });

    if (res.ok) {
      const data = await res.json();
      const nvidiaDrivers = data.nvidia || {};

      const opensourceVersions = nvidiaDrivers['nvidia-opensource'] || [];
      allDriverVersions.value.opensource = opensourceVersions.map(stripVersionSuffix);

      const proprietaryVersions = nvidiaDrivers['nvidia-proprietary'] || [];
      allDriverVersions.value.proprietary = proprietaryVersions.map(stripVersionSuffix);

      // nachdem die items da sind: selection nochmal sauber reinziehen
      normalizeSelectedDriverVersion();
    }
  } catch (e) {
    console.error('Failed to fetch driver versions:', e);
    allDriverVersions.value = { opensource: [], proprietary: [] };
  } finally {
    loadingVersions.value = false;
  }
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
  } catch (e) {
    console.error('Failed to fetch GPUs:', e);
  }
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

      if (data.success === false) {
        const errorMsg = data.output || 'Unknown error';
        if (errorMsg.includes('Command failed') || errorMsg.includes('not found')) {
          gpuError.value =
            'nvidia-smi command not found. Please ensure the NVIDIA driver is installed.';
        } else {
          gpuError.value = `Failed to query GPU data: ${errorMsg}`;
        }
        return;
      }

      const output = data.output;
      if (!output) {
        gpuError.value =
          'No output from nvidia-smi. Please check if the NVIDIA driver is properly installed.';
        return;
      }
      if (typeof output !== 'string') {
        gpuError.value = 'Invalid response format from nvidia-smi.';
        return;
      }

      const gpus = parseNvidiaSmiXml(output);
      if (gpus && gpus.length > 0) {
        gpuError.value = null;
        const configuredUuids = new Set(settings.value.gpus.map((g) => g.uuid));
        for (const gpuDataItem of gpus) {
          if (configuredUuids.has(gpuDataItem.uuid)) {
            gpuData.value[gpuDataItem.uuid] = gpuDataItem;
          }
        }
      } else {
        gpuError.value = 'Failed to parse GPU data from nvidia-smi output.';
      }
    } else {
      gpuError.value = `Failed to fetch GPU data (HTTP ${res.status}). Please check your connection.`;
    }
  } catch (e) {
    gpuError.value = 'Network error while fetching GPU data. Please check your connection.';
  }
};

const fetchProcessData = async () => {
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
      if (data.success === false) {
        processData.value = {};
        return;
      }

      const output = data.output;
      if (!output || typeof output !== 'string') {
        processData.value = {};
        return;
      }

      const processes = parseProcessXml(output);
      const byGpu = {};
      for (const proc of processes) {
        if (!byGpu[proc.gpu_uuid]) byGpu[proc.gpu_uuid] = [];
        byGpu[proc.gpu_uuid].push(proc);
      }
      processData.value = byGpu;
    } else {
      processData.value = {};
    }
  } catch (e) {
    processData.value = {};
  }
};

const fetchAllGpuData = async () => {
  if (!isConfigured.value) return;
  await Promise.all([fetchGpuData(), fetchProcessData()]);
};

const startPolling = () => {
  stopPolling();
  if (!isConfigured.value) return;
  fetchAllGpuData();
  const interval = Math.max(1, settings.value.interval) * 1000;
  pollInterval = setInterval(fetchAllGpuData, interval);
};

const stopPolling = () => {
  if (pollInterval) {
    clearInterval(pollInterval);
    pollInterval = null;
  }
};

const openSettingsDialog = () => {
  settingsDialog.interval = settings.value.interval || 2;
  settingsDialog.selectedGpus = settings.value.gpus.map((g) => g.uuid).filter(Boolean);
  settingsDialog.saving = false;
  settingsDialog.value = true;
};

const saveDriverSettings = async () => {
  saving.value = true;
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        license: selectedLicense.value,
        // branch: selectedBranch.value, // For now deactivated
        driver_version: selectedDriverVersion.value,
        gpus: settings.value.gpus,
        interval: settings.value.interval,
      }),
    });

    if (res.ok) {
      settings.value.license = selectedLicense.value;
      // settings.value.branch = selectedBranch.value; // For now deactivated
      settings.value.driver_version = selectedDriverVersion.value;
    }
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    saving.value = false;
  }
};

const saveGpuSettings = async () => {
  settingsDialog.saving = true;
  validateInterval();

  try {
    const gpus = settingsDialog.selectedGpus.map((uuid) => {
      const gpu = availableGpus.value.find((g) => g.uuid === uuid);
      return { uuid, pci: gpu?.pci || '', name: gpu?.name || '' };
    });

    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        license: settings.value.license,
        // branch: settings.value.branch, // For now deactivated
        driver_version: settings.value.driver_version,
        gpus: gpus,
        interval: settingsDialog.interval,
      }),
    });

    if (res.ok) {
      settings.value.gpus = gpus;
      settings.value.interval = settingsDialog.interval;
      settingsDialog.value = false;
      startPolling();
    }
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    settingsDialog.saving = false;
  }
};

const downloadDriver = async () => {
  downloading.value = true;
  try {
    const res = await fetch('/api/v1/mos/plugins/executefunction', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        plugin: PLUGIN_NAME,
        function: 'driver_download',
        name: 'Driver download',
        restart: true,
      }),
    });

    if (!res.ok) {
      console.error('Failed to execute driver download');
    }
  } catch (e) {
    console.error('Failed to download driver:', e);
  } finally {
    downloading.value = false;
  }
};

watch(selectedLicense, () => {
  normalizeSelectedDriverVersion();
  if (
    selectedDriverVersion.value &&
    !availableDriverVersions.value.includes(selectedDriverVersion.value)
  ) {
    selectedDriverVersion.value = null;
  }
});

watch(availableDriverVersions, (versions) => {
  if (versions.length > 0 && settings.value.driver_version) {
    const normalized = stripVersionSuffix(settings.value.driver_version);
    if (versions.includes(normalized)) {
      selectedDriverVersion.value = normalized;
    }
  }
}, { immediate: true });

onMounted(async () => {
  try {
    // Fetch settings first so selectedDriverVersion and selectedLicense are set
    // before driver versions load and attempt normalization
    await fetchSettings();
    await Promise.all([
      fetchDriverInfo(),
      fetchDriverVersions(),
      fetchAvailableGpus(),
    ]);

    normalizeSelectedDriverVersion();

    if (isConfigured.value) {
      startPolling();
    }
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  stopPolling();
});
</script>
