<script lang="ts">
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  export let id: number;
  export let startTime: number;
  export let duration: number;
  export let frequency: number;
  export let type: string;
  export let trackId: number;
  export let isSelected = false;

  let isDragging = false;
  let isResizing = false;
  let dragStartX = 0;
  let dragStartTime = 0;
  let resizeStartX = 0;
  let resizeStartDuration = 0;
  let element: HTMLElement;

  // ドラッグ開始
  function handleMouseDown(event: MouseEvent) {
    // 入力要素がクリックされた場合はドラッグを開始しない
    const target = event.target as HTMLElement;
    if (target.tagName === 'INPUT' || target.tagName === 'SELECT' || target.tagName === 'BUTTON' || 
        target.closest('input') || target.closest('select') || target.closest('button')) {
      return;
    }
    
    if (event.target === element || element.contains(event.target as Node)) {
      isDragging = true;
      dragStartX = event.clientX;
      dragStartTime = startTime;
      document.addEventListener('mousemove', handleMouseMove);
      document.addEventListener('mouseup', handleMouseUp);
      event.preventDefault();
    }
  }

  // リサイズ開始
  function handleResizeStart(event: MouseEvent) {
    isResizing = true;
    resizeStartX = event.clientX;
    resizeStartDuration = duration;
    document.addEventListener('mousemove', handleResizeMove);
    document.addEventListener('mouseup', handleResizeEnd);
    event.preventDefault();
    event.stopPropagation();
  }

  // ドラッグ移動
  function handleMouseMove(event: MouseEvent) {
    if (!isDragging) return;
    
    const deltaX = event.clientX - dragStartX;
    const pixelsPerSecond = 50; // Timeline.svelteと同じ値
    const deltaTime = deltaX / pixelsPerSecond;
    const newStartTime = Math.max(0, dragStartTime + deltaTime);
    
    dispatch('update', {
      id,
      startTime: newStartTime,
      duration,
      frequency,
      type,
      trackId
    });
  }

  // リサイズ移動
  function handleResizeMove(event: MouseEvent) {
    if (!isResizing) return;
    
    const deltaX = event.clientX - resizeStartX;
    const pixelsPerSecond = 50;
    const deltaTime = deltaX / pixelsPerSecond;
    const newDuration = Math.max(0.1, resizeStartDuration + deltaTime);
    
    dispatch('update', {
      id,
      startTime,
      duration: newDuration,
      frequency,
      type,
      trackId
    });
  }

  // ドラッグ終了
  function handleMouseUp() {
    isDragging = false;
    document.removeEventListener('mousemove', handleMouseMove);
    document.removeEventListener('mouseup', handleMouseUp);
  }

  // リサイズ終了
  function handleResizeEnd() {
    isResizing = false;
    document.removeEventListener('mousemove', handleResizeMove);
    document.removeEventListener('mouseup', handleResizeEnd);
  }

  // ブロック選択
  function handleClick(event: MouseEvent) {
    // 入力フィールドやボタンがクリックされた場合は選択しない
    const target = event.target as HTMLElement;
    if (target.tagName === 'INPUT' || target.tagName === 'SELECT' || target.tagName === 'BUTTON' || 
        target.closest('input') || target.closest('select') || target.closest('button')) {
      return;
    }
    
    if (!isDragging && !isResizing) {
      dispatch('select', { id, trackId });
    }
  }

  // 削除
  function handleDelete(event: MouseEvent) {
    event.stopPropagation();
    dispatch('delete', { id, trackId });
  }

  // 入力フィールドのクリックイベント
  function handleInputClick(event: MouseEvent) {
    event.stopPropagation();
  }

  // セレクトボックスのクリックイベント
  function handleSelectClick(event: MouseEvent) {
    event.stopPropagation();
  }

  // 入力フィールドのフォーカスイベント
  function handleInputFocus(event: FocusEvent) {
    event.stopPropagation();
  }

  // セレクトボックスのフォーカスイベント
  function handleSelectFocus(event: FocusEvent) {
    event.stopPropagation();
  }

  // 入力フィールドのマウスダウンイベント
  function handleInputMouseDown(event: MouseEvent) {
    event.stopPropagation();
  }

  // セレクトボックスのマウスダウンイベント
  function handleSelectMouseDown(event: MouseEvent) {
    event.stopPropagation();
  }

  // 周波数変更
  function handleFrequencyChange(event: Event) {
    const target = event.target as HTMLInputElement;
    const newFrequency = parseInt(target.value);
    dispatch('update', {
      id,
      startTime,
      duration,
      frequency: newFrequency,
      type,
      trackId
    });
  }

  // 波形変更
  function handleTypeChange(event: Event) {
    const target = event.target as HTMLSelectElement;
    const newType = target.value;
    dispatch('update', {
      id,
      startTime,
      duration,
      frequency,
      type: newType,
      trackId
    });
  }

  // 時間フォーマット
  function formatTime(seconds: number): string {
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  }
</script>

