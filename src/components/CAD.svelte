<script>
    import { onMount } from "svelte";
    let baseCanvas, drawCanvas;
    let baseCtx, drawCtx;

    let tool = null; // "pencil" o "eraser"
    let drawing = false;
    let startX, startY;
    let currentAngle = 0; // ángulo calculado automáticamente
    let img, imageX = 50, imageY = 50;

    let customAngle = null; // si presionas Enter para cambiar ángulo manual

    // Array para almacenar el historial de operaciones
    let operationsHistory = [];
    let nextOperationId = 1;

    // Array para almacenar todas las líneas dibujadas
    let drawnLines = [];

    onMount(() => {
        baseCtx = baseCanvas.getContext("2d");
        drawCtx = drawCanvas.getContext("2d");

        img = new Image();
        img.src = "https://upload.wikimedia.org/wikipedia/commons/3/3f/Logo_svelte.svg";
        img.onload = () => {
            baseCtx.drawImage(img, imageX, imageY, 150, 150);
        };
    });

    function selectTool(t) {
        tool = t;
        drawing = false;
        drawCanvas.style.cursor = tool === "pencil" ? "crosshair" : "not-allowed";
    }

    function snapAngle(dx, dy) {
        const angleRad = Math.atan2(dy, dx);
        let angleDeg = angleRad * (180 / Math.PI);

        // Snap a 0°, 45°, 90°, 135°, 180°, etc.
        const snaps = [0, 45, 90, 135, 180, -45, -90, -135];
        let closest = snaps.reduce((prev, curr) =>
            Math.abs(curr - angleDeg) < Math.abs(prev - angleDeg) ? curr : prev
        );
        return closest;
    }

    // Redibujar todas las líneas
    function redrawAllLines() {
        drawCtx.clearRect(0, 0, drawCanvas.width, drawCanvas.height);

        // Dibujar todas las líneas guardadas
        drawnLines.forEach(line => {
            drawCtx.setLineDash([]);
            drawCtx.strokeStyle = "red";
            drawCtx.lineWidth = 2;
            drawCtx.beginPath();
            drawCtx.moveTo(line.startX, line.startY);
            drawCtx.lineTo(line.endX, line.endY);
            drawCtx.stroke();
        });
    }

    function handleCanvasClick(e) {
        if (tool !== "pencil") return;

        const rect = drawCanvas.getBoundingClientRect();
        const mouseX = e.clientX - rect.left;
        const mouseY = e.clientY - rect.top;

        if (!drawing) {
            startX = mouseX;
            startY = mouseY;
            drawing = true;
        } else {
            // Segundo click → dibujar línea final
            const dx = mouseX - startX;
            const dy = mouseY - startY;
            let angle = customAngle !== null ? customAngle : snapAngle(dx, dy);
            const rad = (angle * Math.PI) / 180;
            const dist = Math.sqrt(dx * dx + dy * dy);
            const endX = startX + dist * Math.cos(rad);
            const endY = startY + dist * Math.sin(rad);

            // Guardar la línea
            const newLine = {
                startX, startY, endX, endY, angle
            };
            drawnLines.push(newLine);

            // Redibujar todas las líneas
            redrawAllLines();

            // Guardar la operación en el historial
            operationsHistory.push({
                id: nextOperationId++,
                type: "line",
                startX: Math.round(startX),
                startY: Math.round(startY),
                endX: Math.round(endX),
                endY: Math.round(endY),
                angle: angle,
                length: Math.round(dist),
                timestamp: new Date()
            });

            drawing = false;
            customAngle = null;
        }
    }

    function handleMouseMove(e) {
        if (!drawing) return;

        const rect = drawCanvas.getBoundingClientRect();
        const mouseX = e.clientX - rect.left;
        const mouseY = e.clientY - rect.top;

        // Limpiar solo la previsualización, no las líneas dibujadas
        redrawAllLines();

        // Dibujar la previsualización
        drawCtx.setLineDash([5, 5]);
        drawCtx.strokeStyle = "blue";
        drawCtx.lineWidth = 1;

        let dx = mouseX - startX;
        let dy = mouseY - startY;
        currentAngle = customAngle !== null ? customAngle : snapAngle(dx, dy);
        const rad = (currentAngle * Math.PI) / 180;
        const dist = Math.sqrt(dx * dx + dy * dy);
        const endX = startX + dist * Math.cos(rad);
        const endY = startY + dist * Math.sin(rad);

        drawCtx.beginPath();
        drawCtx.moveTo(startX, startY);
        drawCtx.lineTo(endX, endY);
        drawCtx.stroke();

        // Mostrar ángulo en preview
        drawCtx.fillStyle = "black";
        drawCtx.font = "16px sans-serif";
        drawCtx.fillText(currentAngle + "°", endX + 10, endY - 10);
    }

    // Permite ingresar un ángulo personalizado con Enter
    function handleKeyDown(e) {
        if (drawing && e.key === "Enter") {
            const input = prompt("Ingrese ángulo personalizado en grados:", currentAngle);
            if (input !== null) customAngle = parseFloat(input);
        }
    }

    // Limpiar el historial y las líneas
    function clearHistory() {
        operationsHistory = [];
        drawnLines = [];
        nextOperationId = 1;
        redrawAllLines(); // Limpiar el canvas
    }
