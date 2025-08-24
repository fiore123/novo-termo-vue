<script setup lang="ts">
const emit = defineEmits(['select']);

defineProps<{
  letter?: string;
  state?: 'empty' | 'filled' | 'correct' | 'present' | 'absent';
  isActive?: boolean;
  isRevealing?: boolean;
  isHint?: boolean;
}>();
</script>

<template>
  <div 
    class="tile" 
    :class="[state, { active: isActive, hint: isHint, revealing: isRevealing }]" 
    @click="emit('select')"
  >
    {{ letter }}
  </div>
</template>

<style scoped>
@keyframes pop-in {
  0% { transform: scale(0.8); }
  40% { transform: scale(1.1); }
  100% { transform: scale(1); }
}

@keyframes reveal {
  0% { transform: scale(1); }
  50% { transform: scale(1.2); }
  100% { transform: scale(1); }
}

.tile {
  /* Tamanho agora é controlado pelo GameBoard para consistência */
  width: 100%;
  height: 100%;
  aspect-ratio: 1 / 1; 

  display: flex;
  justify-content: center;
  align-items: center;
  
  font-size: clamp(1.5rem, 4vw, 2rem); /* Fonte responsiva */
  font-weight: 700;
  text-transform: uppercase;
  
  border: 2px solid var(--color-tile-border);
  border-radius: 8px;
  
  transition: background-color 0.3s ease, border-color 0.3s ease, color 0.3s ease;
  cursor: pointer;
}

.tile.filled {
  border-color: #878a8c;
  animation: pop-in 0.1s ease;
}

html.dark .tile.filled {
  border-color: #565758;
}

.tile.active {
  border-color: var(--color-text);
  box-shadow: 0 0 10px rgba(128, 128, 128, 0.5);
}

.tile.hint {
  border-color: var(--color-correct);
  color: var(--color-correct);
}

/* Animação e Cores de Revelação */
.tile.revealing {
  animation: reveal 0.6s ease;
}

.tile.correct {
  background-color: var(--color-correct);
  border-color: var(--color-correct);
  color: white;
}
.tile.present {
  background-color: var(--color-present);
  border-color: var(--color-present);
  color: white;
}
.tile.absent {
  background-color: var(--color-absent);
  border-color: var(--color-absent);
  color: white;
}
</style>