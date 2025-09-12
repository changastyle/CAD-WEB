<script>
  import { onMount } from 'svelte';
  import History from './History.svelte';
  import Canvas from './Canvas.svelte';

  let tool = 'pencil';
  let drawing = false;
  let startX = 0, startY = 0;
  let currentAngle = 0;
  let customAngle = null;
  let selectedLineId = null;

  let drawnLines = [];
  let operationsHistory = [];
  let nextOperationId = 1;

  let preview = null; // { startX, startY, endX, endY, currentAngle }
  let showGrid = true;
  let unit = 'cm'; // 'mm' | 'cm' | 'm'
  let pixelsPerMm = 4; // base scale; can be configurable later
  $: gridSpacingPx = unit === 'mm' ? 1 * pixelsPerMm : unit === 'cm' ? 10 * pixelsPerMm : 1000 * pixelsPerMm;
  let scale = 1;
  let offsetX = 0;
  let offsetY = 0;
  let panning = false;

  const width = 800;
  const height = 600;

  onMount(() => {
    const handleKey = (e) => {
      if (drawing && e.key === 'Enter') {
        const input = prompt('Ingrese ángulo personalizado:', String(currentAngle));
        if (input !== null) customAngle = parseFloat(input);
      }
    };
    window.addEventListener('keydown', handleKey);
    return () => window.removeEventListener('keydown', handleKey);
  });

  function selectTool(t) {
    tool = t;
    drawing = false;
    preview = null;
  }

  function snapAngle(dx, dy) {
    const deg = Math.atan2(dy, dx) * (180 / Math.PI);
    const snaps = [0, 45, 90, 135, 180, -45, -90, -135];
    return snaps.reduce((prev, curr) => Math.abs(curr - deg) < Math.abs(prev - deg) ? curr : prev);
  }

  function handleCanvasClick(point) {
    if (tool !== 'pencil') return;
    // convert screen -> world using current transform
    const mouseX = (point.x - offsetX) / scale;
    const mouseY = (point.y - offsetY) / scale;

    // snap to grid if enabled
    const snappedX = showGrid ? Math.round(mouseX / gridSpacingPx) * gridSpacingPx : mouseX;
    const snappedY = showGrid ? Math.round(mouseY / gridSpacingPx) * gridSpacingPx : mouseY;

    if (!drawing) {
      startX = snappedX; startY = snappedY; drawing = true; preview = null;
    } else {
      const dx = snappedX - startX;
      const dy = snappedY - startY;
      const angle = customAngle !== null ? customAngle : snapAngle(dx, dy);
      const rad = angle * Math.PI / 180;
      const dist = Math.sqrt(dx * dx + dy * dy);
      const endX = startX + dist * Math.cos(rad);
      const endY = startY + dist * Math.sin(rad);

      const line = { id: nextOperationId++, startX, startY, endX, endY, angle, dist };
      drawnLines = [...drawnLines, line];
      operationsHistory = [...operationsHistory, line];
      drawing = false;
      customAngle = null;
      preview = null;
    }
  }

  function handleCanvasMove(point) {
    if (!drawing) return;
    const mouseX = (point.x - offsetX) / scale;
    const mouseY = (point.y - offsetY) / scale;
    const snappedX = showGrid ? Math.round(mouseX / gridSpacingPx) * gridSpacingPx : mouseX;
    const snappedY = showGrid ? Math.round(mouseY / gridSpacingPx) * gridSpacingPx : mouseY;
    const dx = snappedX - startX;
    const dy = snappedY - startY;
    currentAngle = customAngle !== null ? customAngle : snapAngle(dx, dy);
    const rad = currentAngle * Math.PI / 180;
    const dist = Math.sqrt(dx * dx + dy * dy);
    const endX = startX + dist * Math.cos(rad);
    const endY = startY + dist * Math.sin(rad);
    preview = { startX, startY, endX, endY, currentAngle };
  }

  function deleteLine(id) {
    operationsHistory = operationsHistory.filter((l) => l.id !== id);
    drawnLines = drawnLines.filter((l) => l.id !== id);
    if (selectedLineId === id) selectedLineId = null;
  }

  function clearHistory() {
    operationsHistory = [];
    drawnLines = [];
    nextOperationId = 1;
    selectedLineId = null;
    preview = null;
  }

  $: linesCount = drawnLines.length;
  $: totalLength = Math.round(drawnLines.reduce((sum, l) => sum + l.dist, 0));
 </script>

<div class="app-container">
  <History
    operations={operationsHistory}
    selectedId={selectedLineId}
    linesCount={linesCount}
    totalLength={totalLength}
    onSelect={(id) => { selectedLineId = id; }}
    onDelete={(id) => deleteLine(id)}
    onClear={clearHistory}
  />

  <div class="drawing-area">
    <Canvas
      width={width}
      height={height}
      tool={tool}
      drawnLines={drawnLines}
      selectedLineId={selectedLineId}
      preview={preview}
      showGrid={showGrid}
      gridSpacingPx={gridSpacingPx}
      scale={scale}
      offsetX={offsetX}
      offsetY={offsetY}
      panning={panning}
      onPan={(dx,dy) => { offsetX += dx; offsetY += dy; }}
      onZoom={(x,y,deltaY) => {
        const zoomFactor = Math.exp(-deltaY * 0.001);
        const newScale = Math.max(0.1, Math.min(10, scale * zoomFactor));
        const sx = (x - offsetX) / scale;
        const sy = (y - offsetY) / scale;
        offsetX = x - sx * newScale;
        offsetY = y - sy * newScale;
        scale = newScale;
      }}
      onClick={handleCanvasClick}
      onMove={handleCanvasMove}
    />
    <div class="toolbar">
        <button class="tool-btn pencil-btn {tool==='pencil' ? 'active' : ''}"
                on:click={() => selectTool('pencil')}>✏️ Lápiz</button>
        <button class="tool-btn eraser-btn {tool==='eraser' ? 'active' : ''}"
                on:click={() => selectTool('eraser')}>🩹 Goma</button>
        <div class="angle-display">Ángulo: <span>{currentAngle}</span>°</div>
        <label style="margin-left:10px; display:flex; align-items:center; gap:6px;">
          <input type="checkbox" bind:checked={showGrid}>
          Grid
        </label>
        <select bind:value={unit} aria-label="Unidad de grilla" style="margin-left:6px;">
          <option value="mm">1 mm</option>
          <option value="cm">1 cm</option>
          <option value="m">1 m</option>
        </select>
    </div>
  </div>
</div>

    <style>
        .app-container { display:flex; width:100%; max-width:1200px; height:90vh; background:white; border-radius:12px; overflow:hidden; box-shadow:0 10px 30px rgba(0,0,0,0.15); }
        .drawing-area { position:relative; flex:1; padding:20px; display:flex; flex-direction:column; }
        .toolbar { position:absolute; top:30px; left:30px; right:30px; height:60px; background:rgba(255,255,255,0.9); display:flex; gap:10px; align-items:center; padding:0 10px; z-index:10; border-radius:8px; }
        .tool-btn { padding:5px 10px; border:none; border-radius:5px; cursor:pointer; font-weight:bold; }
        .pencil-btn.active { background:#2980b9; color:white; }
        .eraser-btn.active { background:#d35400; color:white; }
        .angle-display { margin-left:auto; padding:5px 10px; background:#ecf0f1; border-radius:5px; }
    </style>
