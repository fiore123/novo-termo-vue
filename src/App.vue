<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed } from 'vue';
import GameBoard from './components/GameBoard.vue';
import Keyboard from './components/Keyboard.vue';
import Modal from './components/Modal.vue';
import StatsDisplay from './components/StatsDisplay.vue';

// --- ÁUDIO ---
const winSound = new Audio("data:audio/wav;base64,UklGRl9vT19XQVZFZm10IBAAAAABAAEAQB8AAEAfAAABAAgAZGF0YSIsanalysis:++++++yamnet_v1_music_embedding,embedding_score:0.5524++++");
const loseSound = new Audio("data:audio/wav;base64,UklGRl9vT19XQVZFZm10IBAAAAABAAEAQB8AAEAfAAABAAgAZGF0YSIsanalysis:++++++yamnet_v1_music_embedding,embedding_score:0.5524++++");

const wordList = ref<string[]>([]);
const deaccentedWordList = ref<string[]>([]); // Lista auxiliar para validação rápida

const gameState = ref<'playing' | 'won' | 'lost'>('playing');
const gameMode = ref(1);
const boards = ref<any[]>([]);
const secretWords = ref<string[]>([]);
const isRevealing = ref(false);
const isLoading = ref(true);

const stats = ref({ played: 0, wins: 0, currentStreak: 0, maxStreak: 0, distribution: Array(9).fill(0) });
const chances = computed(() => gameMode.value + 5);

const activePosition = ref({ boardIndex: 0, rowIndex: 0, colIndex: 0 });

const toast = ref({ show: false, message: '' });
const showEndGameModal = ref(false);
const showStatsModal = ref(false);
const showHintConfirmModal = ref(false);
const showHowToPlayModal = ref(false);
const endGameMessage = ref({ title: '', body: '' });

const isDarkMode = ref(false);
const isMuted = ref(false);

// --- FUNÇÃO HELPER PARA REMOVER ACENTOS ---
const deaccent = (text: string) => {
  return text.normalize('NFD').replace(/[\u0300-\u036f]/g, '');
};

const keyEvaluations = computed(() => {
    const evaluations: Record<string, string[]> = {};
    const submittedRows = activePosition.value.rowIndex;
    if (submittedRows === 0) return {};

    for (let charCode = 65; charCode <= 90; charCode++) {
        const letter = String.fromCharCode(charCode);
        const letterEvals: string[] = Array(gameMode.value).fill('empty');

        boards.value.forEach((board, boardIndex) => {
            let boardStatus = 'empty';
            for (let r = 0; r < submittedRows; r++) {
                for (let c = 0; c < 5; c++) {
                    if (deaccent(board.grid[r][c].letter) === letter) { // Comparação sem acento
                        const status = board.grid[r][c].state;
                        if (status === 'correct') {
                            boardStatus = 'correct';
                            break;
                        } else if (status === 'present') {
                            boardStatus = 'present';
                        } else if (boardStatus !== 'present' && status !== 'empty') {
                            boardStatus = 'absent';
                        }
                    }
                }
                if (boardStatus === 'correct') break;
            }
            letterEvals[boardIndex] = boardStatus;
        });
        evaluations[letter] = letterEvals;
    }
    return evaluations;
});

const playSound = (sound: HTMLAudioElement) => {
  if (!isMuted.value) {
    sound.currentTime = 0;
    sound.play().catch(e => console.error("Erro ao tocar o som:", e));
  }
};

