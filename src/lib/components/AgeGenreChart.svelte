<script>
// @ts-nocheck
  import { onMount } from 'svelte'
  import * as d3 from 'd3'
  import { genreColors } from '$lib/genreColors'
  import { Scroll } from '$lib'
  import { fly, fade } from 'svelte/transition'

  const genreNames = [
    'unknown', 'Action', 'Adventure', 'Animation', "Children's", 'Comedy',
    'Crime', 'Documentary', 'Drama', 'Fantasy', 'Film-Noir', 'Horror',
    'Musical', 'Mystery', 'Romance', 'Sci-Fi', 'Thriller', 'War', 'Western'
  ]

  const ageBuckets = [
    { label: '<18',   min: 0,  max: 17  },
    { label: '18-24', min: 18, max: 24  },
    { label: '25-34', min: 25, max: 34  },
    { label: '35-44', min: 35, max: 44  },
    { label: '45-54', min: 45, max: 54  },
    { label: '55-64', min: 55, max: 64  },
    { label: '65+',   min: 65, max: 999 },
  ]


const steps = [
  {
    label: "The sharpest divide",
    body: "Film-Noir ratings climb steadily as viewers get older. Horror does the exact opposite. They start at similar ratings and pull apart across every age group.",
    genres: ["Film-Noir", "Horror"]
  },
  {
    label: "Film-Noir is not alone",
    body: "Fantasy and Mystery follow the same upward path as Film-Noir. Younger viewers consistently rate all three lower, and older viewers consistently rate them higher. It seems like appreciation for slower, more atmospheric genres builds over time.",
    genres: ["Film-Noir", "Fantasy", "Mystery"]
  },
  {
    label: "Horror has company too",
    body: "Sci-Fi and Animation both skew toward younger viewers as well, though not as dramatically as Horror. Horror's drop is nearly a full point from the youngest group to the oldest.",
    genres: ["Horror", "Sci-Fi", "Animation"]
  },
  {
    label: "The mainstream barely moves",
    body: "Drama, Comedy, and Action are the most rated genres in the dataset by far. But despite all those opinions, age barely shifts how people feel about them. They sit flat around 3.5 to 3.7 across every group.",
    genres: ["Drama", "Comedy", "Action"]
  },
  {
    label: "Documentary peaks in the middle",
    body: "Documentary does not fit either pattern. It starts low for the youngest viewers, peaks around the 35 to 44 age group, then falls again for the oldest. It is the only genre with a clear bell curve shape.",
    genres: ["Documentary"]
  },
  {
    label: "See for yourself",
    body: "Toggle any genres to compare them directly. Dot size shows how many ratings each point is based on, so smaller dots mean less data.",
    genres: []
  }
]



const margin = { top: 40, right: 100, bottom: 40, left: 50 }
const width = 800 - margin.left - margin.right
const height = 380 - margin.top - margin.bottom

let progress = $state(0)
let loading = $state(true)
let error = $state('')
let genreData = $state({})
let manualSelected = $state(['Film-Noir', 'Horror', 'Fantasy'])
let tooltipVisible = $state(false)
let tooltipX = $state(0)
let tooltipY = $state(0)
let tooltipAge = $state('')
let tooltipRows = $state([])
let chartWrap