</script>

<style>
    .app-container {
        display: flex;
        width: 100%;
        height: 100vh;
    }

    .history-panel {
        width: 300px;
        padding: 15px;
        background-color: #f5f5f5;
        border-right: 1px solid #ddd;
        overflow-y: auto;
        height: 100%;
    }

    .history-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 15px;
        padding-bottom: 10px;
        border-bottom: 1px solid #ddd;
    }

    .history-title {
        font-size: 18px;
        font-weight: bold;
        color: #333;
    }

    .clear-btn {
        background-color: #ff4757;
        color: white;
        border: none;
        padding: 5px 10px;
        border-radius: 4px;
        cursor: pointer;
    }

    .clear-btn:hover {
        background-color: #ff6b81;
    }

    .operation-item {
        padding: 10px;
        margin-bottom: 10px;
        background-color: white;
        border-radius: 4px;
        box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }

    .operation-header {
        display: flex;
        justify-content: space-between;
        font-size: 12px;
        color: #888;
        margin-bottom: 5px;
    }

    .operation-content {
        font-size: 14px;
    }

    .operation-coords {
        margin-top: 5px;
        font-size: 12px;
        color: #555;
    }

    .drawing-container {
        position: relative;
        width: 800px;
        height: 600px;
        border: 1px solid #ccc;
    }

    canvas {
        position: absolute;
        top: 0;
        left: 0;
    }

    .toolbar {
        position: absolute;
        top: 5px;
        left: 5px;
        right: 5px;
        height: 50px;
        background: rgba(255, 255, 255, 0.5);
        display: flex;
        gap: 10px;
        align-items: center;
        padding: 5px;
        z-index: 10;
    }

    button {
        padding: 5px 10px;
    }

    .empty-history {
        text-align: center;
        color: #888;
        margin-top: 20px;
        font-style: italic;
    }
</style>

<div class="app-container">
    <!-- Panel de historial a la izquierda -->
    <div class="history-panel">
        <div class="history-header">
            <div class="history-title">Historial de Operaciones</div>
            <button class="clear-btn" on:click={clearHistory}>Limpiar Todo</button>
        </div>

        {#if operationsHistory.length === 0}
            <div class="empty-history">No hay operaciones registradas</div>
        {:else}
            {#each operationsHistory as operation (operation.id)}
                <div class="operation-item">
                    <div class="operation-header">
                        <span>Operación #{operation.id}</span>
                        <span>{operation.timestamp.toLocaleTimeString()}</span>
                    </div>
                    <div class="operation-content">
                        <div>Línea con ángulo: {operation.angle}°</div>
                        <div class="operation-coords">
                            Desde ({operation.startX}, {operation.startY}) hasta ({operation.endX}, {operation.endY})
                        </div>
                        <div class="operation-coords">
                            Longitud: {operation.length}px
                        </div>
                    </div>
                </div>
            {/each}
        {/if}
    </div>

    <!-- Área de dibujo -->
    <div class="drawing-container" tabindex="0" on:keydown={handleKeyDown}>
        <canvas bind:this={baseCanvas} width="800" height="600"></canvas>
        <canvas
                bind:this={drawCanvas}
                width="800"
                height="600"
                on:click={handleCanvasClick}
                on:mousemove={handleMouseMove}
        ></canvas>

        <div class="toolbar">
            <button on:click={() => selectTool("pencil")}>✏️ Lápiz</button>
            <button on:click={() => selectTool("eraser")}>🩹 Goma</button>
            <div>Ángulo: {currentAngle}°</div>
        </div>
    </div>
</div>