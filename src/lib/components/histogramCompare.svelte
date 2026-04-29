<script>
// @ts-nocheck
  import { onMount } from 'svelte'
  const base = import.meta.env.BASE_URL

  export let filterType = "genre"

  // Series A
  export let selectedGenreA = ""
  export let selectedMovieIdA = ""
  export let selectedAgeA = "All"
  export let selectedGenderA = "All"
  export let selectedOccupationA = "All"

  // Series B
  export let selectedGenreB = ""
  export let selectedMovieIdB = ""
  export let selectedAgeB = "All"
  export let selectedGenderB = "All"
  export let selectedOccupationB = "All"

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

  function normalizeGender(g) {
    if (g === "Male") return "M"
    if (g === "Female") return "F"
    return g
  }

  onMount(async () => {
    try {
      const ratingsRes = await fetch(`${base}u.data`)
      const ratingsText = await ratingsRes.text()
      allRatings = ratingsText.trim().split('\n').map(line => {
        const parts = line.trim().split('\t')
        return {
          userId: parseInt(parts[0]),
          movieId: parseInt(parts[1]),
          rating: parseInt(parts[2])
        }
      })

      const itemsRes = await fetch(`${base}u.item`)
      const itemsText = await itemsRes.text()
      itemsText.trim().split('\n').forEach(line => {
        const parts = line.trim().split('|')
        const id = parseInt(parts[0])
        const title = parts[1]
        const genres = []
        for (let i = 0; i < genreNames.length; i++) {
          if (parts[5 + i] === '1') genres.push(genreNames[i])
        }
        movies[id] = { id, title, genres }
      })

      const usersRes = await fetch(`${base}u.user`)
      const usersText = await usersRes.text()
      usersText.trim().split('\n').forEach(line => {
        const parts = line.trim().split('|')
        const id = parseInt(parts[0])
        users[id] = {
          id,
          age: parseInt(parts[1]),
          gender: parts[2].trim(),
          occupation: parts[3].trim(),
          zip: parts[4]
        }
      })

      loading = false
    } catch (e) {
      error = "couldn't load data files - make sure u.data, u.item, and u.user are in the static folder"
      loading = false
    }
  })

  function filterRatings(opts) {
    const { genre, movieId, age, gender, occupation } = opts
    return allRatings.filter(r => {
      if (filterType === "movie") {
        if (movieId === "") return false
        if (r.movieId !== parseInt(movieId)) return false
      } else {
        const movie = movies[r.movieId]
        if (!movie) return false
        if (genre !== "" && !movie.genres.includes(genre)) return false
      }

      const user = users[r.userId]
      if (!user) return false

      if (gender !== "All" && user.gender !== gender) return false
      if (occupation !== "All" && user.occupation !== occupation) return false

      if (age === "Under 25" && user.age >= 25) return false
      if (age === "25-34" && (user.age < 25 || user.age > 34)) return false
      if (age === "35-44" && (user.age < 35 || user.age > 44)) return false
      if (age === "45+" && user.age < 45) return false

      return true
    })
  }

  function getRatingCounts(ratings) {
    return [1, 2, 3, 4, 5].map(star => ({
      star,
      count: ratings.filter(r => r.rating === star).length
    }))
  }

  function getAvg(ratings) {
    if (ratings.length === 0) return "N/A"
    return (ratings.reduce((sum, r) => sum + r.rating, 0) / ratings.length).toFixed(1)
  }

  function getLabel(opts) {
    if (filterType === "movie") {
      const movie = movies[parseInt(opts.movieId)]
      return movie ? movie.title : "Unknown Movie"
    }
    return opts.genre !== "" ? opts.genre : "All Genres"
  }

  function getDemographicTag(opts) {
    return [
      opts.age !== "All" ? opts.age : null,
      opts.gender !== "All" ? (opts.gender === "M" ? "Male" : "Female") : null,
      opts.occupation !== "All" ? opts.occupation : null
    ].filter(Boolean).join(", ")
  }

  $: ratingsA = filterRatings({
    genre: selectedGenreA,
    movieId: selectedMovieIdA,
    age: selectedAgeA,
    gender: normalizeGender(selectedGenderA),
    occupation: selectedOccupationA
  })

  $: ratingsB = filterRatings({
    genre: selectedGenreB,
    movieId: selectedMovieIdB,
    age: selectedAgeB,
    gender: normalizeGender(selectedGenderB),
    occupation: selectedOccupationB
  })

  $: countsA = getRatingCounts(ratingsA)
  $: countsB = getRatingCounts(ratingsB)

  $: maxCount = Math.max(...countsA.map(c => c.count), ...countsB.map(c => c.count), 1)

  $: labelA = getLabel({ genre: selectedGenreA, movieId: selectedMovieIdA })
  $: labelB = getLabel({ genre: selectedGenreB, movieId: selectedMovieIdB })

  $: tagA = getDemographicTag({ age: selectedAgeA, gender: normalizeGender(selectedGenderA), occupation: selectedOccupationA })
  $: tagB = getDemographicTag({ age: selectedAgeB, gender: normalizeGender(selectedGenderB), occupation: selectedOccupationB })
