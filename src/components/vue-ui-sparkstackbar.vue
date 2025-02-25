<script setup>
import { ref, computed, onMounted, watch } from "vue";
import {
    applyDataLabel,
    convertColorToHex, 
    convertCustomPalette, 
    createUid,
    dataLabel,
    error,
    getMissingDatasetAttributes,
    objectIsEmpty,
    palette, 
    setOpacity,
    shiftHue,
    themePalettes,
    XMLNS 
} from "../lib";
import themes from "../themes.json";
import { useNestedProp } from "../useNestedProp";
import Skeleton from "./vue-ui-skeleton.vue";
import { useConfig } from "../useConfig";

const { vue_ui_sparkstackbar: DEFAULT_CONFIG } = useConfig()

const props = defineProps({
    config: {
        type: Object,
        default() {
            return {}
        }
    },
    dataset: {
        type: Array,
        default() {
            return [];
        }
    }
});

const isDataset = computed(() => {
    return !!props.dataset && props.dataset.length;
});

const uid = ref(createUid());

const FINAL_CONFIG = computed({
    get: () => {
        return prepareConfig();
    },
    set: (newCfg) => {
        return newCfg
    }
});

function prepareConfig() {
    const mergedConfig = useNestedProp({
        userConfig: props.config,
        defaultConfig: DEFAULT_CONFIG
    });
    if (mergedConfig.theme) {
        return {
            ...useNestedProp({
                userConfig: themes.vue_ui_sparkstackbar[mergedConfig.theme] || props.config,
                defaultConfig: mergedConfig
            }),
            customPalette: themePalettes[mergedConfig.theme] || palette
        }
    } else {
        return mergedConfig;
    }
}

watch(() => props.config, (_newCfg) => {
    FINAL_CONFIG.value = prepareConfig();
    prepareChart();
}, { deep: true });

watch(() => props.dataset, (_) => {
    safeDatasetCopy.value = props.dataset.map((d, i ) => {
    return {
        ...d,
        color: d.color ? convertColorToHex(d.color) : customPalette.value[i] || palette[i] || palette[i % palette.length]
    }
})
}, { deep: true })

const customPalette = computed(() => {
    return convertCustomPalette(FINAL_CONFIG.value.customPalette);
})

const safeDatasetCopy = ref(props.dataset.map((d, i ) => {
    return {
        ...d,
        value: FINAL_CONFIG.value.style.animation.show ? 0 : d.value || 0,
        color: d.color ? convertColorToHex(d.color) : customPalette.value[i] || palette[i] || palette[i % palette.length]
    }
}));

const isLoading = ref(true);

onMounted(() => {
    prepareChart()

    if (FINAL_CONFIG.value.style.animation.show) {
        const chunks = FINAL_CONFIG.value.style.animation.animationFrames;
        const chunkSet = props.dataset.map((d, i) => d.value / chunks);
        const total = props.dataset.map(d => d.value || 0).reduce((a, b) => a + b, 0);
        let start = 0;
        isLoading.value = true;

        function animate() {
            start += (total / chunks);
            if (start < total) {
                safeDatasetCopy.value = safeDatasetCopy.value.map((d, i) => {
                    return {
                        ...d,
                        value: d.value += chunkSet[i],
                        color: d.color ? convertColorToHex(d.color) : customPalette.value[i] || palette[i] || palette[i % palette.length]
                    }
                });
                requestAnimationFrame(animate)
            } else {
                isLoading.value = false;
                safeDatasetCopy.value = props.dataset.map((d,i) => {
                    return {
                        ...d,
                        value: d.value || 0,
                        color: d.color ? convertColorToHex(d.color) : customPalette.value[i] || palette[i] || palette[i % palette.length],
                        id: createUid(),
                    }
                })
            }
        }
        animate()
    }
});

function prepareChart() {
    if(objectIsEmpty(props.dataset)) {
        error({
            componentName: 'VueUiSparkStackbar',
            type: 'dataset'
        })
    } else {
        props.dataset.forEach((ds, i) => {
            getMissingDatasetAttributes({
                datasetObject: ds,
                requiredAttributes: ['name', 'value']
            }).forEach(attr => {
                error({
                    componentName: 'VueUiSparkStackbar',
                    type: 'datasetSerieAttribute',
                    property: attr,
                    index: i
                });
            });
        });
    }
}

