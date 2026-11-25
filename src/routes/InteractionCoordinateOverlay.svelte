<script>
  import { t } from '#lib/i18n.svelte.js';

  let { coordinate = null } = $props();

  const format = value => Number.isFinite(value) ? value.toFixed(2) : '—';
</script>

{#if coordinate}
  <div
    class="coordinate-probe"
    style="left: {coordinate.clientX}px; top: {coordinate.clientY}px"
    aria-hidden="true"
  >
    <svg viewBox="-8 -52 112 112">
      <line class="axis-x" x1="-8" y1="0" x2="92" y2="0" />
      <path class="axis-x" d="m86 -6 6 6-6 6" />
      <line class="axis-y" x1="0" y1="52" x2="0" y2="-42" />
      <path class="axis-y" d="m-6 -36 6-6 6 6" />
      <circle class="marker-halo" cx="0" cy="0" r="8" />
      <circle class="marker" cx="0" cy="0" r="3" />
      <text class="axis-label axis-x-label" x="76" y="-7">+X</text>
      <text class="axis-label axis-y-label" x="7" y="-32">+Y</text>
    </svg>
    <div class="coordinate-label">
      <span>{t('coordinateCanvas')} X {format(coordinate.coordinates.canvasX)} · Y {format(coordinate.coordinates.canvasY)}</span>
      {#if Number.isFinite(coordinate.coordinates.modelX) && Number.isFinite(coordinate.coordinates.modelY)}
        <span>{coordinate.coordinates.coordinateSpace} X {format(coordinate.coordinates.modelX)} · Y {format(coordinate.coordinates.modelY)}</span>
      {/if}
    </div>
  </div>
{/if}

<style>
  .coordinate-probe {
    position: fixed;
    z-index: var(--z-control);
    width: 0;
    height: 0;
    pointer-events: none;
  }

  svg {
    position: absolute;
    left: -8px;
    top: -52px;
    width: 112px;
    height: 112px;
    overflow: visible;
    filter: drop-shadow(0 1px 3px rgba(0, 0, 0, 0.9));
  }

  line,
  path {
    fill: none;
    stroke-width: 1.8;
    vector-effect: non-scaling-stroke;
  }

  .axis-x {
    stroke: #ff657a;
  }

  .axis-y {
    stroke: #60e08a;
  }

  .marker-halo {
    fill: rgba(255, 196, 64, 0.25);
    stroke: #ffc440;
    stroke-width: 1.5;
    vector-effect: non-scaling-stroke;
  }

  .marker {
    fill: #fff;
    stroke: #232323;
    stroke-width: 1.5;
    vector-effect: non-scaling-stroke;
  }

  .axis-label {
    font: 10px sans-serif;
    font-weight: 700;
    paint-order: stroke;
    stroke: rgba(0, 0, 0, 0.8);
    stroke-width: 2px;
  }

  .axis-x-label {
    fill: #ff8997;
  }

  .axis-y-label {
    fill: #8af0a9;
  }

  .coordinate-label {
    position: absolute;
    top: 10px;
    left: 12px;
    display: grid;
    gap: 2px;
    padding: 5px 7px;
    border: 1px solid rgba(255, 196, 64, 0.75);
    border-radius: 4px;
    background: rgba(20, 20, 20, 0.88);
    color: #fff;
    font: 11px/1.35 ui-monospace, SFMono-Regular, Consolas, monospace;
    white-space: nowrap;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.45);
  }
</style>
