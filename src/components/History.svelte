<script>
  export let operations = [];
  export let selectedId = null;
  export let linesCount = 0;
  export let totalLength = 0;
  export let onSelect = (id) => {};
  export let onDelete = (id) => {};
  export let onClear = () => {};
  export let unit = 'cm';
  export let gridSpacingPx;
</script>

<div class="history-panel bsr">
  <div class="history-header">
    <div>Historial</div>
    <button class="clear-btn" on:click={onClear}>Limpiar Todo</button>
  </div>

  <div class="listado-de-operaciones ">
    {#if operations.length === 0}
      <div class="empty-history">No hay operaciones registradas</div>
    {:else}
      {#each [...operations].slice().reverse() as line}
        <div class="operation-item {selectedId===line.id ? 'selected' : ''}"
             role="button" tabindex="0" aria-pressed={selectedId===line.id}
             on:click={() => onSelect(line.id)}
             on:keydown={(e) => { if (e.key === 'Enter' || e.key === ' ') { onSelect(line.id); } }}>
          Línea #{line.id} | ángulo: {line.angle}° | longitud: {Math.round(line.dist / gridSpacingPx)} {unit}
          <button class="delete-line-btn" on:click|stopPropagation={() => onDelete(line.id)}>Eliminar</button>
        </div>
      {/each}
    {/if}
  </div>

  <div class="stats">
    <div>Líneas: {linesCount}</div>
    <div>Longitud total: {totalLength} {unit}</div>
  </div>
</div>

<style>
  .history-panel { width:300px; background:#2c3e50; color:white; padding:20px; overflow-y:auto; }
  .listado-de-operaciones
  {
    background-color:rgba(0,0,0,0.2);
    height: calc(100% - 100px);
    overflow-y:scroll;
    padding: 12px;
    border-radius: 10px;
  }
  .history-header { display:flex; justify-content:space-between; margin-bottom:20px; }
  .clear-btn { background:#e74c3c; border:none; padding:8px 15px; border-radius:5px; cursor:pointer; font-weight:bold; color:white; }
  .operation-item { padding:10px; margin-bottom:10px; background:#34495e; border-radius:5px; cursor:pointer; }
  .operation-item.selected { border:2px solid #e74c3c; background:#c0392b; }
  .delete-line-btn { margin-top:5px; background:#c0392b; color:white; border:none; padding:5px 10px; border-radius:5px; cursor:pointer; }
  .stats { margin-top:10px; font-size:14px; color:#ecf0f1; }
</style>
