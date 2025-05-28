<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';

  // Game constants
  const COLS = 10;
  const ROWS = 20;
  const BLOCK_SIZE = 30;
  const EMPTY = 'white';
  const LOCAL_STORAGE_KEY = 'bricks-game-high-score';

  // Game state
  let board = createEmptyBoard();
  let score = 0;
  let highScore = 0;
  let level = 1;
  let gameInterval: ReturnType<typeof setInterval>;
  let currentPiece: any = null;
  let nextPiece: any = null;
  let holdPiece: any = null;
  let canHold = true;
  let gameOver = false;
  let isPaused = false;
  
  // Particles for effects
  type Particle = {
    x: number;
    y: number;
    vx: number;
    vy: number;
    color: string;
    size: number;
    life: number;
    maxLife: number;
  };
  
  let particles: Particle[] = [];
  const SPARKLE_COLORS = ['#ffffff', '#ffd6e6', '#ff70b5', '#ff4a9f', '#ffa6c7']; // White and pink sparkles to match the theme

  // Tetromino pieces with their shapes and colors
  const PIECES = [
    { shape: [[1, 1, 1, 1]], color: '#7ecece' },    // I - Light Blue (more saturated)
    { shape: [[1, 1], [1, 1]], color: '#ffb6c1' },  // O - Light Pink (more saturated)
    { shape: [[0, 1, 0], [1, 1, 1]], color: '#c39bd3' }, // T - Lavender (more saturated)
    { shape: [[0, 1, 1], [1, 1, 0]], color: '#9ae7af' },  // S - Mint Green (more saturated)
    { shape: [[1, 1, 0], [0, 1, 1]], color: '#ffafd7' },    // Z - Pink (more saturated)
    { shape: [[1, 0, 0], [1, 1, 1]], color: '#86e3e0' },   // J - Light Cyan (more saturated)
    { shape: [[0, 0, 1], [1, 1, 1]], color: '#ffd280' }  // L - Peach (more saturated)
  ];

  // Create an empty game board
  function createEmptyBoard() {
    return Array.from({ length: ROWS }, () => Array(COLS).fill(EMPTY));
  }

  // Generate a random piece
  function getRandomPiece() {
    const pieceIndex = Math.floor(Math.random() * PIECES.length);
    const piece = JSON.parse(JSON.stringify(PIECES[pieceIndex]));
    
    // Set initial position (centered at top)
    piece.x = Math.floor((COLS - piece.shape[0].length) / 2);
    piece.y = 0;
    
    return piece;
  }

  // Check if the piece can move to the specified position
  function isValidMove(piece: any, offsetX: number, offsetY: number, newShape: any[][] | null = null) {
    const shape = newShape || piece.shape;
    
    for (let y = 0; y < shape.length; y++) {
      for (let x = 0; x < shape[y].length; x++) {
        if (shape[y][x]) {
          const newX = piece.x + x + offsetX;
          const newY = piece.y + y + offsetY;
          
          if (
            newX < 0 ||
            newX >= COLS ||
            newY >= ROWS ||
            (newY >= 0 && board[newY][newX] !== EMPTY)
          ) {
            return false;
          }
        }
      }
    }
    
    return true;
  }

  // Lock the piece in place
  function lockPiece() {
    for (let y = 0; y < currentPiece.shape.length; y++) {
      for (let x = 0; x < currentPiece.shape[y].length; x++) {
        if (currentPiece.shape[y][x]) {
          const boardY = currentPiece.y + y;
          if (boardY < 0) {
            endGame();
            return;
          }
          board[boardY][currentPiece.x + x] = currentPiece.color;
        }
      }
    }
    
    clearLines();
    canHold = true;
    currentPiece = nextPiece;
    nextPiece = getRandomPiece();
    
    // Check if the new piece can be placed on the board
    if (!isValidMove(currentPiece, 0, 0)) {
      endGame();
    }
  }
  
  // End the game and update high score
  function endGame() {
    gameOver = true;
    clearInterval(gameInterval);
    
    // Update high score if needed
    if (score > highScore) {
      highScore = score;
      if (browser) {
        localStorage.setItem(LOCAL_STORAGE_KEY, highScore.toString());
      }
    }
  }
  
  // Create sparkle effect
  function createSparkles(row: number) {
    const particleCount = Math.floor(Math.random() * 10) + 20; // 20-30 particles
    
    for (let x = 0; x < COLS; x++) {
      // Create multiple particles for each cell in the cleared row
      for (let i = 0; i < particleCount / COLS; i++) {
        const particle: Particle = {
          x: (x + 0.5) * BLOCK_SIZE,
          y: (row + 0.5) * BLOCK_SIZE,
          vx: (Math.random() - 0.5) * 3,
          vy: (Math.random() - 0.5) * 3 - 1, // Slight upward bias
          color: SPARKLE_COLORS[Math.floor(Math.random() * SPARKLE_COLORS.length)],
          size: Math.random() * 3 + 1,
          life: 1,
          maxLife: Math.random() * 0.5 + 0.5 // 0.5 to 1 second
        };
        particles.push(particle);
      }
    }
  }
  
  // Update particle positions and lifetimes
  function updateParticles(deltaTime: number) {
    particles = particles.filter(p => {
      // Update position
      p.x += p.vx;
      p.y += p.vy;
      
      // Decrease life
      p.life -= deltaTime;
      
      // Return false to remove dead particles
      return p.life > 0;
    });
  }
  
  // Draw particles
  function drawParticles(ctx: CanvasRenderingContext2D) {
    for (const p of particles) {
      ctx.globalAlpha = p.life / p.maxLife; // Fade out as life decreases
      ctx.fillStyle = p.color;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.globalAlpha = 1;
  }

  // Clear completed lines
  function clearLines() {
    let linesCleared = 0;
    
    for (let y = ROWS - 1; y >= 0; y--) {
      if (board[y].every(cell => cell !== EMPTY)) {
        // Create sparkle effect before removing the line
        createSparkles(y);
        
        board.splice(y, 1);
        board.unshift(Array(COLS).fill(EMPTY));
        linesCleared++;
        y++; // Check the same row again after shifting
      }
    }
    
    if (linesCleared > 0) {
      // Calculate score based on number of lines cleared
      const points = [0, 100, 300, 500, 800][linesCleared] * level;
      score += points;
      
      // Level up every 10 lines
      level = Math.floor(score / 1000) + 1;
      updateGameSpeed();
    }
  }

  // Move the current piece
  function movePiece(dx: number, dy: number) {
    if (!currentPiece || gameOver || isPaused) return;
    
    if (isValidMove(currentPiece, dx, dy)) {
      currentPiece.x += dx;
      currentPiece.y += dy;
    } else if (dy > 0) {
      // If can't move down, lock the piece
      lockPiece();
    }
  }

  // Rotate the current piece
  function rotatePiece() {
    if (!currentPiece || gameOver || isPaused) return;
    
    // Skip rotation for O piece
    if (currentPiece.shape.length === 2 && currentPiece.shape[0].length === 2) {
      return;
    }
    
    const newShape: number[][] = [];
    for (let x = 0; x < currentPiece.shape[0].length; x++) {
      const newRow: number[] = [];
      for (let y = currentPiece.shape.length - 1; y >= 0; y--) {
        newRow.push(currentPiece.shape[y][x]);
      }
      newShape.push(newRow);
    }
    
    // Try wall kicks in this order: normal position, right shift, left shift, up shift
    const kicks = [
      [0, 0],   // Normal position
      [1, 0],   // Right shift
      [-1, 0],  // Left shift
      [0, -1],  // Up shift
      [1, -1],  // Right + Up shift
      [-1, -1], // Left + Up shift
    ];

    // Special handling for I piece
    if (currentPiece.shape.length === 1 || currentPiece.shape[0].length === 1) {
      // Center the I piece rotation around its middle
      const centerOffset = Math.floor((currentPiece.shape.length === 1 ? 
                                     currentPiece.shape[0].length : 
                                     currentPiece.shape.length) / 2);
      currentPiece.x += centerOffset;
      if (!isValidMove(currentPiece, 0, 0, newShape)) {
        currentPiece.x -= centerOffset;
      }
    }

    // Try each wall kick position
    for (const [kickX, kickY] of kicks) {
      if (isValidMove(currentPiece, kickX, kickY, newShape)) {
        currentPiece.shape = newShape;
        currentPiece.x += kickX;
        currentPiece.y += kickY;
        return;
      }
    }
  }

  // Hard drop the piece
  function hardDrop() {
    if (!currentPiece || gameOver || isPaused) return;
    
    let dropDistance = 0;
    while (isValidMove(currentPiece, 0, dropDistance + 1)) {
      dropDistance++;
    }
    
    currentPiece.y += dropDistance;
    lockPiece();
  }

  // Hold the current piece
  function holdCurrentPiece() {
    if (!canHold || !currentPiece || gameOver || isPaused) return;
    
    if (holdPiece === null) {
      holdPiece = {
        shape: currentPiece.shape,
        color: currentPiece.color
      };
      currentPiece = nextPiece;
      nextPiece = getRandomPiece();
    } else {
      const temp = {
        shape: currentPiece.shape,
        color: currentPiece.color
      };
      
      currentPiece = {
        shape: holdPiece.shape,
        color: holdPiece.color,
        x: Math.floor((COLS - holdPiece.shape[0].length) / 2),
        y: 0
      };
      
      holdPiece = temp;
    }
    
    canHold = false;
  }

  // Calculate game speed based on level with a gentler curve
  function calculateGameSpeed(level: number): number {
    // Base speed is 1000ms (1 second)
    const baseSpeed = 1000;
    
    if (level <= 5) {
      // For levels 1-5, decrease by 100ms per level (same as before but only up to level 5)
      return Math.max(500, baseSpeed - (level - 1) * 100);
    } else if (level <= 10) {
      // For levels 6-10, decrease by 50ms per level
      const level5Speed = 500; // Speed at level 5
      return Math.max(250, level5Speed - (level - 5) * 50);
    } else if (level <= 15) {
      // For levels 11-15, decrease by 20ms per level
      const level10Speed = 250; // Speed at level 10
      return Math.max(150, level10Speed - (level - 10) * 20);
    } else {
      // For levels 16+, decrease by 5ms per level with a minimum of 100ms
      const level15Speed = 150; // Speed at level 15
      return Math.max(100, level15Speed - (level - 15) * 5);
    }
  }

  // Update game speed based on level
  function updateGameSpeed() {
    if (gameInterval) {
      clearInterval(gameInterval);
    }
    
    const speed = calculateGameSpeed(level);
    gameInterval = setInterval(() => {
      movePiece(0, 1);
    }, speed);
  }

  // Render the game board
  function drawBoard(ctx: CanvasRenderingContext2D) {
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    
    // Draw the border
    ctx.strokeStyle = '#ff4a9f';
    ctx.lineWidth = 4;
    ctx.strokeRect(0, 0, COLS * BLOCK_SIZE, ROWS * BLOCK_SIZE);
    
    // Draw the board
    for (let y = 0; y < ROWS; y++) {
      for (let x = 0; x < COLS; x++) {
        drawBlock(ctx, x, y, board[y][x]);
      }
    }
    
    // Draw the current piece
    if (currentPiece) {
      for (let y = 0; y < currentPiece.shape.length; y++) {
        for (let x = 0; x < currentPiece.shape[y].length; x++) {
          if (currentPiece.shape[y][x]) {
            const boardX = currentPiece.x + x;
            const boardY = currentPiece.y + y;
            if (boardY >= 0) { // Only draw if on screen
              drawBlock(ctx, boardX, boardY, currentPiece.color);
            }
          }
        }
      }
    }
    
    // Draw particles
    drawParticles(ctx);
  }

  // Draw a single block
  function drawBlock(ctx: CanvasRenderingContext2D, x: number, y: number, color: string) {
    ctx.fillStyle = color;
    ctx.fillRect(x * BLOCK_SIZE, y * BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE);
    ctx.strokeStyle = '#777';
    ctx.strokeRect(x * BLOCK_SIZE, y * BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE);
  }

  // Draw the next piece preview
  function drawNextPiece(ctx: CanvasRenderingContext2D) {
    if (!nextPiece) return;
    
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    
    const blockSize = Math.min(
      Math.floor(ctx.canvas.width / 4),
      Math.floor(ctx.canvas.height / 4)
    );
    
    const offsetX = (ctx.canvas.width - nextPiece.shape[0].length * blockSize) / 2;
    const offsetY = (ctx.canvas.height - nextPiece.shape.length * blockSize) / 2;
    
    for (let y = 0; y < nextPiece.shape.length; y++) {
      for (let x = 0; x < nextPiece.shape[y].length; x++) {
        if (nextPiece.shape[y][x]) {
          ctx.fillStyle = nextPiece.color;
          ctx.fillRect(
            offsetX + x * blockSize,
            offsetY + y * blockSize,
            blockSize,
            blockSize
          );
          ctx.strokeStyle = '#777';
          ctx.strokeRect(
            offsetX + x * blockSize,
            offsetY + y * blockSize,
            blockSize,
            blockSize
          );
        }
      }
    }
  }

  // Draw the hold piece
  function drawHoldPiece(ctx: CanvasRenderingContext2D) {
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    
    if (!holdPiece) return;
    
    const blockSize = Math.min(
      Math.floor(ctx.canvas.width / 4),
      Math.floor(ctx.canvas.height / 4)
    );
    
    const offsetX = (ctx.canvas.width - holdPiece.shape[0].length * blockSize) / 2;
    const offsetY = (ctx.canvas.height - holdPiece.shape.length * blockSize) / 2;
    
    // Draw with a dimmed color if can't hold
    const color = canHold ? holdPiece.color : holdPiece.color + '80';
    
    for (let y = 0; y < holdPiece.shape.length; y++) {
      for (let x = 0; x < holdPiece.shape[y].length; x++) {
        if (holdPiece.shape[y][x]) {
          ctx.fillStyle = color;
          ctx.fillRect(
            offsetX + x * blockSize,
            offsetY + y * blockSize,
            blockSize,
            blockSize
          );
          ctx.strokeStyle = '#777';
          ctx.strokeRect(
            offsetX + x * blockSize,
            offsetY + y * blockSize,
            blockSize,
            blockSize
          );
        }
      }
    }
  }

  // Handle keyboard input
  function handleKeyDown(e: KeyboardEvent) {
    if (gameOver) return;
    
    switch (e.key) {
      case 'ArrowLeft':
        movePiece(-1, 0);
        break;
      case 'ArrowRight':
        movePiece(1, 0);
        break;
      case 'ArrowDown':
        hardDrop();
        break;
      case 'ArrowUp':
        movePiece(0, 1); // Soft drop
        break;
      case ' ':
        rotatePiece();
        break;
      case 'h':
      case 'H':
        holdCurrentPiece();
        break;
      case 'p':
      case 'P':
        togglePause();
        break;
    }
  }

  function togglePause() {
    isPaused = !isPaused;
    if (isPaused) {
      clearInterval(gameInterval);
    } else {
      updateGameSpeed();
    }
  }

  // Start a new game
  function startGame() {
    board = createEmptyBoard();
    score = 0;
    level = 1;
    gameOver = false;
    isPaused = false;
    currentPiece = getRandomPiece();
    nextPiece = getRandomPiece();
    holdPiece = null;
    canHold = true;
    particles = [];
    
    updateGameSpeed();
  }
  
  // Load high score from local storage
  function loadHighScore() {
    if (browser) {
      const storedScore = localStorage.getItem(LOCAL_STORAGE_KEY);
      if (storedScore) {
        highScore = parseInt(storedScore, 10);
      }
    }
  }

  let boardCtx: CanvasRenderingContext2D;
  let nextCtx: CanvasRenderingContext2D;
  let holdCtx: CanvasRenderingContext2D;
  let animationId: number;
  let lastTime = 0;

  // Main game loop
  function gameLoop(timestamp: number) {
    // Calculate delta time
    const deltaTime = (timestamp - lastTime) / 1000; // Convert to seconds
    lastTime = timestamp;
    
    if (!gameOver && !isPaused) {
      // Update particles
      updateParticles(deltaTime);
      
      // Draw everything
      drawBoard(boardCtx);
      drawNextPiece(nextCtx);
      drawHoldPiece(holdCtx);
    }
    
    animationId = requestAnimationFrame(gameLoop);
  }

  onMount(() => {
    if (!browser) return;
    
    // Load high score
    loadHighScore();
    
    const boardCanvas = document.getElementById('boardCanvas') as HTMLCanvasElement;
    const nextCanvas = document.getElementById('nextCanvas') as HTMLCanvasElement;
    const holdCanvas = document.getElementById('holdCanvas') as HTMLCanvasElement;
    
    if (boardCanvas && nextCanvas && holdCanvas) {
      boardCtx = boardCanvas.getContext('2d') as CanvasRenderingContext2D;
      nextCtx = nextCanvas.getContext('2d') as CanvasRenderingContext2D;
      holdCtx = holdCanvas.getContext('2d') as CanvasRenderingContext2D;
      
      boardCanvas.width = COLS * BLOCK_SIZE;
      boardCanvas.height = ROWS * BLOCK_SIZE;
      
      window.addEventListener('keydown', handleKeyDown);
      
      startGame();
      lastTime = performance.now();
      animationId = requestAnimationFrame(gameLoop);
    }
  });

  onDestroy(() => {
    if (!browser) return;
    window.removeEventListener('keydown', handleKeyDown);
    clearInterval(gameInterval);
    cancelAnimationFrame(animationId);
  });

  function handleMoveButtonClick(direction: string) {
    if (direction === 'left') movePiece(-1, 0);
    else if (direction === 'right') movePiece(1, 0);
  }

  function handleRotateButtonClick() {
    rotatePiece();
  }

  function handleSoftDropButtonClick() {
    movePiece(0, 1);
  }

  function handleHardDropButtonClick() {
    hardDrop();
  }

  function handleHoldButtonClick() {
    holdCurrentPiece();
  }
</script>

<svelte:head>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous">
  <link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;500&display=swap" rel="stylesheet">
  <link href="https://fonts.googleapis.com/css2?family=Pacifico&display=swap" rel="stylesheet">
</svelte:head>

<div class="container">
  <div class="logo-container">
    <div class="logo">glitter blocks</div>
    <div class="logo-sparkles">
      <div class="sparkle">✦</div>
      <div class="sparkle">★</div>
      <div class="sparkle">✦</div>
      <div class="sparkle">★</div>
      <div class="sparkle">✦</div>
    </div>
  </div>

  <div class="game-container">
    <div class="sidebar">
      <div class="panel">
        <div class="panel-header">NEXT</div>
        <div class="panel-content">
          <canvas id="nextCanvas" width="100" height="100"></canvas>
        </div>
      </div>
      
      <div class="panel">
        <div class="panel-header">HOLD</div>
        <div class="panel-content">
          <canvas id="holdCanvas" width="100" height="100"></canvas>
        </div>
      </div>
      
      <div class="panel">
        <div class="panel-header">HIGH SCORE</div>
        <div class="panel-content">
          <span>{highScore}</span>
        </div>
      </div>
    </div>

    <div class="board-container">
      <canvas id="boardCanvas"></canvas>
    </div>

    <div class="sidebar right">
      <div class="panel">
        <div class="panel-header">LEVEL</div>
        <div class="panel-content">
          <span>{level}</span>
        </div>
      </div>
      
      <div class="panel">
        <div class="panel-header">SCORE</div>
        <div class="panel-content">
          <span>{score}</span>
        </div>
      </div>
    </div>
  </div>

  <div class="controls">
    <button on:click={() => handleMoveButtonClick('left')}>←<span>Move</span></button>
    <button on:click={handleRotateButtonClick}>Space<span>Rotate</span></button>
    <button on:click={handleSoftDropButtonClick}>↑<span>Soft</span></button>
    <button on:click={handleHardDropButtonClick}>↓<span>Hard</span></button>
    <button on:click={handleHoldButtonClick}>H<span>Hold</span></button>
  </div>

  {#if gameOver}
    <div class="game-over">
      <div class="message">Game Over</div>
      <button on:click={startGame}>Play Again</button>
    </div>
  {/if}
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    background-color: #FFD6E7; /* Original color from the sparkle image */
    font-family: Arial, sans-serif;
  }

  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 20px;
    max-width: 800px;
    margin: 0 auto;
  }
  
  .logo-container {
    text-align: center;
    width: 100%;
    margin-bottom: 10px;
    padding: 20px 0;
    position: relative;
  }

  .logo {
    font-size: 60px;
    font-weight: normal;
    color: white;
    margin-bottom: 30px;
    text-align: center;
    font-family: 'Pacifico', cursive;
    letter-spacing: 1px;
    line-height: 1.1;
    position: relative;
    text-shadow: 
      3px 3px 0 #ff70b5,
      6px 6px 0 #ff4a9f,
      0px 1px 5px rgba(0, 0, 0, 0.1);
    display: inline-block;
    transform: rotate(-5deg);
  }
  
  .logo::before {
    content: "✦";
    position: absolute;
    top: -15px;
    right: -15px;
    font-size: 24px;
    color: white;
  }
  
  .logo::after {
    content: "★";
    position: absolute;
    bottom: 0px;
    left: -15px;
    font-size: 20px;
    color: white;
  }
  
  .logo-sparkles {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    pointer-events: none;
    z-index: 1;
  }
  
  .sparkle {
    position: absolute;
    color: white;
    font-size: 12px;
  }
  
  .sparkle:nth-child(1) { top: 10%; left: 20%; animation: twinkle 2s infinite; }
  .sparkle:nth-child(2) { top: 15%; left: 80%; animation: twinkle 3s infinite 0.5s; }
  .sparkle:nth-child(3) { top: 80%; left: 15%; animation: twinkle 2.5s infinite 1s; }
  .sparkle:nth-child(4) { top: 60%; left: 90%; animation: twinkle 2s infinite 0.7s; }
  .sparkle:nth-child(5) { top: 50%; left: 50%; animation: twinkle 3s infinite 0.3s; }
  
  @keyframes twinkle {
    0% { opacity: 0; transform: scale(0.5); }
    50% { opacity: 1; transform: scale(1.2); }
    100% { opacity: 0; transform: scale(0.5); }
  }

  .game-container {
    display: flex;
    justify-content: center;
    margin-bottom: 20px;
  }

  .board-container {
    border: 4px solid #ff4a9f;
    background-color: #fff;
    border-radius: 8px;
    overflow: hidden;
  }

  canvas {
    display: block;
  }

  .sidebar {
    display: flex;
    flex-direction: column;
    margin: 0 20px;
  }

  .right {
    align-items: flex-start;
  }

  .panel {
    margin-bottom: 20px;
    width: 100px;
    border-radius: 8px;
    overflow: hidden;
  }

  .panel-header {
    background-color: #ff4a9f;
    color: white;
    padding: 5px;
    text-align: center;
    font-weight: bold;
  }

  .panel-content {
    background-color: white;
    height: 100px;
    width: 100px;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 24px;
    font-weight: bold;
  }

  .panel-content canvas {
    width: 100%;
    height: 100%;
  }

  .controls {
    display: flex;
    justify-content: center;
    margin-top: 10px;
  }

  .controls button {
    background-color: #ff4a9f;
    color: white;
    border: none;
    padding: 8px 12px;
    margin: 0 5px;
    cursor: pointer;
    display: flex;
    flex-direction: column;
    align-items: center;
    border-radius: 8px;
  }

  .controls button span {
    font-size: 12px;
  }

  .game-over {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.7);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    z-index: 10;
  }

  .message {
    color: white;
    font-size: 48px;
    margin-bottom: 20px;
  }

  .game-over button {
    background-color: #ff4a9f;
    color: white;
    border: none;
    padding: 10px 20px;
    font-size: 18px;
    cursor: pointer;
    border-radius: 8px;
  }
</style>
