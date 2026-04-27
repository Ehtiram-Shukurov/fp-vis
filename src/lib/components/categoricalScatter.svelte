<script>
//@ts-nocheck
  import { onMount } from 'svelte'
  import * as d3 from "d3"
  import { derived } from 'svelte/store';
  import { fly, fade } from 'svelte/transition';
  import Scroll from '$lib/components/Scroll.svelte';

  let allRatings = $state([]);
  let movies = $state({});
  let users = $state({});
  let loading = $state(true);
  let error = $state("");

  let myProgress = $state(0);

  //sticky scroll
const thresholds = [0, 45, 95];
  let activeIndex = $derived(Math.max(0, thresholds.findLastIndex(t => myProgress >= t)));

  let ageOnX = $derived(activeIndex === 0);
  let GenderOnX = $derived(activeIndex === 1);
  let OccupationOnX = $derived(activeIndex === 2);

  let myProgressSetUp = $state(0);

  const genreNames = [
    "unknown", "Action", "Adventure", "Animation", "Children's", "Comedy",
    "Crime", "Documentary", "Drama", "Fantasy", "Film-Noir", "Horror",
    "Musical", "Mystery", "Romance", "Sci-Fi", "Thriller", "War", "Western"
  ]

  onMount(async () => {
    try {
      const base = import.meta.env.BASE_URL
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
  });

  let allGenres = $derived.by(() => {
    let genres = []
    Object.values(movies).forEach(movie => {
      movie.genres.forEach(genre => {
        if (!genres.includes(genre) && genre !== "unknown") {
          genres.push(genre)
        }
      })
    })
    return genres.sort()
  });

  let allOccupations = $derived.by(() => {
    let occs = []
    Object.values(users).forEach(user => {
      if (!occs.includes(user.occupation)) {
        occs.push(user.occupation)
      }
    })
    return occs.sort()
  });

  let selectedMovieId = $state("");

  let filteredRatings = $derived(allRatings.filter(r => {
    if (selectedMovieId === "") return false
    if (r.movieId !== parseInt(selectedMovieId)) return false
    return true
  }));

  let ratingCounts = $derived([1, 2, 3, 4, 5].map(star => {
    let count = filteredRatings.filter(r => r.rating === star).length
    return { star, count }
  }));

  let maxCount = $derived(Math.max(...ratingCounts.map(c => c.count), 1));

  let avgRating = $derived(filteredRatings.length > 0
    ? (filteredRatings.reduce((sum, r) => sum + r.rating, 0) / filteredRatings.length).toFixed(1)
    : "N/A");

  let userAge= $state("");
  let userGender= $state("");
  let userOccupation= $state("");
  let userRating= $state(-8);

  const margin = {
    top: 15,
    bottom: 50,
    left: 30,
    right: 10,
  };

  let xAxis = $state("Age");

  let height = 500;
  let width = $derived(xAxis === "Occupation" ? 1200 : 900);

  const usableArea = $derived({
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  });

  let xAxisCategories = $derived.by(() => {
    if (xAxis === "Gender") return ["Male", "Female"]
    else if (xAxis === "Age") return ["Under 25", "25-34", "35-44", "45+"]
    else return allOccupations
  });

  let xScale = $derived(d3.scaleBand().domain(xAxisCategories).range([usableArea.left, usableArea.right]));
  let yScale = $derived(d3.scaleBand().domain(["1", "2", "3", "4", "5"]).range([usableArea.bottom, usableArea.top]));

  function getDataCategory(reviewerDemographic) {
    if (xAxis === "Gender") return reviewerDemographic.gender === "M" ? "Male" : "Female";
    else if (xAxis === "Age") {
      if (reviewerDemographic.age < 25) return "Under 25";
      else if (reviewerDemographic.age < 35) return "25-34";
      else if (reviewerDemographic.age < 45) return "35-44";
      else return "45+";
    }
    else return reviewerDemographic.occupation;
  }

  let graphPoints = $derived.by(() => {
    const graphPoints = {};

    filteredRatings.forEach(rating => {
      const reviewer = users[rating.userId];
      const dataCategory = getDataCategory(reviewer);

      const index = `${dataCategory}, ${rating.rating}`;
      if (!graphPoints[index]) {
        graphPoints[index] = {dataCategory: dataCategory, userRating: rating.rating, count: 1};
      }
      else {
        graphPoints[index].count += 1;
      }
    })

    return Object.values(graphPoints);
  })

  let currentUserData = $derived.by(() => {
    if (xAxis === "Gender") return userGender;
    else if (xAxis === "Age") return userAge;
    else return userOccupation;
  });

  $effect(() => {
    if (ageOnX) xAxis="Age";
    if (GenderOnX) xAxis="Gender";
    if (OccupationOnX) xAxis="Occupation";
  })
</script>

<Scroll bind:progress={myProgressSetUp} --scrolly-story-width="0">
  <svelte:fragment slot="viz">
    <div class="setup-center">
      {#if loading}
        <p style="color: var(--text-secondary)">Loading data...</p>
      {:else if error}
        <p style="color: red">{error}</p>
      {:else}
        <div class="setup-story">
          <h3>Enter your details</h3>
          <p>Pick a movie you've seen. Then, enter your demographic info and personal rating. In the following visualization, your personal dot will appear highlighted green and/or red on the chart where you would fit in.</p>
          <p>Bigger dots mean more people gave that rating.</p>
        </div>

        <div class="setup-form">
          <div class="form-row">
            <label for="movie">Movie</label>
            <select id="movie" bind:value={selectedMovieId}>
              <option value="">-- select a movie --</option>
              {#each Object.values(movies) as movie}
                <option value={movie.id}>{movie.title}</option>
              {/each}
            </select>
          </div>
          <div class="form-row">
            <label for="age">Age</label>
            <select id="age" bind:value={userAge}>
              <option>Under 25</option>
              <option>25-34</option>
              <option>35-44</option>
              <option>45+</option>
            </select>
          </div>
          <div class="form-row">
            <label for="gender">Gender</label>
            <select id="gender" bind:value={userGender}>
              <option value="Male">Male</option>
              <option value="Female">Female</option>
            </select>
          </div>
          <div class="form-row">
            <label for="occupation">Occupation</label>
            <select id="occupation" bind:value={userOccupation}>
              {#each allOccupations as occ}
                <option value={occ}>{occ}</option>
              {/each}
            </select>
          </div>
          <div class="form-row">
            <label for="rating">Rating</label>
            <select id="rating" bind:value={userRating}>
              <option value=1>1</option>
              <option value=2>2</option>
              <option value=3>3</option>
              <option value=4>4</option>
              <option value=5>5</option>
            </select>
          </div>
        </div>
      {/if}
    </div>
  </svelte:fragment>
</Scroll>

<Scroll bind:progress={myProgress} --scrolly-viz-width="1.4fr" debounce={50}>
  
  <div class="sticky-text-wrap">
    <div class="card-container">
      {#key activeIndex}
        <div class="scroll-card active" in:fly={{ y: 20, duration: 600, delay: 200 }} out:fly={{ y: -20, duration: 400 }}>
          <span class="step-num">0{activeIndex + 1}</span>
          
          {#if activeIndex === 0}
            <h3>By age</h3>
            <p>Consider your placement in this visualization, sorted with age groups on the bottom. Do you match the biggest rating group within those of a similar age?</p>
            <br/>
            <p>Is age a big factor of your rating, or do you think it's something else?</p>
          {:else if activeIndex === 1}
            <h3>By gender</h3>
            <p>Now the graph switches to sort by gender. Does your rating match up with others who share yours?</p>
            <br/>
            <p>Did your gender, and the culture tied to it, have any impact on how you rated this film?</p>
          {:else}
            <h3>By occupation</h3>
            <p>Finally, sorted by occupation. Do you match those who share your field?</p>
            <br/>
            <p>Demographics affecting movie ratings runs deeper than it seems. it's the product of everything that shaped who you are.</p>
          {/if}
        </div>
      {/key}
    </div>
  </div>

  <div class="scroll-track">
    <div class="spacer"></div>
    <div class="spacer"></div>
    <div class="spacer"></div>
  </div>

  <svelte:fragment slot="viz">
    <div class="center">
      <p style="color: var(--text-secondary); font-size: 18px; margin-bottom: 12px;">
        Total ratings: {filteredRatings.length} | Average: {avgRating}
      </p>
      
      <div class="border glass-panel">
        <svg {height} width=100% viewBox={`0 0 ${width} ${height}`}>
          <g>
            {#each graphPoints as point}
              <circle class="data"
                transition:fly={{ y: 200, duration: 1000 }} 
                cx={xScale(point.dataCategory) + xScale.bandwidth() / 2}
                cy={yScale(String(point.userRating)) + yScale.bandwidth() / 2}
                r={0.4 * point.count}
                fill={point.dataCategory===currentUserData && String(point.userRating)===String(userRating) ? "#6d49ff" : "steelblue"}
                stroke="var(--bg-dark)"
                stroke-width="2"
                opacity={0.8}
              />
              <text class="data"
                transition:fly={{ y: 200, duration: 1000 }} 
                x={xScale(point.dataCategory) + xScale.bandwidth() / 2}
                y={yScale(String(point.userRating)) + yScale.bandwidth() / 2 - (0.4 * point.count + 6)} 
                text-anchor="middle"
                dominant-baseline="middle"
                fill="var(--text-primary)">
                {point.count}
              </text>
            {/each}

            {#if userRating != -8}
              {@const cx = xScale(currentUserData) + xScale.bandwidth() / 2}
              {@const cy = yScale(String(userRating)) + yScale.bandwidth() / 2}
              {@const r = 20}
              <g style="transform: translate({cx}px, {cy}px); transition: transform 1s ease-in-out;">
                <polygon
                  class="pulse"
                  points={Array.from({length: 5}, (_, i) => {
                    const outer = `${r * Math.sin((i * 4 * Math.PI) / 5)},${r * Math.cos((i * 4 * Math.PI) / 5)}`;
                    const inner = `${(r/2) * Math.sin(((i * 4 + 2) * Math.PI) / 5)},${(r/2) * Math.cos(((i * 4 + 2) * Math.PI) / 5)}`;
                    return `${outer} ${inner}`;
                  }).join(' ')}
                  fill="gold"
                  stroke="gold"
                  stroke-width="1.5"
                  opacity={0.95}
                />
              </g>
            {/if}

            

            {#each ["1", "2", "3", "4", "5"] as yMark}
              <text x={usableArea.left - 10} y={yScale(yMark) + yScale.bandwidth() / 2} dominant-baseline="middle" fill="var(--text-muted)">{yMark}</text>
            {/each}

            {#each xAxisCategories as xMark}
              <text x={xScale(xMark) + xScale.bandwidth() / 2} y={usableArea.bottom} text-anchor="middle" transform="rotate(-45, {xScale(xMark) + xScale.bandwidth() / 2}, {usableArea.bottom})" fill="var(--text-muted)">{xMark}</text>
            {/each}

            <line 
              x1={usableArea.left+8}
              x2={usableArea.left+8}
              y1={usableArea.top}
              y2={usableArea.bottom}
              stroke="var(--border-subtle)"
              stroke-width="1"
            ></line>
          </g>
        </svg>
      </div>
    </div>
  </svelte:fragment>
</Scroll>

<style>

  @keyframes pulse {
      0% {
          transform: scale(.8);
      }
      50% {
          transform: scale(1.1);
      }
      100% {
          transform: scale(.8);
      }
    }

  .pulse {
          
    animation: pulse 3s infinite;
    transform-box: fill-box;        /* makes transform-origin relative to the element itself */
    transform-origin: center;
    }
  /* --- SETUP SECTION STYLES --- */
  .setup-center {
    width: 100%;
    min-height: 40vh;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 40px;
    padding: 20px 0;
    margin-bottom: 25vh;
  }

  .setup-story {
    display: flex;
    flex-direction: column;
    gap: 14px;
  }

  .setup-story h3 {
    font-size: 32px;
    font-weight: 500;
    color: var(--text-primary);
    margin: 0;
  }

  .setup-story p {
    font-size: 18px;
    color: var(--text-secondary);
    line-height: 1.7;
    max-width: 450px;
    margin: 0;
  }

  .setup-form {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .form-row {
    display: flex;
    align-items: center;
    gap: 16px;
  }

  .form-row label {
    width: 90px;
    font-size: 15px;
    color: var(--text-muted);
    flex-shrink: 0;
  }

  .form-row select {
    flex: 1;
    background: var(--bg-card);
    color: var(--text-primary);
    border: 1px solid var(--border-subtle);
    border-radius: 6px;
    padding: 8px 12px;
    font-size: 15px;
  }

  /* --- STICKY SCROLLY STYLES --- */
  .scroll-track { width: 100%; }
  .spacer { height: 120vh; }

  .sticky-text-wrap {
    position: sticky;
    top: 30vh;
    width: 100%;
    height: 0;
    z-index: 10;
    overflow: visible;
  }

  .card-container {
    position: relative;
    width: 100%;
  }

  .scroll-card {
    position: absolute; 
    top: 0;
    left: 0;
    width: 100%;
    box-sizing: border-box;
    padding: 32px;
    background: var(--bg-card);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
    border: 1px solid var(--border-subtle);
    border-radius: 12px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  }

  .step-num {
    font-size: 14px;
    color: var(--text-muted);
    font-weight: 600;
    margin-bottom: 12px;
    display: block;
  }

  .scroll-card h3 {
    font-size: 28px;
    font-weight: 500;
    color: var(--text-primary);
    margin: 0 0 12px;
  }

  .scroll-card p {
    font-size: 18px;
    color: var(--text-secondary);
    line-height: 1.6;
    margin: 0;
  }

  /* --- CHART AREA STYLES --- */
  .center {
    display: flex;
    flex-direction: column;
    padding-top: 80px;
  }

  .border {
    border: 1px solid var(--border-subtle);
    background-color: var(--bg-card);
    border-radius: 12px;
    padding: 24px;
  }

  .data {
    transition: all 1s ease;
  }
</style>