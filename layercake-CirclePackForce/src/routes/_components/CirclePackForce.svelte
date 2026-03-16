<!--
  @component
  Cannabis exposure visualization with scroll-driven clustering
-->
<script>
  import { getContext, onMount } from 'svelte';
  import { forceSimulation, forceX, forceY, forceManyBody, forceCollide, forceCenter } from 'd3-force';

  const { data, width, height, zGet } = getContext('LayerCake');

  /** @type {number} */
  let { 
    stage = 0,
    nodeStroke = '#fff',
    nodeStrokeWidth = 1,
    substanceColors = {},
    ageGroups = []
  } = $props();

  const RADIUS = 4;

  // Hover state
  let hoveredPoint = $state(null);
  let mouseX = $state(0);
  let mouseY = $state(0);

  // Make a copy because the simulation will alter the objects
  let initialNodes = $state([]);
  let nodes = $state([]);
  let simulation;

  // Substance order for consistent clustering
  const substanceOrder = [
    'Marijuana: Edible Preparation',
    'Marijuana: Dried Plant',
    'eCigarettes: Marijuana Device Flavor Unknown',
    'Other Marijuana Substances'
  ];

  // Age group positions for stage 2 (grape clusters)
  const getAgeGroupPosition = (ageGroup, w, h) => {
    const positions = {
      '<= 5 Years': { x: w * 0.25, y: h * 0.5 },
      '6 - 12 Years': { x: w * 0.5, y: h * 0.5 },
      '13 - 19 Years': { x: w * 0.75, y: h * 0.5 }
    };
    return positions[ageGroup] || { x: w / 2, y: h / 2 };
  };

  // Get substance position within age group - arranged vertically by substance
  const getSubstanceWithinAgePosition = (substance, ageGroup, w, h) => {
    const agePos = getAgeGroupPosition(ageGroup, w, h);
    const substanceIndex = substanceOrder.indexOf(substance);
    // Stack substances vertically within each age group
    const yOffset = (substanceIndex - 1.5) * 35; // Spread vertically
    return { x: agePos.x, y: agePos.y + yOffset };
  };

  // Substance positions within age groups for stage 3 (more spread out)
  const getSubstancePosition = (substance, ageGroup, w, h) => {
    const agePos = getAgeGroupPosition(ageGroup, w, h);
    const substanceOffsets = {
      'Marijuana: Edible Preparation': { x: -50, y: -50 },
      'Marijuana: Dried Plant': { x: 50, y: -50 },
      'eCigarettes: Marijuana Device Flavor Unknown': { x: -50, y: 50 },
      'Other Marijuana Substances': { x: 50, y: 50 }
    };
    const offset = substanceOffsets[substance] || { x: 0, y: 0 };
    return { x: agePos.x + offset.x, y: agePos.y + offset.y };
  };

  // Substance positions for stage 0 (tighter clustering - all together but colors grouped)
  // isYouth parameter determines if we include grey (adult) cluster
  const getSubstanceClusterPosition = (substance, isYouth, w, h) => {
    // Grey (adult) dots get their own cluster position - clearly separated at bottom right
    if (!isYouth) {
      return { x: w / 2 + Math.min(w, h) * 0.12, y: h / 2 + Math.min(w, h) * 0.12 };
    }
    
    // Youth dots cluster in the upper left area
    const angle = {
      'Marijuana: Edible Preparation': Math.PI * 1.3,       // top-left
      'Marijuana: Dried Plant': Math.PI * 0.7,              // left
      'eCigarettes: Marijuana Device Flavor Unknown': Math.PI * 1.8, // top-right  
      'Other Marijuana Substances': Math.PI * 1.0           // left-bottom
    };
    const a = angle[substance] || 0;
    // Smaller radius to keep dots closer together
    const radius = Math.min(w, h) * 0.05;
    return {
      x: w / 2 - Math.min(w, h) * 0.05 + Math.cos(a) * radius,
      y: h / 2 - Math.min(w, h) * 0.05 + Math.sin(a) * radius
    };
  };

  // Format substance name for display
  function formatSubstance(substance) {
    return substance
      .replace('Marijuana: ', '')
      .replace('eCigarettes: ', '');
  }

  onMount(() => {
    initialNodes = $data.map(d => ({ ...d, x: $width / 2, y: $height / 2 }));
    
    simulation = forceSimulation(initialNodes)
      .force('charge', forceManyBody().strength(0.5))
      .force('collision', forceCollide().radius(RADIUS + 1))
      .force('center', forceCenter($width / 2, $height / 2))
      .on('tick', () => {
        nodes = [...simulation.nodes()];
      });

    updateForces();
  });

  // Update forces when stage changes
  $effect(() => {
    if (simulation && $width && $height) {
      updateForces();
    }
  });

  function updateForces() {
    if (!simulation) return;

    const w = $width;
    const h = $height;

    if (stage === 0) {
      // Stage 0: All dots clustered tightly together, same colors grouped as smaller clusters
      // Grey (adult) dots also cluster together
      simulation
        .force('x', forceX().x(d => getSubstanceClusterPosition(d.substance, d.isYouth, w, h).x).strength(0.25))
        .force('y', forceY().y(d => getSubstanceClusterPosition(d.substance, d.isYouth, w, h).y).strength(0.25))
        .force('charge', forceManyBody().strength(0.8))
        .force('collision', forceCollide().radius(RADIUS + 0.3))
        .force('center', forceCenter(w / 2, h / 2))
        .alpha(0.8)
        .restart();
    } else if (stage === 1) {
      // Stage 1: Grey (adult) dots vanish, youth dots split into 4 color groups centered on screen
      simulation
        .force('x', forceX().x(d => {
          if (!d.isYouth) return w + 500; // Push adults far off screen to the right
          // Split into 4 groups by substance - 2x2 grid centered
          const positions = {
            'Marijuana: Edible Preparation': w / 2 - 80,      // top-left
            'Marijuana: Dried Plant': w / 2 + 80,             // top-right
            'eCigarettes: Marijuana Device Flavor Unknown': w / 2 - 80,  // bottom-left
            'Other Marijuana Substances': w / 2 + 80          // bottom-right
          };
          return positions[d.substance] || w / 2;
        }).strength(0.2))
        .force('y', forceY().y(d => {
          if (!d.isYouth) return h / 2;
          const positions = {
            'Marijuana: Edible Preparation': h / 2 - 80,      // top-left
            'Marijuana: Dried Plant': h / 2 - 80,             // top-right
            'eCigarettes: Marijuana Device Flavor Unknown': h / 2 + 80,  // bottom-left
            'Other Marijuana Substances': h / 2 + 80          // bottom-right
          };
          return positions[d.substance] || h / 2;
        }).strength(0.2))
        .force('charge', forceManyBody().strength(0.5))
        .force('collision', forceCollide().radius(RADIUS + 0.5))
        .force('center', null)
        .alpha(1)
        .restart();
    } else if (stage === 2) {
      // Stage 2: Adults disappear, youth split into 3 age group clusters
      // Colors cluster together within each age group
      simulation
        .force('x', forceX().x(d => {
          if (!d.isYouth) return w + 500; // Push adults far off screen
          return getSubstanceWithinAgePosition(d.substance, d.ageGroup, w, h).x;
        }).strength(0.15))
        .force('y', forceY().y(d => {
          if (!d.isYouth) return h / 2;
          return getSubstanceWithinAgePosition(d.substance, d.ageGroup, w, h).y;
        }).strength(0.15))
        .force('charge', forceManyBody().strength(0.2))
        .force('collision', forceCollide().radius(RADIUS + 0.5))
        .force('center', null)
        .alpha(1)
        .restart();
    } else if (stage === 3) {
      // Stage 3: Split further by substance within each age group (more spread)
      simulation
        .force('x', forceX().x(d => {
          if (!d.isYouth) return w + 500; // Push adults far off screen
          return getSubstancePosition(d.substance, d.ageGroup, w, h).x;
        }).strength(0.2))
        .force('y', forceY().y(d => {
          if (!d.isYouth) return h / 2;
          return getSubstancePosition(d.substance, d.ageGroup, w, h).y;
        }).strength(0.2))
        .force('charge', forceManyBody().strength(0.1))
        .force('collision', forceCollide().radius(RADIUS + 0.3))
        .force('center', null)
        .alpha(1)
        .restart();
    }
  }

  // Get opacity based on stage and youth status
  function getOpacity(d) {
    // Grey (adult) dots vanish starting from stage 1
    if (stage >= 1 && !d.isYouth) {
      return 0;
    }
    return 1;
  }

  // Handle mouse events
  function handleMouseEnter(event, point) {
    hoveredPoint = point;
    const rect = event.target.ownerSVGElement.getBoundingClientRect();
    mouseX = event.clientX - rect.left;
    mouseY = event.clientY - rect.top;
  }

  function handleMouseMove(event) {
    if (hoveredPoint) {
      const rect = event.target.ownerSVGElement.getBoundingClientRect();
      mouseX = event.clientX - rect.left;
      mouseY = event.clientY - rect.top;
    }
  }

  function handleMouseLeave() {
    hoveredPoint = null;
  }
