<script>
// @ts-nocheck
  import { onMount } from 'svelte'
  import { fly, fade } from 'svelte/transition'
  import Scroll from '$lib/components/Scroll.svelte'
  import ScrollyHist from '$lib/components/ScrollyHist.svelte'
  import AgeGenreChart from '$lib/components/AgeGenreChart.svelte'
  import CategoricalScatter from '$lib/components/categoricalScatter.svelte'
  import GlyphChart from '$lib/components/GlyphChart.svelte'

  let introProgress = $state(0)
  let sceneVisible = $state([false, false, false, false])

  onMount(() => {
    const sceneObs = new IntersectionObserver(entries => {
      entries.forEach(e => {
        const i = parseInt(e.target.dataset.i)
        sceneVisible[i] = e.isIntersecting
      })
    }, { 
      threshold: 0.03
    })

    document.querySelectorAll('.scene').forEach((el, i) => {
      el.dataset.i = i
      sceneObs.observe(el)
    })
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

      {#if introProgress > 45}
        <p class="intro-team" in:fly={{ y: 15, duration: 500 }}>
          Dylan Lyon · Genevieve Gray · Ezra Shukurov · Jake O'Shaughnessy · Max Lalonde
        </p>
      {/if}

      {#if introProgress > 65}
        <div class="intro-stats" in:fly={{ y: 20, duration: 600 }}>
          <span class="stat animate" data-target="100000"></span> ratings
          <span class="stat animate" data-target="943"></span> users
          <span class="stat animate" data-target="1682"></span> movies
          <span class="stat animate" data-target="18"></span> genres
        </div>
      {/if}

      {#if introProgress > 80}
        <p class="intro-explain" in:fly={{ y: 15, duration: 500 }}>
          These 100,000 ratings, collected from 1997-98, rate a movie on a solid scale from 1-5. Each rating correlated to a user who self-reports relevant demographic information.
        </p>
      {/if}
      {#if introProgress > 95}
        <p class="intro-explain" in:fly={{ y: 15, duration: 500 }}>
          This data is analyzed and visualized to display concrete patterns and offer possible hypotheses. 
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
<div class="scene" data-i="0" class:scene-visible={sceneVisible[0]}>
  <div class="pinned-header">
    <div class="section-tag">01 · Occupation × Genre</div>
    <h2>Does your job change what you watch?</h2>
    <p>Some occupations rate certain genres noticeably higher or lower than others. Healthcare workers tend to rate Horror lower. Lawyers rate it higher than almost any other group.</p>
  </div>
  <div class="scene-body">
    <ScrollyHist />
  </div>
</div>

<!-- section 02 — age × genre -->
<div class="scene" data-i="1" class:scene-visible={sceneVisible[1]}>
  <div class="pinned-header">
    <div class="section-tag">02 · Age × Genre</div>
    <h2>Does age change what you enjoy?</h2>
    <p>Genre preferences shift noticeably across age groups. Some genres climb steadily with age, others drop off sharply.</p>
  </div>
  <div class="scene-body">
    <AgeGenreChart />
  </div>
</div>


<!-- section 03 — scatter -->
<div class="scene" data-i="2" class:scene-visible={sceneVisible[2]}>
  <div class="pinned-header">
    <div class="section-tag">03 · How you compare</div>
    <h2>How do you compare?</h2>
  </div>
  <div class="scene-body">
    <CategoricalScatter />
  </div>
</div>

<!-- section 04 — glyph -->
<div data-i="3" class:scene-visible={sceneVisible[3]}>
  <div class="pinned-header">
    <div class="section-tag">04 · Glyph exploration</div>
    <h2>Every demographic at a glance.</h2>
    <p>Each glyph represents an age × occupation group. The spokes show genre preferences, longer means higher average rating for that genre.</p>
  </div>
  <div class="scene-body">
    <GlyphChart />
  </div>
</div>

<footer>
  <p>Dimension · CSCI 5609 · Spring 2025 · <a href="https://grouplens.org/datasets/movielens/" target="_blank">MovieLens 100K dataset</a></p>
</footer>

<style>

  @property --num {
    syntax: "<integer>";
    initial-value: 0;
    inherits: false;
  }

  .stat {
    --num: 0;
    counter-reset: num var(--num);
  }

  .stat::after {
    content: counter(num);
  }

  /* One keyframe per unique target number */
  @keyframes count-to-100000 { to { --num: 100000; } }
  @keyframes count-to-943    { to { --num: 943; } }
  @keyframes count-to-1682   { to { --num: 1682; } }
  @keyframes count-to-18     { to { --num: 18; } }

  .stat.animate[data-target="100000"] { animation: count-to-100000 2s ease forwards; }
  .stat.animate[data-target="943"]    { animation: count-to-943    1.5s ease forwards; }
  .stat.animate[data-target="1682"]   { animation: count-to-1682   1.8s ease forwards; }
  .stat.animate[data-target="18"]     { animation: count-to-18     1s ease forwards; }

  :global(.scrolly) {
    padding: 0 60px;
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
    font-size: 15px;
    color: var(--text-muted, #64748b);
    letter-spacing: 0.15em;
    text-transform: uppercase;
    margin: 0 0 24px;
  }

  h1 {
    font-size: clamp(70px, 10vw, 130px);
    font-weight: 500;
    color: var(--text-primary, #e2e8f0);
    margin: 0 0 32px;
    letter-spacing: -0.03em;
  }

  .intro-question {
    font-size: clamp(20px, 2.5vw, 28px);
    color: var(--text-primary, #e2e8f0);
    max-width: 700px;
    line-height: 1.5;
    margin: 0 0 40px;
  }

  .intro-stats {
    display: flex;
    gap: 32px;
    font-size: 18px;
    font-weight: 500;
    color: var(--text-muted, #ffffff);
    margin: 0 0 40px; 
  }

  .intro-team {
    font-size: 15px;
    color: var(--text-secondary, #64748b);
    margin: 0 0 40px;
  }

  .intro-explain {
    font-size: 18px;
    color: var(--text-secondary, #94a3b8);
    max-width: 750px;
    line-height: 1.7;
    margin: 0 0 24px;
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
.scene {
    position: relative;
    width: 100%;
    margin-bottom: 50vh;
    
    opacity: 0;
    transform: translateY(40px);
    transition: opacity 0.8s ease-out, transform 0.8s ease-out;
  }

  .scene.scene-visible {
    opacity: 1;
    transform: translateY(0);
  }

  .pinned-header {
    position: sticky;
    top: 0;
    z-index: 20;
    padding: 40px 60px 60px 60px; 
    width: 100%; 
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    align-items: center; 
    text-align: center;
    background: linear-gradient(to bottom, var(--bg-dark) 75%, transparent);
    pointer-events: none; 
  }

  .scene-body {
    position: relative;
    z-index: 10;
    pointer-events: auto;
    margin-top: 20vh; 
  }

  .section-tag {
    font-size: 14px;
    color: var(--text-muted);
    letter-spacing: 0.15em;
    text-transform: uppercase;
    margin-bottom: 16px;
  }

  .pinned-header h2 {
    font-size: 40px;
    color: var(--text-primary);
    margin: 0 0 16px;
    font-weight: 500;
  }

  .pinned-header p {
    font-size: 18px;
    color: var(--text-secondary);
    max-width: 800px;
    line-height: 1.6;
    margin: 0;
  }

  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(6px); }
  }

  footer {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 32px 60px;
    font-size: 14px;
    color: #2d3748;
    border-top: 1px solid #1e2530;
    margin-top: 80px;
  }

  footer a { color: #2d3748; }
</style>