const thresholds = [0, 19, 38, 57, 76, 95];
let activeIndex = $derived(Math.max(0, thresholds.findLastIndex(t => progress >= t)))
let chartIndex = $derived(Math.max(activeIndex, 0))
let isExploreStep = $derived(chartIndex === steps.length - 1)
let selected = $derived(isExploreStep ? manualSelected : steps[chartIndex].genres)
  let xScale = $derived(
    d3.scalePoint()
      .domain(ageBuckets.map(b => b.label))
      .range([0, width])
      .padding(0.3)
  )

  let yScale = $derived(
    d3.scaleLinear()
      .domain([2.5, 4.5])
      .range([height, 0])
  )

  let yTicks = $derived(yScale.ticks(5))

  onMount(async () => {
    try {
      const base = import.meta.env.BASE_URL
      const [ratingsText, itemsText, usersText] = await Promise.all([
        fetch(`${base}u.data`).then(r => r.text()),
        fetch(`${base}u.item`).then(r => r.text()),
        fetch(`${base}u.user`).then(r => r.text()),
      ])

      // build user age map
      const userMap = {}
      usersText.trim().split('\n').forEach(line => {
        const parts = line.split('|')
        userMap[parseInt(parts[0])] = { age: parseInt(parts[1]) }
      })

      // build movie genre map
      const movieMap = {}
      itemsText.trim().split('\n').forEach(line => {
        const parts = line.split('|')
        const genres = genreNames.filter((_, i) => parts[5 + i] === '1')
        movieMap[parseInt(parts[0])] = genres
      })

      // accumulate ratings by genre + age bucket
      const acc = {}
      genreNames.forEach(g => {
        acc[g] = {}
        ageBuckets.forEach(b => acc[g][b.label] = { sum: 0, count: 0 })
      })

      ratingsText.trim().split('\n').forEach(line => {
        const parts = line.split('\t')
        const user = userMap[parseInt(parts[0])]
        const genres = movieMap[parseInt(parts[1])]
        const rating = parseInt(parts[2])
        if (!user || !genres) return

        const bucket = ageBuckets.find(b => user.age >= b.min && user.age <= b.max)
        if (!bucket) return

        genres.forEach(g => {
          acc[g][bucket.label].sum += rating
          acc[g][bucket.label].count += 1
        })
      })

      // compute averages
      const result = {}
      genreNames.forEach(g => {
        result[g] = ageBuckets.map(b => {
          const e = acc[g][b.label]
          return { age: b.label, avg: e.count > 0 ? parseFloat((e.sum / e.count).toFixed(2)) : null, count: e.count }
        })
      })

      genreData = result
      loading = false
    } catch (e) {
      error = "couldn't load data files"
      loading = false
    }
  })

  function getLinePath(genre) {
    const points = genreData[genre]
    if (!points) return ''
    return points
      .filter(p => p.avg !== null)
      .map((p, i) => `${i === 0 ? 'M' : 'L'} ${xScale(p.age)} ${yScale(p.avg)}`)
      .join(' ')
  }

  function getDots(genre) {
    return (genreData[genre] || []).filter(p => p.avg !== null).map(p => ({
      cx: xScale(p.age),
      cy: yScale(p.avg),
      r: Math.sqrt(p.count) * 0.25,
      age: p.age, avg: p.avg, count: p.count
    }))
  }

function animateLine(path) {
  const len = path.getTotalLength()
  d3.select(path)
    .attr('stroke-dasharray', len)
    .attr('stroke-dashoffset', len)
    .transition()
    .duration(800)
    .ease(d3.easeQuadOut)
    .attr('stroke-dashoffset', 0)
}

  function toggleGenre(genre) {
  if (manualSelected.includes(genre)) {
    manualSelected = manualSelected.filter(g => g !== genre)
  } else {
    manualSelected = [...manualSelected, genre]
  }
}

  function handleMouseMove(event, ageLabel) {
    const rect = chartWrap.getBoundingClientRect()
    tooltipX = event.clientX - rect.left + 12
    tooltipY = event.clientY - rect.top - 10
    tooltipAge = ageLabel
    tooltipVisible = true
    tooltipRows = selected
      .map(genre => {
        const point = genreData[genre]?.find(d => d.age === ageLabel)
        return point?.avg != null ? { genre, color: genreColors[genre], avg: point.avg, count: point.count } : null
      })
      .filter(Boolean)
  }

  function handleMouseLeave() {
    tooltipVisible = false
  }
</script>