</script>

{#each nodes as point (point.id)}
  <circle
    class="node"
    class:hovered={hoveredPoint?.id === point.id}
    role="img"
    aria-label="{formatSubstance(point.substance)}, {point.ageGroup}"
    r={hoveredPoint?.id === point.id ? RADIUS * 1.5 : RADIUS}
    fill={point.isYouth ? (substanceColors[point.substance] || $zGet(point)) : '#666'}
    stroke={hoveredPoint?.id === point.id ? '#fff' : nodeStroke}
    stroke-width={hoveredPoint?.id === point.id ? 2 : nodeStrokeWidth}
    cx={point.x}
    cy={point.y}
    opacity={getOpacity(point)}
    style="transition: opacity 0.8s ease, r 0.15s ease, stroke-width 0.15s ease;"
    onmouseenter={(e) => handleMouseEnter(e, point)}
    onmousemove={handleMouseMove}
    onmouseleave={handleMouseLeave}
  />
{/each}

<!-- Tooltip -->
{#if hoveredPoint}
  <g class="tooltip" transform="translate({mouseX + 15}, {mouseY - 10})">
    <rect
      x="0"
      y="-40"
      width="180"
      height="55"
      rx="6"
      fill="rgba(10, 10, 15, 0.95)"
      stroke={substanceColors[hoveredPoint.substance]}
      stroke-width="2"
    />
    <text x="10" y="-22" class="tooltip-label">Substance:</text>
    <text x="10" y="-6" class="tooltip-value" fill={substanceColors[hoveredPoint.substance]}>
      {formatSubstance(hoveredPoint.substance)}
    </text>
    <text x="10" y="10" class="tooltip-age">
      Age: {hoveredPoint.ageGroup}
    </text>
  </g>
{/if}

<style>
  .node {
    cursor: pointer;
  }

  .node.hovered {
    filter: drop-shadow(0 0 6px currentColor);
  }

  .age-label {
    fill: #f0f0f5;
    font-family: 'Space Mono', monospace;
    font-size: 14px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  .age-count {
    fill: #E9C46A;
    font-family: 'Space Mono', monospace;
    font-size: 20px;
    font-weight: 700;
  }

  .tooltip-label {
    fill: #808090;
    font-family: 'DM Sans', sans-serif;
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  .tooltip-value {
    font-family: 'DM Sans', sans-serif;
    font-size: 11px;
    font-weight: 600;
  }

  .tooltip-age {
    fill: #f0f0f5;
    font-family: 'Space Mono', monospace;
    font-size: 11px;
  }
</style>
