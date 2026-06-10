<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let animFrame: number;
	let keys: Record<string, boolean> = {};
	let score = 0;
	let gameWon = false;
	let confettiParticles: { x: number; y: number; vx: number; vy: number; color: string; life: number; size: number }[] = [];

	// --- Game dimensions ---
	const W = 960;
	const H = 640;
	const GRAVITY = 0.6;
	const JUMP_FORCE = -12.5;
	const MOVE_SPEED = 4;
	const GROUND_Y = 600;

	// --- Player (DeepSeek whale) ---
	const player = {
		x: 200,
		y: GROUND_Y - 40,
		w: 56,
		h: 40,
		vx: 0,
		vy: 0,
		facingRight: true,
		grounded: false
	};

	// --- Collectible coins ---
	const coins: { x: number; y: number; r: number; collected: boolean }[] = [
		{ x: 400, y: 460, r: 10, collected: false },
		{ x: 550, y: 380, r: 10, collected: false },
		{ x: 750, y: 300, r: 10, collected: false },
		{ x: 300, y: 220, r: 10, collected: false },
		{ x: 650, y: 140, r: 10, collected: false },
	];

	// --- Platforms ---
	const platforms = [
		{ x: 0, y: GROUND_Y, w: W, h: 40 },           // ground
		{ x: 350, y: 500, w: 150, h: 16 },             // platform 1
		{ x: 500, y: 420, w: 120, h: 16 },             // platform 2
		{ x: 200, y: 340, w: 140, h: 16 },             // platform 3
		{ x: 450, y: 260, w: 120, h: 16 },             // platform 4
		{ x: 680, y: 340, w: 150, h: 16 },             // platform 5
		{ x: 100, y: 180, w: 200, h: 16 },             // platform 6
		{ x: 600, y: 180, w: 160, h: 16 },             // platform 7
	];

	// --- Camera ---
	let camX = 0;

	// --- Draw the DeepSeek whale ---
	function drawWhale(x: number, y: number, facingRight: boolean) {
		const ctx = window.__ctx!;
		ctx.save();
		ctx.translate(x, y);
		if (!facingRight) ctx.scale(-1, 1);

		// Body - deep blue oval
		ctx.fillStyle = '#2563EB';
		ctx.beginPath();
		ctx.ellipse(0, 0, 24, 16, 0, 0, Math.PI * 2);
		ctx.fill();

		// Belly - lighter blue
		ctx.fillStyle = '#93C5FD';
		ctx.beginPath();
		ctx.ellipse(-2, 6, 16, 8, 0, 0, Math.PI * 2);
		ctx.fill();

		// Eye
		ctx.fillStyle = '#FFFFFF';
		ctx.beginPath();
		ctx.arc(12, -4, 5, 0, Math.PI * 2);
		ctx.fill();
		ctx.fillStyle = '#1E293B';
		ctx.beginPath();
		ctx.arc(14, -4, 2.5, 0, Math.PI * 2);
		ctx.fill();

		// Smile
		ctx.strokeStyle = '#1E40AF';
		ctx.lineWidth = 1.5;
		ctx.beginPath();
		ctx.arc(10, 2, 6, 0.2, Math.PI - 0.2);
		ctx.stroke();

		// Tail fin
		ctx.fillStyle = '#2563EB';
		ctx.beginPath();
		ctx.moveTo(-22, 0);
		ctx.lineTo(-36, -14);
		ctx.lineTo(-36, 14);
		ctx.closePath();
		ctx.fill();

		// Dorsal fin
		ctx.beginPath();
		ctx.moveTo(-6, -14);
		ctx.lineTo(2, -26);
		ctx.lineTo(10, -14);
		ctx.closePath();
		ctx.fill();

		// Blowhole spout
		if (Math.random() < 0.05) {
			ctx.fillStyle = '#BFDBFE';
			ctx.beginPath();
			ctx.arc(0, -20, 3, 0, Math.PI * 2);
			ctx.fill();
			ctx.beginPath();
			ctx.arc(1, -26, 2, 0, Math.PI * 2);
			ctx.fill();
			ctx.beginPath();
			ctx.arc(3, -30, 1.5, 0, Math.PI * 2);
			ctx.fill();
		}

		ctx.restore();
	}

	// --- Draw a coin ---
	function drawCoin(x: number, y: number, r: number) {
		const ctx = window.__ctx!;
		ctx.fillStyle = '#FBBF24';
		ctx.beginPath();
		ctx.arc(x, y, r, 0, Math.PI * 2);
		ctx.fill();
		ctx.fillStyle = '#F59E0B';
		ctx.beginPath();
		ctx.arc(x, y, r - 3, 0, Math.PI * 2);
		ctx.fill();
		ctx.fillStyle = '#FBBF24';
		ctx.beginPath();
		ctx.arc(x - 2, y - 2, 3, 0, Math.PI * 2);
		ctx.fill();
	}

	// --- AABB collision ---
	function rectCollide(
		ax: number, ay: number, aw: number, ah: number,
		bx: number, by: number, bw: number, bh: number
	): boolean {
		return ax < bx + bw && ax + aw > bx && ay < by + bh && ay + ah > by;
	}

	function resolvePlatforms() {
		player.grounded = false;
		for (const p of platforms) {
			if (rectCollide(player.x, player.y, player.w, player.h, p.x, p.y, p.w, p.h)) {
				// From above - landing
				if (player.vy >= 0 && player.y + player.h - player.vy <= p.y + 4) {
					player.y = p.y - player.h;
					player.vy = 0;
					player.grounded = true;
				}
				// From below
				else if (player.vy < 0 && player.y - player.vy >= p.y + p.h - 4) {
					player.y = p.y + p.h;
					player.vy = 0;
				}
				// From left
				else if (player.vx > 0 && player.x + player.w - player.vx <= p.x + 4) {
					player.x = p.x - player.w;
					player.vx = 0;
				}
				// From right
				else if (player.vx < 0 && player.x - player.vx >= p.x + p.w - 4) {
					player.x = p.x + p.w;
					player.vx = 0;
				}
			}
		}
	}

	function collectCoins() {
		for (const c of coins) {
			if (c.collected) continue;
			const dx = (player.x + player.w / 2) - c.x;
			const dy = (player.y + player.h / 2) - c.y;
			if (Math.sqrt(dx * dx + dy * dy) < c.r + 20) {
				c.collected = true;
				score += 10;
			}
		}
		if (coins.every(c => c.collected) && !gameWon) {
			gameWon = true;
			spawnConfetti();
		}
	}

	function spawnConfetti() {
		const colors = ['#FBBF24','#EF4444','#3B82F6','#10B981','#F472B6','#A78BFA','#F97316'];
		for (let i = 0; i < 120; i++) {
			confettiParticles.push({
				x: player.x + player.w / 2,
				y: player.y,
				vx: (Math.random() - 0.5) * 8,
				vy: -Math.random() * 6 - 2,
				color: colors[Math.floor(Math.random() * colors.length)],
				life: 1,
				size: 4 + Math.random() * 6
			});
		}
	}

	function resetGame() {
		gameWon = false;
		score = 0;
		confettiParticles = [];
		player.x = 200;
		player.y = GROUND_Y - 40;
		player.vx = 0;
		player.vy = 0;
		player.facingRight = true;
		player.grounded = false;
		for (const c of coins) c.collected = false;
	}

	function update() {
		if (gameWon) {
			// Animate confetti
			for (const p of confettiParticles) {
				p.x += p.vx;
				p.y += p.vy;
				p.vy += 0.05;
				p.life -= 0.008;
			}
			confettiParticles = confettiParticles.filter(p => p.life > 0);
			return;
		}

		// Input
		if (keys['ArrowLeft'] || keys['KeyA']) player.vx = -MOVE_SPEED;
		else if (keys['ArrowRight'] || keys['KeyD']) player.vx = MOVE_SPEED;
		else player.vx *= 0.7;

		if ((keys['ArrowUp'] || keys['Space'] || keys['KeyW']) && player.grounded) {
			player.vy = JUMP_FORCE;
		}

		// Physics
		player.vy += GRAVITY;
		player.x += player.vx;
		player.y += player.vy;

		// Facing direction
		if (player.vx > 0.3) player.facingRight = true;
		if (player.vx < -0.3) player.facingRight = false;

		resolvePlatforms();
		collectCoins();

		// Keep on screen horizontally
		player.x = Math.max(0, Math.min(W - player.w, player.x));

		// Fall off bottom -> reset
		if (player.y > H + 100) {
			player.x = 200;
			player.y = GROUND_Y - 40;
			player.vx = 0;
			player.vy = 0;
		}

		// Camera
		camX = player.x - W / 3;
		camX = Math.max(0, Math.min(W - W, camX));
	}

	function draw() {
		const c = ctx;
		// Sky gradient
		const grad = c.createLinearGradient(0, 0, 0, H);
		grad.addColorStop(0, '#0F172A');
		grad.addColorStop(0.5, '#1E3A5F');
		grad.addColorStop(1, '#312E81');
		c.fillStyle = grad;
		c.fillRect(0, 0, W, H);

		// Stars
		c.fillStyle = '#FFFFFF33';
		for (let i = 0; i < 40; i++) {
			const sx = ((i * 137 + 50) % W);
			const sy = ((i * 89 + 20) % 200);
			c.beginPath();
			c.arc(sx, sy, 1.2, 0, Math.PI * 2);
			c.fill();
		}

		// Platforms
		for (const p of platforms) {
			c.fillStyle = '#475569';
			c.fillRect(p.x, p.y, p.w, p.h);
			c.fillStyle = '#64748B';
			c.fillRect(p.x, p.y, p.w, 4);
		}

		// Coins
		for (const c of coins) {
			if (!c.collected) drawCoin(c.x, c.y, c.r);
		}

		// Player
		drawWhale(player.x + player.w / 2, player.y + player.h / 2, player.facingRight);

		// Confetti
		for (const p of confettiParticles) {
			c.globalAlpha = p.life;
			c.fillStyle = p.color;
			c.fillRect(p.x - p.size / 2, p.y - p.size / 2, p.size, p.size * 0.6);
		}
		c.globalAlpha = 1;

		// HUD
		c.fillStyle = '#FFFFFF';
		c.font = 'bold 20px monospace';
		c.fillText(`⭐ ${score}`, 20, 36);
		c.fillText('Arrow keys / WASD to move  |  Space / Up to jump', 20, H - 20);
	}

	function gameLoop() {
		update();
		draw();
		animFrame = requestAnimationFrame(gameLoop);
	}

	function keyDown(e: KeyboardEvent) { keys[e.code] = true; e.preventDefault(); }
	function keyUp(e: KeyboardEvent) { keys[e.code] = false; e.preventDefault(); }

	onMount(() => {
		ctx = canvas.getContext('2d')!;
		(window as any).__ctx = ctx;
		window.addEventListener('keydown', keyDown);
		window.addEventListener('keyup', keyUp);
		animFrame = requestAnimationFrame(gameLoop);
	});

	onDestroy(() => {
		if (!browser) return;
		cancelAnimationFrame(animFrame);
		window.removeEventListener('keydown', keyDown);
		window.removeEventListener('keyup', keyUp);
	});
