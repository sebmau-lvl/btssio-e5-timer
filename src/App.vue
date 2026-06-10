<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

type Step = { name: string; minutes: number; start: number; end: number }
type Candidate = {
  id: number
  name: string
  thirdTime: boolean
  autoChain: boolean
  activeStep: number
  running: boolean
  paused: boolean
  endAt: number | null
  remaining: number | null
  timelineStart: number
  customGap: number | null
  completedSteps: number[]
  stepRemaining: Array<number | null>
  deadlineSignaled: boolean
  finished: boolean
  steps: Step[]
}

const baseSteps = [
  { name: 'Préparation', minutes: 30 },
  { name: 'Entretien', minutes: 20 },
  { name: 'Réalisation', minutes: 60 },
  { name: 'Recettage', minutes: 20 },
]

// Etat partagé par toutes les lignes afin de conserver une frise horaire cohérente.
const candidates = ref<Candidate[]>([])
const studentName = ref('')
const globalSound = ref(false)
const firstTimerStarted = ref(false)
const now = ref(new Date())
const theoreticalStartMinute = ref(currentMinute())
let nextCandidateId = 1
let clockInterval: ReturnType<typeof setInterval> | undefined
let timerInterval: ReturnType<typeof setInterval> | undefined

const theoreticalStart = computed({
  get: () => toTime(theoreticalStartMinute.value),
  set: (value: string) => {
    if (!firstTimerStarted.value && value) applyTheoreticalStart(toMin(value))
  },
})

const currentDate = computed(() => {
  const value = now.value.toLocaleDateString('fr-FR', {
    weekday: 'long', day: 'numeric', month: 'long', year: 'numeric',
  })
  return value.charAt(0).toUpperCase() + value.slice(1)
})

const currentTime = computed(() => now.value.toLocaleTimeString('fr-FR', {
  hour: '2-digit', minute: '2-digit', second: '2-digit',
}))

const timelineDuration = computed(() => candidates.value.length
  ? Math.max(...candidates.value.map((candidate, index) =>
    getCandidateVisualOffset(index) + getCandidateDuration(candidate)))
  : 1)

function pad(value: number) {
  return String(value).padStart(2, '0')
}

function toMin(time: string) {
  const [hours, minutes] = time.split(':').map(Number)
  return hours * 60 + minutes
}

function toTime(minutes: number) {
  const normalized = ((minutes % 1440) + 1440) % 1440
  return `${pad(Math.floor(normalized / 60))}:${pad(normalized % 60)}`
}

function formatSeconds(seconds: number) {
  const sign = seconds < 0 ? '-' : ''
  const absolute = Math.abs(seconds)
  return `${sign}${pad(Math.floor(absolute / 60))}:${pad(absolute % 60)}`
}

function secondsFromMs(milliseconds: number) {
  return (milliseconds < 0 ? -1 : 1) * Math.ceil(Math.abs(milliseconds) / 1000)
}

function currentMinute() {
  const date = new Date()
  return date.getHours() * 60 + date.getMinutes()
}

function buildSteps(candidate: Candidate) {
  let start = candidate.timelineStart
  return baseSteps.map((baseStep) => {
    // Le tiers-temps ajoute 33 % à chaque étape, avec un arrondi à la minute.
    const minutes = Math.round(baseStep.minutes * (candidate.thirdTime ? 1.33 : 1))
    const step = { name: baseStep.name, minutes, start, end: start + minutes }
    start += minutes
    return step
  })
}

function getCandidateGap(index: number) {
  const candidate = candidates.value[index]
  if (candidate?.customGap != null) return candidate.customGap
  if (candidate?.thirdTime) return 15
  // Le second candidat démarre 30 minutes après le premier, puis les suivants toutes les 25 minutes.
  return index === 1 ? 30 : 25
}

function getCandidateVisualOffset(index: number) {
  // Le décalage affiché est cumulé depuis le premier candidat.
  let offset = 0
  for (let current = 1; current <= index; current += 1) offset += getCandidateGap(current)
  return offset
}

