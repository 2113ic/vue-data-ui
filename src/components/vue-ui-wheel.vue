<script setup>
import { ref, computed, onMounted, watch, onBeforeUnmount } from "vue";
import themes from "../themes.json";
import Title from "../atoms/Title.vue";
import UserOptions from "../atoms/UserOptions.vue";
import { 
    applyDataLabel,
    checkNaN,
    createUid,
    dataLabel,
    error,
    objectIsEmpty,
    shiftHue,
    XMLNS
} from "../lib";
import { throttle } from "../canvas-lib";
import Skeleton from "./vue-ui-skeleton.vue";
import { useNestedProp } from "../useNestedProp";
import { usePrinter } from "../usePrinter";
import { useResponsive } from "../useResponsive";
import { useConfig } from "../useConfig";

const { vue_ui_wheel: DEFAULT_CONFIG } = useConfig()

const props = defineProps({
    config: {
        type: Object,
        default() {
            return {}
        }
    },
    dataset: {
        type: Object,
        default() {
            return {}
        }
    },
});

const isDataset = computed(() => {
    return !!props.dataset && Object.keys(props.dataset).length
})

const uid = ref(createUid());
const details = ref(null);
const step = ref(0);
const wheelChart = ref(null);
const chartTitle = ref(null);
const titleStep = ref(0);

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
                userConfig: themes.vue_ui_wheel[mergedConfig.theme] || props.config,
                defaultConfig: mergedConfig
            }),
        }
    } else {
        return mergedConfig;
    }
}

watch(() => props.config, (_newCfg) => {
    FINAL_CONFIG.value = prepareConfig();
    prepareChart();
    titleStep.value += 1;
}, { deep: true });

const { isPrinting, isImaging, generatePdf, generateImage } = usePrinter({
    elementId: uid.value,
    fileName: FINAL_CONFIG.value.style.chart.title.text || 'vue-ui-wheel'
});

const svg = ref({
    size: 360,
    height: 360,
    width: 360
});

const baseLabelFontSize = ref(FINAL_CONFIG.value.style.chart.layout.percentage.fontSize)

const wheel = computed(() => {
    return {
        radius: (Math.min(svg.value.width, svg.value.height) * 0.9) / 2,
        centerX: svg.value.width / 2,
        centerY: svg.value.height / 2,
    }
})

function calcTickStart(angle, distance = 1) {
    const angleStart = 29.85;
    return {
        x: wheel.value.centerX + wheel.value.radius * Math.cos(angleStart + angle * Math.PI / 180) * distance,
        y: wheel.value.centerY + wheel.value.radius * Math.sin(angleStart + angle * Math.PI / 180) * distance
    }
}

const activeValue = ref(FINAL_CONFIG.value.style.chart.animation.use ? 0 : (props.dataset.percentage || 0));

watch(() => props.dataset, (v) => {
    if (FINAL_CONFIG.value.style.chart.animation.use) {
        useAnimation(v.percentage);
    } else {
        activeValue.value = v.percentage || 0
    }
}, { deep: true });

const resizeObserver = ref(null);

onMounted(() => {
    prepareChart();
});

function prepareChart() {
    if (objectIsEmpty(props.dataset)) {
        error({
            componentName: 'VueUiWheel',
            type: 'dataset'
        })
    }
    useAnimation(props.dataset.percentage || 0);

    if (FINAL_CONFIG.value.responsive) {
        const handleResize = throttle(() => {
            const { width, height } = useResponsive({
                chart: wheelChart.value,
                title: FINAL_CONFIG.value.style.chart.title.text ? chartTitle.value : null,
            });
            svg.value.width = width;
            svg.value.height = height;
            baseLabelFontSize.value = (FINAL_CONFIG.value.style.chart.layout.percentage.fontSize / 360) * Math.min(width, height);
        });

        resizeObserver.value = new ResizeObserver(handleResize);
        resizeObserver.value.observe(wheelChart.value.parentNode);
    }
}

onBeforeUnmount(() => {
    if (resizeObserver.value) resizeObserver.value.disconnect();
});