</script>

<svelte:head>
	<title>DeepSeek Whale Platformer</title>
	<style>
		body { margin: 0; background: #0F172A; overflow: hidden; }
	</style>
</svelte:head>

<div class="game-container">
	<canvas bind:this={canvas} width={960} height={640}></canvas>

	{#if gameWon}
		<div class="modal-overlay">
			<div class="modal">
				<div class="modal-whale">🐋</div>
				<h1>You Win!</h1>
				<p>The DeepSeek whale collected all the coins!</p>
				<p class="score-final">Score: <strong>{score}</strong></p>
				<button on:click={resetGame}>Play Again</button>
			</div>
		</div>
	{/if}
</div>

<style>
	.game-container {
		display: flex;
		justify-content: center;
		align-items: center;
		width: 100vw;
		height: 100vh;
		background: #0F172A;
	}

	canvas {
		border: 2px solid #334155;
		border-radius: 8px;
		image-rendering: pixelated;
	}

	.modal-overlay {
		position: fixed;
		inset: 0;
		background: rgba(0,0,0,0.7);
		display: flex;
		justify-content: center;
		align-items: center;
		z-index: 100;
		animation: fadeIn 0.3s ease;
	}

	.modal {
		background: linear-gradient(135deg, #1E293B, #312E81);
		border: 2px solid #6366F1;
		border-radius: 16px;
		padding: 40px 48px;
		text-align: center;
		color: #E2E8F0;
		box-shadow: 0 0 60px rgba(99,102,241,0.4);
		animation: popIn 0.4s ease;
	}

	.modal-whale {
		font-size: 64px;
		margin-bottom: 8px;
	}

	.modal h1 {
		font-size: 36px;
		margin: 0 0 8px 0;
		color: #818CF8;
	}

	.modal p {
		margin: 6px 0;
		font-size: 16px;
	}

	.score-final {
		color: #FBBF24;
		font-size: 20px;
	}

	.modal button {
		margin-top: 20px;
		padding: 12px 32px;
		font-size: 18px;
		font-weight: bold;
		background: linear-gradient(135deg, #6366F1, #818CF8);
		color: white;
		border: none;
		border-radius: 8px;
		cursor: pointer;
		transition: transform 0.15s ease, box-shadow 0.15s ease;
	}

	.modal button:hover {
		transform: scale(1.05);
		box-shadow: 0 0 20px rgba(99,102,241,0.6);
	}

	@keyframes fadeIn {
		from { opacity: 0; }
		to { opacity: 1; }
	}

	@keyframes popIn {
		from { transform: scale(0.7); opacity: 0; }
		to { transform: scale(1); opacity: 1; }
	}
</style>