function getCandidateDuration(candidate: Candidate) {
  return candidate.steps.reduce((total, step) => total + step.minutes, 0)
}

function timelineColumns(candidate: Candidate, index: number) {
  const offset = getCandidateVisualOffset(index)
  const duration = getCandidateDuration(candidate)
  return `${offset}fr ${duration}fr ${Math.max(0, timelineDuration.value - offset - duration)}fr`
}

function phaseColumns(candidate: Candidate) {
  return `${candidate.steps[0].minutes + candidate.steps[1].minutes}fr `
    + `${candidate.steps[2].minutes + candidate.steps[3].minutes}fr`
}

function addCandidate() {
  const name = studentName.value.trim()
  if (!name) return
  const index = candidates.value.length
  const previous = candidates.value[index - 1]
  const candidate: Candidate = {
    id: nextCandidateId++,
    name,
    thirdTime: false,
    autoChain: false,
    activeStep: 0,
    running: false,
    paused: false,
    endAt: null,
    remaining: null,
    timelineStart: previous
      ? previous.timelineStart + getCandidateGap(index)
      : theoreticalStartMinute.value,
    customGap: null,
    completedSteps: [],
    stepRemaining: [null, null, null, null],
    deadlineSignaled: false,
    finished: false,
    steps: [],
  }
  candidate.steps = buildSteps(candidate)
  candidates.value.push(candidate)
  studentName.value = ''
}

function deleteCandidate(candidate: Candidate) {
  const index = candidates.value.indexOf(candidate)
  if (index < 0) return
  candidates.value.splice(index, 1)
  // Referme le trou laissé dans le planning sans modifier le premier candidat.
  for (let current = Math.max(1, index); current < candidates.value.length; current += 1) {
    const item = candidates.value[current]
    item.timelineStart = candidates.value[current - 1].timelineStart + getCandidateGap(current)
    item.steps = buildSteps(item)
  }
}

function applyTheoreticalStart(start: number) {
  theoreticalStartMinute.value = start
  candidates.value.forEach((candidate, index) => {
    candidate.timelineStart = index === 0
      ? start
      : candidates.value[index - 1].timelineStart + getCandidateGap(index)
    candidate.steps = buildSteps(candidate)
  })
}

function propagateScheduleFrom(candidate: Candidate) {
  // Un départ réel ou un réglage de durée ne recale que les candidats suivants.
  const reference = candidates.value.indexOf(candidate)
  for (let index = reference + 1; index < candidates.value.length; index += 1) {
    const item = candidates.value[index]
    item.timelineStart = candidates.value[index - 1].timelineStart + getCandidateGap(index)
    item.steps = buildSteps(item)
  }
}

function anchorTimeline(candidate: Candidate, stepIndex: number) {
  // Positionne l'étape choisie sur l'heure courante, même si les étapes précédentes sont ignorées.
  const elapsed = candidate.steps.slice(0, stepIndex)
    .reduce((total, step) => total + step.minutes, 0)
  candidate.timelineStart = currentMinute() - elapsed
  candidate.steps = buildSteps(candidate)
}

function startStep(candidate: Candidate, stepIndex: number) {
  if (candidate.running || candidate.paused) stopCurrentStep(candidate)
  firstTimerStarted.value = true
  anchorTimeline(candidate, stepIndex)
  propagateScheduleFrom(candidate)
  candidate.activeStep = stepIndex
  candidate.finished = false
  candidate.paused = false
  candidate.running = true
  candidate.deadlineSignaled = false
  candidate.completedSteps = candidate.completedSteps.filter(index => index !== stepIndex)
  // Une étape déjà interrompue reprend depuis sa valeur sauvegardée.
  const duration = candidate.steps[stepIndex].minutes * 60 * 1000
  candidate.endAt = Date.now() + (candidate.stepRemaining[stepIndex] ?? duration)
  candidate.stepRemaining[stepIndex] = null
  candidate.remaining = null
}