const loadSettings = () => {
    const savedTheme = localStorage.getItem('termo-theme');
    if (savedTheme === 'dark' || (!savedTheme && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
        isDarkMode.value = true;
        document.documentElement.classList.add('dark');
    }
    const savedMute = localStorage.getItem('termo-muted');
    if (savedMute) {
        isMuted.value = JSON.parse(savedMute);
    }
};

const toggleMute = () => {
    isMuted.value = !isMuted.value;
    localStorage.setItem('termo-muted', JSON.stringify(isMuted.value));
};

const loadStats = () => {
  const savedStats = localStorage.getItem('termo-stats');
  if (savedStats) {
    const parsedStats = JSON.parse(savedStats);
    if (!parsedStats.distribution || parsedStats.distribution.length !== 9) {
      parsedStats.distribution = Array(9).fill(0);
    }
    stats.value = parsedStats;
  }
};

const saveStats = () => {
  localStorage.setItem('termo-stats', JSON.stringify(stats.value));
};

const updateStats = (didWin: boolean, winRowIndex: number | null) => {
  stats.value.played++;
  if (didWin && winRowIndex !== null) {
    stats.value.wins++;
    stats.value.currentStreak++;
    if (stats.value.currentStreak > stats.value.maxStreak) {
      stats.value.maxStreak = stats.value.currentStreak;
    }
    if (winRowIndex < stats.value.distribution.length) {
      stats.value.distribution[winRowIndex]++;
    }
  } else {
    stats.value.currentStreak = 0;
  }
  saveStats();
};

const startGame = (mode: number) => {
  if (wordList.value.length === 0) {
      showToast("Banco de palavras ainda carregando.");
      isLoading.value = true;
      return;
  }
  gameMode.value = mode;
  gameState.value = 'playing';
  secretWords.value = [];
  const usedWords = new Set<string>();
  for (let i = 0; i < mode; i++) {
    let word;
    do {
      word = wordList.value[Math.floor(Math.random() * wordList.value.length)];
    } while (usedWords.has(word));
    usedWords.add(word);
    secretWords.value.push(word);
  }

  const emptyGrid = Array.from({ length: chances.value }, () =>
    Array.from({ length: 5 }, () => ({ letter: '', state: 'empty', isRevealing: false, isHint: false }))
  );
  
  boards.value = Array.from({ length: mode }, (_, i) => ({
    id: i,
    isSolved: false,
    grid: JSON.parse(JSON.stringify(emptyGrid)),
  }));

  activePosition.value = { boardIndex: 0, rowIndex: 0, colIndex: 0 };
  isLoading.value = false;
  showEndGameModal.value = false;
};

const showToast = (message: string) => {
  toast.value = { show: true, message };
  setTimeout(() => { toast.value.show = false; }, 2000);
};

// ATUALIZADO: Lógica de avaliação agora ignora acentos
const evaluateGuess = (guess: string, secret: string) => {
  const result = Array(5).fill('absent');
  const deaccentedSecret = deaccent(secret);
  const secretLetters = deaccentedSecret.split('');
  const guessLetters = deaccent(guess).split('');
  
  guessLetters.forEach((letter, i) => {
    if (secretLetters[i] === letter) {
      result[i] = 'correct';
      secretLetters[i] = ''; 
    }
  });

  guessLetters.forEach((letter, i) => {
    if (result[i] !== 'correct' && secretLetters.includes(letter)) {
      result[i] = 'present';
      secretLetters[secretLetters.indexOf(letter)] = '';
    }
  });
  return result;
};

const checkGameStatus = () => {
    const allSolved = boards.value.every(b => b.isSolved);
    if (allSolved) {
        gameState.value = 'won';
        endGameMessage.value = { title: 'Parabéns!', body: 'Você acertou todas as palavras!' };
        showEndGameModal.value = true;
        updateStats(true, activePosition.value.rowIndex);
        playSound(winSound);
        return;
    }

    if (activePosition.value.rowIndex + 1 >= chances.value) {
        gameState.value = 'lost';
        const wordsText = secretWords.value.join(', ');
        endGameMessage.value = { title: 'Você Perdeu!', body: `As palavras eram: ${wordsText}` };
        showEndGameModal.value = true;
        updateStats(false, null);
        playSound(loseSound);
    }
};

// ATUALIZADO: Lógica de submissão agora valida sem acento e auto-corrige no tabuleiro
const submitGuess = () => {
  if (isRevealing.value || gameState.value !== 'playing') return;
  const { rowIndex } = activePosition.value;
  
  const boardForGuess = boards.value.find(b => !b.isSolved);
  if (!boardForGuess) return;
  const guess = boardForGuess.grid[rowIndex].map((tile: any) => tile.letter).join('');

  if (guess.length !== 5) {
    showToast("A palavra deve ter 5 letras");
    return;
  }
  
  const deaccentedGuess = deaccent(guess);
  const wordIndex = deaccentedWordList.value.indexOf(deaccentedGuess);

  if (wordIndex === -1) {
    showToast("Palavra não encontrada");
    return;
  }

  const canonicalGuess = wordList.value[wordIndex]; // Pega a palavra com acento do banco original

  // Auto-corrige a palavra no tabuleiro antes de animar
  boards.value.forEach(board => {
    if (!board.isSolved) {
      canonicalGuess.split('').forEach((letter, i) => {
        board.grid[rowIndex][i].letter = letter;
      });
    }
  });

  isRevealing.value = true;
  boards.value.forEach((board, boardIndex) => {
    if (board.isSolved) return;
    const evaluation = evaluateGuess(canonicalGuess, secretWords.value[boardIndex]);
    const row = board.grid[rowIndex];
    row.forEach((tile: any, colIndex: number) => {
      setTimeout(() => {
        tile.state = evaluation[colIndex];
        tile.isRevealing = true;
      }, colIndex * 150);
    });
    if(evaluation.every(state => state === 'correct')) {
        board.isSolved = true;
    }
  });

  setTimeout(() => {
    isRevealing.value = false;
    boards.value.forEach(b => {
      if(b.grid[rowIndex]) {
        b.grid[rowIndex].forEach((tile: any) => tile.isRevealing = false);
      }
    });
    checkGameStatus();
    if(gameState.value === 'playing') {
      activePosition.value.rowIndex++;
      activePosition.value.colIndex = 0;
      const firstUnsolved = boards.value.find(b => !b.isSolved);
      if (firstUnsolved) {
        activePosition.value.boardIndex = firstUnsolved.id;
      }
    }
  }, 5 * 150 + 300);
};

const handleTileSelect = (boardIndex: number, rowIndex: number, colIndex: number) => {
  if (gameState.value !== 'playing') return;
  if(rowIndex === activePosition.value.rowIndex) {
    activePosition.value.boardIndex = boardIndex;
    activePosition.value.colIndex = colIndex;
  }
};

const handleKeyPress = (key: string) => {
  if (isRevealing.value || gameState.value !== 'playing') return;
  const { rowIndex, colIndex } = activePosition.value;

  if (key === 'enter') { submitGuess(); return; }

  if (key === 'backspace') {
    const targetCol = colIndex === 5 ? 4 : colIndex - 1;
    if (targetCol >= 0) {
      boards.value.forEach(board => {
        if (board.isSolved) return;
        const tile = board.grid[rowIndex][targetCol];
        tile.letter = '';
        tile.state = 'empty';
        tile.isHint = false;
      });
      activePosition.value.colIndex = targetCol;
    }
    return;
  }

  if (colIndex < 5 && /^[a-zA-Z]$/.test(key)) {
    boards.value.forEach(board => {
      if (board.isSolved) return;
      const tile = board.grid[rowIndex][colIndex];
      tile.letter = key.toUpperCase();
      tile.state = 'filled';
    });
    if (activePosition.value.colIndex < 5) {
      activePosition.value.colIndex++;
    }
  }
};

const handleHint = () => {
    if (isRevealing.value || gameState.value !== 'playing') return;
    const { boardIndex, rowIndex, colIndex } = activePosition.value;

    if (colIndex >= 5) {
      showToast("Não há espaço para a dica.");
      return;
    }
    
    const targetBoard = boards.value[boardIndex];
    if (!targetBoard || targetBoard.isSolved) {
        showToast("Selecione um termo não resolvido.");
        return;
    };

    const secretWord = secretWords.value[targetBoard.id];
    const letterForHint = secretWord[colIndex];

    boards.value.forEach(board => {
      if (board.isSolved) return;
      const tile = board.grid[rowIndex][colIndex];
      tile.letter = letterForHint;
      tile.state = 'filled';
      tile.isHint = true;
    });

    if (activePosition.value.colIndex < 5) {
      activePosition.value.colIndex++;
    }
};

const confirmUseHint = () => {
    handleHint();
    showHintConfirmModal.value = false;
};

const handlePhysicalKeyboard = (e: KeyboardEvent) => {
  if (showEndGameModal.value || showStatsModal.value || showHintConfirmModal.value || showHowToPlayModal.value) return;

  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(e.key)) {
    e.preventDefault();
    const { rowIndex, colIndex } = activePosition.value;
    if (e.key === 'ArrowLeft' && colIndex > 0) activePosition.value.colIndex--;
    if (e.key === 'ArrowRight' && colIndex < 5) activePosition.value.colIndex++;
    return;
  }
  handleKeyPress(e.key.toLowerCase());
};

