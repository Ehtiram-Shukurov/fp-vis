<script>
// @ts-nocheck
	import { onMount } from 'svelte'
	const base = import.meta.env.BASE_URL

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

	$: allGenres = (() => {
		let genres = []
		Object.values(movies).forEach(movie => {
			movie.genres.forEach(genre => {
				if (!genres.includes(genre) && genre !== "unknown") {
					genres.push(genre)
				}
			})
		})
		return genres.sort()
	})()

	$: allOccupations = (() => {
		let occs = ["All"]
		Object.values(users).forEach(user => {
			if (!occs.includes(user.occupation)) {
				occs.push(user.occupation)
			}
		})
		return occs.sort()
	})()

	let filterType = "genre"
	let selectedGenre = ""
	let selectedMovieId = ""

	let selectedAge = "All"
	let selectedGender = "All"
	let selectedOccupation = "All"

	$: filteredRatings = allRatings.filter(r => {
		if (filterType === "movie") {
			if (selectedMovieId === "") return false
			if (r.movieId !== parseInt(selectedMovieId)) return false
		} else {
			let movie = movies[r.movieId]
			if (!movie) return false
			if (selectedGenre !== "" && !movie.genres.includes(selectedGenre)) return false
		}

		let user = users[r.userId]
		if (!user) return false

		if (selectedGender !== "All" && user.gender !== selectedGender) return false
		if (selectedOccupation !== "All" && user.occupation !== selectedOccupation) return false

		if (selectedAge === "Under 25" && user.age >= 25) return false
		if (selectedAge === "25-34" && (user.age < 25 || user.age > 34)) return false
		if (selectedAge === "35-44" && (user.age < 35 || user.age > 44)) return false
		if (selectedAge === "45+" && user.age < 45) return false

		return true
	})

	$: ratingCounts = [1, 2, 3, 4, 5].map(star => {
		let count = filteredRatings.filter(r => r.rating === star).length
		return { star, count }
	})

	$: maxCount = Math.max(...ratingCounts.map(c => c.count), 1)

	$: avgRating = filteredRatings.length > 0
		? (filteredRatings.reduce((sum, r) => sum + r.rating, 0) / filteredRatings.length).toFixed(1)
		: "N/A"
</script>

<div class="container">
	{#if loading}
		<p>Loading data...</p>
	{:else if error}
		<p style="color: red;">{error}</p>
	{:else}
		<div class="filters">
			<div>
				<label>
					<input type="radio" bind:group={filterType} value="genre" />
					Filter by Genre
				</label>
				<label>
					<input type="radio" bind:group={filterType} value="movie" />
					Filter by Movie
				</label>
			</div>

			{#if filterType === "genre"}
				<div>
					<label for="genre">Genre:</label>
					<select id="genre" bind:value={selectedGenre}>
						<option value="">All Genres</option>
						{#each allGenres as genre}
							<option value={genre}>{genre}</option>
						{/each}
					</select>
				</div>
			{:else}
				<div>
					<label for="movie">Movie:</label>
					<select id="movie" bind:value={selectedMovieId}>
						<option value="">-- select a movie --</option>
						{#each Object.values(movies) as movie}
							<option value={movie.id}>{movie.title}</option>
						{/each}
					</select>
				</div>
			{/if}

			<hr />
			<p><strong>Demographics:</strong></p>

			<div>
				<label for="age">Age:</label>
				<select id="age" bind:value={selectedAge}>
					<option>All</option>
					<option>Under 25</option>
					<option>25-34</option>
					<option>35-44</option>
					<option>45+</option>
				</select>
			</div>

			<div>
				<label for="gender">Gender:</label>
				<select id="gender" bind:value={selectedGender}>
					<option>All</option>
					<option value="M">Male</option>
					<option value="F">Female</option>
				</select>
			</div>

			<div>
				<label for="occupation">Occupation:</label>
				<select id="occupation" bind:value={selectedOccupation}>
					{#each allOccupations as occ}
						<option value={occ}>{occ}</option>
					{/each}
				</select>
			</div>
		</div>

		<div class="chart-area">
			<p>Total ratings: {filteredRatings.length} | Average: {avgRating}</p>

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
	.filters {
		background-color: #111827;
		border: 1px solid #1e2530;
		padding: 15px;
		border-radius: 5px;
		margin-bottom: 20px;
		color: #94a3b8;
}

	.filters div {
		margin-bottom: 10px;
	}

	.filters label {
		margin-right: 15px;
		color: #94a3b8;
	}

	select {
		padding: 5px;
		margin-left: 5px;
		background: #1e2530;
		color: #e2e8f0;
		border: 1px solid #2d3748;
		border-radius: 4px;
	}

	hr {
		margin: 10px 0;
		border-color: #1e2530;
	}

	.chart-area {
		border: 1px solid #1e2530;
		padding: 20px;
		border-radius: 5px;
		background: transparent;
		color: #64748b;
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

.bar {
  transition: height 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}
</style>