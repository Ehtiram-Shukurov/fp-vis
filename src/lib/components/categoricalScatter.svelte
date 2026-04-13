<script>
//@ts-nocheck
	import { onMount } from 'svelte'
  import * as d3 from "d3"
  import { derived } from 'svelte/store';
  import { fly } from 'svelte/transition';
  import Scroll from '$lib/components/Scroll.svelte';

	let allRatings = $state([]);
	let movies = $state({});
	let users = $state({});
	let loading = $state(true);
	let error = $state("");

  let myProgress = $state(0);

  let ageOnX = $derived(myProgress <= 50)
  let GenderOnX = $derived(myProgress > 50 && myProgress <= 75)
  let OccupationOnX = $derived(myProgress > 75)

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

    let height=500;
    let width= $derived(xAxis === "Occupation" ? 1200 : 900);
    
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
      if (ageOnX) {
        xAxis="Age";
      }
      if (GenderOnX) {
        xAxis="Gender";
      }
      if (OccupationOnX) {
        xAxis="Occupation";
      }
    })
</script>

<Scroll bind:progress={myProgressSetUp} --scrolly-story-width="0">
  <svelte:fragment slot="viz">
    <div class="setup-center">
      {#if loading}
        <p style="color: #64748b">Loading data...</p>
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

<!-- scrolly-->
<Scroll bind:progress={myProgress} --scrolly-viz-width="3fr">
  <div class="story-scroll">

    <div class="story-block" class:active={myProgress <= 50}>
      <span class="step-num">01</span>
      <h3>By age</h3>
      <p>Consider your placement in this visualization, sorted with age groups on the bottom. Do you match the biggest rating group within those of a similar age?</p>
      <p>Is age a big factor of your rating, or do you think it's something else?</p>
    </div>

    <div class="story-block" class:active={myProgress > 50 && myProgress <= 75}>
      <span class="step-num">02</span>
      <h3>By gender</h3>
      <p>Now the graph switches to sort by gender. Does your rating match up with others who share yours?</p>
      <p>Did your gender, and the culture tied to it, have any impact on how you rated this film?</p>
    </div>

    <div class="story-block" class:active={myProgress > 75}>
      <span class="step-num">03</span>
      <h3>By occupation</h3>
      <p>Finally, sorted by occupation. Do you match those who share your field?</p>
      <p>Demographics affecting movie ratings runs deeper than it seems. it's the product of everything that shaped who you are.</p>
    </div>

  </div>
  <svelte:fragment slot="viz">
    <div class="center">
      <p>Total ratings: {filteredRatings.length} | Average: {avgRating}</p>
      <div class="border">
        <svg {height} width=100% viewBox={`0 0 ${width} ${height}`}>
          <g>
            {#each graphPoints as point}
              <circle class="data"
                transition:fly={{ y: 200, duration: 1000 }} 
                cx={xScale(point.dataCategory) + xScale.bandwidth() / 2}
                cy={yScale(String(point.userRating)) + yScale.bandwidth() / 2}
                r={0.4 * point.count}
                fill={point.dataCategory===currentUserData && String(point.userRating)===userRating ? "red" : "steelblue"}
                stroke="black"
                stroke-width="2"
                opacity={0.8}
              />
              <text class="data"
                transition:fly={{ y: 200, duration: 1000 }} 
                x={xScale(point.dataCategory) + xScale.bandwidth() / 2}
                y={yScale(String(point.userRating)) + yScale.bandwidth() / 2 - (0.4 * point.count + 6)} 
                text-anchor="middle"
                dominant-baseline="middle"
                fill="#e2e8f0">
                {point.count}
              </text>
            {/each}
            <circle class="data"
              cx={xScale(currentUserData) + xScale.bandwidth() / 2}
              cy={(userRating === -8) ? -20 : yScale(String(userRating)) + yScale.bandwidth() / 2}
              r={10}
              fill={(userRating === -8) ? "none" : "green"}
              stroke={(userRating === -8) ? "none" : "green"}
              stroke-width="2"
              opacity={0.9}
            />

            {#each ["1", "2", "3", "4", "5"] as yMark}
              <text x={usableArea.left - 10} y={yScale(yMark) + yScale.bandwidth() / 2} dominant-baseline="middle" fill="#94a3b8">{yMark}</text>
            {/each}

            {#each xAxisCategories as xMark}
              <text x={xScale(xMark) + xScale.bandwidth() / 2} y={usableArea.bottom} text-anchor="middle" transform="rotate(-45, {xScale(xMark) + xScale.bandwidth() / 2}, {usableArea.bottom})" fill="#94a3b8">{xMark}</text>
            {/each}

            <line 
              x1={usableArea.left+8}
              x2={usableArea.left+8}
              y1={usableArea.top}
              y2={usableArea.bottom}
              stroke="#2d3748"
              stroke-width="1"
            ></line>
          </g>
        </svg>
      </div>
    </div>
  </svelte:fragment>
</Scroll>

<style>
.setup-story {
  display: flex;
  flex-direction: column;
  gap: 14px;
  padding: 28px 0;
}

.setup-story h3 {
  font-size: 28px;
  font-weight: 500;
  color: #e2e8f0;
  margin: 0;
}

.setup-story p {
  font-size: 15px;
  color: #64748b;
  line-height: 1.7;
  max-width: 550px;
  margin: 0;
}
  .setup-center {
    width: 100%;
    height: 40vh;
    display: flex;
    align-items: center;
    justify-content: center;
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
    font-size: 14px;
    color: #94a3b8;
    flex-shrink: 0;
  }

  .form-row select {
    flex: 1;
  }
  .story-scroll {
    display: flex;
    flex-direction: column;
  }

  .story-block {
    min-height: 70vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 28px 0;
    border-bottom: 1px solid #1e2530;
    gap: 10px;
    opacity: 0.3;
    transition: opacity 0.3s ease;
  }

  .story-block.active {
    opacity: 1;
  }

  .step-num {
    font-size: 12px;
    color: #4a5568;
    font-weight: 500;
  }

  .story-block h3 {
    font-size: 24px;
    font-weight: 500;
    color: #e2e8f0;
    margin: 0;
  }

  .story-block p {
    font-size: 15px;
    color: #64748b;
    line-height: 1.7;
    max-width: 420px;
    margin: 0;
  }

  .data {
    transition: all 1s ease;
  }

  .center {
    display: flex;
    flex-direction: column;
    padding-top: 80px;
  }

  .border {
    border: 1px solid #1e2530;
    background-color: transparent;
    border-radius: 8px;
    padding: 16px;
  }

  select {
    background: #1e2530;
    color: #e2e8f0;
    border: 1px solid #2d3748;
    border-radius: 4px;
    padding: 4px 8px;
  }

  label {
    color: #94a3b8;
  }

  p {
    color: #64748b;
  }
</style>