const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value;
  document.documentElement.classList.toggle('dark', isDarkMode.value);
  localStorage.setItem('termo-theme', isDarkMode.value ? 'dark' : 'light');
};

async function loadWordList() {
    try {
        const response = await fetch('/palavras.json');
        if (!response.ok) {
            throw new Error(`Erro na rede! Status: ${response.status}`);
        }
        let data = await response.json();

        let wordsToProcess = [];
        if (Array.isArray(data)) {
            wordsToProcess = data;
        } else if (data && Array.isArray(data.palavras)) {
            wordsToProcess = data.palavras;
        } else {
            throw new Error("Formato do JSON de palavras é inválido.");
        }

        const cleanedWords = wordsToProcess
            .map(word => String(word).toUpperCase().trim())
            .filter(word => word.length === 5 && /^[A-ZÇÃÁÀÂÄÉÈÊËÍÌÎÏÓÒÔÕÖÚÙÛÜ]+$/.test(deaccent(word))); // Permite acentos e Ç
        
        if (cleanedWords.length === 0) {
            throw new Error("Nenhuma palavra válida de 5 letras encontrada no banco de palavras.");
        }
        
        wordList.value = cleanedWords;
        deaccentedWordList.value = cleanedWords.map(word => deaccent(word));

    } catch (error) {
        console.error("Erro detalhado ao carregar palavras:", error);
        showToast("Erro fatal: Banco de palavras inválido.");
        isLoading.value = false;
    }
}

