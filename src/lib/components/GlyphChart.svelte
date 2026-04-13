<script>
	// @ts-nocheck
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import { genreColors } from "$lib/genreColors";

	let loading = $state(true);
	let error = $state("");
	let allOccupations = $state([]);
	let glyphData = $state({});
	let glyphsAnimated = $state(false);
	let hoveredKey = $state(null);

	const ageBuckets = [
		{ label: "Under 25", min: 0, max: 24 },
		{ label: "25-34", min: 25, max: 34 },
		{ label: "35-44", min: 35, max: 44 },
		{ label: "45+", min: 45, max: 120 },
	];

	const allAgeLabels = ageBuckets.map(b => b.label);

	const genreNames = [
		"Action", "Adventure", "Animation", "Children's", "Comedy",
		"Crime", "Documentary", "Drama", "Fantasy", "Film-Noir", "Horror",
		"Musical", "Mystery", "Romance", "Sci-Fi", "Thriller", "War", "Western",
	];

	const margin = { top: 40, right: 30, bottom: 80, left: 80 };
	let width = $state(1200);
	let height = $state(450);

	let tooltipVisible = $state(false);
	let tooltipX = $state(0);
	let tooltipY = $state(0);
	let tooltipData = $state(null);

	function handleMouseMove(event, occ, ageLabel) {
		tooltipX = event.clientX + 15;
		tooltipY = event.clientY + 15;
		const stats = glyphData[`${occ}|${ageLabel}`];
		tooltipData = { occ, ageLabel, lines: stats.filter(s => s.avg > 0).sort((a, b) => b.avg - a.avg) };
		tooltipVisible = true;
		hoveredKey = `${occ}|${ageLabel}`;
	}

	function handleMouseLeave() {
		tooltipVisible = false;
		hoveredKey = null;
	}

	onMount(async () => {
		try {
			const base = import.meta.env.BASE_URL;
			const [ratingsText, itemsText, usersText] = await Promise.all([
				fetch(`${base}u.data`).then(r => r.text()),
				fetch(`${base}u.item`).then(r => r.text()),
				fetch(`${base}u.user`).then(r => r.text()),
			]);

			let occSet = new Set();
			let userMap = {};
			usersText.trim().split("\n").forEach(line => {
				const parts = line.split("|");
				const id = parseInt(parts[0]);
				const age = parseInt(parts[1]);
				const occupation = parts[3];
				const bucket = ageBuckets.find(b => age >= b.min && age <= b.max);
				occSet.add(occupation);
				userMap[id] = { occupation, ageLabel: bucket?.label ?? "Unknown" };
			});

			const occupationsArray = Array.from(occSet).sort();
			allOccupations = occupationsArray;

			let movieMap = {};
			itemsText.trim().split("\n").forEach(line => {
				const parts = line.split("|");
				const genres = genreNames.filter((_, i) => parts[6 + i] === "1");
				movieMap[parseInt(parts[0])] = genres;
			});

			let acc = {};
			for (let occ of occupationsArray) {
				for (let ageLabel of allAgeLabels) {
					const key = `${occ}|${ageLabel}`;
					acc[key] = {};
					for (let genre of genreNames) acc[key][genre] = { sum: 0, count: 0 };
				}
			}

			ratingsText.trim().split("\n").forEach(line => {
				const parts = line.split("\t");
				const user = userMap[parseInt(parts[0])];
				const genres = movieMap[parseInt(parts[1])];
				const rating = parseInt(parts[2]);
				if (!user || !genres) return;
				const key = `${user.occupation}|${user.ageLabel}`;
				if (!acc[key]) return;
				genres.forEach(genre => {
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
						return { genre, avg: stat.count > 0 ? stat.sum / stat.count : 0, angle: i * 20 * (Math.PI / 180) };
					});
				}
			}

			glyphData = resultData;
			width = allOccupations.length * 80 + margin.left + margin.right;
			loading = false;

			setTimeout(() => { glyphsAnimated = true }, 300);
		} catch (e) {
			console.error(e);
			error = "Could not load data files.";
			loading = false;
		}
	});

	let xScale = $derived(
		d3.scalePoint().domain(allOccupations).range([margin.left, width - margin.right]).padding(0.5)
	);

	let yScale = $derived(
		d3.scalePoint().domain(allAgeLabels).range([height - margin.bottom, margin.top]).padding(0.5)
	);

	let rScale = $derived(d3.scaleLinear().domain([0, 5]).range([0, 25]));
