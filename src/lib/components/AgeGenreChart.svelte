<script>
// @ts-nocheck
	import { onMount } from 'svelte'
  import * as d3 from "d3"
  import { genreColors } from '$lib/genreColors'

	const genreNames = [
		"Action", "Adventure", "Animation", "Children's", "Comedy",
		"Crime", "Documentary", "Drama", "Fantasy", "Film-Noir", "Horror",
		"Musical", "Mystery", "Romance", "Sci-Fi", "Thriller", "War", "Western"
	]

	const ageBuckets = [
		{ label: "<18",   min: 0,  max: 17  },
		{ label: "18-24", min: 18, max: 24  },
		{ label: "25-34", min: 25, max: 34  },
		{ label: "35-44", min: 35, max: 44  },
		{ label: "45-54", min: 45, max: 54  },
		{ label: "55-64", min: 55, max: 64  },
		{ label: "65+",   min: 65, max: 999 },
	]

	let loading = $state(true);
	let error = $state("");
	let genreData = $state({});
	let selected = $state(["Film-Noir", "Horror", "Fantasy"]);

	let tooltipVisible = $state(false);
	let tooltipX = $state(0);
	let tooltipY = $state(0);
	let tooltipAge = $state("");
	let tooltipRows = $state([]);
	let chartWrap;

	const margin = { top: 20, right: 30, bottom: 40, left: 50 }
	const width = 960 - margin.left - margin.right
	const height = 460 - margin.top - margin.bottom

	onMount(async () => {
		try {
      const base = import.meta.env.BASE_URL

			const ratingsRes = await fetch(`${base}u.data`)
			const ratingsText = await ratingsRes.text()

			const itemsRes = await fetch(`${base}u.item`)
			const itemsText = await itemsRes.text()

			const usersRes = await fetch(`${base}u.user`)
			const usersText = await usersRes.text()

			let userMap = {}
			usersText.trim().split('\n').forEach(line => {
				const parts = line.split('|')
				const id = parseInt(parts[0])
				userMap[id] = { age: parseInt(parts[1]) }
			})

			let movieMap = {}
			itemsText.trim().split('\n').forEach(line => {
				const parts = line.split('|')
				const id = parseInt(parts[0])
				const genres = []
				for (let i = 0; i < genreNames.length; i++) {
					if (parts[5 + i] === '1') {
						genres.push(genreNames[i])
					}
				}
				movieMap[id] = genres
			})

			let acc = {}
			genreNames.forEach(g => {
				acc[g] = {}
				ageBuckets.forEach(b => {
					acc[g][b.label] = { sum: 0, count: 0 }
				})
			})

			ratingsText.trim().split('\n').forEach(line => {
				const parts = line.split('\t')
				const userId = parseInt(parts[0])
				const movieId = parseInt(parts[1])
				const rating = parseInt(parts[2])

				const user = userMap[userId]
				const movieGenres = movieMap[movieId]
				if (!user || !movieGenres) return

				let bucket = null
				for (let i = 0; i < ageBuckets.length; i++) {
					if (user.age >= ageBuckets[i].min && user.age <= ageBuckets[i].max) {
						bucket = ageBuckets[i]
						break
					}
				}
				if (!bucket) return

				movieGenres.forEach(g => {
					acc[g][bucket.label].sum += rating
					acc[g][bucket.label].count += 1
				})
			})

			let result = {}
			genreNames.forEach(g => {
				result[g] = ageBuckets.map(b => {
					const entry = acc[g][b.label]
					return {
						age: b.label,
						avg: entry.count > 0 ? parseFloat((entry.sum / entry.count).toFixed(2)) : null,
						count: entry.count
					}
				})
			})
			genreData = result

			loading = false
		} catch (e) {
			error = "couldn't load data files - make sure u.data, u.item, and u.user are in the static folder"
			loading = false
		}
	});

	let xScale = $derived(
		d3.scalePoint()
			.domain(ageBuckets.map(b => b.label))
			.range([0, width])
			.padding(0.3)
	);

	let yScale = $derived(
		d3.scaleLinear()
			.domain([2.5, 4.5])
			.range([height, 0])
	);

	let yTicks = $derived(yScale.ticks(5));

	function getLinePath(genre) {
		const points = genreData[genre]
		if (!points) return ""
		let d = ""
		let started = false
		points.forEach(p => {
			if (p.avg === null) return
			const x = xScale(p.age)
			const y = yScale(p.avg)
			if (!started) {
				d += `M ${x} ${y}`
				started = true
			} else {
				d += ` L ${x} ${y}`
			}
		})
		return d
	}

	function getDots(genre) {
		const points = genreData[genre]
		if (!points) return []
		return points.filter(p => p.avg !== null).map(p => ({
			cx: xScale(p.age),
			cy: yScale(p.avg),
			r: Math.sqrt(p.count) * 0.25,
			age: p.age,
			avg: p.avg,
			count: p.count
		}))
	}

	// animate the line drawing
	function animateLine(path) {
		const len = path.getTotalLength()
		path.style.strokeDasharray = len
		path.style.strokeDashoffset = len
		path.getBoundingClientRect()
		path.style.transition = "stroke-dashoffset 0.8s ease-out"
		path.style.strokeDashoffset = "0"
	}

	function toggleGenre(genre) {
		if (selected.includes(genre)) {
			selected = selected.filter(g => g !== genre)
		} else {
			selected = [...selected, genre]
		}
	}

	function handleMouseMove(event, ageLabel) {
		const rect = chartWrap.getBoundingClientRect()
		tooltipX = event.clientX - rect.left + 12
		tooltipY = event.clientY - rect.top - 10
		tooltipAge = ageLabel
		tooltipVisible = true

		tooltipRows = selected.map(genre => {
			const point = genreData[genre].find(d => d.age === ageLabel)
			if (!point || point.avg === null) return null
			return { genre, color: genreColors[genre], avg: point.avg, count: point.count }
		}).filter(r => r !== null)
	}

	function handleMouseLeave() {
		tooltipVisible = false
	}
