<script setup lang="ts">
import { ref, watch } from 'vue'
import { useWakeLock, useFullscreen } from '@vueuse/core'
import Timers from './components/Timers.vue';
import Topbar from './components/Topbar.vue';

const etudiants = ref({} as { [key: string]: { name: string, isTiersTemps: boolean } });

let studentNameInput = ref("");

function addStudent(tiersTemps: boolean = false){
  if(studentNameInput.value){
    etudiants.value[Date.now()] = {name: studentNameInput.value, isTiersTemps: tiersTemps};
    studentNameInput.value = '';
  }
}

function removeStudent(key: any){
  delete etudiants.value[key];
}

const { isFullscreen } = useFullscreen()
const { request, release } = useWakeLock()

watch(isFullscreen, (fullscreen) => {
  if (fullscreen) {
    request('screen')
  } else {
    release()
  }
})
</script>

<template>
    <Topbar />

    <div class="text-center">
      <img alt="BTS SIO" class="logo mx-auto m-10 dark:invert" src="./assets/logo-certa.png" />
    </div>

    <div class="py-5">
      <main class="h-full overflow-y-auto">
          <div class="container  mx-auto grid">
            <Timers @finish="() => removeStudent(key)" :key="key" v-for="(value, key) in etudiants" :etudiant="value.name" :isTiersTemps="value.isTiersTemps" />
          </div>
      </main>

      <div class="text-center p-10">
        <input @keyup.enter="addStudent(false)" class="mx-4 p-2.5 text-gray-900 bg-white rounded-lg border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-800 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500" style="margin-right: 10px; margin-bottom: 10px;" type="text" placeholder="Nom étudiant" v-model="studentNameInput"/>
        <button type="button" @click="addStudent(false)" :disabled="!studentNameInput" :class="{'opacity-50': !studentNameInput, 'cursor-not-allowed': !studentNameInput}" class="focus:outline-none text-white bg-green-700 hover:bg-green-800 focus:ring-4 focus:ring-green-300 font-medium rounded-lg text-sm px-5 py-2.5 mr-2 mb-2 dark:bg-green-600 dark:hover:bg-green-700 dark:focus:ring-green-800">Ajouter</button>
        <button type="button" @click="addStudent(true)" :disabled="!studentNameInput" :class="{'opacity-50': !studentNameInput, 'cursor-not-allowed': !studentNameInput}" class="focus:outline-none text-white bg-green-700 hover:bg-green-800 focus:ring-4 focus:ring-green-300 font-medium rounded-lg text-sm px-5 py-2.5 mr-2 mb-2 dark:bg-green-600 dark:hover:bg-green-700 dark:focus:ring-green-800">Tiers Temps</button>
      </div>
    </div>
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

.logo{
  height: 100px;
}
</style>
