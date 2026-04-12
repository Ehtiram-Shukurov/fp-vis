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

  let ageOnX = $derived(myProgress <= 30);
  let GenderOnX = $derived((50 >= myProgress) && (myProgress > 30));
  let OccupationOnX = $derived(myProgress > 50);

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

	// let filterType = "genre"
	// let selectedGenre = "Comedy"
	let selectedMovieId = $state("");
	// let selectedAge = "All"
	// let selectedGender = "All"
	// let selectedOccupation = "All"

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


<Scroll bind:progress={ myProgressSetUp }>
	<!-- Story here -->
  <div class="storyTime2">
    <p>The data and correlations shown before provide a clear visualization of the trends different demographics cause in movie ratings.</p>
    <p>However, getting you, the reader, involved in these comparisons will provide something a lot more relatable and actionable.</p>
    <p>To start, look to the right and pick a movie you've seen, enter your age, gender, occupation, and rating for that same movie. Picking a movie with more ratings, such as Toy Story, will help with better comparisons.</p>
    <p>This information will then be input and sorted in the visualization below. You should see the total data for the movie you selected.</p>
    <p>Bigger dots mean more ratings with the y-axis displaying what specific rating each group has chosen [1, 2, 3, 4, 5]. The x-axis will initially be grouped by age group but will eventually change to age and occupation as you scroll.</p>
    <p>The red and/or green highlight displays where you yourself would fit into the data!</p>
  </div>
	<svelte:fragment slot="viz">
		<div class="totalcenter">
      {#if loading}
        <div class="center">
          <p>Loading data...</p>
        </div>
      {:else if error}
        <div class="center">
          <p style="color: red;">{error}</p>
        </div>
      {:else}
        <div class="border">
          <label for="movie">Movie:</label>
          <select id="movie" bind:value={selectedMovieId}>
            <option value="">-- select a movie --</option>
            {#each Object.values(movies) as movie}
              <option value={movie.id}>{movie.title}</option>
            {/each}
          </select>

          <p><strong>Enter Your Demographics as Well as Your Rating For This Movie</strong></p>

          <div>
            <label for="age">Age:</label>
            <select id="age" bind:value={userAge}>
              <option>Under 25</option>
              <option>25-34</option>
              <option>35-44</option>
              <option>45+</option>
            </select>
          </div>

          <div>
            <label for="gender">Gender:</label>
            <select id="gender" bind:value={userGender}>
              <option value="Male">Male</option>
              <option value="Female">Female</option>
            </select>
          </div>

          <div>
            <label for="occupation">Occupation:</label>
            <select id="occupation" bind:value={userOccupation}>
              {#each allOccupations as occ}
                <option value={occ}>{occ}</option>
              {/each}
            </select>
          </div>

          <div>
            <label for="rating">Rating:</label>
            <select id="rating" bind:value={userRating}>
              <option value=1>1</option>
              <option value=2>2</option>
              <option value=3>3</option>
              <option value=4>4</option>
              <option value=5>5</option>
            </select>
          </div>

          <!-- <div>
            <label for="xAxis">Select an Attribute to Compare Over:</label>
            <select id="xAxis" bind:value={xAxis}>
              <option value="Age">Age</option>
              <option value="Gender">Gender</option>
              <option value="Occupation">Occupation</option>
            </select>
          </div> -->
        </div>

      {/if}
    </div>
	</svelte:fragment>
</Scroll>

<Scroll bind:progress={ myProgress } --scrolly-viz-width="3fr">
	<!-- Story here -->
  <div class="storyTime">
    <p>To start, consider your placement in this visualization sorted with age groups on the bottom.</p>
    <br>
    <br>
    <p>Do you match with the biggest rating group within those of a similar age?</p>
    <br>
    <br>
    <p>Why do you think this is? Do you think different age groups should have wildly different opinions on this movie or not?</p>
    <br>
    <br>
    <p>Is age or associated factors like maturity a big factor of your rating for this movie or do you think it is something else?</p>
    <br>
    <br>
    <p>Now, as you scroll past here, you should see the graph transform to be sorted by gender.</p>
    <br>
    <br>
    <p>Does your rating match up with those within your gender?</p>
    <br>
    <br>
    <p>Does your selected movie show big differences in rating by gender?</p>
    <br>
    <br>
    <p>Think deeply on yourself, did your gender, and the attributes related to your gender such as the culture growing up, seem to have any impact on your personal rating?</p>
    <br>
    <br>
    <p>Finally, we switch the graph to sort by occupation.</p>
    <br>
    <br>
    <p>Do you match those who share your occupation?</p>
    <br>
    <br>
    <p>Is this a movie you think people in your occupation would share feelings on in some way good or bad?</p>
    <br>
    <br>
    <p>Do you think personally that your rating correlates with your choice in occupation? Did something in your life that inspired you to pursure your occupation also eventually affect your rating of this movie?</p>
    <br>
    <br>
    <p>Do all the total result match up with your expectations?</p>
    <br>
    <br>
    <p>Perhaps you felt that everything went as expected and your likes are similar to your most direct peers.</p>
    <br>
    <br>
    <p>Or, maybe, you felt like you were an outcast among your peers and this only confirmed that more.</p>
    <br>
    <br>
    <p>Or, it could be, that none of this provided any sort of conclusion within you.</p>
    <br>
    <br>
    <p>Demographics affecting ratings of movies is likely a way deeper and more complex topic of the total culture and environment that a person experienced growing up with led them to be who they are today.</p>
    <br>
    <br>
    <p>As such, thinking about what led you to like or dislike anything in your life can be very difficult but also very worthwhile to challenge yourself and grow,</p>

  </div>
	<svelte:fragment slot="viz">
		<!-- Visualizations here -->
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

.storyTime {
  text-align: center;
  color: #666;
  height: 420vh; /* Make the div scrollable with a 200% view height */
  border: 2px solid #aaa;
  padding: 10px;
  margin: 10px;
}

.storyTime2 {
  text-align: center;
  color: #666;
  height: 80vh; /* Make the div scrollable with a 200% view height */
  border: 2px solid #aaa;
  padding: 10px;
  margin: 10px;
}

.data {
  transition: all 1s ease;
}
.totalcenter {
  display: flex;
  flex-direction: column;
}

.center {
  display: flex;
  flex-direction: column;
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