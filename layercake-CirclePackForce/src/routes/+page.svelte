<script>
  import { LayerCake, Svg } from 'layercake';
  import { scaleOrdinal } from 'd3-scale';
  import { onMount } from 'svelte';

  import ForceLayout from './_components/CirclePackForce.svelte';
  import data from './_data/cannabis_dots.json';

  // Scroll state
  let scrollY = $state(0);
  let windowHeight = $state(800);
  let currentStage = $state(0);

  // Calculate stage based on scroll position
  $effect(() => {
    const scrollProgress = scrollY / windowHeight;
    if (scrollProgress < 0.8) {
      currentStage = 0;
    } else if (scrollProgress < 2.0) {
      currentStage = 1;
    } else if (scrollProgress < 3.2) {
      currentStage = 2;
    } else {
      currentStage = 3;
    }
  });

  // Color scheme for substances
  const substanceColors = {
    'Marijuana: Edible Preparation': '#E63946',      // Warm red
    'Marijuana: Dried Plant': '#2A9D8F',             // Teal
    'eCigarettes: Marijuana Device Flavor Unknown': '#E9C46A', // Gold
    'Other Marijuana Substances': '#457B9D'          // Steel blue
  };

  const seriesColors = Object.values(substanceColors);
  const substances = Object.keys(substanceColors);

  // Stats
  const totalCases = 542;
  const youthCases = 354;
  const youthPercent = ((youthCases / totalCases) * 100).toFixed(1);

  // Age group labels
  const ageGroups = ['<= 5 Years', '6 - 12 Years', '13 - 19 Years'];

  onMount(() => {
    windowHeight = window.innerHeight;
  });
</script>

<svelte:window bind:scrollY bind:innerHeight={windowHeight} />

<div class="scroll-container">
  <!-- Fixed visualization -->
  <div class="viz-container">
    <div class="chart-wrapper">
      <LayerCake
        {data}
        z="substance"
        zScale={scaleOrdinal()}
        zDomain={substances}
        zRange={seriesColors}
      >
        <Svg>
          <ForceLayout 
            stage={currentStage} 
            nodeStroke="rgba(255,255,255,0.3)" 
            nodeStrokeWidth={0.5}
            {substanceColors}
            {ageGroups}
          />
        </Svg>
      </LayerCake>
    </div>

  </div>

  <!-- Scroll sections (invisible, just for scroll height) -->
  <div class="scroll-section"></div>
  <div class="scroll-section"></div>
  <div class="scroll-section"></div>
  <div class="scroll-section"></div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700&family=Space+Mono:wght@400;700&display=swap');

  :global(body) {
    margin: 0;
    padding: 0;
    background: #0a0a0f;
    color: #f0f0f5;
    font-family: 'DM Sans', sans-serif;
    overflow-x: hidden;
  }

  .scroll-container {
    position: relative;
  }

  .viz-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: radial-gradient(ellipse at center, #1a1a2e 0%, #0a0a0f 70%);
  }

  .chart-wrapper {
    width: 90vw;
    height: 80vh;
    max-width: 1200px;
  }

  .scroll-section {
    height: 100vh;
    pointer-events: none;
  }

  .text-overlay {
    position: absolute;
    text-align: center;
    opacity: 0;
    transform: translateY(20px);
    transition: opacity 0.6s ease, transform 0.6s ease;
    pointer-events: none;
    z-index: 10;
  }

  .text-overlay.visible {
    opacity: 1;
    transform: translateY(0);
  }

  .text-overlay h1 {
    font-family: 'Space Mono', monospace;
    font-size: clamp(1.5rem, 4vw, 3rem);
    font-weight: 700;
    margin: 0 0 0.5rem 0;
    letter-spacing: -0.02em;
    text-transform: uppercase;
    background: linear-gradient(135deg, #f0f0f5 0%, #a0a0b0 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .stat {
    font-size: clamp(2rem, 8vw, 5rem);
    font-weight: 700;
    margin: 0;
    color: #E63946;
    text-shadow: 0 0 40px rgba(230, 57, 70, 0.4);
  }

  .percent {
    font-size: clamp(1.2rem, 3vw, 2rem);
    font-weight: 500;
    margin: 0.5rem 0;
    color: #E9C46A;
  }

  .subtitle {
    font-size: clamp(0.9rem, 2vw, 1.2rem);
    color: #808090;
    margin: 0.5rem 0 0 0;
    font-weight: 400;
  }

  .age-legend, .substance-legend {
    display: flex;
    flex-direction: column;
    gap: 0.8rem;
    margin-top: 1.5rem;
    text-align: left;
  }

  .age-item, .substance-item {
    display: flex;
    align-items: center;
    gap: 1rem;
    padding: 0.5rem 1rem;
    background: rgba(255, 255, 255, 0.05);
    border-radius: 8px;
    backdrop-filter: blur(10px);
  }

  .age-label {
    font-weight: 500;
    min-width: 120px;
  }

  .age-count {
    color: #E9C46A;
    font-family: 'Space Mono', monospace;
  }

  .color-dot {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .substance-label {
    font-size: 0.9rem;
  }

  .scroll-indicator {
    position: absolute;
    bottom: 40px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.5rem;
    opacity: 0.6;
    transition: opacity 0.3s ease;
    animation: pulse 2s ease-in-out infinite;
  }

  .scroll-indicator.hidden {
    opacity: 0;
  }

  .scroll-indicator span {
    font-size: 0.85rem;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: #808090;
  }

  .arrow {
    font-size: 1.5rem;
    animation: bounce 1.5s ease-in-out infinite;
  }

  @keyframes bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(8px); }
  }

  @keyframes pulse {
    0%, 100% { opacity: 0.6; }
    50% { opacity: 0.3; }
  }
</style>