<Scroll bind:progress debounce={50}>
  
  <div class="sticky-text-wrap">
    <div class="card-container">
      {#key activeIndex} 
        <div class="scroll-card active" 
             in:fly={{ y: 20, duration: 600, delay: 200 }} 
             out:fly={{ y: -20, duration: 400 }}>
          
          {#if activeIndex < steps.length - 1}
            <span class="step-num">{String(activeIndex + 1).padStart(2, '0')}</span>
          {/if}
          <h3>{steps[activeIndex].label}</h3>
          <p>{steps[activeIndex].body}</p>
        </div>
      {/key}
    </div>
  </div>

  <div class="scroll-track">
    {#each steps as _, i}
      <div class="spacer"></div>
    {/each}
  </div>

  <svelte:fragment slot="viz">
    <div class="viz-wrapper">
      {#if loading}
        <p style="color: #64748b">Loading data...</p>
      {:else if error}
        <p style="color: red">{error}</p>
			  {:else}

        {#if isExploreStep}
          <div class="toggles" style="padding-top: 40px;">
            {#each genreNames as genre}
              {#if genre !== 'unknown'}
                <button
                  onclick={() => toggleGenre(genre)}
                  style="border-color: {selected.includes(genre) ? genreColors[genre] : '#2d3748'}; color: {selected.includes(genre) ? genreColors[genre] : '#64748b'}"
                >
                  {genre}
                </button>
              {/if}
            {/each}
          </div>
        {/if}

        <div class="chart-wrap" bind:this={chartWrap}>
          <svg width={width + margin.left + margin.right} height={height + margin.top + margin.bottom}>
            <g transform="translate({margin.left}, {margin.top})">

              {#each yTicks as tick}
                <line x1={0} x2={width} y1={yScale(tick)} y2={yScale(tick)} stroke="var(--border-subtle)" stroke-dasharray="3,3" />
                <text x={-10} y={yScale(tick)} text-anchor="end" dominant-baseline="middle" fill="var(--text-muted)" font-size="15px">{tick}</text>
              {/each}

              {#each ageBuckets as bucket}
                <text x={xScale(bucket.label)} y={height + 25} text-anchor="middle" fill="var(--text-muted)" font-size="15px">{bucket.label}</text>
              {/each}

              <line x1={0} x2={width} y1={height} y2={height} stroke="#2d3748" />
              <line x1={0} x2={0} y1={0} y2={height} stroke="#2d3748" />

				{#each selected as genre (genre + activeIndex)}
        {@const dots = getDots(genre)}
        {@const lastDot = dots[dots.length - 1]}

        <path d={getLinePath(genre)} fill="none" stroke={genreColors[genre]} stroke-width="3" use:animateLine />
        
        {#each dots as dot}
          <circle cx={dot.cx} cy={dot.cy} r={dot.r} fill={genreColors[genre]} stroke="var(--bg-dark)" stroke-width="1.5" />
        {/each}

        {#if lastDot}
          <text
            x={lastDot.cx + 12}
            y={lastDot.cy}
            fill={genreColors[genre]}
            font-size="16px"
            font-weight="600"
            dominant-baseline="middle"
            in:fade={{ duration: 400, delay: 800 }}
          >
            {genre}
          </text>
        {/if}
      {/each}

            </g>
          </svg>

          {#if tooltipVisible}
            <div class="tooltip" style="left: {tooltipX}px; top: {tooltipY}px">
              <strong>{tooltipAge}</strong>
              {#each tooltipRows as row}
                <p style="color: {row.color}; margin: 4px 0 0">{row.genre}: {row.avg} ({row.count} ratings)</p>
              {/each}
            </div>
          {/if}
        </div>
      {/if}
    </div>
  </svelte:fragment>
</Scroll>

<style>
.scroll-track { width: 100%; }
  .spacer { height: 120vh; }
  

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
  button {
    background: transparent;
    border: 1px solid var(--border-subtle);
    border-radius: 20px;
    padding: 6px 14px;
    font-size: 14px;
    cursor: pointer;
    transition: all 0.2s;
  }

  .chart-wrap {
    position: relative;
    margin-top: 60px;
  }

  .toggles {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 20px;
  }

  button {
    background: none;
    border: 1px solid;
    border-radius: 20px;
    padding: 5px 14px;
    font-size: 13px;
    cursor: pointer;
  }

  .chart-wrap {
  position: relative;
  margin-top: 100px;
}

  .tooltip {
    position: absolute;
    background: #1e2530;
    border: 1px solid #2d3748;
    border-radius: 5px;
    padding: 8px 12px;
    pointer-events: none;
    font-size: 13px;
    color: #e2e8f0;
  }
</style>