function useAnimation(targetValue) {
    let speed = FINAL_CONFIG.value.style.chart.animation.speed;
    const chunk = Math.abs(targetValue - activeValue.value) / (speed * 120);

    function animate() {
        if(activeValue.value < targetValue) {
            activeValue.value = Math.min(activeValue.value + chunk, targetValue);
        } else if (activeValue.value > targetValue) {
            activeValue.value = Math.max(activeValue.value - chunk, targetValue)
        }
        
        if (activeValue.value !== targetValue) {
            requestAnimationFrame(animate)
        }
    }
    animate()
}

const ticks = computed(() => {
    const tickArray = [];
    const tickAmount = 100;
    for(let i = 0; i < tickAmount; i += 1) {
        const color = activeValue.value > i ? FINAL_CONFIG.value.style.chart.layout.wheel.ticks.activeColor : FINAL_CONFIG.value.style.chart.layout.wheel.ticks.inactiveColor;
        const { x: x1, y: y1 } = calcTickStart((svg.value.size / tickAmount) * i);
        const { x: x2, y: y2 } = calcTickStart((svg.value.size / tickAmount) * i, FINAL_CONFIG.value.style.chart.layout.wheel.ticks.sizeRatio);
        tickArray.push({
            x1,
            y1,
            x2,
            y2,
            color: FINAL_CONFIG.value.style.chart.layout.wheel.ticks.gradient.show ? shiftHue(color, i / tickAmount * (FINAL_CONFIG.value.style.chart.layout.wheel.ticks.gradient.shiftHueIntensity / 100)) : color
        })
    }
    return tickArray
});

const isFullscreen = ref(false)
function toggleFullscreen(state) {
    isFullscreen.value = state;
    step.value += 1;
}

defineExpose({
    generatePdf,
    generateImage
});

</script>

