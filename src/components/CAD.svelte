<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dibujo con Historial - 2 Canvas</title>
    <script src="https://cdn.jsdelivr.net/npm/svelte@3.59.2/index.js"></script>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            margin: 0;
            padding: 20px;
            background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
            display: flex;
            justify-content: center;
            min-height: 100vh;
        }

        .app-container {
            display: flex;
            width: 100%;
            max-width: 1200px;
            height: 90vh;
            background: white;
            border-radius: 12px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.15);
            overflow: hidden;
        }

        .history-panel {
            width: 300px;
            padding: 20px;
            background-color: #2c3e50;
            color: white;
            overflow-y: auto;
            height: 100%;
        }

        .history-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 20px;
            padding-bottom: 15px;
            border-bottom: 2px solid rgba(255, 255, 255, 0.2);
        }

        .history-title {
            font-size: 20px;
            font-weight: bold;
            color: #ecf0f1;
        }

        .clear-btn {
            background-color: #e74c3c;
            color: white;
            border: none;
            padding: 8px 15px;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            transition: background-color 0.3s;
        }

        .clear-btn:hover {
            background-color: #c0392b;
        }

        .operation-item {
            padding: 15px;
            margin-bottom: 15px;
            background-color: #34495e;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            transition: transform 0.2s;
        }

        .operation-item:hover {
            transform: translateY(-2px);
            background-color: #3d566e;
        }

        .operation-header {
            display: flex;
            justify-content: space-between;
            font-size: 13px;
            color: #bdc3c7;
            margin-bottom: 10px;
        }

        .operation-content {
            font-size: 15px;
            color: #ecf0f1;
        }

        .operation-coords {
            margin-top: 8px;
            font-size: 13px;
            color: #95a5a6;
        }

        .drawing-area {
            flex: 1;
            display: flex;
            flex-direction: column;
            padding: 20px;
            background: #ecf0f1;
        }

        .drawing-container {
            position: relative;
            width: 100%;
            height: 100%;
            border: 2px solid #bdc3c7;
            border-radius: 8px;
            overflow: hidden;
            background: white;
        }

        canvas {
            position: absolute;
            top: 0;
            left: 0;
        }

        .toolbar {
            position: absolute;
            top: 10px;
            left: 10px;
            right: 10px;
            height: 60px;
            background: rgba(255, 255, 255, 0.9);
            display: flex;
            gap: 15px;
            align-items: center;
            padding: 0 15px;
            z-index: 10;
            border-radius: 8px;
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
        }

        .tool-btn {
            padding: 8px 16px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            transition: all 0.3s;
        }

        .pencil-btn {
            background-color: #3498db;
            color: white;
        }

        .pencil-btn.active {
            background-color: #2980b9;
            box-shadow: 0 0 0 3px rgba(52, 152, 219, 0.3);
        }

        .eraser-btn {
            background-color: #e67e22;
            color: white;
        }

        .eraser-btn.active {
            background-color: #d35400;
            box-shadow: 0 0 0 3px rgba(230, 126, 34, 0.3);
        }

        .angle-display {
            margin-left: auto;
            font-size: 16px;
            font-weight: bold;
            color: #2c3e50;
            background: #ecf0f1;
            padding: 8px 15px;
            border-radius: 5px;
        }

        .instructions {
            margin-top: 15px;
            padding: 15px;
            background: #d6dbdf;
            border-radius: 8px;
            font-size: 14px;
            color: #2c3e50;
        }

        .instructions h3 {
            margin-top: 0;
            color: #2c3e50;
        }

        .instructions ul {
            padding-left: 20px;
            margin-bottom: 0;
        }

        .empty-history {
            text-align: center;
            color: #7f8c8d;
            margin-top: 40px;
            font-style: italic;
        }

        .stats {
            display: flex;
            justify-content: space-between;
            margin-top: 15px;
            color: #2c3e50;
            font-size: 14px;
        }
    </style>
</head>
<body class="bsb">
<div class="app-container bsr">
    <!-- Panel de historial a la izquierda -->
    <div class="history-panel">
        <div class="history-header">
            <div class="history-title">Historial de Operaciones</div>
            <button class="clear-btn" id="clearBtn">Limpiar Todo</button>
        </div>

        <div id="history-content">
            <div class="empty-history">No hay operaciones registradas</div>
        </div>

        <div class="stats">
            <div id="lines-count">Líneas: 0</div>
            <div id="total-length">Longitud total: 0px</div>
        </div>
    </div>

    <!-- Área de dibujo -->
    <div class="drawing-area">
        <div class="drawing-container">
            <canvas id="baseCanvas" width="800" height="600"></canvas>
            <canvas id="drawCanvas" width="800" height="600"></canvas>

            <div class="toolbar">
                <button class="tool-btn pencil-btn active" id="pencilBtn">✏️ Lápiz</button>
                <button class="tool-btn eraser-btn" id="eraserBtn">🩹 Goma</button>
                <div class="angle-display">Ángulo: <span id="angleValue">0</span>°</div>
            </div>
        </div>

        <div class="instructions">
            <h3>Instrucciones:</h3>
            <ul>
                <li>Haz clic para establecer el punto inicial de la línea</li>
                <li>Haz clic nuevamente para completar la línea</li>
                <li>Presiona Enter mientras dibujas para ingresar un ángulo personalizado</li>
                <li>Usa el botón "Limpiar Todo" para borrar el historial</li>
            </ul>
        </div>
    </div>
</div>

