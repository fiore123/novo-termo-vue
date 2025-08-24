<script setup lang="ts">
import { computed } from 'vue';

const props = defineProps<{
  gameMode: number;
  keyEvaluations: Record<string, string[]>;
}>();

const emit = defineEmits(['key-press']);

const KEYBOARD_LAYOUT = [
  ['q', 'w', 'e', 'r', 't', 'y', 'u', 'i', 'o', 'p'],
  ['a', 's', 'd', 'f', 'g', 'h', 'j', 'k', 'l'],
  ['enter', 'z', 'x', 'c', 'v', 'b', 'n', 'm', 'backspace'],
];

const handleKeyPress = (key: string) => {
  emit('key-press', key);
};

const getKeyEvaluations = (key: string) => {
  return props.keyEvaluations[key.toUpperCase()] || [];
};
</script>

<template>
  <div class="keyboard">
    <div v-for="(row, rowIndex) in KEYBOARD_LAYOUT" :key="rowIndex" class="key-row">
      <button
        v-for="key in row"
        :key="key"
        class="key"
        :class="{ large: key.length > 1 }"
        @click="handleKeyPress(key)"
      >
        <div v-if="key.length === 1" class="color-indicators" :class="`mode-${gameMode}`">
          <div
            v-for="(status, index) in getKeyEvaluations(key)"
            :key="index"
            class="indicator"
            :class="status"
          ></div>
        </div>
        <span class="key-label">{{ key }}</span>
        <svg v-if="key === 'backspace'" class="key-label" xmlns="http://www.w3.org/2000/svg" height="24" viewBox="0 0 24 24" width="24">
            <path fill="currentColor" d="M22 3H7c-.69 0-1.23.35-1.59.88L0 12l5.41 8.12c.36.53.9.88 1.59.88h15c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-3 12.59L17.59 17 14 13.41 10.41 17 9 15.59 12.59 12 9 8.41 10.41 7 14 10.59 17.59 7 19 8.41 15.41 12 19 15.59z"></path>
        </svg>
      </button>
    </div>
  </div>
</template>

<style scoped>
.keyboard {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
}
.key-row {
  display: flex;
  justify-content: center;
  gap: 0.4rem;
}
.key {
  position: relative;
  height: 48px; /* Altura fixa e menor */
  flex-grow: 1;
  font-weight: 600;
  background-color: var(--color-key-bg);
  color: var(--color-text);
  border: none;
  border-radius: 6px;
  cursor: pointer;
  text-transform: uppercase;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  font-size: 1rem;
  /* Fonte monoespaçada para alinhamento perfeito */
  font-family: 'Courier New', Courier, monospace;
}
.key.large {
  flex-grow: 1.5;
  font-size: 0.8rem;
}
.key-label {
  position: relative;
  z-index: 2;
  pointer-events: none;
}
.color-indicators {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  display: flex;
}
.indicator {
  flex: 1;
  height: 100%;
  background-color: var(--color-key-bg);
}
.indicator.correct { background-color: var(--color-correct); }
.indicator.present { background-color: var(--color-present); }
.indicator.absent { background-color: var(--color-absent); }

.color-indicators.mode-4 {
  display: grid;
  grid-template-columns: 1fr 1fr;
  grid-template-rows: 1fr 1fr;
}
.color-indicators.mode-4 .indicator {
  flex: none; 
}
</style>