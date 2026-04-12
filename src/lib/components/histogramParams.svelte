<script>
// @ts-nocheck
	import { onMount } from 'svelte'
	const base = import.meta.env.BASE_URL

	export let filterType = "genre"
	export let selectedGenre = ""
	export let selectedMovieId = ""
	export let selectedAge = "All"
	export let selectedGender = "All"
	export let selectedOccupation = "All"

	let allRatings = []
	let movies = {}
	let users = {}
	let loading = true
	let error = ""

	const genreNames = [
		"unknown", "Action", "Adventure", "Animation", "Children's", "Comedy",
		"Crime", "Documentary", "Drama", "Fantasy", "Film-Noir", "Horror",
		"Musical", "Mystery", "Romance", "Sci-Fi", "Thriller", "War", "Western"
	]

	onMount(async () => {
		try {
			const ratingsRes = await fetch(`${base}u.data`)
			const ratingsText = await ratingsRes.text()
			allRatings = ratingsText.trim().split('\n').map(line => {
				const parts = line.split('\t')
				return {
					userId: parseInt(parts[0]),
					movieId: parseInt(parts[1]),
					rating: parseInt(parts[2])
				}
			})

			const itemsRes = await fetch(`${base}u.item`)
			const itemsText = await itemsRes.text()
			itemsText.trim().split('\n').forEach(line => {
				const parts = line.split('|')
				const id = parseInt(parts[0])
				const title = parts[1]
				const genres = []
				for (let i = 0; i < genreNames.length; i++) {
					if (parts[5 + i] === '1') {
						genres.push(genreNames[i])
					}
				}
				movies[id] = { id, title, genres }
			})

			const usersRes = await fetch(`${base}u.user`)
			const usersText = await usersRes.text()
			usersText.trim().split('\n').forEach(line => {
				const parts = line.split('|')
				const id = parseInt(parts[0])
				users[id] = {
					id,
					age: parseInt(parts[1]),
					gender: parts[2],
					occupation: parts[3],
					zip: parts[4]
				}
			})

			loading = false
		} catch (e) {
			error = "couldn't load data files - make sure u.data, u.item, and u.user are in the static folder"
			loading = false
		}
	})

	$: filteredRatings = allRatings.filter(r => {
		if (filterType === "movie") {
			if (selectedMovieId === "") return false
			if (r.movieId !== parseInt(selectedMovieId)) return false
		} else {
			const movie = movies[r.movieId]
			if (!movie) return false
			if (selectedGenre !== "" && !movie.genres.includes(selectedGenre)) return false
		}

		const user = users[r.userId]
		if (!user) return false

		if (selectedGender !== "All" && user.gender !== selectedGender) return false
		if (selectedOccupation !== "All" && user.occupation !== selectedOccupation) return false

		if (selectedAge === "Under 25" && user.age >= 25) return false
		if (selectedAge === "25-34" && (user.age < 25 || user.age > 34)) return false
		if (selectedAge === "35-44" && (user.age < 35 || user.age > 44)) return false
		if (selectedAge === "45+" && user.age < 45) return false

		return true
	})

	$: ratingCounts = [1, 2, 3, 4, 5].map(star => ({
		star,
		count: filteredRatings.filter(r => r.rating === star).length
	}))

	$: maxCount = Math.max(...ratingCounts.map(c => c.count), 1)

	$: avgRating = filteredRatings.length > 0
		? (filteredRatings.reduce((sum, r) => sum + r.rating, 0) / filteredRatings.length).toFixed(1)
		: "N/A"

	$: chartLabel = (() => {
		if (filterType === "movie") {
			const movie = movies[parseInt(selectedMovieId)]
			return movie ? movie.title : "Unknown Movie"
		}
		return selectedGenre !== "" ? selectedGenre : "All Genres"
	})()
</script>

<div class="container">
	{#if loading}
		<p>Loading data...</p>
	{:else if error}
		<p style="color: red;">{error}</p>
	{:else}
		<div class="chart-area">
			<p class="chart-title">{chartLabel}</p>
			<p class="chart-meta">
				Total ratings: {filteredRatings.length} | Average: {avgRating}
				{#if selectedAge !== "All" || selectedGender !== "All" || selectedOccupation !== "All"}
					<span class="demographic-tag">
						{[
							selectedAge !== "All" ? selectedAge : null,
							selectedGender !== "All" ? (selectedGender === "M" ? "Male" : "Female") : null,
							selectedOccupation !== "All" ? selectedOccupation : null
						].filter(Boolean).join(", ")}
					</span>
				{/if}
			</p>

			{#if filteredRatings.length === 0}
				<p>No ratings found for these filters.</p>
			{:else}
				{#key filteredRatings}
					<div class="histogram">
						{#each ratingCounts as item}
							<div class="bar-wrapper">
								<span class="bar-count">{item.count}</span>
								<div
									class="bar"
									style="
										height: {(item.count / maxCount) * 250}px;
										--bar-height: {(item.count / maxCount) * 250}px;
										background-color: {item.star <= 2 ? '#e74c3c' : item.star === 3 ? '#f39c12' : '#2ecc71'};
										animation-delay: {(item.star - 1) * 150}ms
									"
								></div>
								<span class="bar-label">{item.star} star</span>
							</div>
						{/each}
					</div>
				{/key}
			{/if}
		</div>
	{/if}
</div>

<style>
	.container {
		padding: 20px;
		font-family: Arial, sans-serif;
		background: transparent;
		color: #e2e8f0;
	}

	.chart-area {
		border: 1px solid #1e2530;
		padding: 20px;
		border-radius: 5px;
		background: transparent;
		color: #64748b;
	}

	.chart-title {
		font-size: 18px;
		font-weight: bold;
		color: #e2e8f0;
		margin-bottom: 4px;
	}

	.chart-meta {
		margin-bottom: 16px;
		font-size: 14px;
	}

	.demographic-tag {
		display: inline-block;
		margin-left: 10px;
		background: #1e2530;
		border: 1px solid #2d3748;
		border-radius: 12px;
		padding: 2px 10px;
		font-size: 12px;
		color: #94a3b8;
	}

	.histogram {
		display: flex;
		align-items: flex-end;
		gap: 20px;
		height: 350px;
		padding-top: 30px;
	}

	.bar-wrapper {
		display: flex;
		flex-direction: column;
		align-items: center;
		flex: 1;
	}

	.bar {
		width: 100%;
		min-height: 2px;
		border-radius: 3px 3px 0 0;
		animation: grow 0.8s ease-out forwards;
	}

	.bar-count {
		margin-bottom: 5px;
		font-size: 14px;
		color: #94a3b8;
	}

	.bar-label {
		margin-top: 8px;
		font-size: 13px;
		color: #64748b;
	}

	@keyframes grow {
		from { height: 0; }
		to   { height: var(--bar-height); }
	}
</style>