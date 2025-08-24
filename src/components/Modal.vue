<script setup lang="ts">
const emit = defineEmits(['close', 'action']);
</script>

<template>
  <div class="modal-overlay" @click.self="emit('close')">
    <div class="modal-content">
      <header class="modal-header">
        <h2 class="modal-title">
          <slot name="title"></slot>
        </h2>
        <button class="close-btn" @click="emit('close')">&times;</button>
      </header>
      <main class="modal-body">
        <slot name="body"></slot>
      </main>
      <footer class="modal-footer">
        <button class="action-btn" @click="emit('action')">
          <slot name="action-text">Novo Jogo</slot>
        </button>
      </footer>
    </div>
  </div>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}
.modal-content {
  background-color: var(--color-background);
  padding: 1.5rem;
  border-radius: 12px;
  width: 90%;
  max-width: 450px;
  text-align: center;
}
.modal-header {
  position: relative;
  margin-bottom: 1rem;
}
.modal-title {
  font-size: 1.8rem;
  font-weight: 700;
}
.close-btn {
  position: absolute;
  top: -10px;
  right: -5px;
  background: none;
  border: none;
  font-size: 2.5rem;
  color: var(--color-text);
  cursor: pointer;
}
.modal-body {
  margin-bottom: 1.5rem;
  font-size: 1.1rem;
}
.modal-footer .action-btn {
  width: 100%;
  padding: 12px;
  font-size: 1.2rem;
  font-weight: 600;
  border-radius: 8px;
  cursor: pointer;
  background-color: var(--color-correct);
  color: white;
  border: none;
}
</style>