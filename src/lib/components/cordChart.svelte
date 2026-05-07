<script>
// @ts-nocheck
	import { onMount } from 'svelte'
  	import * as d3 from 'd3'
	import { genreColors } from '$lib/genreColors'
	import { Scroll } from '$lib'

	let loading = $state(true);
	let error = $state("");
	let userRatings = {};
	let userMap = $state([]);
	let allOccupations = $state([]);
	let allRatings = $state([]);
	let clickedCoordinates = $state(new Set());
	
	const demographics = [
		'rounded average rating', 'occupation', 'age', 'gender'
	];

	const genreNames = [
		'unknown', 'Action', 'Adventure', 'Animation', "Children's", 'Comedy',
		'Crime', 'Documentary', 'Drama', 'Fantasy', 'Film-Noir', 'Horror',
		'Musical', 'Mystery', 'Romance', 'Sci-Fi', 'Thriller', 'War', 'Western'
	];

	const genders = [
		'Male', 'Female'
	];

	const ratings = [
		'1', '2', '3', '4', '5'
	]

	const ageBuckets = [
		{ label: '<18',   min: 0,  max: 17  },
		{ label: '18-24', min: 18, max: 24  },
		{ label: '25-34', min: 25, max: 34  },
		{ label: '35-44', min: 35, max: 44  },
		{ label: '45-54', min: 45, max: 54  },
		{ label: '55-64', min: 55, max: 64  },
		{ label: '65+',   min: 65, max: 999 },
	];

	const allAgeLabels = ageBuckets.map(b => b.label);
	const margin = { top: 40, right: 60, bottom: 90, left: 60 };
  	let width = $state(1100); 
  	let height = $state(650);

	function coordKey(x, y)
	{
		return String(x) + " " + String(y);
	}

	function circleClick(x, y)
	{
		if(clickedCoordinates.has(coordKey(x, y)))
		{
			clickedCoordinates.delete(coordKey(x, y));
		}
		else
		{
			clickedCoordinates.add(coordKey(x, y));
		}
		
		console.log(clickedCoordinates);
		clickedCoordinates = new Set(clickedCoordinates);
	}

  	onMount(async () => {
		try {
			const base = import.meta.env.BASE_URL;
			const [ratingsText, itemsText, usersText] = await Promise.all([
				fetch(`${base}u.data`).then(r => r.text()),
				fetch(`${base}u.item`).then(r => r.text()),
				fetch(`${base}u.user`).then(r => r.text()),
			]);

			const ratingsRes = await fetch(`${base}u.data`)
			allRatings = ratingsText.trim().split('\n').map(line => {
				const parts = line.split('\t')
				if (userRatings[parseInt(parts[0])] === undefined)
				{
					userRatings[parseInt(parts[0])] = [parseInt(parts[2]), 1]
				}
				else
				{
					userRatings[parseInt(parts[0])][0] += parseInt(parts[2]);
					userRatings[parseInt(parts[0])][1] += 1;
				}
				return {
					userId: parseInt(parts[0]),
					movieId: parseInt(parts[1]),
					rating: parseInt(parts[2])
				}
			})

			let occSet = new Set();
			userMap = usersText.trim().split("\n").map(line => {
				const parts = line.split("|");
				occSet.add(parts[3]);
				return {
					gender: parts[2] === 'M' ? 'Male' : "Female",
					occupation: parts[3],
					age: ageBuckets.find(b => parseInt(parts[1]) >= b.min && parseInt(parts[1]) <= b.max)?.label ?? "Unknown" ,
					avgRating: Math.round(userRatings[parseInt(parts[0])][0] / userRatings[parseInt(parts[0])][1]),
					id: parseInt(parts[0])
				}
			});

			const occupationsArray = Array.from(occSet).sort();
			allOccupations = occupationsArray

			loading = false;

		} catch (e) {
			console.error(e);
			error = "Could not load data files.";
			loading = false;
		}
	});

	let xScaleDemographic = $derived(
		d3.scalePoint().domain(demographics).range([width - margin.right, margin.left]).padding(0.1)
	);
	
	let yScaleGender = $derived(
		d3.scalePoint().domain(genders).range([height - margin.bottom, margin.top]).padding(0.2)
	);
	let yScaleAge = $derived(
		d3.scalePoint().domain(allAgeLabels).range([height - margin.bottom, margin.top]).padding(0.2)
	);
	let yScaleRatings = $derived(
		d3.scalePoint().domain(ratings).range([height - margin.bottom, margin.top]).padding(0.2)
	);
	let selectedOccupations = $state(['student', 'educator', 'engineer', 'programmer', 'healthcare', 'artist']);

	let filteredUserMap = $derived(userMap.filter(u => selectedOccupations.includes(u.occupation)));

	let yScaleOccupation = $derived(
		d3.scalePoint().domain(selectedOccupations).range([height - margin.bottom, margin.top]).padding(0)
	);

	function toggleOccupation(occ) {
		if (selectedOccupations.includes(occ)) {
			if (selectedOccupations.length > 1) {
				selectedOccupations = selectedOccupations.filter(o => o !== occ);
			}
		} else {
			selectedOccupations = [...selectedOccupations, occ];
		}
	}