const svg = ref({
    width: 500,
    height: 16,
});

const segregated = ref([])

const total = computed(() => {
    return props.dataset.map(d => d.value || 0).filter((ds, i) => !segregated.value.includes(i)).reduce((a, b) => a + b, 0);
});

const absoluteDataset = computed(() => {
    return safeDatasetCopy.value.map((d, i) => {
        return {
            ...d,
            value: d.value || 0,
            proportion: (d.value || 0) / total.value,
            width: (d.value || 0) / total.value * svg.value.width,
            proportionLabel: dataLabel({
                v: (d.value || 0) / total.value * 100,
                s: '%',
                r: FINAL_CONFIG.value.style.legend.percentage.rounding
            }),
        }
    })
});

const mutableDataset = computed(() => {
    return absoluteDataset.value.filter((ds, i) => !segregated.value.includes(i))
});


function segregate(index) {
    if(segregated.value.includes(index)) {
        segregated.value =segregated.value.filter(s => s !== index)
    } else {
        if(segregated.value.length < safeDatasetCopy.value.length - 1) {
            segregated.value.push(index)
        }
    }
}

const drawableDataset = computed(() => {
    let start = 0;
    const datapoints = [];
    for (let i = 0; i < mutableDataset.value.length; i += 1) {
        datapoints.push({
            ...mutableDataset.value[i],
            start
        });
        start += mutableDataset.value[i].width
    }
    return datapoints
});

const emits = defineEmits(['selectDatapoint'])

function selectDatapoint(datapoint, index) {
    emits('selectDatapoint', { datapoint, index })
}

</script>