</script>

<div class="container">
  {#if loading}
    <p>Loading data...</p>
  {:else if error}
    <p style="color: red;">{error}</p>
  {:else}
    <div class="chart-area">

      <div class="legend">
        <div class="legend-item">
          <span class="legend-swatch swatch-a"></span>
          <span class="legend-label">
            {labelA}{tagA ? ` · ${tagA}` : ""}
            <span class="legend-meta">{ratingsA.length} ratings · avg {getAvg(ratingsA)}</span>
          </span>
        </div>
        <div class="legend-item">
          <span class="legend-swatch swatch-b"></span>
          <span class="legend-label">
            {labelB}{tagB ? ` · ${tagB}` : ""}
            <span class="legend-meta">{ratingsB.length} ratings · avg {getAvg(ratingsB)}</span>
          </span>
        </div>
      </div>

      {#if ratingsA.length === 0 && ratingsB.length === 0}
        <p>No ratings found for these filters.</p>
      {:else}
        {#key [countsA, countsB]}
          <div class="histogram">
            {#each [1, 2, 3, 4, 5] as star, i}
              {@const cA = countsA[i].count}
              {@const cB = countsB[i].count}
              <div class="bar-group">
                <div class="counts">
                  {#if cA > 0}<span class="bar-count count-a">{cA}</span>{/if}
                  {#if cB > 0}<span class="bar-count count-b">{cB}</span>{/if}
                </div>
                <div class="bars-overlay">
                  <div
                    class="bar bar-a"
                    style="
                      height: {(cA / maxCount) * 250}px;
                      --bar-height: {(cA / maxCount) * 250}px;
                      animation-delay: {i * 100}ms;
                    "
                  ></div>
                  <div
                    class="bar bar-b"
                    style="
                      height: {(cB / maxCount) * 250}px;
                      --bar-height: {(cB / maxCount) * 250}px;
                      animation-delay: {i * 100 + 50}ms;
                    "
                  ></div>
                </div>
                <span class="bar-label">{star} ★</span>
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
  }

  .legend {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-bottom: 20px;
  }

  .legend-item {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 14px;
    color: #e2e8f0;
  }

  .legend-swatch {
    display: inline-block;
    width: 12px;
    height: 12px;
    border-radius: 2px;
    flex-shrink: 0;
  }

  .swatch-a { background: rgba(99, 179, 237, 0.85); }
  .swatch-b { background: rgba(252, 129, 74, 0.75); }

  .legend-label {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
  }

  .legend-meta {
    font-size: 12px;
    color: #64748b;
  }

  .histogram {
    display: flex;
    align-items: flex-end;
    gap: 20px;
    height: 350px;
    padding-top: 60px;
  }

  .bar-group {
    display: flex;
    flex-direction: column;
    align-items: center;
    flex: 1;
  }

  .counts {
    display: flex;
    gap: 4px;
    margin-bottom: 5px;
    min-height: 20px;
    align-items: flex-end;
  }

  .bar-count {
    font-size: 11px;
  }

  .count-a { color: rgba(99, 179, 237, 0.9); }
  .count-b { color: rgba(252, 129, 74, 0.9); }

  .bars-overlay {
    position: relative;
    width: 100%;
    height: 250px;
    display: flex;
    align-items: flex-end;
    justify-content: center;
  }

  .bar {
    position: absolute;
    bottom: 0;
    min-height: 2px;
    border-radius: 3px 3px 0 0;
    animation: grow 0.7s ease-out forwards;
  }

  .bar-a {
    width: 85%;
    background: rgba(99, 179, 237, 0.55);
    border: 1.5px solid rgba(99, 179, 237, 0.9);
    z-index: 1;
  }

  .bar-b {
    width: 85%;
    background: rgba(252, 129, 74, 0.45);
    border: 1.5px solid rgba(252, 129, 74, 0.85);
    z-index: 2;
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