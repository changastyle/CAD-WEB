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
  /** @type {{ x:number, y:number }|null } */
  let selectedVertex = null;
  /** @type {'vertex'|'edge'|'none'} */
  let selectionMode = 'none'; // 'vertex' | 'edge' | 'none'
  /** @type {{ lineId:number, end:'start'|'end' }|null } */
  let draggingVertex = null; // { lineId, end: 'start'|'end' }

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
      } else if (e.key === 'Escape') {
        // terminar cadena de pluma o cancelar dibujo
        drawing = false;
        preview = null;
        // también cancelar drag de vértice
        draggingVertex = null;
      } else if (e.key === 'p' || e.key === 'P') {
        selectTool('pen');
      } else if (e.key === 'o' || e.key === 'O') {
        selectTool('pencil');
      } else if (e.key === 'e' || e.key === 'E') {
        selectTool('eraser');
      } else if (e.key === '1') {
        selectionMode = 'vertex';
        selectedLineId = null;
      } else if (e.key === '2') {
        selectionMode = 'edge';
        selectedVertex = null;
      } else if (e.key === 'Enter' && selectionMode === 'vertex' && draggingVertex && selectedVertex) {
        // confirmar colocación en el snap actual
        drawnLines = drawnLines.map(l => {
          if (l.id !== draggingVertex.lineId) return l;
          if (draggingVertex.end === 'start') {
            return { ...l, startX: selectedVertex.x, startY: selectedVertex.y, dist: Math.hypot((l.endX - selectedVertex.x),(l.endY - selectedVertex.y)), angle: Math.round(Math.atan2(l.endY - selectedVertex.y, l.endX - selectedVertex.x) * 180/Math.PI) };
          } else {
            return { ...l, endX: selectedVertex.x, endY: selectedVertex.y, dist: Math.hypot((selectedVertex.x - l.startX),(selectedVertex.y - l.startY)), angle: Math.round(Math.atan2(selectedVertex.y - l.startY, selectedVertex.x - l.startX) * 180/Math.PI) };
          }
        });
        operationsHistory = drawnLines;
        draggingVertex = null;
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
    // Selection modes
    if (selectionMode === 'vertex') {
      const xw = (point.x - offsetX) / scale;
      const yw = (point.y - offsetY) / scale;
      const snapped = showGrid ? {
        x: Math.round(xw / gridSpacingPx) * gridSpacingPx,
        y: Math.round(yw / gridSpacingPx) * gridSpacingPx
      } : { x: xw, y: yw };
      // find nearest vertex within tolerance
      const tol = gridSpacingPx * 0.6;
      /** @type {{ x:number, y:number, end:'start'|'end', lineId:number } | null } */
      let bestInfo = null; let bestD = Infinity;
      drawnLines.forEach(line => {
        /** @type {{ x:number, y:number, end:'start'|'end', lineId:number }[]} */
        const candidates = [
          { x: line.startX, y: line.startY, end: 'start', lineId: line.id },
          { x: line.endX, y: line.endY, end: 'end', lineId: line.id }
        ];
        candidates.forEach(p => {
          const d = Math.hypot(p.x - snapped.x, p.y - snapped.y);
          if (d < bestD && d <= tol) { bestInfo = p; bestD = d; }
        });
      });
      selectedVertex = bestInfo ? { x: bestInfo.x, y: bestInfo.y } : null;
      draggingVertex = bestInfo ? { lineId: bestInfo.lineId, end: bestInfo.end } : null;
      return;
    }
    if (selectionMode === 'edge') {
      const xw = (point.x - offsetX) / scale;
      const yw = (point.y - offsetY) / scale;
      // distance point to segment; pick nearest within tolerance
      function distPointToSeg(px, py, x1, y1, x2, y2) {
        const vx = x2 - x1, vy = y2 - y1;
        const wx = px - x1, wy = py - y1;
        const c1 = vx * wx + vy * wy;
        if (c1 <= 0) return Math.hypot(px - x1, py - y1);
        const c2 = vx * vx + vy * vy;
        if (c2 <= c1) return Math.hypot(px - x2, py - y2);
        const b = c1 / c2;
        const bx = x1 + b * vx, by = y1 + b * vy;
        return Math.hypot(px - bx, py - by);
      }
      const tol = gridSpacingPx * 0.6;
      let bestId = null; let bestD = Infinity;
      drawnLines.forEach(line => {
        const d = distPointToSeg(xw, yw, line.startX, line.startY, line.endX, line.endY);
        if (d < bestD && d <= tol) { bestD = d; bestId = line.id; }
      });
      selectedLineId = bestId;
      return;
    }
    if (tool !== 'pencil' && tool !== 'pen') return;
    // convert screen -> world using current transform
    const mouseX = (point.x - offsetX) / scale;
    const mouseY = (point.y - offsetY) / scale;

    // snap to grid if enabled
    const snappedX = showGrid ? Math.round(mouseX / gridSpacingPx) * gridSpacingPx : mouseX;
    const snappedY = showGrid ? Math.round(mouseY / gridSpacingPx) * gridSpacingPx : mouseY;

    // Si es pluma y no estamos dibujando, arrancamos desde el último endpoint
    if (tool === 'pen' && !drawing && operationsHistory.length > 0) {
      const last = operationsHistory[operationsHistory.length - 1];
      startX = last.endX; startY = last.endY; drawing = true; preview = null;
      // y este mismo click crea el primer segmento de la cadena
    }

    if (!drawing) {
      // lápiz: primer click fija inicio
      startX = snappedX; startY = snappedY; drawing = true; preview = null;
      return;
        } else {
      const dx = snappedX - startX;
      const dy = snappedY - startY;
      // Pluma: permite cambiar dirección libremente (sin forzar ángulo anterior)
      const angle = customAngle !== null ? customAngle : snapAngle(dx, dy);
      const rad = angle * Math.PI / 180;
      const dist = Math.sqrt(dx * dx + dy * dy);
      const endX = startX + dist * Math.cos(rad);
      const endY = startY + dist * Math.sin(rad);

            const line = { id: nextOperationId++, startX, startY, endX, endY, angle, dist };
      drawnLines = [...drawnLines, line];
      operationsHistory = [...operationsHistory, line];
      customAngle = null;
      preview = null;
      if (tool === 'pen') {
        // continue chaining from last end point
        startX = endX;
        startY = endY;
        drawing = true;
      } else {
        drawing = false;
      }
    }
  }

  function handleCanvasMove(point) {
    // dragging vertex
    if (selectionMode === 'vertex' && draggingVertex) {
      const xw = (point.x - offsetX) / scale;
      const yw = (point.y - offsetY) / scale;
      const snappedX = showGrid ? Math.round(xw / gridSpacingPx) * gridSpacingPx : xw;
      const snappedY = showGrid ? Math.round(yw / gridSpacingPx) * gridSpacingPx : yw;
      drawnLines = drawnLines.map(l => {
        if (l.id !== draggingVertex.lineId) return l;
        if (draggingVertex.end === 'start') {
          return { ...l, startX: snappedX, startY: snappedY, dist: Math.hypot((l.endX - snappedX),(l.endY - snappedY)), angle: Math.round(Math.atan2(l.endY - snappedY, l.endX - snappedX) * 180/Math.PI) };
        } else {
          return { ...l, endX: snappedX, endY: snappedY, dist: Math.hypot((snappedX - l.startX),(snappedY - l.startY)), angle: Math.round(Math.atan2(snappedY - l.startY, snappedX - l.startX) * 180/Math.PI) };
        }
      });
      operationsHistory = drawnLines;
      selectedVertex = { x: snappedX, y: snappedY };
      return;
    }
    // En pluma: mostrar preview desde el último endpoint aunque no haya empezado la cadena aún
    if (!(drawing || (tool === 'pen' && operationsHistory.length > 0))) return;
    const mouseX = (point.x - offsetX) / scale;
    const mouseY = (point.y - offsetY) / scale;
    const snappedX = showGrid ? Math.round(mouseX / gridSpacingPx) * gridSpacingPx : mouseX;
    const snappedY = showGrid ? Math.round(mouseY / gridSpacingPx) * gridSpacingPx : mouseY;
    let originX = startX;
    let originY = startY;
    if (!drawing && tool === 'pen') {
      const last = operationsHistory[operationsHistory.length - 1];
      originX = last.endX;
      originY = last.endY;
    }
    const dx = snappedX - originX;
    const dy = snappedY - originY;
    currentAngle = customAngle !== null ? customAngle : snapAngle(dx, dy);
    const rad = currentAngle * Math.PI / 180;
    const dist = Math.sqrt(dx * dx + dy * dy);
    const endX = originX + dist * Math.cos(rad);
    const endY = originY + dist * Math.sin(rad);
    preview = { startX: originX, startY: originY, endX, endY, currentAngle };
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
      selectedVertex={selectedVertex}
      showVertices={selectionMode==='vertex'}
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
      onDown={(p) => {
        if (selectionMode === 'vertex') {
          // pick/start drag
          handleCanvasClick(p);
        }
      }}
      onUp={() => {
        // snap and finalize on mouse up
        if (selectionMode === 'vertex' && draggingVertex && selectedVertex) {
          drawnLines = drawnLines.map(l => {
            if (l.id !== draggingVertex.lineId) return l;
            if (draggingVertex.end === 'start') {
              return { ...l, startX: selectedVertex.x, startY: selectedVertex.y, dist: Math.hypot((l.endX - selectedVertex.x),(l.endY - selectedVertex.y)), angle: Math.round(Math.atan2(l.endY - selectedVertex.y, l.endX - selectedVertex.x) * 180/Math.PI) };
            } else {
              return { ...l, endX: selectedVertex.x, endY: selectedVertex.y, dist: Math.hypot((selectedVertex.x - l.startX),(selectedVertex.y - l.startY)), angle: Math.round(Math.atan2(selectedVertex.y - l.startY, selectedVertex.x - l.startX) * 180/Math.PI) };
            }
          });
          operationsHistory = drawnLines;
        }
        draggingVertex = null;
      }}
    />
    <div class="toolbar">
        <button class="tool-btn pencil-btn {tool==='pencil' ? 'active' : ''}"
                on:click={() => selectTool('pencil')}>✏️ Lápiz</button>
        <button class="tool-btn pencil-btn {tool==='pen' ? 'active' : ''}"
                on:click={() => selectTool('pen')}>🖊️ Pluma</button>
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
