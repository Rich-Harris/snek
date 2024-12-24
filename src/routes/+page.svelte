<script lang="ts">
	import gameover from '$lib/audio/gameover.wav';
	import powerup from '$lib/audio/powerup.wav';
	import snek from '$lib/images/snek.png';
	import { innerWidth } from 'svelte/reactivity/window';

	// Grid size
	const GRID_SIZE = 20;
	const CELL_SIZE = $derived(Math.min(20, ((innerWidth.current as number) - 40) / GRID_SIZE));

	// Game state
	let directionBuffer = $state<('up' | 'down' | 'left' | 'right')[]>([]);
	let currentDirection = $state<'up' | 'down' | 'left' | 'right'>('right');
	let lastInputtedDirection = $derived(directionBuffer.at(-1) || currentDirection);
	let snake = $state<[number, number][]>([[0, 0]]); // Array of [x, y] coordinates
	let food = $state<[number, number]>([5, 5]);
	let score = $state(0);
	let gameOver = $state(false);
	let gameLoopInterval = $state<number | null>(null);

	// Derived values
	let gameBoard = $derived.by(() => {
		const board: ('empty' | 'snake' | 'food')[][] = Array(GRID_SIZE)
			.fill('empty')
			.map(() => Array(GRID_SIZE).fill('empty'));

		// Place snake
		snake.forEach(([x, y]) => {
			if (x >= 0 && x < GRID_SIZE && y >= 0 && y < GRID_SIZE) {
				board[y][x] = 'snake';
			}
		});

		// Place food
		board[food[1]][food[0]] = 'food';

		return board;
	});

	// Game logic functions
	function startGame() {
		if (gameLoopInterval) return;

		resetGame();
		gameLoopInterval = window.setInterval(gameLoop, 150);
	}

	function resetGame() {
		snake = [[0, 0]];
		currentDirection = 'right';
		directionBuffer = [];
		score = 0;
		gameOver = false;
		placeFood();
	}

	function placeFood() {
		let newFood: [number, number];
		do {
			newFood = [Math.floor(Math.random() * GRID_SIZE), Math.floor(Math.random() * GRID_SIZE)];
		} while (snake.some(([x, y]) => x === newFood[0] && y === newFood[1]));

		food = newFood;
	}

	function gameLoop() {
		if (gameOver) {
			if (gameLoopInterval) {
				clearInterval(gameLoopInterval);
				gameLoopInterval = null;
			}
			return;
		}

		// Get new current position from Buffer
		currentDirection = directionBuffer.shift() || currentDirection;

		const head = snake[0];
		let newHead: [number, number];

		// Calculate new head position
		switch (currentDirection) {
			case 'up':
				newHead = [head[0], head[1] - 1];
				break;
			case 'down':
				newHead = [head[0], head[1] + 1];
				break;
			case 'left':
				newHead = [head[0] - 1, head[1]];
				break;
			case 'right':
				newHead = [head[0] + 1, head[1]];
				break;
		}

		// Check for collisions
		if (
			newHead[0] < 0 ||
			newHead[0] >= GRID_SIZE ||
			newHead[1] < 0 ||
			newHead[1] >= GRID_SIZE ||
			snake.some(([x, y]) => x === newHead[0] && y === newHead[1])
		) {
			new Audio(gameover).play();
			gameOver = true;
			return;
		}

		// Check if food is eaten
		const ateFood = newHead[0] === food[0] && newHead[1] === food[1];
		if (ateFood) {
			new Audio(powerup).play();
			score += 10;
			placeFood();
		}

		// Update snake position
		snake = [newHead, ...snake.slice(0, ateFood ? snake.length : -1)];
	}

	function handleKeydown(event: KeyboardEvent) {
		switch (event.key) {
			case 'ArrowUp':
				if (lastInputtedDirection !== 'down') directionBuffer.push('up');
				break;
			case 'ArrowDown':
				if (lastInputtedDirection !== 'up') directionBuffer.push('down');
				break;
			case 'ArrowLeft':
				if (lastInputtedDirection !== 'right') directionBuffer.push('left');
				break;
			case 'ArrowRight':
				if (lastInputtedDirection !== 'left') directionBuffer.push('right');
				break;
		}
	}

	function handlePointerdown(event: PointerEvent) {
		let x = event.clientX / window.innerWidth - 0.5;
		let y = event.clientY / window.innerHeight - 0.5;

		if (Math.abs(x) > Math.abs(y)) {
			if (x > 0) {
				if (lastInputtedDirection !== 'left') directionBuffer.push('right');
			} else {
				if (lastInputtedDirection !== 'right') directionBuffer.push('left');
			}
		} else {
			if (y > 0) {
				if (lastInputtedDirection !== 'up') directionBuffer.push('down');
			} else {
				if (lastInputtedDirection !== 'down') directionBuffer.push('up');
			}
		}
	}
</script>

<svelte:window onkeydown={handleKeydown} onpointerdown={handlePointerdown} />

<div class="game-container">
	<div class="score">Score: {score}</div>

	<div style="width: {GRID_SIZE * CELL_SIZE}px; height: {GRID_SIZE * CELL_SIZE}px">
		{#if gameLoopInterval || gameOver}
			<div class="game-board">
				{#each gameBoard as row, y}
					{#each row as cell, x}
						<div
							class="cell {cell}"
							style="
					width: {CELL_SIZE}px;
					height: {CELL_SIZE}px;
					transform: translate({x * CELL_SIZE}px, {y * CELL_SIZE}px)
				"
						></div>
					{/each}
				{/each}
			</div>
		{:else}
			<div class="splash">
				<img class="logo" alt="Snek logo" src={snek} />
				<p>An <a href="https://svelte.dev/blog/advent-of-svelte">Advent of Svelte</a> production</p>
			</div>
		{/if}
	</div>

	{#if gameOver}
		<div class="game-over">
			<h2>Game Over!</h2>
			<p>Final Score: {score}</p>
			<button onclick={startGame}>Play Again</button>
		</div>
	{:else if !gameLoopInterval}
		<button class="start-button" onclick={startGame}>Start Game</button>
	{/if}
</div>

<style>
	.game-container {
		display: grid;
		grid-template-rows: 4em 1fr 4em;
		flex-direction: column;
		align-items: center;
		gap: 20px;
		padding: 20px;
	}

	.score {
		font-weight: bold;
		text-align: center;
	}
	.splash {
		position: relative;
		display: flex;
		flex-direction: column;
		align-items: center;
		width: 100%;
		height: 100%;

		.logo {
			flex: 1;
			aspect-ratio: 1;
			image-rendering: pixelated;
		}
	}

	.game-board {
		position: relative;
		width: 100%;
		height: 100%;
		background-color: #f0f0f0;
		border: 2px solid #333;
	}

	.cell {
		position: absolute;
		box-sizing: border-box;
	}

	.cell.snake {
		background-color: #4caf50;
		border-radius: 4px;
	}

	.cell.food {
		background-color: #ff5722;
		border-radius: 50%;
	}

	.game-over {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		background-color: rgba(255, 255, 255, 0.9);
		padding: 20px;
		border-radius: 8px;
		text-align: center;
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
		color: black;
	}

	button {
		padding: 10px 20px;
		background-color: #4caf50;
		color: white;
		border: none;
		border-radius: 4px;
		cursor: pointer;
	}

	button:hover {
		background-color: #45a049;
	}
</style>
