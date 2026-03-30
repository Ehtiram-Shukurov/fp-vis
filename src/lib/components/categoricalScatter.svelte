<script>
	import { onMount } from 'svelte'
  import * as d3 from "d3";
    import { derived } from 'svelte/store';
    import Page from '../../routes/+page.svelte';

	let allRatings = $state([]);
	let movies = $state({});
	let users = $state({});
	let loading = $state(true);
	let error = $state("");

	const genreNames = [
		"unknown", "Action", "Adventure", "Animation", "Children's", "Comedy",
		"Crime", "Documentary", "Drama", "Fantasy", "Film-Noir", "Horror",
		"Musical", "Mystery", "Romance", "Sci-Fi", "Thriller", "War", "Western"
	]

	onMount(async () => {
		try {
			const ratingsRes = await fetch('/u.data')
			const ratingsText = await ratingsRes.text()
			allRatings = ratingsText.trim().split('\n').map(line => {
				const parts = line.split('\t')
				return {
					userId: parseInt(parts[0]),
					movieId: parseInt(parts[1]),
					rating: parseInt(parts[2])
				}
			})

			const itemsRes = await fetch('/u.item')
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

			const usersRes = await fetch('/u.user')
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
    let width= $derived(xAxis === "Occupation" ? 1200 : 600);
    
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
</script>

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
    <h2>*!Directions for Below(categorical scatterplot)!*</h2>
    <p>This visualization is designed to visualize and get the reader to think about their own life/demographics growing up and how this may have affected them. Below are a series of selection boxes. First off, we would prompt the reader to choose any movie they have seen/are interested in with the first drop down menu. Then we would ask the reader to input some of their demographic information as well as their personal rating for that movie in the following selection boxes.</p>
    <p>Once a movie is selected then the data for that movie will appear on the categorical scatterplot. After the rest of the demographic info is filled in, their personal rating will be highlighted on the screen to help them compare to the surround ratings/demographics. The reader can choose the x-axis with the "Select an attribute to compare over" dropdown in order to consider the three different demographic attributes. </p>
    <p>Our highlighting is done in two parts: first - if there is a circle already created for the same infromation the user provided then the color is changed to red in order to highlight. Second - a different green circle is created on top of the red circle (but smaller) which emphasizes the highlight as well as provides a separate marker to look at if there is no ratings in the dataset with the same demographics.(in this case you would only see the green circle, not the red circle.)</p>
    <p>On top of each circle is a number which is just the numerical value of ratings that are the same rating+demographic attribute for that movie. The size of the dots also communicates this more effectively/interestingly.</p>
    <p>Story Wise, we plan to explore the dataset and make the reader think about how their culture/different demographics may have affected their lives to lead to them enjoying certain movies over others (or not). It is a very complex topic so we just want to try to make this reader think about what in their lives may have caused them to like certain movies over others(and if similar occupations for example may have chose that occupation/liked the same movies because of shared cultural experiences). This visualization just provides them with a direct comparison of a small collection of demographic attributes.</p>
    <div class="border">
      <h1>Categorical Scatterplot and How You Compare!</h1>
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

      <div>
        <label for="xAxis">Select an Attribute to Compare Over:</label>
        <select id="xAxis" bind:value={xAxis}>
          <option value="Age">Age</option>
          <option value="Gender">Gender</option>
          <option value="Occupation">Occupation</option>
        </select>
      </div>
    </div>

    <div class="center">

      <p>Total ratings: {filteredRatings.length} | Average: {avgRating}</p>
      <div class="border">
        <svg {height} {width}>
          <g>
            {#each graphPoints as point}
              <circle
                cx={xScale(point.dataCategory) + xScale.bandwidth() / 2}
                cy={yScale(String(point.userRating)) + yScale.bandwidth() / 2}
                r={0.4 * point.count}
                fill={point.dataCategory===currentUserData && String(point.userRating)===userRating ? "red" : "steelblue"}
                stroke="black"
                stroke-width="2"
                opacity={0.8}
              />
              <circle
                cx={xScale(currentUserData) + xScale.bandwidth() / 2}
                cy={(userRating === -8) ? -20 : yScale(String(userRating)) + yScale.bandwidth() / 2}
                r={10}
                fill={(userRating === -8) ? "white" : "green"}
                stroke={(userRating === -8) ? "white" : "green"}
                stroke-width="2"
                opacity={0.9}
              />
              <text
                x={xScale(point.dataCategory) + xScale.bandwidth() / 2}
                y={yScale(String(point.userRating)) + yScale.bandwidth() / 2 - (0.4 * point.count + 6)} 
                text-anchor="middle"
                dominant-baseline="middle">
                {point.count}
              </text>
            {/each}

            {#each ["1", "2", "3", "4", "5"] as yMark}
              <text x={usableArea.left - 10} y={yScale(yMark) + yScale.bandwidth() / 2} dominant-baseline="middle">{yMark}</text>
            {/each}

            {#each xAxisCategories as xMark}
              <text x={xScale(xMark) + xScale.bandwidth() / 2} y={usableArea.bottom} text-anchor="middle" transform="rotate(-45, {xScale(xMark) + xScale.bandwidth() / 2}, {usableArea.bottom})">{xMark}</text>
            {/each}

            <line 
              x1={usableArea.left+8}
              x2={usableArea.left+8}
              y1={usableArea.top}
              y2={usableArea.bottom}
              stroke="black"
              stroke-width="1"
            ></line>

          </g>
        </svg>
      </div>
    </div>
  {/if}
</div>


<style>
.totalcenter{
  display: flex;
  justify-content: center;
  flex-direction: column;
  align-items: center;
}

.center{
  display: flex;
  justify-content: center;
  flex-direction: column;
  align-items: center;
}

.border{
  border: 2px solid black;
  background-color: rgb(189, 189, 189);
}
</style>