</script>

{#if loading}
	<p style="color: #64748b">Loading data...</p>
{:else if error}
	<p style="color: red;">{error}</p>
{:else}
	<p class="instructions">Toggle genres to compare. Dot size reflects how many ratings that point is based on — smaller dots mean fewer users in that age group.</p>

	<div class="toggles">
		{#each genreNames as genre}
			<button
				onclick={() => toggleGenre(genre)}
				style="border-color: {selected.includes(genre) ? genreColors[genre] : '#2d3748'};
							 color: {selected.includes(genre) ? genreColors[genre] : '#64748b'}"
			>
				{genre}
			</button>
		{/each}
	</div>

	<div class="chart-wrap" bind:this={chartWrap}>
		<svg width={width + margin.left + margin.right} height={height + margin.top + margin.bottom}>
			<g transform="translate({margin.left}, {margin.top})">

				{#each yTicks as tick}
					<line
						x1={0} x2={width}
						y1={yScale(tick)} y2={yScale(tick)}
						stroke="#1e2530"
						stroke-dasharray="3,3"
					/>
				{/each}

				{#each ageBuckets as bucket}
					<text
						x={xScale(bucket.label)}
						y={height + 20}
						text-anchor="middle"
						fill="#64748b"
						font-size="13px"
					>{bucket.label}</text>
				{/each}

				{#each yTicks as tick}
					<text
						x={-10}
						y={yScale(tick)}
						text-anchor="end"
						dominant-baseline="middle"
						fill="#64748b"
						font-size="13px"
					>{tick}</text>
				{/each}

				<line x1={0} x2={width} y1={height} y2={height} stroke="#2d3748" />
				<line x1={0} x2={0} y1={0} y2={height} stroke="#2d3748" />

				{#each selected as genre}
					<path
						d={getLinePath(genre)}
						fill="none"
						stroke={genreColors[genre]}
						stroke-width="2.5"
						use:animateLine
					/>
					{#each getDots(genre) as dot}
						<circle
							cx={dot.cx}
							cy={dot.cy}
							r={dot.r}
							fill={genreColors[genre]}
							stroke="#0d1117"
							stroke-width="1.5"
						/>
					{/each}
				{/each}

				{#each ageBuckets as bucket}
					<rect
						x={xScale(bucket.label) - 30}
						y={0}
						width={60}
						height={height}
						fill="transparent"
						onmousemove={(e) => handleMouseMove(e, bucket.label)}
						onmouseleave={handleMouseLeave}
					/>
				{/each}
			</g>
		</svg>

		{#if tooltipVisible}
			<div class="tooltip" style="left: {tooltipX}px; top: {tooltipY}px">
				<strong>{tooltipAge}</strong>
				{#each tooltipRows as row}
					<p style="color: {row.color}; margin: 4px 0 0 0;">{row.genre}: {row.avg} ({row.count} ratings)</p>
				{/each}
			</div>
		{/if}
	</div>

	<div class="findings">
		<p><strong>What we found:</strong> Film-Noir has the clearest age trend — it goes from about 2.9 for under 18s up to 4.0 for 65+. Horror stays pretty flat across most groups but drops off for the oldest users. Fantasy also trends upward with age, similar to Film-Noir but not as dramatic.</p>
	</div>
{/if}

<style>
	.instructions {
		font-size: 14px;
		color: #4a5568;
		margin-bottom: 14px;
		line-height: 1.6;
		max-width: 700px;
	}

	.toggles {
		display: flex;
		flex-wrap: wrap;
		gap: 8px;
		margin-bottom: 20px;
	}

	button {
		background: none;
		border: 1px solid;
		border-radius: 20px;
		padding: 5px 14px;
		font-size: 13px;
		cursor: pointer;
	}

	.chart-wrap {
		position: relative;
	}

	.tooltip {
		position: absolute;
		background: #1e2530;
		border: 1px solid #2d3748;
		border-radius: 5px;
		padding: 8px 12px;
		pointer-events: none;
		font-size: 13px;
		color: #e2e8f0;
	}

	.findings {
		margin-top: 20px;
	}

	.findings p {
		font-size: 14px;
		color: #64748b;
		line-height: 1.6;
	}
</style>