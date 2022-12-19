<script>
	import * as Pancake from '@sveltejs/pancake';
	import { onMount } from 'svelte';

	let currentMode = 'bangbang';

	let startPoint = 0;
	let setpoint = 1500;
	let maxAcceleration = 5000;
	let outputPercent = 1;
	let outputPercentOff = -1;
	let friction = 0.01;
	let minSpeed = 30;

	let deltaTime = 0.02;
	let simulationTime = 5;

	let graphData = [];

	let minX = 0;
	$: minY = Math.min(-maxAcceleration * deltaTime * 2, setpoint, startPoint);
	$: maxX = simulationTime;
	$: maxY = Math.max(setpoint, startPoint) + maxAcceleration * deltaTime * 2;

	let kP = 0.01;
	let kI = 0.00016;
	let kD = 0.00005;

	$: simulate(currentMode, startPoint, setpoint, maxAcceleration, kP, kI, kD, simulationTime);

	function simulate(
		currentMode,
		startPoint,
		setpoint,
		maxAcceleration,
		kP,
		kI,
		kD,
		simulationTime
	) {
		kP = kP === '' ? 0 : kP;
		kI = kI === '' ? 0 : kI;
		kD = kD === '' ? 0 : kD;
		let data = [];
		let currentVelocity = startPoint;
		let previousError = 0;
		let integral = 0;
		for (let time = 0; time < simulationTime; time += deltaTime) {
			let output = 0;
			if (currentMode === 'bangbang') {
				output = bangBang(currentVelocity, setpoint, outputPercent, outputPercentOff);
			} else if (currentMode === 'pid') {
				const [out, int, error] = pid(
					currentVelocity,
					setpoint,
					integral,
					previousError,
					kP,
					kI,
					kD
				);
				output = Math.max(Math.min(out, 1), -1);
				integral = int;
				previousError = error;
			}
			currentVelocity += output * maxAcceleration * deltaTime;
			// Apply friction
			let frictionOut = friction * currentVelocity;
			currentVelocity -= frictionOut;

			// Minimum speed due to friction
			if (Math.abs(currentVelocity) < minSpeed) {
				currentVelocity = 0;
			}

			// Log data
			data.push({
				time,
				currentVelocity,
				output,
				integral,
				previousError
			});
		}
		console.log(data);
		graphData = data;
	}

	function bangBang(currentSpeed, targetSpeed, outputPercentOn, outputPercentOff) {
		if (currentSpeed < targetSpeed) {
			return outputPercentOn;
		} else {
			return outputPercentOff;
		}
	}

	function pid(currentSpeed, targetSpeed, integral, previous_error, kP, kI, kD) {
		let error = targetSpeed - currentSpeed;
		integral = integral + error * deltaTime;
		let derivative = (error - previous_error) / deltaTime;
		let output = error * kP + integral * kI + derivative * kD;
		return [output, integral, error];
	}

	const pc = (time) => {
		return (100 * (time - minX)) / (maxX - minX);
	};
</script>