function stopCurrentStep(candidate: Candidate) {
  // Changer d'étape fige le compteur courant pour permettre une reprise ultérieure.
  if (candidate.running && candidate.endAt != null) {
    candidate.stepRemaining[candidate.activeStep] = candidate.endAt - Date.now()
  } else if (candidate.paused && candidate.remaining != null) {
    candidate.stepRemaining[candidate.activeStep] = candidate.remaining
  }
  candidate.running = false
  candidate.paused = false
  candidate.endAt = null
  candidate.remaining = null
}

function pauseCandidate(candidate: Candidate) {
  if (!candidate.running || candidate.endAt == null) return
  candidate.remaining = candidate.endAt - Date.now()
  candidate.running = false
  candidate.paused = true
  candidate.endAt = null
}

function resumeCandidate(candidate: Candidate) {
  if (!candidate.paused || candidate.remaining == null) return
  candidate.endAt = Date.now() + candidate.remaining
  candidate.remaining = null
  candidate.running = true
  candidate.paused = false
}

function resetStep(candidate: Candidate, stepIndex: number) {
  if (!window.confirm(`Réinitialiser le timer « ${candidate.steps[stepIndex].name} » ?`)) return
  if (candidate.activeStep === stepIndex && (candidate.running || candidate.paused)) {
    stopCurrentStep(candidate)
  }
  candidate.stepRemaining[stepIndex] = null
  candidate.completedSteps = candidate.completedSteps.filter(index => index !== stepIndex)
  candidate.deadlineSignaled = false
  candidate.finished = false
}

function handleTimerAction(candidate: Candidate, stepIndex: number) {
  if (candidate.activeStep === stepIndex && candidate.running) pauseCandidate(candidate)
  else if (candidate.activeStep === stepIndex && candidate.paused) resumeCandidate(candidate)
  else startStep(candidate, stepIndex)
}

function changeThirdTime(candidate: Candidate) {
  const index = candidates.value.indexOf(candidate)
  if (index > 0) {
    candidate.customGap = null
    candidate.timelineStart = candidates.value[index - 1].timelineStart + getCandidateGap(index)
  }
  candidate.steps = buildSteps(candidate)
  propagateScheduleFrom(candidate)
}

function applyCandidateOffset(index: number, event: Event) {
  if (firstTimerStarted.value || index <= 0) return
  const offset = Number((event.target as HTMLInputElement).value)
  if (!Number.isFinite(offset)) return
  const candidate = candidates.value[index]
  // La saisie représente un décalage cumulé; customGap ne conserve que l'écart local.
  candidate.customGap = Math.max(0, offset - getCandidateVisualOffset(index - 1))
  candidate.timelineStart = candidates.value[index - 1].timelineStart + getCandidateGap(index)
  candidate.steps = buildSteps(candidate)
  propagateScheduleFrom(candidate)
}

function getRemaining(candidate: Candidate, stepIndex: number) {
  const duration = candidate.steps[stepIndex].minutes * 60
  if (candidate.activeStep === stepIndex && candidate.running && candidate.endAt != null) {
    return secondsFromMs(candidate.endAt - now.value.getTime())
  }
  if (candidate.activeStep === stepIndex && candidate.paused && candidate.remaining != null) {
    return secondsFromMs(candidate.remaining)
  }
  const saved = candidate.stepRemaining[stepIndex]
  return saved == null ? duration : secondsFromMs(saved)
}

function hasTimerValue(candidate: Candidate, stepIndex: number) {
  return (candidate.activeStep === stepIndex && (candidate.running || candidate.paused))
    || candidate.stepRemaining[stepIndex] != null
}

function getProgress(candidate: Candidate, stepIndex: number) {
  if (candidate.completedSteps.includes(stepIndex)) return 100
  if (!hasTimerValue(candidate, stepIndex)) return 0
  const duration = candidate.steps[stepIndex].minutes * 60
  // La jauge reste à 100 % pendant un dépassement, tandis que le compteur devient négatif.
  return Math.min(100, Math.max(0,
    ((duration - getRemaining(candidate, stepIndex)) / duration) * 100))
}

