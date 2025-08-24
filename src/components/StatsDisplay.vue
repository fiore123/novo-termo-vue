<script setup lang="ts">
interface Stats {
  played: number;
  wins: number;
  currentStreak: number;
  maxStreak: number;
  distribution: number[];
}
const props = defineProps<{ stats: Stats }>();

const winPercent = (stats: Stats) => {
  return stats.played > 0 ? Math.round((stats.wins / stats.played) * 100) : 0;
};

const maxDistributionValue = Math.max(...props.stats.distribution, 1);
</script>

<template>
  <div class="stats-content">
    <div class="stats-summary">
      <div class="stat-item">
        <div class="stat-value">{{ stats.played }}</div>
        <div class="stat-label">Jogos</div>
      </div>
      <div class="stat-item">
        <div class="stat-value">{{ winPercent(stats) }}%</div>
        <div class="stat-label">Vitórias</div>
      </div>
      <div class="stat-item">
        <div class="stat-value">{{ stats.currentStreak }}</div>
        <div class="stat-label">Sequência Atual</div>
      </div>
      <div class="stat-item">
        <div class="stat-value">{{ stats.maxStreak }}</div>
        <div class="stat-label">Melhor Sequência</div>
      </div>
    </div>
    <h3 class="distribution-title">Distribuição de Tentativas</h3>
    <div class="distribution-chart">
      <div v-for="(count, index) in stats.distribution" :key="index" class="chart-row">
        <div class="chart-label">{{ index + 1 }}</div>
        <div class="chart-bar-container">
          <div 
            class="chart-bar" 
            :style="{ width: (count / maxDistributionValue) * 100 + '%' }"
          >
            {{ count }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.stats-summary {
  display: flex;
  justify-content: space-around;
  text-align: center;
  margin-bottom: 1.5rem;
}
.stat-item {
  min-width: 80px;
}
.stat-value {
  font-size: 2.2rem;
  font-weight: 700;
}
.stat-label {
  font-size: 0.9rem;
  color: #787c7e;
}
.distribution-title {
  text-align: center;
  margin-bottom: 1rem;
  font-weight: 600;
}
.distribution-chart {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.chart-row {
  display: flex;
  align-items: center;
  gap: 8px;
}
.chart-label {
  font-weight: 600;
  width: 20px;
  text-align: center;
}
.chart-bar-container {
  flex-grow: 1;
  background-color: var(--color-key-bg);
  border-radius: 4px;
}
.chart-bar {
  background-color: var(--color-absent);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  min-width: 24px;
  text-align: right;
  font-weight: 600;
}
</style>