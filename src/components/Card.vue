<script lang="ts" setup>
import { ref, computed, watch } from 'vue'

defineProps<{ title: string, value: string, active: boolean, startTime: string, targetTime: string }>()

</script>

<template>
    <div class="flex items-center p-4 bg-white rounded-lg shadow-xs dark:bg-gray-800">
        <div class="hidden md:block p-3 mr-4 text-orange-500 bg-orange-100 rounded-full dark:text-orange-100 dark:bg-orange-500">
            <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 512 512" v-if="!active">
                <path d="M133,440a35.37,35.37,0,0,1-17.5-4.67c-12-6.8-19.46-20-19.46-34.33V111c0-14.37,7.46-27.53,19.46-34.33a35.13,35.13,0,0,1,35.77.45L399.12,225.48a36,36,0,0,1,0,61L151.23,434.88A35.5,35.5,0,0,1,133,440Z"></path>
            </svg>

            <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 512 512" v-if="active">
                <rect width="500" height="500"/>
            </svg>
        </div>
        <div>
            <p class="mb-2 text-left text-sm font-medium text-gray-600 dark:text-white">
                {{title}}
            </p>
            <p :class="{'text-4xl': true, 'font-semibold': active, 'text-gray-700': true, 'dark:text-white': true}">
                {{value}}
                <span class="small">{{ startTime }} → {{ targetTime }}</span>
            </p>
        </div>
    </div>
</template>

<style scoped>

.small {
    font-size: 0.5em;
    display: block;
    text-align: left;
}

</style>