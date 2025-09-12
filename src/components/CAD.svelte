<script>
  import { onMount } from 'svelte';

  let baseCanvasEl;
  let drawCanvasEl;

  let tool = 'pencil';
  let drawing = false;
  let startX = 0, startY = 0;
  let currentAngle = 0;
  let customAngle = null;
  let selectedLineId = null;

  let drawnLines = [];
  let operationsHistory = [];
  let nextOperationId = 1;

  let baseCtx;
  let drawCtx;

  const width = 800;
  const height = 600;

  onMount(() => {
    baseCtx = baseCanvasEl.getContext('2d');
    drawCtx = drawCanvasEl.getContext('2d');

    const img = new Image();
    img.src = 'https://upload.wikimedia.org/wikipedia/commons/3/3f/Logo_svelte.svg';
    img.onload = () => {
      baseCtx.drawImage(img, 50, 50, 150, 150);
    };

    updateCursor();

    const handleKey = (e) => {
      if (drawing && e.key === 'Enter') {
        const input = prompt('Ingrese ángulo personalizado:', String(currentAngle));
        if (input !== null) customAngle = parseFloat(input);
      }
    };
    window.addEventListener('keydown', handleKey);
    return () => window.removeEventListener('keydown', handleKey);
  });

  function updateCursor() {
    if (drawCanvasEl) {
      drawCanvasEl.style.cursor = tool === 'pencil' ? 'crosshair' : 'not-allowed';
    }
  }

  function selectTool(t) {
    tool = t;
    drawing = false;
    updateCursor();
  }

  function snapAngle(dx, dy) {
    const deg = Math.atan2(dy, dx) * (180 / Math.PI);
    const snaps = [0, 45, 90, 135, 180, -45, -90, -135];
    return snaps.reduce((prev, curr) => Math.abs(curr - deg) < Math.abs(prev - deg) ? curr : prev);
  }

  function handleCanvasClick(e) {
    if (tool !== 'pencil') return;
    const rect = drawCanvasEl.getBoundingClientRect();
    const mouseX = e.clientX - rect.left;
    const mouseY = e.clientY - rect.top;

    if (!drawing) {
      startX = mouseX; startY = mouseY; drawing = true;
    } else {
      const dx = mouseX - startX;
      const dy = mouseY - startY;
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
      redrawAllLines();
    }
  }

  function handleCanvasMove(e) {
    if (!drawing) return;
    redrawAllLines();
    const rect = drawCanvasEl.getBoundingClientRect();
    const mouseX = e.clientX - rect.left;
    const mouseY = e.clientY - rect.top;
    const dx = mouseX - startX;
    const dy = mouseY - startY;
    currentAngle = customAngle !== null ? customAngle : snapAngle(dx, dy);
    const rad = currentAngle * Math.PI / 180;
    const dist = Math.sqrt(dx * dx + dy * dy);
    const endX = startX + dist * Math.cos(rad);
    const endY = startY + dist * Math.sin(rad);

    drawCtx.setLineDash([5, 5]);
    drawCtx.strokeStyle = 'blue';
    drawCtx.lineWidth = 1;
    drawCtx.beginPath();
    drawCtx.moveTo(startX, startY);
    drawCtx.lineTo(endX, endY);
    drawCtx.stroke();
    drawCtx.fillStyle = 'black';
    drawCtx.font = '16px sans-serif';
    drawCtx.fillText(currentAngle + '°', endX + 10, endY - 10);
  }

  function redrawAllLines() {
    if (!drawCtx || !drawCanvasEl) return;
    drawCtx.clearRect(0, 0, drawCanvasEl.width, drawCanvasEl.height);
    drawnLines.forEach((line) => {
      drawCtx.setLineDash([]);
      drawCtx.strokeStyle = (selectedLineId === line.id) ? 'orange' : 'red';
      drawCtx.lineWidth = 2;
      drawCtx.beginPath();
      drawCtx.moveTo(line.startX, line.startY);
      drawCtx.lineTo(line.endX, line.endY);
      drawCtx.stroke();
    });
  }

  function deleteLine(id) {
    operationsHistory = operationsHistory.filter((l) => l.id !== id);
    drawnLines = drawnLines.filter((l) => l.id !== id);
    if (selectedLineId === id) selectedLineId = null;
    redrawAllLines();
  }

  function clearHistory() {
    operationsHistory = [];
    drawnLines = [];
    nextOperationId = 1;
    selectedLineId = null;
    redrawAllLines();
  }

  $: linesCount = drawnLines.length;
  $: totalLength = Math.round(drawnLines.reduce((sum, l) => sum + l.dist, 0));