<script>
    // Variables globales
    let baseCtx, drawCtx;
    let tool = "pencil";
    let drawing = false;
    let startX, startY;
    let currentAngle = 0;
    let customAngle = null;

    // Arrays para almacenamiento
    let operationsHistory = [];
    let drawnLines = [];
    let nextOperationId = 1;

    // Elementos del DOM
    const baseCanvas = document.getElementById('baseCanvas');
    const drawCanvas = document.getElementById('drawCanvas');
    const historyContent = document.getElementById('history-content');
    const clearBtn = document.getElementById('clearBtn');
    const pencilBtn = document.getElementById('pencilBtn');
    const eraserBtn = document.getElementById('eraserBtn');
    const angleValue = document.getElementById('angleValue');
    const linesCount = document.getElementById('lines-count');
    const totalLength = document.getElementById('total-length');

    // Inicialización
    window.onload = function() {
        baseCtx = baseCanvas.getContext("2d");
        drawCtx = drawCanvas.getContext("2d");

        // Cargar imagen de fondo
        const img = new Image();
        img.src = "https://upload.wikimedia.org/wikipedia/commons/3/3f/Logo_svelte.svg";
        img.onload = () => {
            baseCtx.drawImage(img, 50, 50, 150, 150);
        };

        // Establecer manejadores de eventos
        drawCanvas.addEventListener('click', handleCanvasClick);
        drawCanvas.addEventListener('mousemove', handleMouseMove);
        window.addEventListener('keydown', handleKeyDown);
        clearBtn.addEventListener('click', clearHistory);
        pencilBtn.addEventListener('click', () => selectTool('pencil'));
        eraserBtn.addEventListener('click', () => selectTool('eraser'));

        // Estilo inicial del cursor
        drawCanvas.style.cursor = "crosshair";
    };

    // Funciones de herramientas
    function selectTool(t) {
        tool = t;
        drawing = false;

        // Actualizar botones
        pencilBtn.classList.toggle('active', tool === 'pencil');
        eraserBtn.classList.toggle('active', tool === 'eraser');

        // Cambiar cursor
        drawCanvas.style.cursor = tool === "pencil" ? "crosshair" : "not-allowed";
    }

    // Funciones de dibujo
    function snapAngle(dx, dy) {
        const angleRad = Math.atan2(dy, dx);
        let angleDeg = angleRad * (180 / Math.PI);

        const snaps = [0, 45, 90, 135, 180, -45, -90, -135];
        return snaps.reduce((prev, curr) =>
            Math.abs(curr - angleDeg) < Math.abs(prev - angleDeg) ? curr : prev
        );
    }

    function redrawAllLines() {
        drawCtx.clearRect(0, 0, drawCanvas.width, drawCanvas.height);

        drawnLines.forEach(line => {
            drawCtx.setLineDash([]);
            drawCtx.strokeStyle = "red";
            drawCtx.lineWidth = 2;
            drawCtx.beginPath();
            drawCtx.moveTo(line.startX, line.startY);
            drawCtx.lineTo(line.endX, line.endY);
            drawCtx.stroke();
        });

        updateStats();
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
            const dx = mouseX - startX;
            const dy = mouseY - startY;
            let angle = customAngle !== null ? customAngle : snapAngle(dx, dy);
            const rad = (angle * Math.PI) / 180;
            const dist = Math.sqrt(dx * dx + dy * dy);
            const endX = startX + dist * Math.cos(rad);
            const endY = startY + dist * Math.sin(rad);

            // Guardar la línea
            drawnLines.push({ startX, startY, endX, endY, angle, dist });

            // Redibujar
            redrawAllLines();

            // Guardar en historial
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

            updateHistoryDisplay();

            drawing = false;
            customAngle = null;
        }
    }

    function handleMouseMove(e) {
        if (!drawing) return;

        const rect = drawCanvas.getBoundingClientRect();
        const mouseX = e.clientX - rect.left;
        const mouseY = e.clientY - rect.top;

        // Redibujar líneas existentes
        redrawAllLines();

        // Dibujar previsualización
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

        // Mostrar ángulo
        drawCtx.fillStyle = "black";
        drawCtx.font = "16px sans-serif";
        drawCtx.fillText(currentAngle + "°", endX + 10, endY - 10);

        // Actualizar display de ángulo
        angleValue.textContent = currentAngle;
    }

    function handleKeyDown(e) {
        if (drawing && e.key === "Enter") {
            const input = prompt("Ingrese ángulo personalizado en grados:", currentAngle);
            if (input !== null) customAngle = parseFloat(input);
        }
    }

    // Funciones de historial
    function updateHistoryDisplay() {
        if (operationsHistory.length === 0) {
            historyContent.innerHTML = '<div class="empty-history">No hay operaciones registradas</div>';
            return;
        }

        let html = '';
        operationsHistory.slice().reverse().forEach(op => {
            html += `
                <div class="operation-item">
                    <div class="operation-header">
                        <span>Operación #${op.id}</span>
                        <span>${op.timestamp.toLocaleTimeString()}</span>
                    </div>
                    <div class="operation-content">
                        <div>Línea con ángulo: ${op.angle}°</div>
                        <div class="operation-coords">
                            Desde (${op.startX}, ${op.startY}) hasta (${op.endX}, ${op.endY})
                        </div>
                        <div class="operation-coords">
                            Longitud: ${op.length}px
                        </div>
                    </div>
                </div>
                `;
        });

        historyContent.innerHTML = html;
    }

    function updateStats() {
        linesCount.textContent = `Líneas: ${drawnLines.length}`;

        const totalLen = drawnLines.reduce((sum, line) => sum + line.dist, 0);
        totalLength.textContent = `Longitud total: ${Math.round(totalLen)}px`;
    }

    function clearHistory() {
        operationsHistory = [];
        drawnLines = [];
        nextOperationId = 1;
        redrawAllLines();
        updateHistoryDisplay();
    }
</script>
</body>
</html>