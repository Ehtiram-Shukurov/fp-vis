<script lang="ts">
  import { Scroll } from "$lib";
  import RatingsChart from "./histogramParams.svelte";

  let progress = $state(0);

  const SECTION_SIZE = 100 / 8;

  let activeIndex = $derived(
    Math.min(Math.max(Math.floor(progress / SECTION_SIZE), 0), 7)
  );

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
      label: "Children's",
      description: "With an average of 3.4, this genre notably shows a mode where 3-star and 4-star ratings are almost tied, contrasting the usual trend where 4-star ratings dominate significantly.",
      filterType: "genre",
      selectedGenre: "Children's",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Documentary",
      description: "Averaging 3.7, documentaries benefit from a niche audience. Since viewers typically seek out specific topics they already enjoy, the genre avoids the 'casual dislike' often seen in broader categories.",
      filterType: "genre",
      selectedGenre: "Documentary",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Drama",
      description: "At a 3.7 average, Drama is a heavyweight. As a popular and broad genre, its distribution closely mirrors the shape of the overall dataset, representing a reliable baseline for audience expectations.",
      filterType: "genre",
      selectedGenre: "Drama",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Fantasy",
      description: "A lower average of 3.2. This may stem from the 'source material curse'—readers often feel book-to-film adaptations fall short—and the inherent difficulty of executing complex world-building.",
      filterType: "genre",
      selectedGenre: "Fantasy",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Film-Noir",
      description: "The highest average at 3.9. Much like documentaries, Film-Noir attracts a specific cinephile demographic, resulting in fewer low ratings and a very high concentration of critical acclaim.",
      filterType: "genre",
      selectedGenre: "Film-Noir",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "Horror",
      description: "A lower average of 3.3. This is likely tied to the genre's objective: it is designed to provoke unease or fear, a visceral reaction that doesn't always translate to a high numerical comfort rating.",
      filterType: "genre",
      selectedGenre: "Horror",
      selectedMovieId: "",
      selectedAge: "All",
      selectedGender: "All",
      selectedOccupation: "All",
    },
    {
      label: "War",
      description: "Strong performance with a 3.8 average. The distribution is top-heavy, showing nearly identical counts for 4 and 5-star ratings, indicating high satisfaction among its viewership.",
      filterType: "genre",
      selectedGenre: "War",
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
      <span class="step-num">{String(i + 1).padStart(2, '0')}</span>
      <h3>{section.label}</h3>
      <p>{section.description}</p>
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
    min-height: 70vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    padding: 28px 0;
    border-bottom: 1px solid #1e2530;
    transition: all 0.3s ease;
  }

  .scroll-card.active h3 {
    color: #e2e8f0;
  }

  .scroll-card.active p {
    color: #94a3b8;
  }

  .step-num {
    font-size: 13px;
    color: #4a5568;
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
    color: #475569;
    line-height: 1.7;
    max-width: 420px;
    margin: 0;
    transition: color 0.3s ease;
  }

  .viz-wrapper {
    width: 100%;
    height: 100%;
  }
</style>