</script>

{#if loading}
	<p style="color: #64748b; text-align: center;">Loading data...</p>
{:else if error}
	<p style="color: red; text-align: center;">{error}</p>
{:else}
	<div class="scroll-wrap">
		<svg {width} {height}>
			<g>
				{#each allAgeLabels as yLabel}
					<line x1={margin.left} x2={width - margin.right} y1={yScale(yLabel)} y2={yScale(yLabel)} stroke="#1e2530" stroke-width="1" stroke-dasharray="2,2" />
					<text x={margin.left - 15} y={yScale(yLabel)} text-anchor="end" dominant-baseline="middle" fill="#94a3b8" font-size="13px">{yLabel}</text>
				{/each}

				{#each allOccupations as xLabel}
					<line x1={xScale(xLabel)} x2={xScale(xLabel)} y1={margin.top} y2={height - margin.bottom} stroke="#1e2530" stroke-width="1" stroke-dasharray="2,2" />
					<text x={xScale(xLabel)} y={height - margin.bottom + 20} text-anchor="end" transform={`rotate(-45, ${xScale(xLabel)}, ${height - margin.bottom + 20})`} fill="#94a3b8" font-size="13px">{xLabel}</text>
				{/each}

				{#each allOccupations as occ}
					{#each allAgeLabels as ageLabel}
						{#if glyphData[`${occ}|${ageLabel}`]}
							{@const isHovered = hoveredKey === `${occ}|${ageLabel}`}
							<g
								transform="translate({xScale(occ)}, {yScale(ageLabel)})"
								role="img"
								onmousemove={(e) => handleMouseMove(e, occ, ageLabel)}
								onmouseleave={handleMouseLeave}
								style="cursor: pointer;"
							>
								<circle r="25" fill="transparent" />
								<circle r={isHovered ? 3 : 2} fill={isHovered ? "#e2e8f0" : "#2d3748"} style="transition: r 0.2s ease;" />

								{#each glyphData[`${occ}|${ageLabel}`] as genreStat}
									{#if genreStat.avg > 0}
										<line
											x1="0" y1="0"
											x2={glyphsAnimated ? Math.cos(genreStat.angle) * rScale(genreStat.avg) * (isHovered ? 1.5 : 1) : 0}
											y2={glyphsAnimated ? Math.sin(genreStat.angle) * rScale(genreStat.avg) * (isHovered ? 1.5 : 1) : 0}
											stroke={genreColors[genreStat.genre]}
											stroke-width={isHovered ? 3.5 : 2.5}
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

	{#if tooltipVisible && tooltipData}
		<div class="tooltip" style="left: {tooltipX}px; top: {tooltipY}px">
			<strong style="display: block; margin-bottom: 6px;">{tooltipData.occ} ({tooltipData.ageLabel})</strong>
			{#each tooltipData.lines as row}
				<div style="color: {genreColors[row.genre]}; margin: 2px 0;">
					{row.genre}: {row.avg.toFixed(2)}
				</div>
			{/each}
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

	.scroll-wrap::-webkit-scrollbar { height: 8px; }
	.scroll-wrap::-webkit-scrollbar-track { background: #0d1117; border-radius: 4px; }
	.scroll-wrap::-webkit-scrollbar-thumb { background: #2d3748; border-radius: 4px; }

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
		box-shadow: 0 4px 6px rgba(0,0,0,0.3);
	}
</style>