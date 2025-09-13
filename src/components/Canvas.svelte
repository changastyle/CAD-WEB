<script>
  import { onMount } from 'svelte';

  /** @typedef {{ id:number, startX:number, startY:number, endX:number, endY:number, angle:number, dist:number }} Line */
  /** @typedef {{ startX:number, startY:number, endX:number, endY:number, currentAngle:number }} Preview */

  /** @type {number} */
  export let width = 800;
  /** @type {number} */
  export let height = 600;
  /** @type {string} */
  export let tool = 'pencil';
  /** @type {Line[]} */
  export let drawnLines = [];
  /** @type {number|null} */
  export let selectedLineId = null;
  /** @type {{ x:number, y:number }|null} */
  export let selectedVertex = null;
  /** @type {boolean} */
  export let showVertices = false;
  /** @type {Preview|null} */
  export let preview = null; // { startX, startY, endX, endY, currentAngle }
  /** @type {string} */
  export let backgroundImageUrl = 'https://upload.wikimedia.org/wikipedia/commons/3/3f/Logo_svelte.svg';
  /** @type {boolean} */
  export let showGrid = true;
  /** @type {number} */
  export let gridSpacingPx = 40; // default ~1cm if 4px per mm
  /** @type {string} */
  export let gridColor = 'rgba(0,0,0,0.08)';
  /** @type {number} */
  export let gridBoldEvery = 5; // draw a bolder line every N cells
  /** @type {number} */
  export let scale = 1;
  /** @type {number} */
  export let offsetX = 0;
  /** @type {number} */
  export let offsetY = 0;
  /** @type {boolean} */
  export let panning = false;
  /** @type {string} */
  export let unit = 'cm';

  // Callbacks provided by parent
  export let onClick = (point) => {};
  export let onMove = (point) => {};
  export let onDown = (point) => {};
  export let onUp = () => {};

  let baseCanvasEl;
  let drawCanvasEl;
  let baseCtx;
  let drawCtx;
  let bgImage = null;

  onMount(() => {
    baseCtx = baseCanvasEl.getContext('2d');
    drawCtx = drawCanvasEl.getContext('2d');
    // Load background image
    if (backgroundImageUrl) {
      bgImage = new Image();
      bgImage.src = backgroundImageUrl;
      bgImage.onload = () => {
        drawBase();
      };
    } else {
      drawBase();
    }
    updateCursor();
  });

  $: updateCursor();
  function updateCursor() {
    if (drawCanvasEl) {
      drawCanvasEl.style.cursor = tool === 'pencil' ? 'crosshair' : 'not-allowed';
    }
  }

  function drawGrid() {
    if (!baseCtx) return;
    baseCtx.save();
    baseCtx.clearRect(0, 0, width, height);
    // background image under grid transformed
    if (bgImage && bgImage.complete) {
      const imgX = 50 * scale + offsetX;
      const imgY = 50 * scale + offsetY;
      const imgW = 150 * scale;
      const imgH = 150 * scale;
      baseCtx.drawImage(bgImage, imgX, imgY, imgW, imgH);
    }
    if (!showGrid || gridSpacingPx <= 0) {
      baseCtx.restore();
      return;
    }
    // grid lines (screen space)
    const spacing = gridSpacingPx * scale;
    const startX = ((-offsetX) % spacing + spacing) % spacing;
    const startY = ((-offsetY) % spacing + spacing) % spacing;
    for (let x = startX, i = 0; x <= width; x += spacing, i++) {
      baseCtx.beginPath();
      baseCtx.strokeStyle = (i % gridBoldEvery === 0) ? 'rgba(0,0,0,0.18)' : gridColor;
      baseCtx.lineWidth = (i % gridBoldEvery === 0) ? 1 : 0.5;
      baseCtx.moveTo(x, 0);
      baseCtx.lineTo(x, height);
      baseCtx.stroke();
    }
    for (let y = startY, j = 0; y <= height; y += spacing, j++) {
      baseCtx.beginPath();
      baseCtx.strokeStyle = (j % gridBoldEvery === 0) ? 'rgba(0,0,0,0.18)' : gridColor;
      baseCtx.lineWidth = (j % gridBoldEvery === 0) ? 1 : 0.5;
      baseCtx.moveTo(0, y);
      baseCtx.lineTo(width, y);
      baseCtx.stroke();
    }
    baseCtx.restore();
  }

  function drawBase() {
    if (!baseCtx) return;
    drawGrid();
  }

  // Redraw base when grid props or transform change (don't short-circuit on bgImage)
  $: {
    baseCtx; width; height; showGrid; gridSpacingPx; gridColor; gridBoldEvery; scale; offsetX; offsetY; bgImage;
    if (baseCtx) drawBase();
  }

  // Redraw lines when inputs change
  $: if (drawCtx && drawCanvasEl) {
    drawCtx.clearRect(0, 0, drawCanvasEl.width, drawCanvasEl.height);

    // Primero dibujamos todas las líneas
    drawnLines.forEach((line) => {
      drawCtx.setLineDash([]);
      drawCtx.strokeStyle = (selectedLineId === line.id) ? 'orange' : 'red';
      drawCtx.lineWidth = 2;
      drawCtx.beginPath();
      drawCtx.moveTo(line.startX * scale + offsetX, line.startY * scale + offsetY);
      drawCtx.lineTo(line.endX * scale + offsetX, line.endY * scale + offsetY);
      drawCtx.stroke();

      // Dibujar la cota de la línea
      const midX = (line.startX + line.endX) / 2 * scale + offsetX;
      const midY = (line.startY + line.endY) / 2 * scale + offsetY;
      const distInUnits = line.dist / gridSpacingPx;
      drawCtx.font = '12px Arial';
      drawCtx.fillStyle = 'black';
      drawCtx.textAlign = 'center';
      drawCtx.textBaseline = 'bottom';
      const offset = 10;
      drawCtx.fillText(`${Math.round(distInUnits)} ${unit}`, midX, midY - offset);
    });

    // Luego dibujamos los vértices si están activados
    if (showVertices) {
      drawCtx.fillStyle = 'rgba(0,0,0,0.6)';
      drawnLines.forEach((line) => {
        const points = [
          { x: line.startX, y: line.startY },
          { x: line.endX, y: line.endY },
        ];
        points.forEach((p) => {
          const sx = p.x * scale + offsetX;
          const sy = p.y * scale + offsetY;
          drawCtx.beginPath();
          drawCtx.arc(sx, sy, 3, 0, Math.PI * 2);
          drawCtx.fill();
        });
      });
      if (selectedVertex) {
        drawCtx.fillStyle = 'orange';
        const sx = selectedVertex.x * scale + offsetX;
        const sy = selectedVertex.y * scale + offsetY;
        drawCtx.beginPath();
        drawCtx.arc(sx, sy, 5, 0, Math.PI * 2);
        drawCtx.fill();
      }
    }

    // Finalmente, si hay preview, la dibujamos encima
    if (preview) {
      // Dibujar línea preview
      drawCtx.setLineDash([5, 5]);
      drawCtx.strokeStyle = 'blue';
      drawCtx.lineWidth = 1;
      drawCtx.beginPath();
      drawCtx.moveTo(preview.startX * scale + offsetX, preview.startY * scale + offsetY);
      drawCtx.lineTo(preview.endX * scale + offsetX, preview.endY * scale + offsetY);
      drawCtx.stroke();

      // Calcular y mostrar la longitud
      const dx = preview.endX - preview.startX;
      const dy = preview.endY - preview.startY;
      const distInPixels = Math.sqrt(dx*dx + dy*dy);
      const lengthInUnits = distInPixels / gridSpacingPx;

      // Posición para mostrar la longitud
      const midX = (preview.startX + preview.endX) / 2 * scale + offsetX;
      const midY = (preview.startY + preview.endY) / 2 * scale + offsetY;

      // Mostrar longitud en azul (para distinguirla de las cotas finales)
      drawCtx.font = '14px Arial';
      drawCtx.fillStyle = 'blue';
      drawCtx.textAlign = 'center';
      drawCtx.textBaseline = 'bottom';
      drawCtx.fillText(`${Math.round(lengthInUnits)} ${unit}`, midX, midY - 15);

      // Mostrar ángulo cerca del punto final
      drawCtx.fillStyle = 'black';
      drawCtx.font = '12px Arial';
      const angle = preview.currentAngle;
      if (Math.abs(angle % 90) < 5 || Math.abs(angle % 90) > 85) {
        const size = 8;
        drawCtx.fillRect(preview.endX * scale + offsetX - size/2, preview.endY * scale + offsetY - size/2, size, size);
      } else {
        drawCtx.beginPath();
        drawCtx.arc(preview.endX * scale + offsetX, preview.endY * scale + offsetY, 6, 0, Math.PI * 2);
        drawCtx.fill();
        drawCtx.fillText(`${Math.round(angle)}°`, preview.endX * scale + offsetX + 15, preview.endY * scale + offsetY);
      }
    }
  }

  function handleClick(e) {
    const rect = drawCanvasEl.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    onClick({ x, y });
  }

  function handleMove(e) {
    const rect = drawCanvasEl.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    onMove({ x, y });
  }

  // Pan & Zoom support
  export let onPan = (dx, dy) => {};
  export let onZoom = (x, y, deltaY) => {};
  let isPanning = false;
  let lastX = 0, lastY = 0;
  function handleMouseDown(e) {
    if (panning || e.button === 1 || e.button === 2) {
      isPanning = true;
      lastX = e.clientX;
      lastY = e.clientY;
      e.preventDefault();
    } else {
      const rect = drawCanvasEl.getBoundingClientRect();
      const x = e.clientX - rect.left;
      const y = e.clientY - rect.top;
      onDown({ x, y, button: e.button });
    }
  }
  function handleMouseMove(e) {
    if (isPanning) {
      const dx = e.clientX - lastX;
      const dy = e.clientY - lastY;
      lastX = e.clientX;
      lastY = e.clientY;
      onPan(dx, dy);
      e.preventDefault();
    } else {
      handleMove(e);
    }
  }
  function handleMouseUp() { isPanning = false; onUp(); }
  function handleContextMenu(e) { if (panning) e.preventDefault(); }
  function handleWheel(e) {
    e.preventDefault();
    const rect = drawCanvasEl.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    onZoom(x, y, e.deltaY);
  }
</script>

<div class="drawing-container">
  <canvas bind:this={baseCanvasEl} width={width} height={height}></canvas>
  <canvas bind:this={drawCanvasEl} width={width} height={height}
          on:mousedown={handleMouseDown}
          on:mousemove={handleMouseMove}
          on:mouseup={handleMouseUp}
          on:mouseleave={handleMouseUp}
          on:contextmenu={handleContextMenu}
          on:click={handleClick}
          on:wheel={handleWheel}></canvas>
</div>

<style>
  .drawing-container { position:relative; width:100%; height:100%; border:2px solid #bdc3c7; border-radius:8px; overflow:hidden; background:white; }
  canvas { position:absolute; top:0; left:0; }
</style>
