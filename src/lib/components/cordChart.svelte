<script>
// @ts-nocheck
	import { onMount } from 'svelte'
  	import * as d3 from 'd3'
	import { genreColors } from '$lib/genreColors'
	import { Scroll } from '$lib'

	let loading = $state(true);
	let error = $state("");
	let userMap = {};
	let allOccupations = $state([]);
	let allRatings = $state([]);
	
	const demographics = [
		'rating', 'occupation', 'age', 'gender'
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

  	const margin = { top: 40, right: 30, bottom: 80, left: 80 };
	let width = $state(2000);
	let height = $state(1200);

  	onMount(async () => {
		try {
			const base = import.meta.env.BASE_URL;
			const [ratingsText, itemsText, usersText] = await Promise.all([
				fetch(`${base}u.data`).then(r => r.text()),
				fetch(`${base}u.item`).then(r => r.text()),
				fetch(`${base}u.user`).then(r => r.text()),
			]);

			let occSet = new Set();
			usersText.trim().split("\n").forEach(line => {
				const parts = line.split("|");
				const id = parseInt(parts[0]);
				const gender = parts[2];
				const age = parseInt(parts[1]);
				const occupation = parts[3];
				const bucket = ageBuckets.find(b => age >= b.min && age <= b.max);
				occSet.add(occupation);
				userMap[id] = {gender: gender === 'M' ? 'Male' : "Female", occupation, ageLabel: bucket?.label ?? "Unknown" };
			});

			const occupationsArray = Array.from(occSet).sort();
			allOccupations = occupationsArray

			const ratingsRes = await fetch(`${base}u.data`)
			allRatings = ratingsText.trim().split('\n').map(line => {
				const parts = line.split('\t')
				return {
					userId: parseInt(parts[0]),
					movieId: parseInt(parts[1]),
					rating: parseInt(parts[2])
				}
			})

			loading = false;

		} catch (e) {
			console.error(e);
			error = "Could not load data files.";
			loading = false;
		}
	});

	let xScaleDemographic = $derived(
		d3.scalePoint().domain(demographics).range([width - margin.right, margin.left]).padding(1)
	);
	let yScaleGender = $derived(
		d3.scalePoint().domain(genders).range([height, 0]).padding(1)
	);
	let yScaleAge = $derived(
		d3.scalePoint().domain(allAgeLabels).range([height, 0]).padding(1)
	);
	let yScaleOccupation = $derived(
		d3.scalePoint().domain(allOccupations).range([height, 0]).padding(1)
	);
	let yScaleRatings = $derived(
		d3.scalePoint().domain(ratings).range([height, 0]).padding(1)
	);
</script>

{#if loading}
	<p style="color: #64748b; text-align: center;">Loading data...</p>
{:else if error}
	<p style="color: red; text-align: center;">{error}</p>
{:else}
	<div class="scroll-wrap">
		<svg {width} {height}>
			<g>
				{#each allRatings as r}
					{#if userMap[r.userId].gender === 'Male'}
						<path
							d={"M " + xScaleDemographic('gender') + " " + (yScaleGender(userMap[r.userId].gender))
							+ " L " + xScaleDemographic('age') + " " + (yScaleAge(userMap[r.userId].ageLabel))
							+ " L " + xScaleDemographic('occupation') + " " + (yScaleOccupation(userMap[r.userId].occupation))
							+ " L " + xScaleDemographic('rating') + " " + (yScaleRatings(String(r.rating)))}
							stroke="#00aaff"
							stroke-width="2"
							stroke-opacity="0.02"
							fill="none"
						/>
					{:else}
						<path
							d={"M " + xScaleDemographic('gender') + " " + (yScaleGender(userMap[r.userId].gender))
							+ " L " + xScaleDemographic('age') + " " + (yScaleAge(userMap[r.userId].ageLabel))
							+ " L " + xScaleDemographic('occupation') + " " + (yScaleOccupation(userMap[r.userId].occupation))
							+ " L " + xScaleDemographic('rating') + " " + (yScaleRatings(String(r.rating)))}
							stroke="#f700a9"
							stroke-width="2"
							stroke-opacity="0.02"
							fill="none"
						/>
					{/if}
				{/each}
				{#each demographics as xLabel}
					<line x1={xScaleDemographic(xLabel)} x2={xScaleDemographic(xLabel)} y1="0" y2={height - 13} stroke="#808080" stroke-width="1" />
					<text x={xScaleDemographic(xLabel)} y={height - 10} text-anchor="middle" dominant-baseline="middle" fill="#94a3b8" font-size="20px">{xLabel}</text>
				{/each}
				{#each genders as yLabel}
					<text x={xScaleDemographic('gender') - 1} y={yScaleGender(yLabel)} text-anchor="end" dominant-baseline="middle" fill="#94a3b8" font-size="13px">{yLabel}</text>
				{/each}
				{#each allAgeLabels as yLabel}
					<text x={xScaleDemographic('age') - 1} y={yScaleAge(yLabel)} text-anchor="end" dominant-baseline="middle" fill="#94a3b8" font-size="13px">{yLabel}</text>
				{/each}
				{#each allOccupations as yLabel}
					<text x={xScaleDemographic('occupation') - 1} y={yScaleOccupation(yLabel)} text-anchor="end" dominant-baseline="middle" fill="#94a3b8" font-size="13px">{yLabel}</text>
				{/each}
				{#each ratings as yLabel}
					<text x={xScaleDemographic('rating') - 1} y={yScaleRatings(yLabel)} text-anchor="end" dominant-baseline="middle" fill="#94a3b8" font-size="13px">{yLabel}</text>
				{/each}
			</g>
		</svg>
	</div>
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

</style>