onMounted(async () => {
  loadStats();
  loadSettings();
  await loadWordList();
  startGame(gameMode.value);
  window.addEventListener('keydown', handlePhysicalKeyboard);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handlePhysicalKeyboard);
});
</script>

<template>
  <div id="app-container">
    <div v-if="toast.show" class="toast">{{ toast.message }}</div>
    
    <Modal v-if="showEndGameModal" @close="showEndGameModal = false" @action="startGame(gameMode)">
      <template #title>{{ endGameMessage.title }}</template>
      <template #body>{{ endGameMessage.body }}</template>
    </Modal>

    <Modal v-if="showStatsModal" @close="showStatsModal = false" @action="showStatsModal = false">
        <template #title>Estatísticas</template>
        <template #body><StatsDisplay :stats="stats" /></template>
        <template #action-text>Fechar</template>
    </Modal>
    
    <Modal v-if="showHintConfirmModal" @close="showHintConfirmModal = false" @action="confirmUseHint">
        <template #title>Usar Dica?</template>
        <template #body>Isso revelará uma letra correta. Deseja continuar?</template>
        <template #action-text>Sim, usar dica</template>
    </Modal>

    <Modal v-if="showHowToPlayModal" @close="showHowToPlayModal = false" @action="showHowToPlayModal = false">
        <template #title>Como Jogar</template>
        <template #body>
            <div class="how-to-play-content">
                <p>Adivinhe a(s) palavra(s) secreta(s) em um número limitado de tentativas.</p>
                <p>As cores das peças indicam o quão perto você está da resposta.</p>
                <hr>
                <div class="example">
                    <div class="example-row">
                        <div class="example-tile correct">V</div><div class="example-tile">E</div><div class="example-tile">R</div><div class="example-tile">D</div><div class="example-tile">E</div>
                    </div>
                    <p>A letra <strong>V</strong> está na palavra e na <strong>posição correta</strong>.</p>
                </div>
                <div class="example">
                    <div class="example-row">
                        <div class="example-tile">V</div><div class="example-tile present">O</div><div class="example-tile">G</div><div class="example-tile">A</div><div class="example-tile">L</div>
                    </div>
                    <p>A letra <strong>O</strong> está na palavra, mas na <strong>posição errada</strong>.</p>
                </div>
                <div class="example">
                    <div class="example-row">
                        <div class="example-tile">P</div><div class="example-tile">U</div><div class="example-tile">L</div><div class="example-tile">A</div><div class="example-tile absent">R</div>
                    </div>
                    <p>A letra <strong>R</strong> <strong>não faz parte</strong> da palavra.</p>
                </div>
                <hr>
                <p>Jogue com 1, 2, 3 ou 4 tabuleiros simultaneamente. Cada modo de jogo adiciona uma tentativa extra!</p>
            </div>
        </template>
        <template #action-text>Entendi!</template>
    </Modal>

    <header class="app-header">
      <div class="header-content">
        <div class="header-left">
          <button class="icon-btn" @click="showHowToPlayModal = true">
            <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"><path d="M0 0h24v24H0V0z" fill="none"/><path d="M11 18h2v-2h-2v2zm1-16C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8zm0-14c-2.21 0-4 1.79-4 4h2c0-1.1.9-2 2-2s2 .9 2 2c0 2-3 1.75-3 5h2c0-2.25 3-2.5 3-5 0-2.21-1.79-4-4-4z"/></svg>
          </button>
          <button class="icon-btn" @click="showHintConfirmModal = true">
            <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"><path d="M0 0h24v24H0V0z" fill="none"/><path d="M9 21c0 .55.45 1 1 1h4c.55 0 1-.45 1-1v-1H9v1zm3-19C8.14 2 5 5.14 5 9c0 2.38 1.19 4.47 3 5.74V17c0 .55.45 1 1 1h4c.55 0 1-.45 1-1v-2.26c1.81-1.27 3-3.36 3-5.74 0-3.86-3.14-7-7-7zm2.85 11.1l-.85.6V16h-4v-2.3l-.85-.6C7.8 12.16 7 10.63 7 9c0-2.76 2.24-5 5-5s5 2.24 5 5c0 1.63-.8 3.16-2.15 4.1z"/></svg>
          </button>
        </div>
        
        <div class="header-center">
          <h1 class="app-title">TERMO</h1>
          <div class="difficulty-selector">
            <button @click="startGame(1)" :class="{ active: gameMode === 1 }">1</button>
            <button @click="startGame(2)" :class="{ active: gameMode === 2 }">2</button>
            <button @click="startGame(3)" :class="{ active: gameMode === 3 }">3</button>
            <button @click="startGame(4)" :class="{ active: gameMode === 4 }">4</button>
          </div>
        </div>

        <div class="header-right">
          <button class="icon-btn" @click="showStatsModal = true">
            <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"><path d="M0 0h24v24H0V0z" fill="none"/><path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zM9 17H7v-7h2v7zm4 0h-2V7h2v10zm4 0h-2v-4h2v4z"/></svg>
          </button>
          <button class="icon-btn" @click="toggleTheme">
            <svg v-if="isDarkMode" xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"><path d="M0 0h24v24H0V0z" fill="none"/><path d="M12 3c-4.97 0-9 4.03-9 9s4.03 9 9 9 9-4.03 9-9c0-.46-.04-.92-.11-1.36-.98 1.37-2.58 2.26-4.39 2.26-2.98 0-5.4-2.42-5.4-5.4 0-1.81.89-3.42 2.26-4.39C12.92 3.04 12.46 3 12 3z"/></svg>
            <svg v-else xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"><path d="M0 0h24v24H0V0z" fill="none"/><path d="M20 8.69V4h-4.69L12 .69 8.69 4H4v4.69L.69 12 4 15.31V20h4.69L12 23.31 15.31 20H20v-4.69L23.31 12 20 8.69zM12 18c-3.31 0-6-2.69-6-6s2.69-6 6-6 6 2.69 6 6-2.69 6-6 6zm0-10c-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4-1.79-4-4-4z"/></svg>
          </button>
          <button class="icon-btn" @click="toggleMute">
            <svg v-if="isMuted" xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"><path d="M0 0h24v24H0V0z" fill="none"/><path d="M16.5 12c0-1.77-1.02-3.29-2.5-4.03v2.21l2.45 2.45c.03-.2.05-.41.05-.63zm2.5 0c0 .94-.2 1.82-.54 2.64l1.51 1.51C20.63 14.91 21 13.5 21 12c0-4.28-2.99-7.86-7-8.77v2.06c2.89.86 5 3.54 5 6.71zM4.27 3L3 4.27 7.73 9H3v6h4l5 5v-6.73l4.25 4.25c-.67.52-1.42.93-2.25 1.18v2.06c1.38-.31 2.63-.95 3.69-1.81L19.73 21 21 19.73l-9-9L4.27 3zM12 4L9.91 6.09 12 8.18V4z"/></svg>
            <svg v-else xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 0 24 24" width="24px"><path d="M0 0h24v24H0V0z" fill="none"/><path d="M3 9v6h4l5 5V4L7 9H3zm13.5 3c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77z"/></svg>
          </button>
        </div>
      </div>
    </header>

    <main class="game-area">
      <div v-if="isLoading" class="loading-text">Carregando banco de palavras...</div>
      <div v-else class="boards-container" :class="`mode-${gameMode}`">
        <div v-for="board in boards" :key="board.id" class="game-board-wrapper" :class="{ solved: board.isSolved }">
          <GameBoard
            :boardState="board"
            :activePosition="activePosition"
            @tile-select="(boardId, rowIndex, colIndex) => handleTileSelect(boardId, rowIndex, colIndex)"
          />
        </div>
      </div>
    </main>

    <footer class="keyboard-area">
      <Keyboard 
        v-if="!isLoading"
        :key-evaluations="keyEvaluations" 
        :game-mode="gameMode" 
        @key-press="handleKeyPress" 
      />
    </footer>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;600;700&display=swap');