function progressLevel(progress: number) {
  if (progress >= 90) return 'danger'
  if (progress >= 70) return 'warning'
  return 'normal'
}

function timerAction(candidate: Candidate, stepIndex: number) {
  if (candidate.activeStep === stepIndex && candidate.running) return 'pause'
  if (candidate.activeStep === stepIndex && candidate.paused) return 'resume'
  return 'start'
}

function tickTimers() {
  now.value = new Date()
  candidates.value.forEach((candidate) => {
    if (!candidate.running || candidate.endAt == null || candidate.endAt > Date.now()) return
    if (!candidate.deadlineSignaled) {
      // Le signal ne doit être joué qu'une fois, même si le compteur reste négatif.
      candidate.deadlineSignaled = true
      if (globalSound.value) playBeep()
    }
    if (!candidate.autoChain) return
    // En mode automatique, l'étape terminée est figée puis la suivante démarre immédiatement.
    candidate.running = false
    if (!candidate.completedSteps.includes(candidate.activeStep)) {
      candidate.completedSteps.push(candidate.activeStep)
    }
    candidate.stepRemaining[candidate.activeStep] = 0
    candidate.endAt = null
    candidate.activeStep += 1
    if (candidate.activeStep < candidate.steps.length) {
      candidate.running = true
      candidate.deadlineSignaled = false
      candidate.endAt = Date.now() + candidate.steps[candidate.activeStep].minutes * 60 * 1000
    } else candidate.finished = true
  })
}

function playBeep() {
  const AudioContextClass = window.AudioContext
    || (window as typeof window & { webkitAudioContext?: typeof AudioContext }).webkitAudioContext
  if (!AudioContextClass) return
  const context = new AudioContextClass()
  const oscillator = context.createOscillator()
  const gain = context.createGain()
  oscillator.type = 'sine'
  oscillator.frequency.value = 880
  gain.gain.value = 0.05
  oscillator.connect(gain)
  gain.connect(context.destination)
  oscillator.start()
  gain.gain.exponentialRampToValueAtTime(0.001, context.currentTime + 3)
  oscillator.stop(context.currentTime + 3)
  oscillator.addEventListener('ended', () => context.close())
}

onMounted(() => {
  clockInterval = setInterval(() => { now.value = new Date() }, 1000)
  timerInterval = setInterval(tickTimers, 250)
})

onBeforeUnmount(() => {
  clearInterval(clockInterval)
  clearInterval(timerInterval)
})
</script>

