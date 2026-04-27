<script>
  import { onMount } from 'svelte'
  import { fly, fade } from 'svelte/transition'
  import Scroll from '$lib/components/Scroll.svelte'
  import ScrollyHist from '$lib/components/ScrollyHist.svelte'
  import AgeGenreChart from '$lib/components/AgeGenreChart.svelte'
  import CategoricalScatter from '$lib/components/categoricalScatter.svelte'
  import GlyphChart from '$lib/components/GlyphChart.svelte'

  let introProgress = $state(0)
  let visible = $state([false, false, false, false])
  let bodyVisible = $state([false, false, false, false]) // NEW: Tracks the charts

  onMount(() => {
    const headerObs = new IntersectionObserver(entries => {
      entries.forEach(e => {
        const i = parseInt(e.target.dataset.i)
        visible[i] = e.isIntersecting
      })
    }, { threshold: 0.65 })

    document.querySelectorAll('.section-header').forEach((el, i) => {
      el.dataset.i = i
      headerObs.observe(el)
    })

    const bodyObs = new IntersectionObserver(entries => {
      entries.forEach(e => {
        const i = parseInt(e.target.dataset.i)
        bodyVisible[i] = e.isIntersecting
      })
    }, { threshold: 0.03 }) // Triggers right as the body slides into view

    document.querySelectorAll('.section-body').forEach((el, i) => {
      el.dataset.i = i
      bodyObs.observe(el)
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
        <div class="intro-stats" in:fly={{ y: 20, duration: 600 }}>
          <span>100K ratings</span>
          <span>943 users</span>
          <span>1,682 movies</span>
          <span>18 genres</span>
        </div>
      {/if}

      {#if introProgress > 65}
        <p class="intro-team" in:fly={{ y: 15, duration: 500 }}>
          Dylan Lyon · Genevieve Gray · Ezra Shukurov · Jake O'Shaughnessy · Max Lalonde
        </p>
      {/if}

      {#if introProgress > 80}
        <p class="intro-explain" in:fly={{ y: 15, duration: 500 }}>
          The following exploration is built on 100,000 movie ratings collected from a rating site in 1997 and 1998. 
          Each rating is a score from 1 to 5, and every reviewer self-reported their own demographic information.
        </p>
      {/if}
      {#if introProgress > 95}
        <p class="intro-explain" in:fly={{ y: 15, duration: 500 }}>
          We will analyze this data, demonstrate the patterns we found, and offer some hypotheses along the way. 
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
<div class="section-header" data-i="0">
  <div class="header-content">
    {#if visible[0]}
      <div in:fly={{ y: 30, duration: 800 }} out:fade={{ duration: 400 }}>
        <div class="section-tag">01 · Occupation × Genre</div>
        <h2>Does your job change what you watch?</h2>
        <p>Some occupations rate certain genres noticeably higher or lower than others. Healthcare workers tend to rate Horror lower. Lawyers rate it higher than almost any other group.</p>
      </div>
    {/if}
  </div>
</div>

<div class="section-body" data-i="0" class:body-visible={bodyVisible[0]}>
  <ScrollyHist />
</div>

<!-- section 02 — age × genre -->
<div class="section-header" data-i="1">
  <div class="header-content">
    {#if visible[1]}
      <div in:fly={{ y: 30, duration: 800 }} out:fade={{ duration: 400 }}>
        <div class="section-tag">02 · Age × Genre</div>
        <h2>Does age change what you enjoy?</h2>
        <p>Genre preferences shift noticeably across age groups. Some genres climb steadily with age, others drop off sharply.</p>
      </div>
    {/if}
  </div>
</div>
<div class="section-body" data-i="1" class:body-visible={bodyVisible[1]}>
  <AgeGenreChart />
</div>


<!-- section 03 — scatter -->
<div class="section-header" data-i="2">
  <div class="header-content">
    {#if visible[2]}
      <div in:fly={{ y: 30, duration: 800 }} out:fade={{ duration: 400 }}>
        <div class="section-tag">03 · How you compare</div>
        <h2>How do you compare?</h2>
      </div>
    {/if}
  </div>
</div>
<div class="section-body" data-i="2" class:body-visible={bodyVisible[2]}>
  <CategoricalScatter />
</div>

<!-- section 04 — glyph -->
<div class="section-header" data-i="3">
  <div class="header-content">
    {#if visible[3]}
      <div in:fly={{ y: 30, duration: 800 }} out:fade={{ duration: 400 }}>
        <div class="section-tag">04 · Glyph exploration</div>
        <h2>Every demographic at a glance.</h2>
        <p>Each glyph represents an age × occupation group. The spokes show genre preferences, longer means higher average rating for that genre.</p>
      </div>
    {/if}
  </div>
</div>
<div class="section-body" data-i="3" class:body-visible={bodyVisible[3]}>
  <GlyphChart />
</div>

<footer>
  <p>Dimension · CSCI 5609 · Spring 2025 · <a href="https://grouplens.org/datasets/movielens/" target="_blank">MovieLens 100K dataset</a></p>
</footer>

<style>
  :global(.scrolly) {
    padding: 0 60px;
  }
  .section-body {
    opacity: 0;
    transition: opacity 0.4s ease-out;
  }
  
  .section-body.body-visible {
    opacity: 1;
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
    font-size: 16px;
    font-weight: 500;
    color: var(--text-muted, #94a3b8);
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