<script lang="ts">
  import { Scroll } from "$lib";
  import RatingsChart from "./histogramParams.svelte";
  import RatingsHistogram from "./histogram.svelte";
  import { fly } from "svelte/transition";

  let progress = $state(0);
  let showComparison = $state(false);

  const SECTION_SIZE = 100 / 9;

  let activeIndex = $derived(
    Math.min(Math.max(Math.floor(progress / SECTION_SIZE), 0), 8)
  );

  let isExploreStep = $derived(activeIndex === 8);

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
      selectedOccupation: "healthcare",
    },
    {
      label: "Unemployed people",
      description: "On the opposite end of the spectrum are unemplyed people who had the highest average rating of 3.8 across all the genres.",
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
      description: "",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "executive",
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

<Scroll bind:progress debounce={150}>
  {#each sections as section, i}
    <div class="scroll-card" class:active={i === activeIndex}>
      {#if i < 8}
        <span class="step-num">{String(i + 1).padStart(2, '0')}</span>
      {/if}
      <h3>{section.label}</h3>
      <p>{section.description}</p>
    </div>
  {/each}

  <svelte:fragment slot="viz">
    <div class="viz-wrapper">
      {#if !isExploreStep}
        <RatingsChart
          filterType={sections[activeIndex].filterType}
          selectedGenre={sections[activeIndex].selectedGenre}
          selectedMovieId={sections[activeIndex].selectedMovieId}
          selectedAge={sections[activeIndex].selectedAge}
          selectedGender={sections[activeIndex].selectedGender}
          selectedOccupation={sections[activeIndex].selectedOccupation}
        />
      {:else}
        <div class="explore-wrap" in:fly={{ y: 20, duration: 500 }}>
          <button
            onclick={() => showComparison = !showComparison}
            class="compare-btn"
          >
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
      {/if}
    </div>
  </svelte:fragment>
</Scroll>

<style>
  .scroll-card {
    min-height: 55vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 28px 0;
    border-bottom: 1px solid #1e2530;
    opacity: 0.3;
    transition: opacity 0.3s ease;
  }

  .scroll-card.active {
    opacity: 1;
  }

  .step-num {
    font-size: 12px;
    color: #4a5568;
    font-weight: 500;
    margin-bottom: 8px;
  }

  h3 {
    font-size: 24px;
    font-weight: 500;
    color: #e2e8f0;
    margin: 0 0 10px;
  }

  p {
    font-size: 15px;
    color: #64748b;
    line-height: 1.7;
    max-width: 420px;
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