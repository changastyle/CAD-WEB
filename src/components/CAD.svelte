<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dibujo con Historial y Eliminación de Líneas</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin:0; padding:20px; background:#f0f2f5; display:flex; justify-content:center; min-height:100vh; }
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
</head>
<body>
<div class="app-container">
    <div class="history-panel">
        <div class="history-header">
            <div>Historial</div>
            <button id="clearBtn" class="clear-btn">Limpiar Todo</button>
        </div>
        <div id="history-content">
            <div class="empty-history">No hay operaciones registradas</div>
        </div>
        <div class="stats">
            <div id="lines-count">Líneas: 0</div>
            <div id="total-length">Longitud total: 0px</div>
        </div>
    </div>

    <div class="drawing-area">
        <div class="drawing-container">
            <canvas id="baseCanvas" width="800" height="600"></canvas>
            <canvas id="drawCanvas" width="800" height="600"></canvas>
            <div class="toolbar">
                <button id="pencilBtn" class="tool-btn pencil-btn active">✏️ Lápiz</button>
                <button id="eraserBtn" class="tool-btn eraser-btn">🩹 Goma</button>
                <div class="angle-display">Ángulo: <span id="angleValue">0</span>°</div>
            </div>
        </div>
    </div>
</div>

<script>
    let baseCtx = document.getElementById('baseCanvas').getContext('2d');
    let drawCtx = document.getElementById('drawCanvas').getContext('2d');

    let tool = 'pencil';
    let drawing = false;
    let startX = 0, startY = 0;
    let currentAngle = 0;
    let customAngle = null;
    let selectedLineId = null;

    let drawnLines = [];
    let operationsHistory = [];
    let nextOperationId = 1;

    // DOM elements
    const drawCanvas = document.getElementById('drawCanvas');
    const historyContent = document.getElementById('history-content');
    const pencilBtn = document.getElementById('pencilBtn');
    const eraserBtn = document.getElementById('eraserBtn');
    const clearBtn = document.getElementById('clearBtn');
    const angleValue = document.getElementById('angleValue');
    const linesCount = document.getElementById('lines-count');
    const totalLength = document.getElementById('total-length');

    // Cargar imagen de fondo
    const img = new Image();
    img.src = "https://upload.wikimedia.org/wikipedia/commons/3/3f/Logo_svelte.svg";
    img.onload = () => { baseCtx.drawImage(img,50,50,150,150); }

    // Herramientas
    pencilBtn.onclick = () => selectTool('pencil');
    eraserBtn.onclick = () => selectTool('eraser');
    clearBtn.onclick = clearHistory;

    function selectTool(t){
        tool = t; drawing = false;
        pencilBtn.classList.toggle('active', tool==='pencil');
        eraserBtn.classList.toggle('active', tool==='eraser');
        drawCanvas.style.cursor = tool==='pencil' ? 'crosshair' : 'not-allowed';
    }

    // Dibujo
    function snapAngle(dx,dy){
        const deg = Math.atan2(dy,dx)*(180/Math.PI);
        const snaps=[0,45,90,135,180,-45,-90,-135];
        return snaps.reduce((prev,curr)=>Math.abs(curr-deg)<Math.abs(prev-deg)?curr:prev);
    }

    drawCanvas.addEventListener('click', e=>{
        if(tool!=='pencil') return;
        const rect = drawCanvas.getBoundingClientRect();
        const mouseX = e.clientX-rect.left;
        const mouseY = e.clientY-rect.top;

        if(!drawing){
            startX = mouseX; startY = mouseY; drawing=true;
        } else {
            const dx = mouseX-startX;
            const dy = mouseY-startY;
            const angle = customAngle!==null?customAngle:snapAngle(dx,dy);
            const rad = angle*Math.PI/180;
            const dist = Math.sqrt(dx*dx + dy*dy);
            const endX = startX + dist*Math.cos(rad);
            const endY = startY + dist*Math.sin(rad);

            const line = { id: nextOperationId++, startX, startY, endX, endY, angle, dist };
            drawnLines.push(line);
            operationsHistory.push(line);
            drawing=false;
            customAngle=null;
            redrawAllLines();
            updateHistoryDisplay();
        }
    });

    drawCanvas.addEventListener('mousemove', e=>{
        if(!drawing) return;
        redrawAllLines();
        const rect = drawCanvas.getBoundingClientRect();
        const mouseX = e.clientX-rect.left;
        const mouseY = e.clientY-rect.top;
        let dx = mouseX-startX, dy = mouseY-startY;
        currentAngle = customAngle!==null?customAngle:snapAngle(dx,dy);
        const rad = currentAngle*Math.PI/180;
        const dist = Math.sqrt(dx*dx+dy*dy);
        const endX = startX+dist*Math.cos(rad);
        const endY = startY+dist*Math.sin(rad);

        drawCtx.setLineDash([5,5]);
        drawCtx.strokeStyle='blue';
        drawCtx.lineWidth=1;
        drawCtx.beginPath(); drawCtx.moveTo(startX,startY); drawCtx.lineTo(endX,endY); drawCtx.stroke();
        drawCtx.fillStyle='black';
        drawCtx.font='16px sans-serif';
        drawCtx.fillText(currentAngle+'°', endX+10,endY-10);
        angleValue.textContent = currentAngle;
    });

    window.addEventListener('keydown', e=>{
        if(drawing && e.key==='Enter'){
            const input = prompt("Ingrese ángulo personalizado:", currentAngle);
            if(input!==null) customAngle=parseFloat(input);
        }
    });

    function redrawAllLines(){
        drawCtx.clearRect(0,0,drawCanvas.width,drawCanvas.height);
        drawnLines.forEach(line=>{
            drawCtx.setLineDash([]);
            drawCtx.strokeStyle = (selectedLineId===line.id)?'orange':'red';
            drawCtx.lineWidth = 2;
            drawCtx.beginPath();
            drawCtx.moveTo(line.startX,line.startY);
            drawCtx.lineTo(line.endX,line.endY);
            drawCtx.stroke();
        });
        updateStats();
    }

    // Historial
    function updateHistoryDisplay(){
        if(operationsHistory.length===0){
            historyContent.innerHTML='<div class="empty-history">No hay operaciones registradas</div>';
            return;
        }
        let html='';
        operationsHistory.slice().reverse().forEach(line=>{
            html+=`
        <div class="operation-item ${selectedLineId===line.id?'selected':''}" data-id="${line.id}">
            Línea #${line.id} | ángulo: ${line.angle}° | longitud: ${Math.round(line.dist)}px
            <button class="delete-line-btn" data-id="${line.id}">Eliminar</button>
        </div>`;
        });
        historyContent.innerHTML=html;

        document.querySelectorAll('.operation-item').forEach(item=>{
            item.onclick = ()=>{
                selectedLineId=parseInt(item.dataset.id);
                redrawAllLines();
                updateHistoryDisplay();
            }
        });
        document.querySelectorAll('.delete-line-btn').forEach(btn=>{
            btn.onclick = e=>{
                e.stopPropagation();
                const id=parseInt(btn.dataset.id);
                operationsHistory = operationsHistory.filter(l=>l.id!==id);
                drawnLines = drawnLines.filter(l=>l.id!==id);
                if(selectedLineId===id) selectedLineId=null;
                redrawAllLines();
                updateHistoryDisplay();
            }
        });
    }

    function clearHistory(){
        operationsHistory=[]; drawnLines=[]; nextOperationId=1; selectedLineId=null;
        redrawAllLines(); updateHistoryDisplay();
    }

    function updateStats(){
        linesCount.textContent='Líneas: '+drawnLines.length;
        totalLength.textContent='Longitud total: '+Math.round(drawnLines.reduce((sum,l)=>sum+l.dist,0))+'px';
    }
</script>
</body>
</html>
