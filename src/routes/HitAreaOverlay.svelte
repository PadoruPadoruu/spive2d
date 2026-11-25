<script>
  import { onMount } from 'svelte';
  import { appState } from '#lib/appState.svelte.js';
  import { getRenderer } from '#lib/rendererStore.svelte.js';

  const STROKE_COLOR = '#ff5a8a';
  const FILL_COLOR = 'rgba(255, 90, 138, 0.15)';
  const MARGIN_COLOR = 'rgba(255, 196, 64, 0.16)';
  const MARGIN_STROKE_COLOR = '#ffc440';
  const LABEL_BACKGROUND = 'rgba(0, 0, 0, 0.7)';
  const LINE_PX = 1.5;
  const HIT_MARGIN_PX = 7;

  let canvas = $state();
  let frame = 0;
  let painted = false;

  function clear() {
    if (!painted || !canvas) return;
    canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
    painted = false;
  }

  function convexHull(flat) {
    const points = [];
    for (let i = 0; i + 1 < flat.length; i += 2) {
      const x = flat[i], y = flat[i + 1];
      if (Number.isFinite(x) && Number.isFinite(y)) points.push({ x, y });
    }
    const unique = [...new Map(points.map(point => [`${point.x},${point.y}`, point])).values()];
    if (unique.length < 3) return null;

    const pivotIndex = unique.reduce((best, point, index, all) =>
      point.y < all[best].y || (point.y === all[best].y && point.x < all[best].x) ? index : best, 0);
    const [pivot] = unique.splice(pivotIndex, 1);
    unique.sort((a, b) => {
      const angle = Math.atan2(a.y - pivot.y, a.x - pivot.x) - Math.atan2(b.y - pivot.y, b.x - pivot.x);
      if (angle !== 0) return angle;
      return Math.hypot(a.x - pivot.x, a.y - pivot.y) - Math.hypot(b.x - pivot.x, b.y - pivot.y);
    });

    const cross = (origin, a, b) =>
      (a.x - origin.x) * (b.y - origin.y) - (a.y - origin.y) * (b.x - origin.x);
    const hull = [pivot];
    for (const point of unique) {
      while (hull.length > 1 && cross(hull[hull.length - 2], hull[hull.length - 1], point) <= 0) hull.pop();
      hull.push(point);
    }
    if (hull.length < 3) return null;
    return new Float32Array(hull.flatMap(point => [point.x, point.y]));
  }

  function expandHull(flat, margin) {
    let centerX = 0, centerY = 0;
    for (let i = 0; i + 1 < flat.length; i += 2) {
      centerX += flat[i];
      centerY += flat[i + 1];
    }
    const count = flat.length / 2;
    centerX /= count;
    centerY /= count;
    const expanded = new Float32Array(flat.length);
    for (let i = 0; i + 1 < flat.length; i += 2) {
      const dx = flat[i] - centerX;
      const dy = flat[i + 1] - centerY;
      const length = Math.hypot(dx, dy) || 1;
      expanded[i] = flat[i] + dx / length * margin;
      expanded[i + 1] = flat[i + 1] + dy / length * margin;
    }
    return expanded;
  }

  function tracePolygon(ctx, polygon) {
    ctx.beginPath();
    ctx.moveTo(polygon[0], polygon[1]);
    for (let i = 2; i + 1 < polygon.length; i += 2) ctx.lineTo(polygon[i], polygon[i + 1]);
    ctx.closePath();
  }

  function paint() {
    frame = requestAnimationFrame(paint);
    const renderer = getRenderer();
    if (!canvas || !appState.showHitAreas || !appState.initialized || appState.processing || typeof renderer?.getHitAreas !== 'function') {
      clear();
      return;
    }
    let areas = [];
    try {
      areas = renderer.getHitAreas() ?? [];
    } catch (e) {
      console.warn('[hitAreas]', e);
    }
    const dpr = window.devicePixelRatio || 1;
    const width = Math.max(1, Math.round(window.innerWidth * dpr));
    const height = Math.max(1, Math.round(window.innerHeight * dpr));
    if (canvas.width !== width || canvas.height !== height) {
      canvas.width = width;
      canvas.height = height;
    }
    const ctx = canvas.getContext('2d');
    ctx.setTransform(1, 0, 0, 1, 0, 0);
    ctx.clearRect(0, 0, width, height);
    painted = areas.length > 0;
    if (!painted) return;
    const rect = renderer.getCanvas().getBoundingClientRect();
    ctx.setTransform(dpr, 0, 0, dpr, rect.left * dpr, rect.top * dpr);
    ctx.lineWidth = LINE_PX;
    ctx.font = `${getComputedStyle(document.documentElement).getPropertyValue('--ui-font-size').trim() || '12px'} sans-serif`;
    ctx.textBaseline = 'top';
    for (const { name, polygon } of areas) {
      const hull = convexHull(polygon);
      if (!hull) continue;
      const margin = expandHull(hull, HIT_MARGIN_PX);
      tracePolygon(ctx, margin);
      ctx.fillStyle = MARGIN_COLOR;
      ctx.fill();
      ctx.strokeStyle = MARGIN_STROKE_COLOR;
      ctx.lineWidth = LINE_PX;
      ctx.stroke();
      tracePolygon(ctx, hull);
      ctx.fillStyle = FILL_COLOR;
      ctx.fill();
      ctx.strokeStyle = STROKE_COLOR;
      ctx.lineWidth = LINE_PX;
      ctx.stroke();
      let x = Infinity;
      let y = Infinity;
      for (let i = 0; i + 1 < hull.length; i += 2) {
        if (hull[i + 1] < y || (hull[i + 1] === y && hull[i] < x)) {
          x = hull[i];
          y = hull[i + 1];
        }
      }
      const metrics = ctx.measureText(name);
      const labelHeight = (metrics.actualBoundingBoxDescent || 10) + 4;
      ctx.fillStyle = LABEL_BACKGROUND;
      ctx.fillRect(x, y, metrics.width + 6, labelHeight);
      ctx.fillStyle = STROKE_COLOR;
      ctx.fillText(name, x + 3, y + 2);
    }
  }

  onMount(() => {
    frame = requestAnimationFrame(paint);
    return () => cancelAnimationFrame(frame);
  });
</script>

<canvas class="hit-area-overlay" bind:this={canvas}></canvas>

<style>
  .hit-area-overlay {
    display: block;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: var(--z-overlay);
    pointer-events: none;
  }
</style>
