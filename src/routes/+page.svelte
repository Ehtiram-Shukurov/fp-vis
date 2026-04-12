<script>
  import RatingsHistogram from '$lib/components/histogram.svelte'
  import ScrollyHist from '$lib/components/ScrollyHist.svelte';
  import AgeGenreChart from '$lib/components/AgeGenreChart.svelte'
  import CategoricalScatter from '$lib/components/categoricalScatter.svelte'
  let showComparison = false;
</script>

<div class="page">

  <header>
    <p class="subtitle">CSCI 5609 · MovieLens 100K · 1997–98</p>
    <h1>Dimension</h1>
    <p class="question">How does the demographic of users change how they watch movies?</p>
    <p class="info">100K ratings · 943 users · 1,682 movies · 18 genres</p>
    <p class="team">Dylan Lyon · Genevieve Gray · Ezra Shukurov · Jake O'Shaughnessy · Max Lalonde</p>
  </header>

  <section>
    <h2>Does your job change what you watch?</h2>
    <p>Some occupations rate certain genres noticeably higher or lower than others. Healthcare workers tend to rate Horror lower. Lawyers rate it higher than almost any other group. Filter by genre and occupation to explore.</p>
    
    <div style="flex: 1; min-width: 0;">
      <ScrollyHist />
    </div>

    <p>Your turn to explore the data</p>
    <button 
      on:click={() => showComparison = !showComparison}
      style="
        background: #111827;
        color: #94a3b8;
        border: 1px solid #1e2530;
        padding: 6px 14px;
        border-radius: 4px;
        cursor: pointer;
        margin-bottom: 20px;
        font-size: 14px;
      "
    >
      {showComparison ? 'Remove comparison' : '+ Add comparison'}
    </button>

    <div style="display: flex; gap: 20px;">
      <div style="flex: 1; min-width: 0;">
        <RatingsHistogram />
      </div>
      {#if showComparison}
        <div style="flex: 1; min-width: 0;">
          <RatingsHistogram />
        </div>
      {/if}
    </div>
  </section>

  <section>
    <h2>Does age change what you enjoy?</h2>
    <p>Genre preferences shift a lot across age groups. Film-Noir ratings climb steadily from under 3.0 for the youngest users to 4.0 for the oldest. Horror goes the other way. Toggle genres below to compare.</p>
    <AgeGenreChart />
  </section>
  
    <section>
    <h2>How do you compare?</h2>
    <p>Pick a movie you've seen and enter your age, gender, and occupation. Your rating gets highlighted on the chart so you can see how it compares to everyone else who rated the same film. The dot sizes show how many people share that rating, bigger means more people. Use the dropdown to switch between age, gender, and occupation on the x-axis.</p>
    <p>The green dot marks where you land. If others in your group gave the same rating, that dot turns red so you can spot it easily.</p>
    <CategoricalScatter />
  </section>

  <footer>
    <p>Dimension · CSCI 5609 · Spring 2025 · <a href="https://grouplens.org/datasets/movielens/" target="_blank">MovieLens 100K dataset</a></p>
  </footer>

</div>

<style>
  :global(body) {
    margin: 0;
    background: #0d1117;
    color: #e2e8f0;
    font-family: sans-serif;
  }

  .page {
    max-width: 1100px;
    margin: 0 auto;
    padding: 0 40px;
  }

  header {
    padding: 60px 0 50px;
    border-bottom: 1px solid #1e2530;
  }

  .subtitle {
    font-size: 13px;
    color: #4a5568;
    margin: 0 0 12px;
  }

  h1 {
    font-size: 64px;
    font-weight: 500;
    color: #e2e8f0;
    margin: 0 0 16px;
  }

  .question {
    font-size: 18px;
    color: #64748b;
    line-height: 1.6;
    margin: 0 0 20px;
  }

  .info {
    font-size: 14px;
    color: #4a5568;
    margin: 0 0 12px;
  }

  .team {
    font-size: 13px;
    color: #334155;
    margin: 0;
  }

  section {
    padding: 50px 0;
    border-bottom: 1px solid #1e2530;
  }

  h2 {
    font-size: 28px;
    font-weight: 500;
    color: #e2e8f0;
    margin: 0 0 10px;
  }

  section p {
    font-size: 16px;
    color: #64748b;
    margin: 0 0 30px;
    max-width: 620px;
    line-height: 1.7;
  }

  footer {
    padding: 24px 0;
    font-size: 13px;
    color: #2d3748;
  }

  footer a {
    color: #2d3748;
  }
</style>