:root {
  --color-background: #F8F9FA;
  --color-text: #212529;
  --color-border: #D3D6DA;
  --color-key-bg: #EAECEE;
  --color-key-bg-active: #D3D6DA;
  --color-tile-border: #B2BABB;
  --color-correct: #6AAA64;
  --color-present: #C9B458;
  --color-absent: #787C7E;
  --icon-color: #888;
}
html.dark {
  --color-background: #121213;
  --color-text: #F8F9FA;
  --color-border: #3A3A3C;
  --color-key-bg: #3A3A3C;
  --color-key-bg-active: #565758;
  --color-tile-border: #565758;
  --color-correct: #538D4E;
  --color-present: #B59F3B;
  --color-absent: #3A3A3C;
  --icon-color: #bbb;
}
* { box-sizing: border-box; margin: 0; padding: 0; font-family: 'Poppins', sans-serif; -webkit-tap-highlight-color: transparent; }
body { background-color: var(--color-background); color: var(--color-text); transition: background-color 0.3s ease, color 0.3s ease; }

#app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  max-height: 100vh;
  margin: 0 auto;
}

.app-header {
  flex-shrink: 0;
  padding: 0.5rem 1rem;
  border-bottom: 1px solid var(--color-border);
  width: 100%;
}
.header-content {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  width: 1200px;
  max-width: 1200px;
  margin: 0 auto;
}
.header-left, .header-right { display: flex; gap: 0.5rem; }
.header-left { justify-content: flex-start; }
.header-right { justify-content: flex-end; }
.header-center { display: flex; flex-direction: column; align-items: center; }
.app-title { font-size: 1.5rem; font-weight: 700; letter-spacing: 0.05em; padding-bottom: 0.25rem; }
.icon-btn { background: none; border: none; font-size: 1.25rem; cursor: pointer; color: var(--icon-color); padding: 4px; border-radius: 50%; display: flex; align-items: center; justify-content: center; }
.icon-btn svg { width: 24px; height: 24px; fill: currentColor; }
.icon-btn:hover { background-color: var(--color-key-bg); }

