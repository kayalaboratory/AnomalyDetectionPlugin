<template>
  <v-container fluid class="pa-0">
    <!-- Header -->
    <v-card v-if="!hideSettings" class="mb-4">
      <v-card-title>
        <div class="text-subtitle-1">
          {{ nameToDisplay(submodelElementData, 'en', 'Time Series Data (LinkedSegment)') }}
        </div>
      </v-card-title>
      <v-card-text v-if="descriptionToDisplay(submodelElementData)" class="pt-0">
        {{ descriptionToDisplay(submodelElementData) }}
      </v-card-text>
    </v-card>

    <!-- Configuration Panel -->
    <v-card v-if="!hideSettings || editDialog" class="mb-4">
      <v-list nav class="py-0">
        <v-list-item class="pb-0">
          <template #title>
            <div class="text-subtitle-2">Preview Configuration:</div>
          </template>
        </v-list-item>
      </v-list>
      <v-card-text class="pt-1">
        <!-- Segment (LinkedSegment) -->
        <v-select
          v-model="selectedSegment"
          variant="outlined"
          density="compact"
          clearable
          label="Segment"
          :items="segments"
          item-title="idShort"
          item-value="idShort"
          return-object
        />

        <!-- Time & Y-Variable (LinkedSegment) -->
        <v-row>
          <v-col cols="12" md="6">
            <v-select
              v-model="timeVariable"
              variant="outlined"
              density="compact"
              clearable
              label="time-value"
              :items="records"
              item-title="idShort"
              item-value="idShort"
              return-object
            />
          </v-col>
          <v-col cols="12" md="6">
            <v-select
              v-model="yVariables"
              variant="outlined"
              density="compact"
              clearable
              label="y-value(s)"
              :items="records"
              multiple
              item-title="idShort"
              item-value="idShort"
              return-object
            />
          </v-col>
        </v-row>


        <v-select
          v-model="selectedAlgorithm"
          :items="algorithmOptions"
          variant="outlined"
          density="compact"
          clearable
          label="Anomaly Algorithm"
          class="mt-4"
        />

        
        <template v-if="selectedAlgorithm === 'MahalanobisDistance' && !hideSettings">
          <div
            v-for="(yv, idx) in yVariables"
            :key="yv.idShort"
            class="mt-3"
          >
            <v-list-subheader>
              {{ yv.idShort }} - Mahalanobis (1D)
            </v-list-subheader>
            <v-row>
              <v-col cols="12" sm="4">
                <v-text-field
                  v-model="mahParamsByIdShort[yv.idShort].mean"
                  label="Mahalanobis Mean"
                  variant="outlined"
                  hide-details
                  density="compact"
                />
              </v-col>
              <v-col cols="12" sm="4">
                <v-text-field
                  v-model="mahParamsByIdShort[yv.idShort].invCov"
                  label="Mahalanobis InvCov"
                  variant="outlined"
                  hide-details
                  density="compact"
                />
              </v-col>
              <v-col cols="12" sm="4">
                <v-text-field
                  v-model="mahParamsByIdShort[yv.idShort].thresh"
                  label="Mahalanobis Threshold"
                  variant="outlined"
                  hide-details
                  density="compact"
                />
              </v-col>
            </v-row>
          </div>
        </template>

        
        <template v-if="selectedAlgorithm === 'ZScore' && !hideSettings">
          <div
            v-for="(yv, idx) in yVariables"
            :key="yv.idShort"
            class="mt-3"
          >
            <v-list-subheader>
              {{ yv.idShort }} - Z-Score (1D)
            </v-list-subheader>
            <v-row>
              <v-col cols="12" sm="4">
                <v-text-field
                  v-model="zParamsByIdShort[yv.idShort].mean"
                  label="Z-Score Mean"
                  variant="outlined"
                  hide-details
                  density="compact"
                />
              </v-col>
              <v-col cols="12" sm="4">
                <v-text-field
                  v-model="zParamsByIdShort[yv.idShort].stdDev"
                  label="Z-Score StdDev"
                  variant="outlined"
                  hide-details
                  density="compact"
                />
              </v-col>
              <v-col cols="12" sm="4">
                <v-text-field
                  v-model="zParamsByIdShort[yv.idShort].thresh"
                  label="Z-Score Threshold"
                  variant="outlined"
                  hide-details
                  density="compact"
                />
              </v-col>
            </v-row>
          </div>
        </template>
      </v-card-text>
      <v-divider></v-divider>

      <!-- Toggle Button for Starting/Stopping the Fetch -->
      <v-list nav class="pt-0">
        <v-list-item>
          <template #append>
            <v-btn size="small" color="primary" @click="toggleFetch">
              {{ isFetching ? 'Stop Fetching' : 'Start Fetching' }}
            </v-btn>
          </template>
        </v-list-item>
      </v-list>
    </v-card>

    <!-- Chart / Dashboard -->
    <v-card :flat="hideSettings">
      <v-list v-if="!hideSettings || editDialog" nav class="py-0">
        <v-list-item>
          <template #title>
            <div class="text-subtitle-2">Preview Chart:</div>
          </template>
          <template #append>
            <v-btn
              v-if="!hideSettings"
              color="primary"
              size="small"
              variant="elevated"
              append-icon="mdi-plus"
              @click="createObject"
            >
              Dashboard
            </v-btn>
          </template>
        </v-list-item>
      </v-list>
      <v-card-text>
        <!-- Range (ms) -->
        <v-row v-if="!hideSettings || editDialog" align="center">
          <v-col cols="auto">
            <v-text-field
              v-model="range"
              type="number"
              hide-details
              density="compact"
              label="Range"
              variant="outlined"
              suffix="ms"
              @blur="changeRange"
              @keydown.enter="changeRange"
            />
          </v-col>
        </v-row>

        <!-- ApexChart configured as a scatter plot -->
        <apexchart
          ref="apexchartRef"
          type="scatter"
          height="350"
          :series="chartSeries"
          :options="chartOptions"
        />
      </v-card-text>
    </v-card>
  </v-container>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount } from 'vue';
