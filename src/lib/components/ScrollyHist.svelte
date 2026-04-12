<script lang="ts">
  import { Scroll } from "$lib";
  import RatingsChart from "./histogramParams.svelte";

  let progress = $state(0);

  const SECTION_SIZE = 100 / 7;

  let activeIndex = $derived(
    Math.min(Math.max(Math.floor(progress / SECTION_SIZE), 0), 6)
  );

  const sections = [
    {
      label: "FILLER TEXT",
      filterType: "genre",
      selectedGenre: "",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "FILLER TEXT",
      filterType: "genre",
      selectedGenre: "Action",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "FILLER TEXT",
      filterType: "genre",
      selectedGenre: "Drama",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "FILLER TEXT",
      filterType: "genre",
      selectedGenre: "Comedy",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "FILLER TEXT",
      filterType: "genre",
      selectedGenre: "Romance",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "FILLER TEXT",
      filterType: "genre",
      selectedGenre: "Thriller",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "FILLER TEXT",
      filterType: "genre",
      selectedGenre: "Sci-Fi",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
  ];
</script>

<Scroll bind:progress>
  {#each sections as section, i}
    <div class="scroll-card" class:active={i === activeIndex}>
      <span class="step-num">0{i + 1}</span>
      <h3>{section.label}</h3>
      <p>FILLER TEXT FILLER TEXT FILLER TEXT FILLER TEXT FILLER TEXT FILLER TEXT FILLER TEXT.</p>
    </div>
  {/each}

  <svelte:fragment slot="viz">
    <div class="viz-wrapper">
      <RatingsChart
        filterType={sections[activeIndex].filterType}
        selectedGenre={sections[activeIndex].selectedGenre}
        selectedMovieId={sections[activeIndex].selectedMovieId}
        selectedAge={sections[activeIndex].selectedAge}
        selectedGender={sections[activeIndex].selectedGender}
        selectedOccupation={sections[activeIndex].selectedOccupation}
      />
    </div>
  </svelte:fragment>
</Scroll>

<style>
  .scroll-card {
    padding: 28px 0;
    border-bottom: 1px solid #1e2530;
    transition: all 0.3s ease;
  }

  .scroll-card.active h3 {
    color: #e2e8f0;
  }

  .scroll-card.active p {
    color: #64748b;
  }

  .step-num {
    font-size: 13px;
    color: #2d3748;
    font-weight: 500;
  }

  h3 {
    font-size: 22px;
    font-weight: 500;
    color: #334155;
    margin: 8px 0 10px;
    transition: color 0.3s ease;
  }

  p {
    font-size: 15px;
    color: #334155;
    line-height: 1.7;
    max-width: 420px;
    margin: 0;
    transition: color 0.3s ease;
  }

  .footer-card {
    border-bottom: none;
    padding-top: 28px;
  }

  .viz-wrapper {
    width: 100%;
    height: 100%;
  }
</style>