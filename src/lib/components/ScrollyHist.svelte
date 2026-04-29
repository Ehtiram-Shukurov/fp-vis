<script lang="ts">
  import { Scroll } from "$lib";
  import RatingsChart from "./histogramParams.svelte";
  import RatingsHistogram from "./histogram.svelte";
  import { fly,fade } from "svelte/transition";

  let progress = $state(0);
  let showComparison = $state(false);

  const thresholds = [0, 12, 25, 35, 45, 55, 65, 75, 85, 95]
  let activeIndex = $derived(
    thresholds.findLastIndex(t => progress >= t)
  )

  let isExploreStep = $derived(activeIndex === 9);

  const sections = [
    {
      label: "All Genres",
      description: "Over all movie ratings in the dataset, we see an average rating of 3.5. This distribution is about 1 point above the true middle (2.5), likely because active raters tend to be movie enthusiasts or are influenced by central tendency bias.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Healthcare workers",
      description: "With an average of 2.9, Healthcare workers have the lowest average rating with an astounding 569 1-star ratings across the occupation. Furthermore they are one of the few demographics with matching median and mode.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "Healthcare",
    },
    {
      label: "Unemployed people",
      description: "On the opposite end of the spectrum are unemployed people who had the highest average rating of 3.8 across all the genres. Unemployed people may include younger viewers who could have higher enjoyment in general.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "none",
    },
    {
      label: "Homemakers",
      description: "Homemakers disliked the children's genre much more than the other occupations. While we cannot be certain of the reasoning for this, one plausible idea is that homemakers who care for children are probably a bit sick of watching the same childrens movie over and over.",
      filterType: "genre",
      selectedGenre: "Children's",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "homemaker",
    },
    {
      label: "The documentary genre speaks to different people",
      description: "This is the distribution of ratings across all occupations for the documentary genre.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Artists seem to really enjoy the genre!",
      description: "",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "artist",
    },
    {
      label: "The same can't be said for the executives...",
      description: "Executive may simply have higher standards in life which could cause this. Or, the focus on work that comes with being an executive may rebound with less enjoyment for 'leisure' activities.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "executive",
    },
    {
      label: "Gender Effects in Horror",
      description: "Comparing male and female audiences under 25, the female distribution shows a significant skew towards lower ratings. Despite having less than half the total ratings of the male group, the female group has nearly as many 1-star reviews.",
      filterType: "genre",
      selectedGenre: "Horror",
      selectedAge: "Under 25",
      selectedGender: "M",
      selectedOccupation: "All",
      compare: true,
      secondParams: { gender: "F" }
    },
    {
      label: "Gender Effects in Documentaries",
      description: "While the male distribution is fairly standard, the female distribution is more proportionally skewed toward extremes (1s and 5s), suggesting a more 'love-it-or-hate-it' relationship with the genre.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedAge: "All",
      selectedGender: "M",
      selectedOccupation: "All",
      compare: true,
      secondParams: { gender: "F" }
    },
    {
      label: "Now explore it yourself.",
      description: "As you can see, there are many differences and patterns to be found when analyzing the ratings of certain demographics and sub-demographics. Filter by any genre, age group, gender, or occupation. Compare two distributions side by side to find your own patterns.",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
  ];
</script>

<Scroll bind:progress debounce={50}>
  
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