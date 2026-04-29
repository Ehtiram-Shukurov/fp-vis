<script>
	// @ts-nocheck
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import { genreColors } from "$lib/genreColors";

	let loading = $state(true);
	let error = $state("");
	let allOccupations = $state([]);
	let activeOccupations = $state({});
	let visibleOccupations = $derived(
		allOccupations.filter((occ) => activeOccupations[occ] !== false),
	);
	let glyphData = $state({});
	let glyphsAnimated = $state(false);
	let hoveredKey = $state(null);
	let selectedGlyphData = $state(null);

	const ageBuckets = [
		{ label: "Under 25", min: 0, max: 24 },
		{ label: "25-34", min: 25, max: 34 },
		{ label: "35-44", min: 35, max: 44 },
		{ label: "45+", min: 45, max: 120 },
	];

	const allAgeLabels = ageBuckets.map((b) => b.label);

	const genreNames = [
		"Action",
		"Adventure",
		"Animation",
		"Children's",
		"Comedy",
		"Crime",
		"Documentary",
		"Drama",
		"Fantasy",
		"Film-Noir",
		"Horror",
		"Musical",
		"Mystery",
		"Romance",
		"Sci-Fi",
		"Thriller",
		"War",
		"Western",
	];

	const margin = { top: 40, right: 30, bottom: 80, left: 80 };
	let width = $derived(
		Math.max(
			400,
			visibleOccupations.length * 80 + margin.left + margin.right,
		),
	);
	let height = $state(450);

	let tooltipVisible = $state(false);
	let tooltipX = $state(0);
	let tooltipY = $state(0);
	let tooltipData = $state(null);

	function handleMouseMove(event, occ, ageLabel) {
		tooltipX = event.clientX + 15;
		tooltipY = event.clientY + 15;
		const stats = glyphData[`${occ}|${ageLabel}`];
		tooltipData = {
			occ,
			ageLabel,
			lines: stats.filter((s) => s.avg > 0).sort((a, b) => b.avg - a.avg),
		};
		tooltipVisible = true;
		hoveredKey = `${occ}|${ageLabel}`;
	}

	function handleMouseLeave() {
		tooltipVisible = false;
		hoveredKey = null;
	}

	function handleGlyphClick(occ, ageLabel) {
		const stats = glyphData[`${occ}|${ageLabel}`];
		let totalSum = 0;
		let totalCount = 0;
		stats.forEach(s => {
			if (s.avg > 0) {
				totalSum += s.avg * s.count;
				totalCount += s.count;
			}
		});
		const overallAvg = totalCount > 0 ? totalSum / totalCount : 0;
		
		selectedGlyphData = {
			occ,
			ageLabel,
			overallAvg,
			lines: stats.filter((s) => s.avg > 0).sort((a, b) => b.avg - a.avg),
		};
	}

	function closeBox() {
		selectedGlyphData = null;
	}

	onMount(async () => {
		try {
			const base = import.meta.env.BASE_URL;
			const [ratingsText, itemsText, usersText] = await Promise.all([
				fetch(`${base}u.data`).then((r) => r.text()),
				fetch(`${base}u.item`).then((r) => r.text()),
				fetch(`${base}u.user`).then((r) => r.text()),
			]);

			let occSet = new Set();
			let userMap = {};
			usersText
				.trim()
				.split("\n")
				.forEach((line) => {
					const parts = line.split("|");
					const id = parseInt(parts[0]);
					const age = parseInt(parts[1]);
					const occupation = parts[3];
					const bucket = ageBuckets.find(
						(b) => age >= b.min && age <= b.max,
					);
					occSet.add(occupation);
					userMap[id] = {
						occupation,
						ageLabel: bucket?.label ?? "Unknown",
					};
				});

			const occupationsArray = Array.from(occSet).sort();
			allOccupations = occupationsArray;

			let movieMap = {};
			itemsText
				.trim()
				.split("\n")
				.forEach((line) => {
					const parts = line.split("|");
					const genres = genreNames.filter(
						(_, i) => parts[6 + i] === "1",
					);
					movieMap[parseInt(parts[0])] = genres;
				});

			let acc = {};
			for (let occ of occupationsArray) {
				for (let ageLabel of allAgeLabels) {
					const key = `${occ}|${ageLabel}`;
					acc[key] = {};
					for (let genre of genreNames)
						acc[key][genre] = { sum: 0, count: 0 };
				}
			}

			ratingsText
				.trim()
				.split("\n")
				.forEach((line) => {
					const parts = line.split("\t");
					const user = userMap[parseInt(parts[0])];
					const genres = movieMap[parseInt(parts[1])];
					const rating = parseInt(parts[2]);
					if (!user || !genres) return;
					const key = `${user.occupation}|${user.ageLabel}`;
					if (!acc[key]) return;
					genres.forEach((genre) => {
						if (acc[key][genre]) {
							acc[key][genre].sum += rating;
							acc[key][genre].count += 1;
						}
					});
				});

			let resultData = {};
			for (let occ of occupationsArray) {
				for (let ageLabel of allAgeLabels) {
					const key = `${occ}|${ageLabel}`;
					resultData[key] = genreNames.map((genre, i) => {
						const stat = acc[key][genre];
						return {
							genre,
							avg: stat.count > 0 ? stat.sum / stat.count : 0,
							count: stat.count,
							angle: i * 20 * (Math.PI / 180),
						};
					});
				}
			}

			glyphData = resultData;
			loading = false;

			setTimeout(() => {
				glyphsAnimated = true;
			}, 300);
		} catch (e) {
			console.error(e);
			error = "Could not load data files.";
			loading = false;
		}
	});

	let xScale = $derived(
		d3
			.scalePoint()
			.domain(visibleOccupations)
			.range([margin.left, width - margin.right])
			.padding(0.5),
	);

	let yScale = $derived(
		d3
			.scalePoint()
			.domain(allAgeLabels)
			.range([height - margin.bottom, margin.top])
			.padding(0.5),
	);

	let rScale = $derived(d3.scaleLinear().domain([0, 5]).range([0, 25]));
