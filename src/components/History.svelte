<script>
  export let operations = [];
  export let selectedId = null;
  export let linesCount = 0;
  export let totalLength = 0;
  export let onSelect = (id) => {};
  export let onDelete = (id) => {};
  export let onClear = () => {};
</script>

<div class="history-panel">
  <div class="history-header">
    <div>Historial</div>
    <button class="clear-btn" on:click={onClear}>Limpiar Todo</button>
  </div>

  {#if operations.length === 0}
    <div class="empty-history">No hay operaciones registradas</div>
  {:else}
    {#each [...operations].slice().reverse() as line}
      <div class="operation-item {selectedId===line.id ? 'selected' : ''}"
           role="button" tabindex="0" aria-pressed={selectedId===line.id}
           on:click={() => onSelect(line.id)}
           on:keydown={(e) => { if (e.key === 'Enter' || e.key === ' ') { onSelect(line.id); } }}>
        Línea #{line.id} | ángulo: {line.angle}° | longitud: {Math.round(line.dist)}px
        <button class="delete-line-btn" on:click|stopPropagation={() => onDelete(line.id)}>Eliminar</button>
      </div>
    {/each}
  {/if}

  <div class="stats">
    <div>Líneas: {linesCount}</div>
    <div>Longitud total: {totalLength}px</div>
  </div>
</div>

<style>
  .history-panel { width:300px; background:#2c3e50; color:white; padding:20px; overflow-y:auto; }
  .history-header { display:flex; justify-content:space-between; margin-bottom:20px; }
  .clear-btn { background:#e74c3c; border:none; padding:8px 15px; border-radius:5px; cursor:pointer; font-weight:bold; }
  .operation-item { padding:10px; margin-bottom:10px; background:#34495e; border-radius:5px; cursor:pointer; }
  .operation-item.selected { border:2px solid #e74c3c; background:#c0392b; }
  .delete-line-btn { margin-top:5px; background:#c0392b; color:white; border:none; padding:5px 10px; border-radius:5px; cursor:pointer; }
  .stats { margin-top:10px; font-size:14px; color:#ecf0f1; display:flex; justify-content:space-between; }
</style>