<div
  bind:this={element}
  class="oscillator-block"
  class:selected={isSelected}
  class:dragging={isDragging}
  class:resizing={isResizing}
  style="left: {startTime * 50}px; width: {duration * 50}px;"
  on:mousedown={handleMouseDown}
  on:click={handleClick}
>
  <div class="block-content">
    <div class="block-header">
      <span class="frequency">{frequency}Hz</span>
      <button class="delete-btn" on:click={handleDelete}>×</button>
    </div>
    <div class="block-body">
      <input
        type="number"
        value={frequency}
        on:input={handleFrequencyChange}
        on:click={handleInputClick}
        on:focus={handleInputFocus}
        on:mousedown={handleInputMouseDown}
        placeholder="周波数"
        min="20"
        max="20000"
        class="frequency-input"
      />
      <select 
        value={type} 
        on:change={handleTypeChange} 
        on:click={handleSelectClick}
        on:focus={handleSelectFocus}
        on:mousedown={handleSelectMouseDown}
        class="type-select"
      >
        <option value="sine">正弦波</option>
        <option value="triangle">三角波</option>
        <option value="square">矩形波</option>
        <option value="sawtooth">ノコギリ波</option>
      </select>
    </div>
    <div class="block-footer">
      <span class="duration">{formatTime(duration)}</span>
    </div>
  </div>
  <div class="resize-handle" on:mousedown={handleResizeStart}></div>
</div>

<style>
  .oscillator-block {
    position: absolute;
    height: 50px;
    background: linear-gradient(135deg, #4CAF50, #45a049);
    border: 2px solid #2E7D32;
    border-radius: 6px;
    cursor: move;
    user-select: none;
    transition: all 0.2s ease;
    min-width: 60px;
  }

  .oscillator-block:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
  }

  .oscillator-block.selected {
    border-color: #FFD700;
    box-shadow: 0 0 0 2px rgba(255, 215, 0, 0.5);
  }

  .oscillator-block.dragging {
    opacity: 0.8;
    z-index: 1000;
  }

  .oscillator-block.resizing {
    opacity: 0.8;
    z-index: 1000;
  }

  .block-content {
    height: 100%;
    display: flex;
    flex-direction: column;
    padding: 4px;
    color: white;
    font-size: 0.8rem;
  }

  .block-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2px;
  }

  .frequency {
    font-weight: bold;
    font-family: monospace;
  }

  .delete-btn {
    background: rgba(255, 255, 255, 0.2);
    border: none;
    color: white;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    cursor: pointer;
    font-size: 0.7rem;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .delete-btn:hover {
    background: rgba(255, 255, 255, 0.3);
  }

  .block-body {
    display: flex;
    flex-direction: column;
    gap: 2px;
    flex: 1;
  }

  .frequency-input,
  .type-select {
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: white;
    padding: 2px 4px;
    border-radius: 3px;
    font-size: 0.7rem;
    width: 100%;
  }

  .frequency-input::placeholder {
    color: rgba(255, 255, 255, 0.6);
  }

  .block-footer {
    display: flex;
    justify-content: center;
    margin-top: 2px;
  }

  .duration {
    font-size: 0.6rem;
    color: rgba(255, 255, 255, 0.8);
    font-family: monospace;
  }

  .resize-handle {
    position: absolute;
    right: -4px;
    top: 0;
    bottom: 0;
    width: 8px;
    cursor: ew-resize;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 2px;
  }

  .resize-handle:hover {
    background: rgba(255, 255, 255, 0.4);
  }
</style> 