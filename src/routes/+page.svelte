<script>
  import { onMount } from 'svelte'
  import { fly } from 'svelte/transition'
  import Scroll from '$lib/components/Scroll.svelte'
  import ScrollyHist from '$lib/components/ScrollyHist.svelte'
  import AgeGenreChart from '$lib/components/AgeGenreChart.svelte'
  import CategoricalScatter from '$lib/components/categoricalScatter.svelte'
  import GlyphChart from '$lib/components/GlyphChart.svelte'

  let introProgress = $state(0)
  let visible = $state([false, false, false, false])

  onMount(() => {
    const observer = new IntersectionObserver(entries => {
      entries.forEach(e => {
        const i = parseInt(e.target.dataset.i)
        if (e.isIntersecting) {
          visible[i] = true
          observer.unobserve(e.target)
        }
      })
    }, { threshold: 0.1 })

    document.querySelectorAll('.section-anchor').forEach(el => observer.observe(el))
  })
</script>

<!-- intro -->
<Scroll bind:progress={introProgress} --scrolly-story-width="0">
  <div id="intro-virtual"></div>
  <svelte:fragment slot="viz">
    <div class="intro-hero">
      <p class="eyebrow">CSCI 5609 · MovieLens 100K · 1997–98</p>
      <h1>Dimension</h1>

      {#if introProgress > 25}
        <p class="intro-question" in:fly={{ y: 30, duration: 700 }}>
          How does the demographic of users change how they watch movies?
        </p>
      {/if}

      {#if introProgress > 55}
        <div class="intro-stats" in:fly={{ y: 20, duration: 600 }}>
          <span>100K ratings</span>
          <span>943 users</span>
          <span>1,682 movies</span>
          <span>18 genres</span>
        </div>
      {/if}

      {#if introProgress > 75}
        <p class="intro-team" in:fly={{ y: 15, duration: 500 }}>
          Dylan Lyon · Genevieve Gray · Ezra Shukurov · Jake O'Shaughnessy · Max Lalonde
        </p>
      {/if}

      {#if introProgress > 85}
        <p class="intro-explain" in:fly={{ y: 15, duration: 500 }}>
          The following exploration considers a dataset 100,000 strong. It consists of data taken from a movie rating site wherein ratings for individual movies are one of [1, 2, 3, 4, 5] and each reviewer has self-reported their own demographic information.
        </p>
      {/if}
      {#if introProgress > 95}
        <p class="intro-explain" in:fly={{ y: 15, duration: 500 }}>
          We will analyze this data, demonstrate the facts in the dataset, and provide some small hypotheses for explanation.
          Please continue scrolling and enjoy!
        </p>
      {/if}

      {#if introProgress < 15}
        <div class="scroll-hint">
          <span>scroll</span>
          <div class="scroll-arrow"></div>
        </div>
      {/if}
    </div>
  </svelte:fragment>
</Scroll>

<!-- section 01 — occupation × genre -->
<div class="section-header">
  <div class="section-anchor" data-i="0"></div>
  {#if visible[0]}
    <div in:fly={{ y: 30, duration: 600 }}>
      <div class="section-tag">01 · Occupation × Genre</div>
      <h2>Does your job change what you watch?</h2>
      <p>Some occupations rate certain genres noticeably higher or lower than others. Healthcare workers tend to rate Horror lower. Lawyers rate it higher than almost any other group.</p>
    </div>
  {/if}
</div>
<ScrollyHist />

<!-- section 02 — age × genre -->
<div class="section-header">
  <div class="section-anchor" data-i="1"></div>
  {#if visible[1]}
    <div in:fly={{ y: 30, duration: 600 }}>
      <div class="section-tag">02 · Age × Genre</div>
      <h2>Does age change what you enjoy?</h2>
      <p>Genre preferences shift noticeably across age groups. Some genres climb steadily with age, others drop off sharply.</p>
    </div>
  {/if}
</div>
<AgeGenreChart />

<!-- section 03 — scatter -->
<div class="section-header">
  <div class="section-anchor" data-i="2"></div>
  {#if visible[2]}
    <div in:fly={{ y: 30, duration: 600 }}>
      <div class="section-tag">03 · How you compare</div>
      <h2>How do you compare?</h2>
    </div>
  {/if}
</div>
<CategoricalScatter />

<!-- section 04 — glyph -->
<div class="section-header">
  <div class="section-anchor" data-i="3"></div>
  {#if visible[3]}
    <div in:fly={{ y: 30, duration: 600 }}>
      <div class="section-tag">04 · Glyph exploration</div>
      <h2>Every demographic at a glance.</h2>
      <p>Each glyph represents an age × occupation group. The spokes show genre preferences, longer means higher average rating for that genre.</p>
    </div>
  {/if}
</div>
<GlyphChart />

<footer>
  <p>Dimension · CSCI 5609 · Spring 2025 · <a href="https://grouplens.org/datasets/movielens/" target="_blank">MovieLens 100K dataset</a></p>
</footer>

<style>
  :global(body) {
    margin: 0;
    background: #0d1117;
    color: #e2e8f0;
    font-family: sans-serif;
    overflow-x: hidden;
  }

  :global(.scrolly) {
    padding: 0 60px;
  }

  .section-header {
    max-width: 1100px;
    margin: 0 auto;
    padding: 60px 60px 40px;
    border-top: 1px solid #1e2530;
    min-height: 60px;
  }

  .section-anchor {
    height: 0;
    width: 0;
  }

  .section-tag {
    font-size: 12px;
    color: #4a5568;
    text-transform: uppercase;
    letter-spacing: 0.12em;
    margin-bottom: 14px;
  }

  .section-header h2 {
    font-size: 36px;
    font-weight: 500;
    color: #e2e8f0;
    margin: 0 0 12px;
    letter-spacing: -0.01em;
  }

  .section-header p {
    font-size: 17px;
    color: #64748b;
    max-width: 600px;
    line-height: 1.7;
    margin: 0;
  }

  #intro-virtual { height: 200vh; }

  .intro-hero {
    position: relative;
    width: 100%;
    height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    padding: 0 40px;
    box-sizing: border-box;
    background: #0d1117;
  }

  .eyebrow {
    font-size: 12px;
    color: #4a5568;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    margin: 0 0 20px;
  }

  h1 {
    font-size: clamp(60px, 10vw, 120px);
    font-weight: 500;
    color: #e2e8f0;
    margin: 0 0 24px;
    letter-spacing: -0.03em;
  }

  .intro-question {
    font-size: clamp(16px, 2vw, 22px);
    color: #64748b;
    max-width: 580px;
    line-height: 1.6;
    margin: 0 0 28px;
  }

  .intro-explain {
    font-size: 14px;
    color: #64748b;
    max-width: 580px;
    line-height: 1.6;
    margin: 0 0 28px;
  }

  .intro-stats {
    display: flex;
    gap: 28px;
    font-size: 13px;
    color: #4a5568;
    margin: 0 0 20px;
  }

  .intro-team {
    font-size: 12px;
    color: #2d3748;
    margin: 0;
  }

  .scroll-hint {
    position: absolute;
    bottom: 40px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    font-size: 11px;
    color: #2d3748;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    animation: bounce 2s ease-in-out infinite;
  }

  .scroll-arrow {
    width: 1px;
    height: 36px;
    background: linear-gradient(to bottom, #2d3748, transparent);
  }

  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(6px); }
  }

  footer {
    padding: 32px 60px;
    font-size: 12px;
    color: #2d3748;
    border-top: 1px solid #1e2530;
    margin-top: 80px;
  }

  footer a { color: #2d3748; }
</style>