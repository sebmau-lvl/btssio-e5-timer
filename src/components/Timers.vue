<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import Card from './Card.vue'
import Close from './Close.vue';
defineEmits(['finish'])

const props = defineProps<{ etudiant: string, isTiersTemps: boolean }>()
let addedAt = new Date()
let step = ref("")
let running = ref(false)

const phases = ['preparation', 'entretien', 'realisation', 'recettage'] as const

const timer: any = ref({
    "preparation": 1800,
    "entretien": 1200,
    "realisation": 3600,
    "recettage": 1200,
})

const endTime: any = ref({
    "preparation": "--:--",
    "entretien": "--:--",
    "realisation": "--:--",
    "recettage": "--:--",
})

const startTime: any = ref({
    "preparation": "--:--",
    "entretien": "--:--",
    "realisation": "--:--",
    "recettage": "--:--",
})

if(props.isTiersTemps){
    for (const key in timer.value) {
        timer.value[key] = timer.value[key] + (timer.value[key] / 3)
    }
}

// Précalcul des heures théoriques dès la création
calculateEndTimes(addedAt.getTime())

const addedAtFormated = computed(() => addedAt.toLocaleTimeString("fr-FR", {hour: '2-digit', minute:'2-digit', hour12: false}))

function toggleTimer(targetStep: string){
    const stepIndex = phases.indexOf(targetStep)

    if(step.value == targetStep && running.value){
        running.value = false
        calculateEndTimes(Date.now(), stepIndex)
    } else {
        running.value = true
        if(timer.value[targetStep] > 0){
            const now = Date.now()
            const endMs = now + timer.value[targetStep] * 1000
            startTime.value[targetStep] = formatEndDate(now)
            endTime.value[targetStep] = formatEndDate(endMs)
            calculateEndTimes(endMs, stepIndex + 1)
        } else {
            startTime.value[targetStep] = "--:--"
            endTime.value[targetStep] = "--:--"
            calculateEndTimes(Date.now(), stepIndex + 1)
        }
    }

    step.value = targetStep
}

const timeInterval = ref();
watch(running, (value) => {
    if(!value){
        clearInterval(timeInterval.value)
    } else {
        timeInterval.value = setInterval(() => {
            timer.value[step.value]--;
        }, 1000)
    }
})

function fancyTimeFormat(s: number): string
{
    if(s < 0){
        s = Math.abs(s)
        return '-' + fancyTimeFormat(s)
    }

    return(s-(s%=60))/60+(9<s?':':':0')+s
}

function formatEndDate(ms: number): string
{
    return new Date(ms).toLocaleTimeString("fr-FR", {hour: '2-digit', minute:'2-digit', hour12: false})
}

function calculateEndTimes(baseTimeMs: number, fromPhaseIndex: number = 0) {
    let cumulative = baseTimeMs
    for (let i = fromPhaseIndex; i < phases.length; i++) {
        const phase = phases[i]
        if (timer.value[phase] > 0) {
            startTime.value[phase] = formatEndDate(cumulative)
            cumulative += timer.value[phase] * 1000
            endTime.value[phase] = formatEndDate(cumulative)
        } else {
            startTime.value[phase] = "--:--"
            endTime.value[phase] = "--:--"
        }
    }
}

</script>

<template>
    <h3 class="text-left dark:text-white font-bold text-xl">
        <Close @close="$emit('finish')" />
        À {{addedAtFormated}} : {{etudiant}}
    </h3>
    <div class="grid mb-8 gap-2 md:gap-2 grid-cols-4 md:grid-cols-4">
        <Card class="pointer" :startTime="startTime.preparation" :targetTime="endTime.preparation" @click="toggleTimer('preparation')" :active="running && step === 'preparation'" title="Preparation" :value="fancyTimeFormat(timer.preparation)"></Card>
        <Card class="pointer" :startTime="startTime.entretien" :targetTime="endTime.entretien" @click="toggleTimer('entretien')" :active="running && step === 'entretien'" title="Entretien" :value="fancyTimeFormat(timer.entretien)"></Card>
        <Card class="pointer" :startTime="startTime.realisation" :targetTime="endTime.realisation" @click="toggleTimer('realisation')" :active="running && step === 'realisation'" title="Réalisation" :value="fancyTimeFormat(timer.realisation)"></Card>
        <Card class="pointer" :startTime="startTime.recettage" :targetTime="endTime.recettage" @click="toggleTimer('recettage')" :active="running && step === 'recettage'" title="Recettage" :value="fancyTimeFormat(timer.recettage)"></Card>
    </div>
</template>

<style scoped>

.fixed-width {
    width: 100px;
}

.pointer{
    cursor: pointer;
}

</style>