<template>
    <div :style="`width:100%; background:${FINAL_CONFIG.style.backgroundColor}`">
        <!-- TITLE -->
        <div data-cy="sparkstackbar-title-wrapper" v-if="FINAL_CONFIG.style.title.text" :style="`width:calc(100% - 12px);background:transparent;margin:0 auto;margin:${FINAL_CONFIG.style.title.margin};padding: 0 6px;text-align:${FINAL_CONFIG.style.title.textAlign}`">
            <div data-cy="sparkstackbar-title" :style="`font-size:${FINAL_CONFIG.style.title.fontSize}px;color:${FINAL_CONFIG.style.title.color};font-weight:${FINAL_CONFIG.style.title.bold ? 'bold' : 'normal'}`">
                {{ FINAL_CONFIG.style.title.text }}
            </div>
            <div data-cy="sparkstackbar-subtitle" v-if="FINAL_CONFIG.style.title.subtitle.text" :style="`font-size:${FINAL_CONFIG.style.title.subtitle.fontSize}px;color:${FINAL_CONFIG.style.title.subtitle.color};font-weight:${FINAL_CONFIG.style.title.subtitle.bold ? 'bold' : 'normal'}`">
                {{ FINAL_CONFIG.style.title.subtitle.text }}
            </div>
            
        </div>
        <!-- CHART -->
        <svg :xmlns="XMLNS" v-if="isDataset" width="100%" :viewBox="`0 0 ${svg.width} ${svg.height}`">
        <defs>
            <linearGradient v-for="(rect, i) in drawableDataset" :key="`stack_gradient_${i}`" gradientTransform="rotate(90)" :id="`stack_gradient_${i}_${uid}`">
                <stop offset="0%" :stop-color="rect.color"/>
                <stop offset="50%" :stop-color="setOpacity(shiftHue(rect.color, 0.05), 100 - FINAL_CONFIG.style.bar.gradient.intensity)"/>
                <stop offset="100%" :stop-color="rect.color"/>
            </linearGradient>
            <clipPath id="stackPill" clipPathUnits="objectBoundingBox">
                <rect
                    x="0.005"
                    y="-2"
                    width="0.99"
                    height="5"
                    rx="3"
                    ry="3"
                    :fill="FINAL_CONFIG.style.backgroundColor"
                />
            </clipPath>
        </defs>
            <g clip-path="url(#stackPill)">
                <rect 
                    :x="0" 
                    :y="0" 
                    :height="svg.height" 
                    :width="drawableDataset.map(ds => ds.width).reduce((a, b) => a + b, 0)" 
                    :fill="FINAL_CONFIG.style.bar.gradient.underlayerColor"
                    :class="{'animated': !isLoading}"
                />
                <rect 
                    v-for="(rect, i) in drawableDataset" :key="`stack_${i}`"
                    @click="() => selectDatapoint(rect, i)"
                    :x="rect.start"
                    :y="0"
                    :width="rect.width"
                    :height="svg.height"
                    :fill="FINAL_CONFIG.style.bar.gradient.show ? `url(#stack_gradient_${i}_${uid})` : rect.color"
                    :stroke="FINAL_CONFIG.style.backgroundColor"
                    stroke-linecap="round"
                    :class="{'animated': !isLoading}"
                />
            </g>
        </svg>

        <Skeleton
            v-if="!isDataset"
            :config="{
                type: 'sparkStackbar',
                style: {
                    backgroundColor: FINAL_CONFIG.style.backgroundColor,
                    sparkStackbar: {
                        color: '#CCCCCC'
                    }
                }
            }"
        />
        <div 
            v-if="FINAL_CONFIG.style.legend.show" 
            data-cy="sparkstackbar-legend" 
            :style="`background:transparent;margin:0 auto;margin:${FINAL_CONFIG.style.legend.margin};justify-content:${FINAL_CONFIG.style.legend.textAlign === 'left' ? 'flex-start' : FINAL_CONFIG.style.legend.textAlign === 'right' ? 'flex-end' : 'center'}`" 
            class="vue-ui-sparkstackbar-legend"
        >
            <div 
                v-for=" (rect, i) in absoluteDataset" 
                :style="`font-size:${FINAL_CONFIG.style.legend.fontSize}px;`" 
                :class="{'vue-ui-sparkstackbar-legend-item': true, 'vue-ui-sparkstackbar-legend-item-unselected': segregated.includes(i)}" 
                @click="segregate(i); selectDatapoint(rect, i)"

            >
                <div style="display:flex;flex-direction:row;align-items:center;gap:4px;justify-content:center" >
                    <svg 
                        :height="`${FINAL_CONFIG.style.legend.fontSize}px`" 
                        :width="`${FINAL_CONFIG.style.legend.fontSize}px`" 
                        viewBox="0 0 10 10"
                    >
                        <circle :cx="5" :cy="5" :r="5" :fill="rect.color"/>
                    </svg>
                    <span :style="`color:${FINAL_CONFIG.style.legend.name.color}; font-weight:${FINAL_CONFIG.style.legend.name.bold ? 'bold' : 'normal'}`">
                        {{ rect.name }}
                    </span>
                    <span 
                        v-if="FINAL_CONFIG.style.legend.percentage.show" 
                        :style="`font-weight:${FINAL_CONFIG.style.legend.percentage.bold ? 'bold': 'normal'};color:${FINAL_CONFIG.style.legend.percentage.color}`"
                    >
                        {{ segregated.includes(i) ? ' - ' : rect.proportionLabel }}
                    </span>
                    <span 
                        v-if="FINAL_CONFIG.style.legend.value.show" 
                        :style="`font-weight:${FINAL_CONFIG.style.legend.value.bold ? 'bold' : 'normal'};color:${FINAL_CONFIG.style.legend.value.color}`"
                    >
                        ({{ applyDataLabel(
                            FINAL_CONFIG.style.legend.value.formatter,
                            rect.value,
                            dataLabel({
                                p: FINAL_CONFIG.style.legend.value.prefix,
                                v: rect.value,
                                s: FINAL_CONFIG.style.legend.value.suffix,
                                r: FINAL_CONFIG.style.legend.value.rounding
                            }),
                            { datapoint: rect, seriesIndex: i}
                            )  
                        }})
                    </span> 
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.vue-ui-sparkstackbar-legend {
    display: flex;
    flex-wrap: wrap;
    column-gap: 12px;
    width: calc(100% - 12px);
    padding: 0 6px;
}
.vue-ui-sparkstackbar-legend-item {
    cursor: pointer;
    transition: opacity 0.2s ease-in-out;
}
.vue-ui-sparkstackbar-legend-item-unselected {
    opacity: 0.3;
}
rect.animated {
    transition: all 0.3s ease-in-out !important;
}
</style>