</script>

<div class="app-container">
  <div class="history-panel">
    <div class="history-header">
      <div>Historial</div>
      <button class="clear-btn" on:click={clearHistory}>Limpiar Todo</button>
    </div>

    {#if operationsHistory.length === 0}
      <div class="empty-history">No hay operaciones registradas</div>
    {:else}
      {#each [...operationsHistory].slice().reverse() as line}
        <div class="operation-item {selectedLineId===line.id ? 'selected' : ''}"
             role="button" tabindex="0" aria-pressed={selectedLineId===line.id}
             on:click={() => { selectedLineId = line.id; redrawAllLines(); }}
             on:keydown={(e) => { if (e.key === 'Enter' || e.key === ' ') { selectedLineId = line.id; redrawAllLines(); } }}>
          Línea #{line.id} | ángulo: {line.angle}° | longitud: {Math.round(line.dist)}px
          <button class="delete-line-btn" on:click|stopPropagation={() => deleteLine(line.id)}>Eliminar</button>
        </div>
      {/each}
    {/if}

    <div class="stats">
      <div>Líneas: {linesCount}</div>
      <div>Longitud total: {totalLength}px</div>
    </div>
  </div>

  <div class="drawing-area">
    <div class="drawing-container">
      <canvas bind:this={baseCanvasEl} width={width} height={height}></canvas>
      <canvas bind:this={drawCanvasEl} width={width} height={height}
              on:click={handleCanvasClick}
              on:mousemove={handleCanvasMove}></canvas>
      <div class="toolbar">
        <button class="tool-btn pencil-btn {tool==='pencil' ? 'active' : ''}"
                on:click={() => selectTool('pencil')}>✏️ Lápiz</button>
        <button class="tool-btn eraser-btn {tool==='eraser' ? 'active' : ''}"
                on:click={() => selectTool('eraser')}>🩹 Goma</button>
        <div class="angle-display">Ángulo: <span>{currentAngle}</span>°</div>
      </div>
    </div>
  </div>
</div>

    <style>
        .app-container { display:flex; width:100%; max-width:1200px; height:90vh; background:white; border-radius:12px; overflow:hidden; box-shadow:0 10px 30px rgba(0,0,0,0.15); }
        .history-panel { width:300px; background:#2c3e50; color:white; padding:20px; overflow-y:auto; }
        .history-header { display:flex; justify-content:space-between; margin-bottom:20px; }
        .clear-btn { background:#e74c3c; border:none; padding:8px 15px; border-radius:5px; cursor:pointer; font-weight:bold; }
        .operation-item { padding:10px; margin-bottom:10px; background:#34495e; border-radius:5px; cursor:pointer; }
        .operation-item.selected { border:2px solid #e74c3c; background:#c0392b; }
        .delete-line-btn { margin-top:5px; background:#c0392b; color:white; border:none; padding:5px 10px; border-radius:5px; cursor:pointer; }
        .drawing-area { flex:1; padding:20px; display:flex; flex-direction:column; }
        .drawing-container { position:relative; width:100%; height:100%; border:2px solid #bdc3c7; border-radius:8px; overflow:hidden; background:white; }
        canvas { position:absolute; top:0; left:0; }
        .toolbar { position:absolute; top:10px; left:10px; right:10px; height:60px; background:rgba(255,255,255,0.9); display:flex; gap:10px; align-items:center; padding:0 10px; z-index:10; border-radius:8px; }
        .tool-btn { padding:5px 10px; border:none; border-radius:5px; cursor:pointer; font-weight:bold; }
        .pencil-btn.active { background:#2980b9; color:white; }
        .eraser-btn.active { background:#d35400; color:white; }
        .angle-display { margin-left:auto; padding:5px 10px; background:#ecf0f1; border-radius:5px; }
        .stats { margin-top:10px; font-size:14px; color:#ecf0f1; display:flex; justify-content:space-between; }
    </style>
