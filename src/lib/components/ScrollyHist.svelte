<script lang="ts">
  import { Scroll } from "$lib";
  import RatingsChart from "./histogramParams.svelte";
  import RatingsHistogram from "./histogram.svelte";
  import { fly,fade } from "svelte/transition";

  let progress = $state(0);
  let showComparison = $state(false);

  const thresholds = [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 97];
  let activeIndex = $derived(
    thresholds.findLastIndex(t => progress >= t)
  )

  let isExploreStep = $derived(activeIndex === 10);

  const sections = [
    {
      label: "Most of us think we're objective viewers. We're not.",
      description: "Across all ratings in the dataset, the average sits at 3.5, which a full point above the true midpoint of 2.5. Active raters skew enthusiastic by nature. But look closer: the real story isn't the average. It's how differently that distribution shifts depending on who's watching.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Students watch to feel. Critics watch to judge.",
      description: "Students rate generously, their distribution clusters around 4s and 5s. For them, a movie is an experience to enjoy, not evaluate. The more a profession demands critical thinking and high standards, the harder it becomes to simply switch that off.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "student",
    },
    {
      label: "Executives are harder to please.",
      description: "Executives show a notably flatter, more skeptical distribution. When your days are spent making high-stakes judgments, you may bring that same scrutiny to the cinema. Time is scarce and mediocre films don't make the cut.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "executive",
    },
    {
      label: "Healthcare workers are the toughest crowd of all.",
      description: "With an average rating of just 2.9, healthcare workers rate lower than any other occupation, including a striking 569 one-star ratings. When your job confronts you with real human suffering daily, it may raise the bar for what feels meaningful or worth your time on screen.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "healthcare",
    },
    {
      label: "And then there are the unemployed.",
      description: "At the opposite extreme, unemployed viewers average 3.8, the highest of any group. When a film isn't an escape from work but simply your afternoon, maybe it just hits differently. The pressure to justify the time is gone.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "none",
    },
    {
      label: "Genre reveals the gap even more.",
      description: "Overall averages only tell part of the story. When we filter by genre, occupational identity starts to look less like a quirk and more like a lens, one that different genres either magnify or dissolve entirely.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "For artists, documentaries aren't leisure, they're research.",
      description: "Artists skew heavily toward 4 and 5-star ratings for documentaries. The genre aligns with how they already move through the world: attentive to reality, searching for meaning, drawn to non-fiction as a creative form in itself.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "artist",
    },
    {
      label: "Executives, on the other hand, are unmoved.",
      description: "The executive distribution for documentaries is notably flat and skeptical. Whether it's higher personal standards, less patience for slow-burn storytelling, or simply less appetite for work-adjacent content during leisure time, documentaries don't land the same way.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "executive",
    },
    {
      label: "Gender shapes how you experience a documentary.",
      description: "Men rate documentaries with a familiar peak around 4. Women's ratings, however, spread toward both extremes, more 5s and 1s. The documentary genre seems to polarize female viewers in a way it doesn't for men: a stronger love-it-or-hate-it reaction.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedAge: "All",
      selectedGender: "M",
      selectedOccupation: "All",
      compare: true,
      secondParams: { gender: "F" }
    },
    {
      label: "In horror, the gender gap is even starker.",
      description: "Among viewers under 25, women rate horror significantly lower than men. The female group has less than half the total ratings, yet nearly as many 1-star reviews. Horror, as a genre, is not a shared experience. Who you are changes what you're watching.",
      filterType: "genre",
      selectedGenre: "Horror",
      selectedAge: "Under 25",
      selectedGender: "M",
      selectedOccupation: "All",
      compare: true,
      secondParams: { gender: "F" }
    },
    {
      label: "Now find your own patterns.",
      description: "Who you are changes what you watch. Explore it yourself, filter by genre, age, gender, or occupation and compare two distributions side by side.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
  ];
</script>

<Scroll bind:progress debounce={50} --scrolly-viz-width="2fr" --scrolly-story-width="1.5fr">
  
  <div class="sticky-text-wrap">
    <div class="card-container">
      {#key activeIndex} 
        <div class="scroll-card active" 
             in:fly={{ y: 20, duration: 600, delay: 200 }} 
             out:fly={{ y: -20, duration: 400 }}>
          
          {#if activeIndex < 12}
            <span class="step-num">{String(activeIndex + 1).padStart(2, '0')}</span>
          {/if}
          <h3>{sections[activeIndex].label}</h3>
          <p>{sections[activeIndex].description}</p>
        </div>
      {/key}
    </div>
  </div>

  <div class="scroll-track">
    {#each sections.slice(0, -1) as _, i}
      <div class="spacer"></div>
    {/each}
  </div>

  <svelte:fragment slot="viz">
    <div class="viz-wrapper">
      {#if isExploreStep}
        <div class="explore-wrap" in:fly={{ y: 20, duration: 500 }}>
          <button onclick={() => (showComparison = !showComparison)} class="compare-btn">
            {showComparison ? 'Remove comparison' : '+ Add comparison'}
          </button>
          <div class="side-by-side">
            <div class="chart-col"><RatingsHistogram /></div>
            {#if showComparison}
              <div class="chart-col" in:fly={{ x: 20, duration: 400 }}>
                <RatingsHistogram />
              </div>
            {/if}
          </div>
        </div>
      {:else if sections[activeIndex].compare}
        <div class="side-by-side" in:fly={{ y: 10, duration: 400 }}>
          <div class="chart-col">
            <RatingsChart
              {...sections[activeIndex]}
            />
          </div>
          <div class="chart-col">
            <RatingsChart
              {...sections[activeIndex]}
              selectedGender={sections[activeIndex].secondParams.gender}
            />
          </div>
        </div>
      {:else}
        <RatingsChart
          filterType={sections[activeIndex].filterType}
          selectedGenre={sections[activeIndex].selectedGenre}
          selectedMovieId={sections[activeIndex].selectedMovieId}
          selectedAge={sections[activeIndex].selectedAge}
          selectedGender={sections[activeIndex].selectedGender}
          selectedOccupation={sections[activeIndex].selectedOccupation}
        />
      {/if}
    </div>
  </svelte:fragment>
</Scroll>

<style>
.scroll-track {
    width: 100%;
  }
  .spacer {
    height: 120vh;
  }
  .spacer:last-child { 
    height: 40vh; 
  }

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

  h3 {
    font-size: 28px;
    font-weight: 500;
    color: var(--text-primary);
    margin: 0 0 12px;
  }

  p {
    font-size: 18px;
    color: var(--text-secondary);
    line-height: 1.6;
    margin: 0;
  }

  .viz-wrapper {
    width: 100%;
    height: 100%;
    padding-top: 80px;
  }

  .explore-wrap {
    padding: 20px 0;
    width: 100%;
  }

  .compare-btn {
    background: #111827;
    color: #94a3b8;
    border: 1px solid #1e2530;
    padding: 6px 14px;
    border-radius: 4px;
    cursor: pointer;
    margin-bottom: 16px;
    font-size: 13px;
    transition: background 0.2s;
  }

  .compare-btn:hover {
    background: #2d3748;
  }

  .side-by-side {
    display: flex;
    gap: 16px;
  }

  .chart-col {
    flex: 1;
    min-width: 0;
  }
</style>