.difficulty-selector { display: flex; align-items: center; gap: 0.5rem; }
.difficulty-selector button {
  width: 28px;
  height: 28px;
  border: 2px solid var(--color-border);
  background-color: transparent;
  color: var(--color-text);
  font-size: 0.9rem;
  font-weight: 600;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.2s, border-color 0.2s;
}
.difficulty-selector button.active { background-color: var(--color-key-bg-active); border-color: var(--color-key-bg-active); }

.game-area {
  flex-grow: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
  overflow: hidden;
}
.loading-text {
  font-size: 1.5rem;
  font-weight: 600;
  color: var(--icon-color);
}
.boards-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  gap: clamp(10px, 5.5vh, 50px);
}
.game-board-wrapper {
  width: var(--board-width);
  transition: opacity 0.3s;
}
.boards-container.mode-1 { --board-width: clamp(300px, 45vh, 380px); }
.boards-container.mode-2 { --board-width: clamp(260px, 38vh, 340px); }
.boards-container.mode-3 { --board-width: clamp(260px, 38vh, 340px); }
.boards-container.mode-4 { --board-width: clamp(260px, 38vh, 340px); }

.game-board-wrapper.solved {
  opacity: 0.6;
  pointer-events: none;
}

.keyboard-area {
  flex-shrink: 0;
  padding: 0.5rem 0.25rem;
}