{#if currentMode === 'bangbang'}
	<h1>Bang Bang (No PID Control)</h1>
{:else if currentMode === 'pid'}
	<h1>PID Control</h1>
{/if}

<select bind:value={currentMode}>
	<option value="bangbang">Bang Bang</option>
	<option value="pid">PID</option>
</select>

<div>
	<label class="form-label" for="setpoint">Start Speed</label>
	<input bind:value={startPoint} type="number" name="setpoint" step={250} />
</div>
<div>
	<label class="form-label" for="setpoint">Setpoint</label>
	<input bind:value={setpoint} type="number" name="setpoint" step={250} />
</div>
<div>
	<label class="form-label" for="simtime">Simulation Time</label>
	<input bind:value={simulationTime} type="number" name="simtime" step={1} min={1} max={50} />
</div>

{#if currentMode === 'pid'}
	<div>
		<label for="kP" class="form-label">P</label>
		<input bind:value={kP} name="kP" type="number" step={0.01} />
	</div>
	<div>
		<label for="kI" class="form-label">I</label>
		<input bind:value={kI} name="kI" type="number" step={0.00001} />
	</div>
	<div>
		<label for="kD" class="form-label">D</label>
		<input bind:value={kD} name="kD" type="number" step={0.00001} />
	</div>
{/if}

<div class="chart">
	<Pancake.Chart x1={minX} x2={maxX} y1={minY} y2={maxY}>
		<Pancake.Grid horizontal count={5} let:value let:last>
			<div class="grid-line horizontal">
				<span>{value} {last ? 'rpm' : ''}</span>
			</div>
		</Pancake.Grid>
		<Pancake.Grid vertical count={simulationTime} let:value>
			<div class="grid-line vertical" />
			<span class="time-label">{value}s</span>
		</Pancake.Grid>
		<Pancake.Svg>
			<Pancake.SvgLine data={graphData} x={(d) => d.time} y={(d) => setpoint} let:d>
				<path class="setpoint" {d} />
			</Pancake.SvgLine>
			<Pancake.SvgLine data={graphData} x={(d) => d.time} y={(d) => d.output * setpoint} let:d>
				<path class="output" {d} />
			</Pancake.SvgLine>
			<Pancake.SvgLine data={graphData} x={(d) => d.time} y={(d) => d.currentVelocity} let:d>
				<path class="velocity" {d} />
			</Pancake.SvgLine>
			{#if currentMode === 'pid'}
				<Pancake.SvgLine data={graphData} x={(d) => d.time} y={(d) => d.integral} let:d>
					<path class="integral" {d} />
				</Pancake.SvgLine>
				<Pancake.SvgLine data={graphData} x={(d) => d.time} y={(d) => d.previousError} let:d>
					<path class="previousError" {d} />
				</Pancake.SvgLine>
			{/if}
		</Pancake.Svg>
		<Pancake.Quadtree data={graphData} x={(d) => d.time} y={(d) => d.currentVelocity} let:closest>
			{#if closest}
				<Pancake.Point x={closest.time} y={closest.currentVelocity} let:d>
					<div class="focus" />
					<div class="tooltip" style="transform: translate-{pc(closest.time)}%, 0">
						<strong>{closest.currentVelocity} rpm</strong>
						<span>{Math.round(closest.time * 100) / 100} s</span>
					</div>
				</Pancake.Point>
			{/if}
		</Pancake.Quadtree>
	</Pancake.Chart>
</div>

<style>
	.chart {
		height: 450px;
		padding: 3em 0 2em 2em;
		margin: 0 0 36px 0;
		max-width: 80em;
	}
	.grid-line {
		position: relative;
		display: block;
	}
	.grid-line.horizontal {
		width: calc(100% + 2em);
		left: -2em;
		border-bottom: 1px dashed #ccc;
	}
	.grid-line.vertical {
		height: 100%;
		border-left: 1px dashed #ccc;
	}
	.grid-line span {
		position: absolute;
		left: 0;
		bottom: 2px;
		line-height: 1;
		font-family: sans-serif;
		font-size: 14px;
		color: #999;
	}

	path.setpoint {
		stroke: black;
		opacity: 1;
		stroke-linejoin: round;
		stroke-linecap: round;
		stroke-width: 3px;
		fill: none;
		stroke-dasharray: 8;
	}
	path.velocity {
		stroke: red;
		opacity: 1;
		stroke-linejoin: round;
		stroke-linecap: round;
		stroke-width: 3px;
		fill: none;
	}
	path.output {
		stroke: blue;
		opacity: 1;
		stroke-linejoin: round;
		stroke-linecap: round;
		stroke-width: 2px;
		fill: none;
	}

	path.previousError {
		stroke: #999;
		opacity: 1;
		stroke-linejoin: round;
		stroke-linecap: round;
		stroke-width: 1px;
		fill: none;
	}
	path.integral {
		stroke: green;
		opacity: 0.6;
		stroke-linejoin: round;
		stroke-linecap: round;
		stroke-width: 1px;
		fill: none;
	}

	div:has(.form-label) {
		padding: 0.25em;
	}

	.form-label {
		width: 10em;
		display: inline-block;
	}

	.focus {
		position: absolute;
		width: 10px;
		height: 10px;
		left: -5px;
		top: -5px;
		border: 1px solid black;
		border-radius: 50%;
		box-sizing: border-box;
	}
	.tooltip {
		position: absolute;
		white-space: nowrap;
		width: 8em;
		bottom: 1em;
		/* background-color: white; */
		line-height: 1;
		text-shadow: 0 0 10px white, 0 0 10px white, 0 0 10px white, 0 0 10px white, 0 0 10px white,
			0 0 10px white, 0 0 10px white;
	}
	.tooltip strong {
		font-size: 1.4em;
		display: block;
	}

	@media (min-width: 800px) {
		.chart {
			height: 600px;
		}
	}
</style>
