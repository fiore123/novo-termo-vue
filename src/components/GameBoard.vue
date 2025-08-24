<script setup lang="ts">
import GameTile from './GameTile.vue';

const emit = defineEmits(['tile-select']);

interface BoardState {
  id: number;
  grid: { letter: string; state: any; isRevealing?: boolean, isHint?: boolean }[][];
}
interface ActivePosition {
  boardIndex: number;
  rowIndex: number;
  colIndex: number;
}
const props = defineProps<{
  boardState: BoardState;
  activePosition: ActivePosition;
}>();

const handleTileSelect = (rowIndex: number, colIndex: number) => {
  // Agora emite também o ID do board, que é o boardIndex
  emit('tile-select', props.boardState.id, rowIndex, colIndex);
};
</script>

<template>
  <div class="game-board">
    <div 
      v-for="(row, rowIndex) in boardState.grid" 
      :key="rowIndex" 
      class="board-row"
      :class="{
        'active-row': activePosition.boardIndex === boardState.id && activePosition.rowIndex === rowIndex,
        'submitted-row': rowIndex < activePosition.rowIndex
      }"
    >
      <GameTile
        v-for="(tile, colIndex) in row"
        :key="colIndex"
        :letter="tile.letter"
        :state="tile.state"
        :is-revealing="tile.isRevealing"
        :is-hint="tile.isHint"
        :is-active="
          activePosition.boardIndex === boardState.id &&
          activePosition.rowIndex === rowIndex &&
          activePosition.colIndex === colIndex
        "
        @select="handleTileSelect(rowIndex, colIndex)"
      />
    </div>
  </div>
</template>

<style scoped>
.game-board {
  display: grid;
  gap: 5px;
  width: 100%;
}
.board-row {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 5px;
}
.board-row:not(.active-row) > * {
  opacity: 0.7;
}
.submitted-row > * {
  opacity: 1;
}
</style>