<template>
  <div class="page">
    <header class="toolbar">
      <div class="title-group">
        <div class="header-info title-info">
          <svg class="header-icon title-icon" viewBox="0 0 24 24" aria-hidden="true">
            <rect x="3" y="4" width="18" height="13" rx="2" />
            <path d="M8 21h8M12 17v4" />
          </svg>
          <h1>BTS SIO - Epreuve E6</h1>
        </div>
        <div class="header-info">
          <svg class="header-icon" viewBox="0 0 24 24" aria-hidden="true">
            <rect x="3" y="5" width="18" height="16" rx="2" />
            <path d="M7 3v4M17 3v4M3 10h18" />
          </svg>
          <time>{{ currentDate }}</time>
        </div>
        <div class="header-info">
          <svg class="header-icon" viewBox="0 0 24 24" aria-hidden="true">
            <circle cx="12" cy="12" r="9" />
            <path d="M12 7v5l3 2" />
          </svg>
          <time class="current-time">{{ currentTime }}</time>
        </div>
      </div>

      <div class="global-options">
        <label class="theoretical-start">
          Début théorique
          <input v-model="theoreticalStart" type="time" :disabled="firstTimerStarted">
        </label>
        <label><input v-model="globalSound" type="checkbox"> Signal sonore</label>
        <input v-model="studentName" type="text" placeholder="Nom étudiant" @keyup.enter="addCandidate">
        <button type="button" :disabled="!studentName.trim()" @click="addCandidate">Ajouter</button>
      </div>
    </header>

    <main class="candidates">
      <section v-for="(candidate, candidateIndex) in candidates" :key="candidate.id" class="candidate">
        <div class="candidate-header">
          <div class="candidate-title">
            <button type="button" class="delete-candidate"
              :aria-label="`Supprimer ${candidate.name}`" @click="deleteCandidate(candidate)">
              &times;
            </button>
            <span>{{ candidate.name }}</span>
          </div>
          <div class="candidate-options">
            <label><input v-model="candidate.autoChain" type="checkbox"> Enchaînement timers auto</label>
            <label>
              <input v-model="candidate.thirdTime" type="checkbox" @change="changeThirdTime(candidate)">
              Tiers temps
            </label>
          </div>
        </div>

        <div class="timeline-lane" :style="{ gridTemplateColumns: timelineColumns(candidate, candidateIndex) }">
          <div class="timeline-offset">
            <label v-if="candidateIndex > 0" class="offset-editor">
              <span>+</span>
              <input type="number" class="offset-input"
                :min="getCandidateVisualOffset(candidateIndex - 1)"
                :value="getCandidateVisualOffset(candidateIndex)"
                :disabled="firstTimerStarted"
                @change="applyCandidateOffset(candidateIndex, $event)">
              <span>min</span>
            </label>
          </div>

          <div class="phases" :style="{ gridTemplateColumns: phaseColumns(candidate) }">
            <fieldset v-for="phase in [
              { name: 'PHASE 1', indexes: [0, 1] },
              { name: 'PHASE 2', indexes: [2, 3] },
            ]" :key="phase.name" class="phase">
              <legend>{{ phase.name }}</legend>
              <div class="phase-steps" :style="{ gridTemplateColumns:
                phase.indexes.map(index => `${candidate.steps[index].minutes}fr`).join(' ') }">
                <div v-for="stepIndex in phase.indexes" :key="stepIndex" class="step"
                  :class="{
                    running: candidate.activeStep === stepIndex && candidate.running,
                    paused: candidate.activeStep === stepIndex && candidate.paused,
                  }">
                  <div class="step-controls">
                    <button type="button" class="timer-btn"
                      :class="timerAction(candidate, stepIndex)"
                      @click="handleTimerAction(candidate, stepIndex)" />
                    <button type="button" class="reset-btn"
                      @click="resetStep(candidate, stepIndex)">RESET</button>
                  </div>
                  <div class="step-content">
                    <div class="step-name">{{ candidate.steps[stepIndex].name }}</div>
                    <div class="step-details">
                      <div class="step-time" :class="{
                        overtime: getRemaining(candidate, stepIndex) < 0,
                        'overtime-running': candidate.activeStep === stepIndex
                          && candidate.running && getRemaining(candidate, stepIndex) < 0,
                      }">{{ formatSeconds(getRemaining(candidate, stepIndex)) }}</div>
                      <div class="step-range">
                        {{ toTime(candidate.steps[stepIndex].start) }}
                        →
                        {{ toTime(candidate.steps[stepIndex].end) }}
                      </div>
                    </div>
                  </div>
                  <div class="progress-row">
                    <div class="progress-track" role="progressbar"
                      :aria-valuenow="Math.floor(getProgress(candidate, stepIndex))"
                      aria-valuemin="0" aria-valuemax="100">
                      <div class="progress-fill"
                        :class="progressLevel(getProgress(candidate, stepIndex))"
                        :style="{ width: `${getProgress(candidate, stepIndex)}%` }" />
                    </div>
                    <span class="progress-value">{{ Math.floor(getProgress(candidate, stepIndex)) }}%</span>
                  </div>
                </div>
              </div>
            </fieldset>
          </div>
          <div class="timeline-tail" />
        </div>
      </section>
    </main>
  </div>
</template>
