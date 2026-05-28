# Fix: Dynamic cascade of start/end times in Timers.vue

## Context

In `Timers.vue`, the 4 phases (preparation → entretien → realisation → recettage) are sequential timers. Each phase's start time should cascade from the previous phase's end time. Currently two bugs break this:

1. **Starting a phase doesn't cascade subsequent phases.** `toggleTimer` only updates `startTime[X]` and `endTime[X]` for the clicked phase — phases after X keep stale times from the initial calculation.

2. **Stopping recalculates all phases from now, including past ones.** `calculateEndTimes(Date.now())` iterates from phase 0, overwriting the start/end times of phases that have already run.

## Fix

### File to modify
`src/components/Timers.vue`

### Change 1 — Add `fromPhaseIndex` param to `calculateEndTimes`

```typescript
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
```

The initial call `calculateEndTimes(addedAt.getTime())` stays unchanged (defaults to index 0).

### Change 2 — Fix `toggleTimer` to cascade correctly

```typescript
function toggleTimer(targetStep: string) {
    const stepIndex = phases.indexOf(targetStep)

    if (step.value == targetStep && running.value) {
        // Pause: recalculate from this phase onward (not from 0)
        running.value = false
        calculateEndTimes(Date.now(), stepIndex)
    } else {
        running.value = true
        if (timer.value[targetStep] > 0) {
            const now = Date.now()
            const endMs = now + timer.value[targetStep] * 1000
            startTime.value[targetStep] = formatEndDate(now)
            endTime.value[targetStep] = formatEndDate(endMs)
            // Cascade all phases after this one
            calculateEndTimes(endMs, stepIndex + 1)
        } else {
            startTime.value[targetStep] = "--:--"
            endTime.value[targetStep] = "--:--"
            calculateEndTimes(Date.now(), stepIndex + 1)
        }
    }

    step.value = targetStep
}
```

## Why this is correct

- **Start phase X at time T**: endTime[X] = T + remaining[X]. All phases after X cascade from that endTime. Past phases are untouched.
- **Pause phase X at time T**: startTime[X] = T, endTime[X] = T + remaining[X]. All phases after X cascade. Past phases are untouched.
- **Initial load**: unchanged — all 4 phases calculated sequentially from `addedAt`.

## Verification

1. Run `npm run dev`
2. Add a student, observe the 4 phases show cascaded start→end times
3. Start "Preparation" → check that entretien/realisation/recettage start times shift to after preparation's new endTime
4. Pause preparation midway → check that subsequent phases shift again based on remaining preparation time
5. Start "Entretien" directly (skipping preparation) → realisation/recettage should cascade from entretien's endTime; preparation's times should remain unchanged