</script>

{#if loading}
	<p style="color: #64748b; text-align: center;">Loading data...</p>
{:else if error}
	<p style="color: red; text-align: center;">{error}</p>
{:else}
	<div
		class="chart-container"
		style="display: flex; gap: 24px; align-items: flex-start; max-width: 100%;"
	>
		<div class="scroll-wrap" style="flex: 1; min-width: 0; margin: 0;">
			<svg {width} {height}>
				<g>
					{#each allAgeLabels as yLabel}
						<line
							x1={margin.left}
							x2={width - margin.right}
							y1={yScale(yLabel)}
							y2={yScale(yLabel)}
							stroke="#1e2530"
							stroke-width="1"
							stroke-dasharray="2,2"
						/>
						<text
							x={margin.left - 15}
							y={yScale(yLabel)}
							text-anchor="end"
							dominant-baseline="middle"
							fill="#94a3b8"
							font-size="13px">{yLabel}</text
						>
					{/each}

					{#each visibleOccupations as xLabel}
						<line
							x1={xScale(xLabel)}
							x2={xScale(xLabel)}
							y1={margin.top}
							y2={height - margin.bottom}
							stroke="#1e2530"
							stroke-width="1"
							stroke-dasharray="2,2"
						/>
						<text
							x={xScale(xLabel)}
							y={height - margin.bottom + 20}
							text-anchor="end"
							transform={`rotate(-45, ${xScale(xLabel)}, ${height - margin.bottom + 20})`}
							fill="#94a3b8"
							font-size="13px">{xLabel}</text
						>
					{/each}

					{#each visibleOccupations as occ}
						{#each allAgeLabels as ageLabel}
							{#if glyphData[`${occ}|${ageLabel}`]}
								{@const isHovered =
									hoveredKey === `${occ}|${ageLabel}`}
								<g
									transform="translate({xScale(occ)}, {yScale(
										ageLabel,
									)})"
									role="button"
									tabindex="0"
									onmousemove={(e) =>
										handleMouseMove(e, occ, ageLabel)}
									onmouseleave={handleMouseLeave}
									onclick={() =>
										handleGlyphClick(occ, ageLabel)}
									onkeydown={(e) =>
										e.key === "Enter" &&
										handleGlyphClick(occ, ageLabel)}
									style="cursor: pointer;"
								>
									<circle r="25" fill="transparent" />
									<circle
										r={isHovered ? 3 : 2}
										fill={isHovered ? "#e2e8f0" : "#2d3748"}
										style="transition: r 0.2s ease;"
									/>

									{#each glyphData[`${occ}|${ageLabel}`] as genreStat}
										{#if genreStat.avg > 0}
											<line
												x1="0"
												y1="0"
												x2={glyphsAnimated
													? Math.cos(
															genreStat.angle,
														) *
														rScale(genreStat.avg) *
														(isHovered ? 1.5 : 1)
													: 0}
												y2={glyphsAnimated
													? Math.sin(
															genreStat.angle,
														) *
														rScale(genreStat.avg) *
														(isHovered ? 1.5 : 1)
													: 0}
												stroke={genreColors[
													genreStat.genre
												]}
												stroke-width={isHovered
													? 3.5
													: 2.5}
												stroke-linecap="round"
												style="transition: x2 0.6s ease-out, y2 0.6s ease-out, stroke-width 0.2s ease;"
											/>
										{/if}
									{/each}
								</g>
							{/if}
						{/each}
					{/each}
				</g>
			</svg>
		</div>

		<div
			class="legend-wrap"
			style="flex: 0 0 350px; padding: 24px 10px; border: 1px solid #1e2530; border-radius: 8px; background: transparent; text-align: center; position: sticky; top: 20px;"
		>
			<h3
				style="color: #e2e8f0; margin-top: 0; margin-bottom: 24px; font-size: 16px; font-weight: 600;"
			>
				Genre Key
			</h3>
			<svg
				width="330"
				height="300"
				style="overflow: visible; margin: 0 auto; display: block;"
			>
				<g transform="translate(165, 150)">
					<circle r="3" fill="#94a3b8" />
					{#each genreNames as genre, i}
						{@const angle = i * 20 * (Math.PI / 180)}
						{@const r1 = 30}
						{@const r2 = 35 + Math.abs(Math.sin(angle)) * 80}
						{@const x1 = Math.cos(angle) * r1}
						{@const y1 = Math.sin(angle) * r1}
						{@const x2 = Math.cos(angle) * r2}
						{@const y2 = Math.sin(angle) * r2}
						{@const isRight = Math.cos(angle) >= 0}
						{@const x3 = x2 + (isRight ? 15 : -15)}
						{@const y3 = y2}

						<line
							x1="0"
							y1="0"
							x2={x1}
							y2={y1}
							stroke={genreColors[genre]}
							stroke-width="3"
							stroke-linecap="round"
						/>
						<polyline
							points="{x1},{y1} {x2},{y2} {x3},{y3}"
							fill="none"
							stroke="#475569"
							stroke-width="1.5"
						/>
						<text
							x={x3 + (isRight ? 6 : -6)}
							y={y3}
							fill="#94a3b8"
							font-size="12px"
							dominant-baseline="middle"
							text-anchor={isRight ? "start" : "end"}
						>
							{genre}
						</text>
					{/each}
				</g>
			</svg>
		</div>
	</div>

	<div
		class="occupation-key"
		style="margin-top: 24px; padding: 16px; border: 1px solid #1e2530; border-radius: 8px; background: transparent;"
	>
		<h3
			style="color: #e2e8f0; margin-top: 0; margin-bottom: 16px; font-size: 16px; font-weight: 600;"
		>
			Filter Occupations
		</h3>
		<div style="display: flex; flex-wrap: wrap; gap: 8px;">
			{#each allOccupations as occ}
				<button
					class="occ-btn"
					class:active={activeOccupations[occ] !== false}
					onclick={() =>
						(activeOccupations[occ] =
							activeOccupations[occ] === false ? true : false)}
				>
					{occ}
				</button>
			{/each}
		</div>
	</div>

	{#if tooltipVisible && tooltipData}
		<div class="tooltip" style="left: {tooltipX}px; top: {tooltipY}px">
			<strong style="display: block; margin-bottom: 6px;"
				>{tooltipData.occ} ({tooltipData.ageLabel})</strong
			>
			{#each tooltipData.lines as row}
				<div style="color: {genreColors[row.genre]}; margin: 2px 0;">
					{row.genre}: {row.avg.toFixed(2)}
				</div>
			{/each}
		</div>
	{/if}

	{#if selectedGlyphData}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div class="modal-backdrop" onclick={closeBox}>
			<div class="modal-box" onclick={(e) => e.stopPropagation()}>
				<div class="modal-header">
					<h2>
						{selectedGlyphData.occ} ({selectedGlyphData.ageLabel})
					</h2>
					<button onclick={closeBox}>&times;</button>
				</div>
				<div class="modal-content">
					<div class="modal-glyph">
						<svg width="240" height="240">
							<g transform="translate(120, 120)">
								{#if selectedGlyphData.overallAvg > 0}
									<circle 
										r={rScale(selectedGlyphData.overallAvg) * 3 + 2.5} 
										fill="none" 
										stroke="#475569" 
										stroke-width="1.5" 
										stroke-dasharray="4 4" 
									/>
								{/if}
								<circle r="6" fill="#e2e8f0" />
								{#each glyphData[`${selectedGlyphData.occ}|${selectedGlyphData.ageLabel}`] as genreStat}
									{#if genreStat.avg > 0}
										<line
											x1="0"
											y1="0"
											x2={Math.cos(genreStat.angle) *
												rScale(genreStat.avg) *
												3}
											y2={Math.sin(genreStat.angle) *
												rScale(genreStat.avg) *
												3}
											stroke={genreColors[
												genreStat.genre
											]}
											stroke-width="5"
											stroke-linecap="round"
										/>
									{/if}
								{/each}
							</g>
						</svg>
					</div>
					<div class="modal-labels">
						{#each selectedGlyphData.lines as row}
							<div
								class="modal-label-row"
								style="color: {genreColors[row.genre]}"
							>
								<span class="genre-name">{row.genre}</span>
								<span class="genre-value"
									>{row.avg.toFixed(2)} ({row.count} ratings)</span
								>
							</div>
						{/each}
					</div>
				</div>
				{#if selectedGlyphData.overallAvg > 0}
					<div style="margin-top: 24px; text-align: center; font-size: 13px; color: #94a3b8; font-style: italic;">
						* The faint dotted circle indicates the overall average rating ({selectedGlyphData.overallAvg.toFixed(2)}) for this group.
					</div>
				{/if}
			</div>
		</div>
	{/if}
{/if}

<style>
	.scroll-wrap {
		width: 100%;
		max-width: 100%;
		overflow-x: auto;
		border: 1px solid #1e2530;
		border-radius: 8px;
		background: transparent;
		padding: 10px 0;
		box-sizing: border-box;
	}

	.scroll-wrap::-webkit-scrollbar {
		height: 8px;
	}
	.scroll-wrap::-webkit-scrollbar-track {
		background: #0d1117;
		border-radius: 4px;
	}
	.scroll-wrap::-webkit-scrollbar-thumb {
		background: #2d3748;
		border-radius: 4px;
	}

	.tooltip {
		position: fixed;
		background: #1e2530;
		border: 1px solid #2d3748;
		border-radius: 5px;
		padding: 10px 14px;
		pointer-events: none;
		font-size: 13px;
		color: #e2e8f0;
		z-index: 100;
		box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
	}

	.modal-backdrop {
		position: fixed;
		top: 0;
		left: 0;
		width: 100vw;
		height: 100vh;
		background: rgba(0, 0, 0, 0.7);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 1000;
	}
	.modal-box {
		background: #1e2530;
		border: 1px solid #2d3748;
		border-radius: 12px;
		padding: 32px;
		width: 600px;
		max-width: 90vw;
		max-height: 90vh;
		display: flex;
		flex-direction: column;
		box-shadow: 0 10px 25px rgba(0, 0, 0, 0.5);
	}
	.modal-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 32px;
		color: #e2e8f0;
	}
	.modal-header h2 {
		margin: 0;
		font-size: 20px;
		font-weight: 600;
	}
	.modal-header button {
		background: transparent;
		border: none;
		color: #94a3b8;
		font-size: 28px;
		cursor: pointer;
		line-height: 1;
		padding: 0;
		display: flex;
		align-items: center;
	}
	.modal-header button:hover {
		color: #e2e8f0;
	}
	.modal-content {
		display: flex;
		align-items: flex-start;
		gap: 40px;
	}
	.modal-glyph {
		flex-shrink: 0;
		display: flex;
		justify-content: center;
		align-items: center;
		background: #161b22;
		border-radius: 50%;
		border: 1px solid #2d3748;
		width: 240px;
		height: 240px;
	}
	.modal-labels {
		flex-grow: 1;
		display: flex;
		flex-direction: column;
		gap: 8px;
		max-height: 240px;
		overflow-y: auto;
		padding-right: 12px;
	}
	.modal-labels::-webkit-scrollbar {
		width: 6px;
	}
	.modal-labels::-webkit-scrollbar-track {
		background: #0d1117;
		border-radius: 3px;
	}
	.modal-labels::-webkit-scrollbar-thumb {
		background: #2d3748;
		border-radius: 3px;
	}
	.modal-label-row {
		display: flex;
		justify-content: space-between;
		font-size: 14px;
		font-weight: 500;
		background: rgba(255, 255, 255, 0.05);
		padding: 8px 12px;
		border-radius: 6px;
	}

	.occ-btn {
		background: #1e2530;
		border: 1px solid #2d3748;
		color: #94a3b8;
		padding: 6px 12px;
		border-radius: 20px;
		font-size: 13px;
		cursor: pointer;
		transition: all 0.2s ease;
	}
	.occ-btn:hover {
		background: #2d3748;
		color: #e2e8f0;
	}
	.occ-btn.active {
		background: #3b82f6;
		border-color: #2d3748;
		color: white;
	}
</style>