import { useRoute } from 'vue-router';
import { useTheme } from 'vuetify';

import { useConceptDescriptionHandling } from '@/composables/AAS/ConceptDescriptionHandling';
import { useReferableUtils } from '@/composables/AAS/ReferableUtils';
import { useDashboardHandling } from '@/composables/DashboardHandling';
import { useRequestHandling } from '@/composables/RequestHandling';
import { useAASStore } from '@/store/AASDataStore';
import { useEnvStore } from '@/store/EnvironmentStore';
import { useNavigationStore } from '@/store/NavigationStore';

import _ from 'lodash';
import ApexChart from 'vue3-apexcharts';

interface DataPoint {
  time: Date;
  value: number;
  anomaly?: boolean;
}

interface MahParams {
  mean: string;         // For multi-dim: store each dimension's mean splitted by commas, if needed.
  invCov: string;       // Each sensor might store 1 row of NxN inverse cov.
  thresh: string;       // Single numeric threshold for Mahalanobis distance.
}

interface ZParams {
  mean: string;
  stdDev: string;
  thresh: string;
}

export default defineComponent({
  name: 'LinkedSegmentAnomalyDetectionToggle',
  semanticId: 'http://submodel_template/AnomalyDetection',

  components: { apexchart: ApexChart },

  props: ['submodelElementData', 'configData', 'editDialog', 'loadTrigger'],

  setup() {
    const aasStore = useAASStore();
    const envStore = useEnvStore();
    const navigationStore = useNavigationStore();
    const theme = useTheme();
    const route = useRoute();

    const { fetchCds } = useConceptDescriptionHandling();
    const { checkIdShort, descriptionToDisplay, nameToDisplay } = useReferableUtils();
    const { getRequest, postRequest } = useRequestHandling();
    const { dashboardAdd } = useDashboardHandling();

    return {
      aasStore,
      envStore,
      navigationStore,
      theme,
      route,
      fetchCds,
      checkIdShort,
      descriptionToDisplay,
      nameToDisplay,
      getRequest,
      postRequest,
      dashboardAdd,
    };
  },

  data() {
    return {
      segments: [] as any[],
      selectedSegment: null as any,
      records: [] as any[],
      timeVariable: null as any,
      yVariables: [] as any[],

      // Default token
      apiToken:
        'S18VeAlq042B4naMX31oqIaSGmUmOLAC-DV3VIdkxDJuAhTXLTVFEchyTSmCcUAmB7Wu94KgExzV8gJaDjzR3Q==',
      showTokenInput: true,

      // Chart data
      timeSeriesValues: [] as DataPoint[][], // array of arrays, each sub-array is one sensor's time-series
      chartSeries: [] as any[],
      // Default range = 60000
      range: 60000,
      chartOptions: {} as any,
      localChartOptions: {} as any,

      yVariableTemplate: '{{y-value}}',
      isFetching: false,
      autoFetchInterval: null as any,

      // Anomaly algorithm
      algorithmOptions: ['Default', 'MahalanobisDistance', 'ZScore'],
      selectedAlgorithm: 'Default',

      // Per-yVariable anomaly params
      mahParamsByIdShort: {} as Record<string, MahParams>,
      zParamsByIdShort: {} as Record<string, ZParams>,
      windowSize: 50,
      thresholdMultiplier: 3,
      burnInPeriod: 20,
    };
  },

  computed: {
    hideSettings(): boolean {
      return this.route.name === 'DashboardGroup';
    },
    isDark(): boolean {
      return this.theme.global.current.value.dark;
    },
  },

  watch: {
    loadTrigger() {
      this.initializeSegments();
      this.initDashboardConfig();
      const token = this.envStore.getEnvInfluxdbToken;
      if (token) {
        this.apiToken = token;
        this.showTokenInput = false;
      }
    },
    isDark() {
      this.applyTheme();
    },
    selectedSegment() {
      
      if (this.selectedSegment && this.selectedSegment.value) {
        // Token
        const tokenEl = this.selectedSegment.value.find((smc: any) =>
          this.checkIdShort(smc, 'Token')
        );
        if (tokenEl?.value) {
          this.apiToken = tokenEl.value;
          this.showTokenInput = false; 
        }
      }

      if (this.isFetching) {
        this.restartAutoFetch();
      }
    },
    timeVariable() {
      if (this.isFetching) {
        this.restartAutoFetch();
      }
    },
    yVariables: {
      immediate: true,
      handler(newVal) {
        if (!newVal) return;
        newVal.forEach((yv: any) => {
          if (!this.mahParamsByIdShort[yv.idShort]) {
            this.mahParamsByIdShort[yv.idShort] = {
              mean: '',
              invCov: '',
              thresh: '',
            };
          }
          if (!this.zParamsByIdShort[yv.idShort]) {
            this.zParamsByIdShort[yv.idShort] = {
              mean: '',
              stdDev: '',
              thresh: '',
            };
          }
        });
        if (this.isFetching) {
          this.restartAutoFetch();
        }
      },
    },
  },

  mounted() {
    this.initializeSegments();
    this.initDashboardConfig();
    const token = this.envStore.getEnvInfluxdbToken;
    if (token) {
      this.apiToken = token;
      this.showTokenInput = false;
    }
    this.$nextTick(() => {
      this.setupApexChart();
    });
  },

  beforeUnmount() {
    if (this.autoFetchInterval) {
      clearInterval(this.autoFetchInterval);
      this.autoFetchInterval = null;
    }
  },

  methods: {
    /* -----------------------------------------------------------------------
       1) SUBMODEL SETUP
    -----------------------------------------------------------------------*/
    initializeSegments(): void {
      if (!this.submodelElementData?.submodelElements) return;
      const segmentsSMC = this.submodelElementData.submodelElements.find((smc: any) =>
        this.checkIdShort(smc, 'Segments')
      );
      if (segmentsSMC?.value) {
        this.segments = segmentsSMC.value;
      }
      const metadataSMC = this.submodelElementData.submodelElements.find((smc: any) =>
        this.checkIdShort(smc, 'Metadata')
      );
      if (metadataSMC) {
        const recordSMC = metadataSMC.value.find((smc: any) =>
          this.checkIdShort(smc, 'Record')
        );
        if (recordSMC?.value) {
          this.records = recordSMC.value;
          this.loadQualifiersFromProperties(recordSMC.value);
          // Fetch concept descriptions to obtain unit info
          this.records.forEach((record: any) => {
            this.fetchCds(record).then((response: any) => {
              if (response) {
                record.conceptDescriptions = response;
                this.initializeSeries();
              }
            });
          });
        }
      }
    },

    loadQualifiersFromProperties(recordArray: any[]) {
      const scanElement = (elem: any) => {
        if (elem.modelType === 'SubmodelElementCollection' && elem.value) {
          elem.value.forEach((child: any) => scanElement(child));
        } else if (elem.modelType === 'Property') {
          if (!this.mahParamsByIdShort[elem.idShort]) {
            this.mahParamsByIdShort[elem.idShort] = {
              mean: '',
              invCov: '',
              thresh: '',
            };
          }
          if (!this.zParamsByIdShort[elem.idShort]) {
            this.zParamsByIdShort[elem.idShort] = {
              mean: '',
              stdDev: '',
              thresh: '',
            };
          }
          if (elem.qualifiers) {
            elem.qualifiers.forEach((q: any) => {
              if (
                q.kind === 'ValueQualifier' &&
                q.value !== undefined &&
                q.value !== null &&
                q.value !== ''
              ) {
                // Mahalanobis
                if (q.type === 'MahalanobisMeanVector') {
                  // e.g. "0.5,0.3" or separate for each sensor
                  this.mahParamsByIdShort[elem.idShort].mean = q.value;
                }
                if (q.type === 'MahalanobisInvCovMatrix') {
                  // e.g. each sensor might store a row: "1.0,0.01,..."
                  this.mahParamsByIdShort[elem.idShort].invCov = q.value;
                }
                if (q.type === 'MahalanobisThreshold') {
                  this.mahParamsByIdShort[elem.idShort].thresh = q.value;
                }
                // ZScore
                if (q.type === 'ZScoreMean') {
                  this.zParamsByIdShort[elem.idShort].mean = q.value;
                }
                if (q.type === 'ZScoreStdDev') {
                  this.zParamsByIdShort[elem.idShort].stdDev = q.value;
                }
                if (q.type === 'ZScoreThreshold') {
                  this.zParamsByIdShort[elem.idShort].thresh = q.value;
                }
              }
            });
          }
        }
      };
      recordArray.forEach((item: any) => scanElement(item));
    },

    initDashboardConfig(): void {
      // If in dashboard, load config
      if (!this.hideSettings) return;
      this.selectedSegment = this.configData?.configObject?.segment;
      this.timeVariable = this.configData?.configObject?.timeVal;
      this.yVariables = this.configData?.configObject?.yvals;
      if (this.configData?.configObject?.apiToken) {
        this.apiToken = this.configData.configObject.apiToken;
        this.showTokenInput = false;
      }
    },

    createObject(): void {
      const dashboardElement: any = {
        title: this.submodelElementData.idShort,
        segment: this.selectedSegment,
        timeVal: this.timeVariable,
        yvals: this.yVariables,
      };
      if (this.apiToken) {
        dashboardElement.apiToken = this.apiToken;
      }
      this.dashboardAdd(dashboardElement);
    },

    /* -----------------------------------------------------------------------
       2) FETCH TOGGLE
    -----------------------------------------------------------------------*/
    toggleFetch(): void {
      this.isFetching = !this.isFetching;
      if (this.isFetching) {
        this.restartAutoFetch();
      } else {
        this.stopFetchInterval();
      }
    },
    restartAutoFetch(): void {
      this.stopFetchInterval();
      if (!this.isFetching) return;
      this.autoFetchInterval = setInterval(() => {
        this.autoFetchLinkedSegment();
      }, 3000);
    },
    stopFetchInterval(): void {
      if (this.autoFetchInterval) {
        clearInterval(this.autoFetchInterval);
        this.autoFetchInterval = null;
      }
    },

    autoFetchLinkedSegment(): void {
      if (!this.selectedSegment) return;
      const endpoint = this.selectedSegment.value.find((smc: any) =>
        this.checkIdShort(smc, 'Endpoint')
      )?.value;
      let query = this.selectedSegment.value.find((smc: any) =>
        this.checkIdShort(smc, 'Query')
      )?.value;

      
      const tokenEl = this.selectedSegment.value.find((smc: any) =>
        this.checkIdShort(smc, 'Token')
      );
      let segmentToken = tokenEl?.value || '';
      if (!segmentToken) {
        
        segmentToken = this.apiToken;
      }

      if (!endpoint || !query || !segmentToken) return;

      // If the query uses {{y-value}}, replace with first y-variable’s idShort
      if (this.yVariables?.length > 0) {
        query = query.replace(this.yVariableTemplate, this.yVariables[0].idShort);
      }

      const headers = new Headers();
      headers.append('Authorization', 'Token ' + segmentToken);
      headers.append('Accept', 'application/csv');
      headers.append('Content-Type', 'application/vnd.flux');

      this.postRequest(endpoint, query, headers, 'fetching data', false, true).then((resp: any) => {
        if (resp.success && typeof resp.data === 'string') {
          this.convertInfluxCSVtoArray(resp.data);
        }
      });
    },

    /* -----------------------------------------------------------------------
       3) CSV PARSING & ANOMALIES
    -----------------------------------------------------------------------*/

    // This is the main anomaly dispatcher.
    detectAnomalies(data: DataPoint[], recordIdShort: string): boolean[] {
      if (!data.length) return [];
      if (this.selectedAlgorithm === 'MahalanobisDistance') {
        // 1D fallback for old usage
        return this.detectMahalanobis1D(data, recordIdShort);
      }
      if (this.selectedAlgorithm === 'ZScore') {
        return this.detectZScore(data, recordIdShort);
      }
      return this.detectDefault(data);
    },

    
    detectMahalanobis1D(data: DataPoint[], recordIdShort: string): boolean[] {
      const recParams = this.mahParamsByIdShort[recordIdShort] || { mean: '0', invCov: '1', thresh: '999999' };
      const meanNum = parseFloat(recParams.mean || '0');
      const invCovNum = parseFloat(recParams.invCov || '1');
      const thresholdNum = parseFloat(recParams.thresh || '999999');
      return data.map(pt => ((pt.value - meanNum) ** 2 * invCovNum) > thresholdNum);
    },

    // multi-dimensional approach for entire set of yVariables.
    detectMahalanobisMultiDim(allSeries: DataPoint[][]): boolean[] {
      // Suppose we have N sensors => allSeries.length = N.
      // We assume each sensor has T points => allSeries[i].length = T.
      const T = allSeries[0]?.length || 0;
      const N = allSeries.length;
      if (N === 0 || T === 0) {
        return [];
      }

      // parse threshold from the first sensor or define a universal threshold
      const firstIdShort = this.yVariables[0].idShort;
      const firstParams = this.mahParamsByIdShort[firstIdShort] || { mean: '', invCov: '', thresh: '999999' };
      const thresholdNum = parseFloat(firstParams.thresh || '999999');

      // (1) gather mean vector
      // if each sensor stores a single dimension => parse it as float
      // if sensor provides CSV of means => parse that row as well
      const meanVector = new Array(N).fill(0);
      for (let i = 0; i < N; i++) {
        const idShort = this.yVariables[i].idShort;
        const param = this.mahParamsByIdShort[idShort];
        if (param && param.mean) {
          // check if we have multiple means splitted by comma
          const splitted = param.mean.split(",").map((val: string) => parseFloat(val.trim()));
          if (splitted.length === 1) {
            // single dimension stored
            meanVector[i] = splitted[0];
          } else if (splitted.length === N) {
            // if the sensor provides the entire mean vector => pick splitted[i]
            // or you might store partial means
            // but typically each sensor only stores 1 dimension
            meanVector[i] = splitted[i] || 0;
          } else {
            // fallback
            meanVector[i] = parseFloat(param.mean) || 0;
          }
        }
      }

      // (2) gather NxN inverse cov matrix by reading each sensor's "row"
      const invCovMatrix: number[][] = [];
      for (let i = 0; i < N; i++) {
        invCovMatrix[i] = new Array(N).fill(0);
      }

      for (let i = 0; i < N; i++) {
        const idShort = this.yVariables[i].idShort;
        const param = this.mahParamsByIdShort[idShort];
        if (!param || !param.invCov) {
          // missing => fill with identity?
          // For safety, fill the diagonal with 1, else 0.
          invCovMatrix[i][i] = 1;
          continue;
        }
        // parse row
        const rowValues = param.invCov.split(",").map((val: string) => parseFloat(val.trim()));
        // handle dimension mismatch
        for (let j = 0; j < N; j++) {
          invCovMatrix[i][j] = rowValues[j] !== undefined ? rowValues[j] : 0;
        }
      }

      // (3) compute Mahalanobis distance for each time step
      const anomalies = new Array(T).fill(false);

      // Optimization: we can do this in a single pass.
      for (let tIndex = 0; tIndex < T; tIndex++) {
        // build x(t) => N-d vector
        const x: number[] = [];
        for (let i = 0; i < N; i++) {
          x.push(allSeries[i][tIndex]?.value ?? 0);
        }
        const md2 = this.mahalanobisDistanceSquared(x, meanVector, invCovMatrix);
        anomalies[tIndex] = md2 > thresholdNum;
      }
      return anomalies;
    },

    // helper function: compute (x - mu)^T * M * (x - mu)
    mahalanobisDistanceSquared(x: number[], mu: number[], invCov: number[][]): number {
      const N = x.length;
      // y = x - mu
      const y = new Array(N);
      for (let i = 0; i < N; i++) {
        y[i] = x[i] - mu[i];
      }

      // z = invCov * y
      const z = new Array(N).fill(0);
      for (let i = 0; i < N; i++) {
        let sum = 0;
        for (let j = 0; j < N; j++) {
          sum += invCov[i][j] * y[j];
        }
        z[i] = sum;
      }

      // result = y^T * z
      let result = 0;
      for (let i = 0; i < N; i++) {
        result += y[i] * z[i];
      }
      return result;
    },

    detectDefault(data: DataPoint[]): boolean[] {
      const anomalies: boolean[] = [];

      
      const wSize = this.windowSize || data.length;
      const threshold = this.thresholdMultiplier || 3;
      const burnIn = this.burnInPeriod || 0;

      for (let i = 0; i < data.length; i++) {
        // Burn-in 
        if (i < burnIn) {
          anomalies.push(false);
          continue;
        }

        
        const startIndex = Math.max(burnIn, i - wSize + 1);
        const windowData = data.slice(startIndex, i + 1).map(pt => pt.value);

        
        const sortedWindow = [...windowData].sort((a, b) => a - b);
        const mid = Math.floor(sortedWindow.length / 2);
        const median = sortedWindow.length % 2 !== 0
          ? sortedWindow[mid]
          : (sortedWindow[mid - 1] + sortedWindow[mid]) / 2;

        
        const absDevs = windowData.map(v => Math.abs(v - median));
        const sortedAbs = absDevs.sort((a, b) => a - b);
        const mad = sortedAbs.length % 2 !== 0
          ? sortedAbs[mid]
          : (sortedAbs[mid - 1] + sortedAbs[mid]) / 2;

        
        if (mad !== 0 && Math.abs(data[i].value - median) > threshold * mad) {
          anomalies.push(true);
        } else {
          anomalies.push(false);
        }
      }

      return anomalies;
    },

    detectZScore(data: DataPoint[], recordIdShort: string): boolean[] {
      const recParams = this.zParamsByIdShort[recordIdShort] || { mean: '0', stdDev: '1', thresh: '3' };
      const meanNum = parseFloat(recParams.mean || '0');
      const stdNum = parseFloat(recParams.stdDev || '1');
      const threshNum = parseFloat(recParams.thresh || '3');
      return data.map(pt => Math.abs((pt.value - meanNum) / stdNum) > threshNum);
    },

    convertInfluxCSVtoArray(csvData: string): void {
      if (typeof csvData !== 'string') return;
      const lines = csvData.trim().split('\n');
      const datasets: Record<string, DataPoint[]> = {};
      let currentDataset: string[] = [];
      let currentTable: string | null = null;
      let headerLine = '';

      lines.forEach((line: string) => {
        const cols = line.split(',');
        // If we see "result" in col[1], it's the header line
        if (cols[1] === 'result') {
          headerLine = line;
          return;
        }
        const table = cols[2];
        if (currentTable === null) {
          currentTable = table;
          currentDataset.push(line);
        } else if (table !== currentTable) {
          const topic = this.extractTopic(currentDataset[0]);
          const processed = this.processInfluxDataset(headerLine, currentDataset);
          datasets[topic] = processed;
          currentDataset = [line];
          currentTable = table;
        } else {
          currentDataset.push(line);
        }
      });

      // Process the last dataset
      if (currentDataset.length > 0) {
        const topic = this.extractTopic(currentDataset[0]);
        const processed = this.processInfluxDataset(headerLine, currentDataset);
        datasets[topic] = processed;
      }

      // Filter only datasets relevant to yVariables
      const dsKeys = Object.keys(datasets);
      const relevantKeys = dsKeys.filter(key =>
        this.yVariables.some(yv => key.includes(yv.idShort))
      );

      // Check for missing yVariables
      const missingYVars = this.yVariables.filter(
        yv => !relevantKeys.some(key => key.includes(yv.idShort))
      );
      if (missingYVars.length > 0) {
        const names = missingYVars.map(v => v.idShort).join(', ');
        this.navigationStore.dispatchSnackbar({
          status: true,
          timeout: 4000,
          color: 'warning',
          btnColor: 'buttonText',
          text: `y-values "${names}" not found in the LinkedSegment data!`,
        });
      }

      // Re-order final data to match yVariables order
      const finalData: DataPoint[][] = this.yVariables
        .map(yv => relevantKeys.find(k => k.includes(yv.idShort)))
        .filter(k => k !== undefined)
        .map(k => datasets[k as string]);

      this.timeSeriesValues = finalData;
      this.initializeSeries();
    },

    extractTopic(headerLine: string): string {
      const cols = headerLine.split(',');
      return cols[cols.length - 1];
    },

    processInfluxDataset(headerLine: string, datasetLines: string[]): DataPoint[] {
      const headers = headerLine.split(',');
      const valueIndex = headers.indexOf('_value');
      const timeIndex = headers.indexOf('_time');

      return datasetLines.slice(1).map((line: string) => {
        const c = line.split(',');
        const dateObj = this.parseTimestamp(c[timeIndex]);
        return {
          time: dateObj,
          value: parseFloat(c[valueIndex]) || 0,
        };
      });
    },

    /* -----------------------------------------------------------------------
       4) CHART SETUP
    -----------------------------------------------------------------------*/
    setupApexChart(): void {
      this.chartOptions = {
        chart: {
          id: 'linkedSegmentChart',
          type: 'scatter',
          height: 350,
          background: '#ffffff00',
        },
        dataLabels: { enabled: false },
        xaxis: {
          type: 'datetime',
          range: 60000,
          tickAmount: 16,
          labels: {
            datetimeUTC: false,
            formatter: (val: number) => new Date(val).toLocaleTimeString(),
            rotate: -45,
            rotateAlways: true,
            hideOverlappingLabels: true,
          },
        },
        yaxis: {
          decimalsInFloat: 2,
        },
        markers: {
          size: 4,
          strokeColors: this.isDark ? '#1E1E1E' : '#ffffff',
        },
        grid: {
          xaxis: { lines: { show: false } },
        },
        tooltip: {
          x: {
            formatter: (val: number) => new Date(val).toLocaleString(),
          },
        },
        legend: {
          show: true,
          showForSingleSeries: true,
        },
        theme: {
          mode: this.isDark ? 'dark' : 'light',
        },
      };
      this.initializeSeries();
    },

    initializeSeries(): void {
      
      if (this.selectedAlgorithm === 'MahalanobisDistance' && this.yVariables.length > 1) {
        // handle multi-dim only if we have multiple sensors.
        
        const allSeries = this.timeSeriesValues;
        // ensure all series are same length?
        const lengths = allSeries.map(arr => arr.length);
        const uniqueLengths = new Set(lengths);
        if (uniqueLengths.size !== 1) {
          // dimension mismatch => fallback.
          console.warn('Mahalanobis multi-dim: sensor timeseries have different lengths. Fallback to 1D.');
          this.buildFallbackSeries();
          return;
        }
        // multi-dim detection:
        const anomaliesMulti = this.detectMahalanobisMultiDim(allSeries);
        const combined: any[] = [];
        const T = anomaliesMulti.length;
        const N = allSeries.length;

        for (let i = 0; i < N; i++) {
          const record = this.yVariables[i];
          const baseLabel = record?.idShort || `Series ${i}`;
          const unit = this.getUnit(record);
          const labelWithUnit = unit ? `${baseLabel}[${unit}]` : baseLabel;
          const dataSet = allSeries[i];

          const normalPoints = [];
          const anomalyPoints = [];

          for (let t = 0; t < T; t++) {
            const val = dataSet[t].value;
            const timeVal = dataSet[t].time.getTime();
            if (anomaliesMulti[t]) {
              anomalyPoints.push([timeVal, val]);
            } else {
              normalPoints.push([timeVal, val]);
            }
          }

          combined.push({
            name: `${labelWithUnit} (Normal)`,
            type: 'scatter',
            data: normalPoints,
            color: '#00E396',
            markers: { size: 4 },
          });
          combined.push({
            name: `${labelWithUnit} (Anomaly)`,
            type: 'scatter',
            data: anomalyPoints,
            color: '#FF0000',
            markers: { size: 5 },
          });
        }

        this.chartSeries = combined;
        if (this.$refs.apexchartRef) {
          (this.$refs.apexchartRef as any).updateSeries(combined);
          (this.$refs.apexchartRef as any).updateOptions(this.chartOptions);
        }
      } else {
        // fallback to current logic (or single-sensor 1D)
        this.buildFallbackSeries();
      }
    },

    buildFallbackSeries(): void {
      const combined: any[] = [];
      this.timeSeriesValues.forEach((dataset: DataPoint[], idx: number) => {
        const record = this.yVariables[idx];
        const baseLabel = record?.idShort || `Series ${idx}`;
        const unit = this.getUnit(record);
        const labelWithUnit = unit ? `${baseLabel}[${unit}]` : baseLabel;

        const anomalies = this.detectAnomalies(dataset, record.idShort);

        const normalPoints = dataset
          .filter((pt, i) => !anomalies[i])
          .map(pt => [pt.time.getTime(), pt.value]);

        const anomalyPoints = dataset
          .filter((pt, i) => anomalies[i])
          .map(pt => [pt.time.getTime(), pt.value]);

        combined.push({
          name: `${labelWithUnit} (Normal)`,
          type: 'scatter',
          data: normalPoints,
          color: '#00E396',
          markers: { size: 4 },
        });
        combined.push({
          name: `${labelWithUnit} (Anomaly)`,
          type: 'scatter',
          data: anomalyPoints,
          color: '#FF0000',
          markers: { size: 5 },
        });
      });

      this.chartSeries = combined;
      if (this.$refs.apexchartRef) {
        (this.$refs.apexchartRef as any).updateSeries(combined);
        (this.$refs.apexchartRef as any).updateOptions(this.chartOptions);
      }
    },

    changeRange(): void {
      const newVal = Number(this.range);
      if (!newVal || newVal <= 0) {
        this.range = 60000; // reset if invalid
      }
      const newOptions = { xaxis: { range: this.range } };
      (this.$refs.apexchartRef as any).updateOptions(newOptions);
      const merged = _.merge({}, this.localChartOptions, newOptions);
      this.localChartOptions = merged;
    },

    applyTheme(): void {
      if (!this.$refs.apexchartRef) return;
      const mode = this.isDark ? 'dark' : 'light';
      const strokeColors = this.isDark ? '#1E1E1E' : '#ffffff';
      (this.$refs.apexchartRef as any).updateOptions({
        theme: { mode },
        markers: { strokeColors },
      });
    },

    /**
     * Convert numeric or ISO string timestamps into a Date.
     * If < 1e12, treat as seconds; else treat as milliseconds.
     */
    parseTimestamp(timestamp: string): Date {
      if (!timestamp) return new Date();
      const val = Number(timestamp);
      if (!isNaN(val)) {
        return new Date(val);
      }
      return new Date(timestamp);
    },

    // Retrieve 'unit' from conceptDescriptions if present
    getUnit(record: any): string {
      if (record?.conceptDescriptions && record.conceptDescriptions.length > 0) {
        const cd = record.conceptDescriptions[0];
        if (cd.unit && cd.unit.trim() !== '') {
          return cd.unit;
        }
        if (
          cd.embeddedDataSpecifications &&
          cd.embeddedDataSpecifications.length > 0 &&
          cd.embeddedDataSpecifications[0].dataSpecificationContent &&
          cd.embeddedDataSpecifications[0].dataSpecificationContent.unit
        ) {
          return cd.embeddedDataSpecifications[0].dataSpecificationContent.unit;
        }
      }
      return '';
    },
  },
});
</script>

<style scoped>
/* Additional styling if needed */
</style>