</script>

{#if loading}
	<p style="color: #64748b; text-align: center;">Loading data...</p>
{:else if error}
	<p style="color: red; text-align: center;">{error}</p>
{:else}
<div class="cord-layout">
	<div class="glass-panel" style="padding: 20px; width: 100%; max-width: 1400px; margin: 0 auto; box-sizing: border-box;">
    <div class="scroll-wrap" style="overflow-x: auto; overflow-y: visible;">
      <svg viewBox="0 0 {width} {height}" style="width: 100%; height: auto; min-width: 1000px; display: block;">
        <g>
				{#each filteredUserMap as user}
					<path
						d={"M " + xScaleDemographic('gender') + " " + (yScaleGender(user.gender))
						+ " L " + xScaleDemographic('age') + " " + (yScaleAge(user.age))
						+ " L " + xScaleDemographic('occupation') + " " + (yScaleOccupation(user.occupation))
						+ " L " + xScaleDemographic('rounded average rating') + " " + (yScaleRatings(String(user.avgRating)))}
						stroke={clickedCoordinates.has(coordKey(xScaleDemographic('gender'), yScaleGender(user.gender))) ||
								clickedCoordinates.has(coordKey(xScaleDemographic('age'), yScaleAge(user.age))) || 
								clickedCoordinates.has(coordKey(xScaleDemographic('occupation'), yScaleOccupation(user.occupation))) || 
								clickedCoordinates.has(coordKey(xScaleDemographic('rounded average rating'), yScaleRatings(String(user.avgRating)))) 
								? '#c6cf5d' : user.gender === 'Male' ? "#00aaff" : "#f700a9"}
						stroke-width="2"
						stroke-opacity={clickedCoordinates.has(coordKey(xScaleDemographic('gender'), yScaleGender(user.gender))) ||
										clickedCoordinates.has(coordKey(xScaleDemographic('age'), yScaleAge(user.age))) || 
										clickedCoordinates.has(coordKey(xScaleDemographic('occupation'), yScaleOccupation(user.occupation))) || 
										clickedCoordinates.has(coordKey(xScaleDemographic('rounded average rating'), yScaleRatings(String(user.avgRating)))) 
										? "1" : "0.08"}
						fill="none"
					/>
				{/each}
				{#each demographics as xLabel}
					<line x1={xScaleDemographic(xLabel)} x2={xScaleDemographic(xLabel)} y1="0" y2={height - 13} stroke="#808080" stroke-width="1" />
					<text x={xScaleDemographic(xLabel)} y={height - 10} text-anchor="middle" dominant-baseline="middle" fill="#94a3b8" font-size="20px">{xLabel}</text>
				{/each}
				{#each genders as yLabel}
					<circle
						cx={xScaleDemographic('gender')}
						cy={yScaleGender(yLabel)}
						r="17"
						onclick={() => circleClick(xScaleDemographic('gender'), yScaleGender(yLabel))}
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('gender'), yScaleGender(yLabel))) ? '#c6cf5d' : '#0f6e56'}
						stroke="black"
						stroke-width="2"
						opacity="1"
					/>
					<text
						x={xScaleDemographic('gender')}
						y={yScaleGender(yLabel)}
						onclick={() => circleClick(xScaleDemographic('gender'), yScaleGender(yLabel))}
						text-anchor="middle"
						dominant-baseline="middle"
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('gender'), yScaleGender(yLabel))) ? "#000000": "#e2e8f0"}
						font-size="10px">
						{yLabel}
					</text>
				{/each}
				{#each allAgeLabels as yLabel}
					<circle
						cx={xScaleDemographic('age')}
						cy={yScaleAge(yLabel)}
						r="17"
						onclick={() => circleClick(xScaleDemographic('age'), yScaleAge(yLabel))}
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('age'), yScaleAge(yLabel))) ? '#c6cf5d' : '#0f6e56'}
						stroke="black"
						stroke-width="2"
						opacity="1"
					/>
					<text
						x={xScaleDemographic('age')}
						y={yScaleAge(yLabel)}
						onclick={() => circleClick(xScaleDemographic('age'), yScaleAge(yLabel))}
						text-anchor="middle"
						dominant-baseline="middle"
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('age'), yScaleAge(yLabel))) ? "#000000": "#e2e8f0"}
						font-size="10px">
						{yLabel}
					</text>
				{/each}
				{#each selectedOccupations as yLabel}
					<circle
						cx={xScaleDemographic('occupation')}
						cy={yScaleOccupation(yLabel)}
						r="12"
						onclick={() => circleClick(xScaleDemographic('occupation'), yScaleOccupation(yLabel))}
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('occupation'), yScaleOccupation(yLabel))) ? '#c6cf5d' : '#0f6e56'}
						stroke="black"
						stroke-width="2"
						opacity="1"
					/>
					<text
						x={xScaleDemographic('occupation')}
						y={yScaleOccupation(yLabel)}
						onclick={() => circleClick(xScaleDemographic('occupation'), yScaleOccupation(yLabel))}
						text-anchor="middle"
						dominant-baseline="middle"
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('occupation'), yScaleOccupation(yLabel))) ? "#000000": "#e2e8f0"}
						font-size="10px">
						{yLabel}
					</text>
				{/each}
				{#each ratings as yLabel}
					<circle
						cx={xScaleDemographic('rounded average rating')}
						cy={yScaleRatings(yLabel)}
						r="17"
						onclick={() => circleClick(xScaleDemographic('rounded average rating'), yScaleRatings(yLabel))}
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('rounded average rating'), yScaleRatings(yLabel))) ? '#c6cf5d' : '#0f6e56'}
						stroke="black"
						stroke-width="2"
						opacity="1"
					/>
					<text
						x={xScaleDemographic('rounded average rating')}
						y={yScaleRatings(yLabel)}
						onclick={() => circleClick(xScaleDemographic('rounded average rating'), yScaleRatings(yLabel))}
						text-anchor="middle"
						dominant-baseline="middle"
						fill={clickedCoordinates.has(coordKey(xScaleDemographic('rounded average rating'), yScaleRatings(yLabel))) ? "#000000": "#e2e8f0"}
						font-size="10px">
						{yLabel}
					</text>
				{/each}
			</g>
		</svg>
	</div>
	</div>
	<div class="filter-section">
			<span class="filter-label">Occupations:</span>
			<div class="pill-container">
				{#each allOccupations as occ}
					<button 
						class="simple-pill {selectedOccupations.includes(occ) ? 'active' : ''}"
						onclick={() => toggleOccupation(occ)}
					>
						{occ}
					</button>
				{/each}
			</div>
		</div>

	</div>
{/if}

<style>
	.scroll-wrap {
		width: 100%;
		max-width: 100%;
		overflow-x: auto;
		padding: 10px 0;
		box-sizing: border-box;
	}

	.scroll-wrap::-webkit-scrollbar { height: 8px; }
	.scroll-wrap::-webkit-scrollbar-track { background: #0d1117; border-radius: 4px; }
	.scroll-wrap::-webkit-scrollbar-thumb { background: #2d3748; border-radius: 4px; }
	.cord-layout {
		display: flex;
		align-items: flex-start;
		gap: 24px;
		max-width: 1400px;
		margin: 0 auto 40px auto;
	}

	.filter-section {
		position: sticky;
		top: 220px; 
		z-index: 50; 
		width: 200px;
		flex-shrink: 0;
	}

	.filter-label {
		font-size: 13px;
		font-weight: 600;
		color: #64748b;
		text-transform: uppercase;
		letter-spacing: 0.05em;
		margin-bottom: 12px; 
		display: block;
		padding-left: 4px;
	}

	.pill-container {
		display: flex;
		flex-direction: column; 
		gap: 6px; 
		max-height: 55vh;
		overflow-y: auto;
		padding-right: 8px;
	}

	.pill-container::-webkit-scrollbar { 
		width: 4px; 
	}
	.pill-container::-webkit-scrollbar-track { 
		background: transparent; 
	}
	.pill-container::-webkit-scrollbar-thumb { 
		background: rgba(255, 255, 255, 0.1); 
		border-radius: 4px; 
	}
	.pill-container::-webkit-scrollbar-thumb:hover { 
		background: rgba(255, 255, 255, 0.2); 
	}

	.simple-pill {
		width: 100%;
		text-align: left;
		background: #1e2530;
		border: 1px solid #2d3748;
		border-radius: 20px;
		padding: 6px 12px;
		font-size: 13px;
		color: #94a3b8;
		cursor: pointer;
		transition: all 0.2s ease;
	}

	.simple-pill:hover {
		background: #2d3748;
		color: #e2e8f0;
	}

	.simple-pill.active {
		background: #3b82f6;
		border-color: #2d3748;
		color: white;
	}

</style>