.toast { position: fixed; top: 20px; left: 50%; transform: translateX(-50%); background-color: var(--color-text); color: var(--color-background); padding: 10px 20px; border-radius: 8px; z-index: 1000; font-weight: 600; animation: fade-in-out 2s ease-in-out; }
@keyframes fade-in-out { 0%, 100% { opacity: 0; transform: translate(-50%, -20px); } 10%, 90% { opacity: 1; transform: translate(-50%, 0); } }
.how-to-play-content { text-align: left; }
.how-to-play-content p { margin-bottom: 1rem; font-size: 1rem; line-height: 1.5; }
.how-to-play-content hr { border: none; border-top: 1px solid var(--color-border); margin: 1.5rem 0; }
.example { margin-bottom: 1.5rem; }
.example-row { display: flex; gap: 5px; margin-bottom: 0.5rem; }
.example-tile { width: 40px; height: 40px; display: flex; justify-content: center; align-items: center; font-size: 1.5rem; font-weight: 700; border: 2px solid var(--color-border); border-radius: 6px; }
.example-tile.correct { background-color: var(--color-correct); border-color: var(--color-correct); color: white; }
.example-tile.present { background-color: var(--color-present); border-color: var(--color-present); color: white; }
.example-tile.absent { background-color: var(--color-absent); border-color: var(--color-absent); color: white; }
</style>