<template>
    <div 
        class="vue-ui-wheel" 
        ref="wheelChart"
        :id="uid"
        :style="`font-family:${FINAL_CONFIG.style.fontFamily};width:100%; text-align:center;background:${FINAL_CONFIG.style.chart.backgroundColor};${FINAL_CONFIG.responsive ? 'height:100%' : ''}`"
    >
        <div ref="chartTitle" v-if="FINAL_CONFIG.style.chart.title.text" :style="`width:100%;background:transparent;padding-bottom:12px`">
            <Title
                :key="`title_${titleStep}`"
                :config="{
                    title: {
                        cy: 'wheel-title',
                        ...FINAL_CONFIG.style.chart.title
                    },
                    subtitle: {
                        cy: 'wheel-subtitle',
                        ...FINAL_CONFIG.style.chart.title.subtitle
                    },
                }"
            />
        </div>

        <UserOptions
            ref="details"
            :key="`user_options_${step}`"
            v-if="FINAL_CONFIG.userOptions.show && isDataset"
            :backgroundColor="FINAL_CONFIG.style.chart.backgroundColor"
            :color="FINAL_CONFIG.style.chart.color"
            :isPrinting="isPrinting"
            :isImaging="isImaging"
            :uid="uid"
            :hasPdf="FINAL_CONFIG.userOptions.buttons.pdf"
            :hasImg="FINAL_CONFIG.userOptions.buttons.img"
            :hasFullscreen="FINAL_CONFIG.userOptions.buttons.fullscreen"
            :hasXls="false"
            :isFullscreen="isFullscreen"
            :position="FINAL_CONFIG.userOptions.position"
            :titles="{ ...FINAL_CONFIG.userOptions.buttonTitles }"
            :chartElement="wheelChart"
            @toggleFullscreen="toggleFullscreen"
            @generatePdf="generatePdf"
            @generateImage="generateImage"
        >
            <template #optionPdf v-if="$slots.optionPdf">
                <slot name="optionPdf" />
            </template>
            <template #optionImg v-if="$slots.optionImg">
                <slot name="optionImg" />
            </template>
            <template v-if="$slots.optionFullscreen" template #optionFullscreen="{ toggleFullscreen, isFullscreen }">
                <slot name="optionFullscreen" v-bind="{ toggleFullscreen, isFullscreen }"/>
            </template>
        </UserOptions>

        <svg :xmlns="XMLNS" v-if="isDataset" :class="{ 'vue-data-ui-fullscreen--on': isFullscreen, 'vue-data-ui-fulscreen--off': !isFullscreen }" data-cy="wheel-svg" :viewBox="`0 0 ${svg.width <= 0 ? 10 : svg.width} ${svg.height <= 0 ? 10 : svg.height}`" :style="`max-width:100%;overflow:visible;background:transparent;color:${FINAL_CONFIG.style.chart.color}`">
            <line 
                v-for="(tick, i) in ticks"
                :x1="tick.x1"
                :x2="tick.x2"
                :y1="tick.y1"
                :y2="tick.y2"
                :stroke="tick.color"
                :stroke-width="(5 / 360) * Math.min(svg.width, svg.height)"
                :stroke-linecap="FINAL_CONFIG.style.chart.layout.wheel.ticks.rounded ? 'round' : 'butt'"
                :class="{ 'vue-ui-tick-animated': FINAL_CONFIG.style.chart.animation.use && i <= activeValue }"
            />
            <circle 
                v-if="FINAL_CONFIG.style.chart.layout.innerCircle.show"
                :cx="wheel.centerX"
                :cy="wheel.centerY"
                :r="wheel.radius * 0.8 <= 0 ? 0.0001 : wheel.radius * 0.8"
                :stroke="FINAL_CONFIG.style.chart.layout.innerCircle.stroke"
                :stroke-width="FINAL_CONFIG.style.chart.layout.innerCircle.strokeWidth"
                fill="none"
            />
            <text
                v-if="FINAL_CONFIG.style.chart.layout.percentage.show"
                :x="wheel.centerX"
                :y="wheel.centerY + baseLabelFontSize / 3"
                :font-size="baseLabelFontSize"
                :fill="FINAL_CONFIG.style.chart.layout.wheel.ticks.gradient.show ? shiftHue(FINAL_CONFIG.style.chart.layout.wheel.ticks.activeColor, activeValue / 100 * (FINAL_CONFIG.style.chart.layout.wheel.ticks.gradient.shiftHueIntensity / 100)) : FINAL_CONFIG.style.chart.layout.wheel.ticks.activeColor"
                text-anchor="middle"
                :font-weight="FINAL_CONFIG.style.chart.layout.percentage.bold ? 'bold' : 'normal'"
                style="font-variant-numeric:tabluar-nums"
            >
                {{ applyDataLabel(
                    FINAL_CONFIG.style.chart.layout.percentage.formatter,
                    checkNaN(activeValue),
                    dataLabel({
                        v: checkNaN(activeValue),
                        s: '%',
                        r: FINAL_CONFIG.style.chart.layout.percentage.rounding
                    }))
                }}
            </text>
            <slot name="svg" :svg="svg"/>
        </svg>

        <div v-if="$slots.watermark" class="vue-data-ui-watermark">
            <slot name="watermark" v-bind="{ isPrinting: isPrinting || isImaging }"/>
        </div>

        <Skeleton
            v-if="!isDataset"
            :config="{
                type: 'wheel',
                style: {
                    backgroundColor: FINAL_CONFIG.style.chart.backgroundColor,
                    wheel: {
                        color: '#CCCCCC'
                    }
                }
            }"
        />
    </div>
</template>

<style scoped>
@import "../vue-data-ui.css";
.vue-ui-wheel *{
    transition: unset;
}
.vue-ui-wheel {
    user-select: none;
    position: relative;
}
.vue-ui-tick-animated {
    animation: animate-tick 0.3s ease-in;
    transform-origin: center;
}

@keyframes animate-tick {
    0% {
        stroke-width: 8;
    }
    80% {
        stroke-width: 6;
    }
    100% {
        stroke-width: 5;
    }
}
</style>