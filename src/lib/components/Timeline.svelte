<script lang="ts">
  import { onMount } from 'svelte';
  import OscillatorBlock from './OscillatorBlock.svelte';

  // タイムラインの状態管理
  let currentTime = $state(0);
  let duration = $state(60); // 60秒
  let isPlaying = $state(false);
  let zoom = $state(1); // ズームレベル
  let tracks = $state([
    { id: 1, name: 'Track 1', frequency: 440, type: 'sine' },
    { id: 2, name: 'Track 2', frequency: 880, type: 'triangle' }
  ]);

  // オシレーターブロック管理
  let blocks = $state([
    { id: 1, trackId: 1, startTime: 0, duration: 4, frequency: 440, type: 'sine' },
    { id: 2, trackId: 1, startTime: 6, duration: 3, frequency: 660, type: 'triangle' },
    { id: 3, trackId: 2, startTime: 2, duration: 5, frequency: 880, type: 'square' }
  ]);
  let selectedBlockId: number | null = null;
  let nextBlockId = 4;

  // 右クリックメニュー管理
  let contextMenu = $state({
    visible: false,
    x: 0,
    y: 0,
    trackId: null as number | null,
    clickTime: 0
  });

  // Web Audio API関連
  let audioContext: AudioContext | null = null;
  let oscillators: Map<number, OscillatorNode> = new Map();
  let gainNodes: Map<number, GainNode> = new Map();
  let masterGain: GainNode | null = null;

  // タイムラインの設定
  const pixelsPerSecond = 50 * zoom;
  const timelineHeight = 400;
  const trackHeight = 100;

  // 再生制御
  let animationFrameId: number;
  let startTime: number;

  // Web Audio API初期化
  function initAudioContext() {
    if (!audioContext) {
      audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
      masterGain = audioContext.createGain();
      masterGain.connect(audioContext.destination);
      masterGain.gain.value = 0.3; // マスターボリューム
    }
  }

  // オシレーター作成（ブロックベース）
  function createOscillator(blockId: number, frequency: number, type: OscillatorType) {
    if (!audioContext || !masterGain) return;

    // 既存のオシレーターを停止・削除
    stopOscillator(blockId);

    const oscillator = audioContext.createOscillator();
    const gainNode = audioContext.createGain();
    
    oscillator.frequency.setValueAtTime(frequency, audioContext.currentTime);
    oscillator.type = type;
    
    gainNode.gain.setValueAtTime(0.1, audioContext.currentTime);
    
    oscillator.connect(gainNode);
    gainNode.connect(masterGain);
    
    oscillators.set(blockId, oscillator);
    gainNodes.set(blockId, gainNode);
    
    oscillator.start();
  }

  // オシレーター停止
  function stopOscillator(blockId: number) {
    const oscillator = oscillators.get(blockId);
    const gainNode = gainNodes.get(blockId);
    
    if (oscillator) {
      oscillator.stop();
      oscillators.delete(blockId);
    }
    
    if (gainNode) {
      gainNodes.delete(blockId);
    }
  }

  // 全オシレーター停止
  function stopAllOscillators() {
    oscillators.forEach((oscillator, blockId) => {
      oscillator.stop();
    });
    oscillators.clear();
    gainNodes.clear();
  }

  // 現在再生中のブロックを取得
  function getActiveBlocks(): typeof blocks {
    return blocks.filter(block => {
      const blockEnd = block.startTime + block.duration;
      return currentTime >= block.startTime && currentTime < blockEnd;
    });
  }

  // 再生中のブロックを更新
  function updateActiveOscillators() {
    const activeBlocks = getActiveBlocks();
    
    // 現在アクティブでないブロックのオシレーターを停止
    blocks.forEach(block => {
      const isActive = activeBlocks.some(activeBlock => activeBlock.id === block.id);
      if (!isActive) {
        stopOscillator(block.id);
      }
    });
    
    // アクティブなブロックのオシレーターを開始
    activeBlocks.forEach(block => {
      if (!oscillators.has(block.id)) {
        createOscillator(block.id, block.frequency, block.type as OscillatorType);
      }
    });
  }

  function togglePlay() {
    if (isPlaying) {
      stopPlayback();
    } else {
      startPlayback();
    }
  }

  function startPlayback() {
    if (!audioContext) {
      initAudioContext();
    }
    
    isPlaying = true;
    startTime = performance.now() - (currentTime * 1000);
    animate();
  }

  function stopPlayback() {
    isPlaying = false;
    stopAllOscillators();
    if (animationFrameId) {
      cancelAnimationFrame(animationFrameId);
    }
  }

  function animate() {
    if (!isPlaying) return;
    
    currentTime = (performance.now() - startTime) / 1000;
    
    if (currentTime >= duration) {
      currentTime = 0;
      stopPlayback();
      return;
    }
    
    updateActiveOscillators();
    animationFrameId = requestAnimationFrame(animate);
  }

  function seekTo(event: MouseEvent) {
    // ブロック選択中はシーク機能を無効化
    if (selectedBlockId !== null) {
      return;
    }
    
    const target = event.currentTarget as HTMLElement;
    const rect = target.getBoundingClientRect();
    const x = event.clientX - rect.left;
    currentTime = (x / pixelsPerSecond);
    currentTime = Math.max(0, Math.min(currentTime, duration));
    
    if (isPlaying) {
      updateActiveOscillators();
    }
  }

  function addTrack() {
    const newTrack = {
      id: tracks.length + 1,
      name: `Track ${tracks.length + 1}`,
      frequency: 440,
      type: 'sine'
    };
    tracks = [...tracks, newTrack];
  }

  function removeTrack(trackId: number) {
    tracks = tracks.filter(track => track.id !== trackId);
    // トラックに関連するブロックも削除
    blocks = blocks.filter(block => block.trackId !== trackId);
  }

  // ブロック追加
  function addBlock(trackId: number, startTime: number) {
    const newBlock = {
      id: nextBlockId++,
      trackId,
      startTime,
      duration: 4,
      frequency: 440,
      type: 'sine'
    };
    blocks = [...blocks, newBlock];
  }

  // ブロック更新
  function updateBlock(event: CustomEvent) {
    const { id, startTime, duration, frequency, type, trackId } = event.detail;
    blocks = blocks.map(block => 
      block.id === id ? { ...block, startTime, duration, frequency, type, trackId } : block
    );
    
    // 再生中なら即座に更新
    if (isPlaying) {
      const block = blocks.find(b => b.id === id);
      if (block) {
        const isActive = currentTime >= block.startTime && currentTime < block.startTime + block.duration;
        if (isActive) {
          createOscillator(block.id, block.frequency, block.type as OscillatorType);
        } else {
          stopOscillator(block.id);
        }
      }
    }
  }

  // ブロック削除
  function deleteBlock(event: CustomEvent) {
    const { id } = event.detail;
    blocks = blocks.filter(block => block.id !== id);
    stopOscillator(id);
  }

  // ブロック選択
  function selectBlock(event: CustomEvent) {
    const { id } = event.detail;
    selectedBlockId = id;
  }

  // 右クリックメニュー処理
  function handleContextMenu(event: MouseEvent, trackId: number) {
    event.preventDefault();
    
    const target = event.currentTarget as HTMLElement;
    const rect = target.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const clickTime = x / pixelsPerSecond;
    
    contextMenu = {
      visible: true,
      x: event.clientX,
      y: event.clientY,
      trackId,
      clickTime
    };
  }

  // メニューを閉じる
  function closeContextMenu() {
    contextMenu.visible = false;
  }

  // オシレーター追加
  function addOscillator() {
    if (contextMenu.trackId && contextMenu.visible) {
      addBlock(contextMenu.trackId, contextMenu.clickTime);
      closeContextMenu();
    }
  }

  // キーボードイベント処理
  function handleKeyDown(event: KeyboardEvent) {
    if (event.key === 'Escape') {
      closeContextMenu();
      // ブロック選択解除
      selectedBlockId = null;
    }
  }

  // メニュー外クリック処理
  function handleGlobalClick(event: MouseEvent) {
    const menuElement = document.querySelector('.context-menu');
    if (menuElement && !menuElement.contains(event.target as Node)) {
      closeContextMenu();
    }
    
    // ブロック選択外クリックで選択解除
    const blockElements = document.querySelectorAll('.oscillator-block');
    let clickedOnBlock = false;
    
    blockElements.forEach(blockElement => {
      if (blockElement.contains(event.target as Node)) {
        clickedOnBlock = true;
      }
    });
    
    if (selectedBlockId !== null && !clickedOnBlock) {
      selectedBlockId = null;
    }
  }

  function updateTrackFrequency(trackId: number, frequency: number) {
    tracks = tracks.map(track => 
      track.id === trackId ? { ...track, frequency } : track
    );
  }

  function updateTrackType(trackId: number, type: string) {
    tracks = tracks.map(track => 
      track.id === trackId ? { ...track, type } : track
    );
  }

  // 時間フォーマット
  function formatTime(seconds: number): string {
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  }

  onMount(() => {
    // グローバルイベントリスナーを追加
    document.addEventListener('keydown', handleKeyDown);
    document.addEventListener('click', handleGlobalClick);
    
    return () => {
      if (animationFrameId) {
        cancelAnimationFrame(animationFrameId);
      }
      stopAllOscillators();
      if (audioContext) {
        audioContext.close();
      }
      document.removeEventListener('keydown', handleKeyDown);
      document.removeEventListener('click', handleGlobalClick);
    };
  });
</script>

<div class="timeline-container">
  <!-- ヘッダー -->
  <div class="timeline-header">
    <div class="controls">
      <button class="play-button" on:click={togglePlay}>
        {isPlaying ? '⏸' : '▶'}
      </button>
      <button class="stop-button" on:click={stopPlayback}>⏹</button>
      <span class="time-display">{formatTime(currentTime)} / {formatTime(duration)}</span>
    </div>
    <div class="zoom-controls">
      <button on:click={() => zoom = Math.max(0.5, zoom - 0.5)}>-</button>
      <span>{zoom}x</span>
      <button on:click={() => zoom = Math.min(4, zoom + 0.5)}>+</button>
    </div>
  </div>

  <!-- タイムライン本体 -->
  <div class="timeline-main">
    <!-- トラックリスト -->
    <div class="track-list">
      <div class="track-header">
        <span>Track</span>
        <button class="add-track-btn" on:click={addTrack}>+</button>
      </div>
      {#each tracks as track (track.id)}
        <div class="track-item">
          <div class="track-info">
            <span class="track-name">{track.name}</span>
            <button class="remove-track-btn" on:click={() => removeTrack(track.id)}>×</button>
          </div>
        </div>
      {/each}
    </div>

    <!-- タイムライン表示エリア -->
    <div class="timeline-display">
      <!-- 時間軸ラベル -->
      <div class="time-ruler">
        {#each Array.from({ length: Math.ceil(duration) + 1 }) as _, i}
          <div 
            class="time-mark" 
            style="left: {i * pixelsPerSecond}px"
          >
            {formatTime(i)}
          </div>
        {/each}
      </div>

      <!-- 再生ヘッド -->
      <div 
        class="playhead" 
        style="left: {currentTime * pixelsPerSecond}px"
      ></div>

      <!-- トラック表示エリア -->
      <div 
        class="tracks-display" 
        class:selection-mode={selectedBlockId !== null}
        on:click={seekTo}
      >
        {#each tracks as track (track.id)}
          <div class="track-display" style="height: {trackHeight}px;">
            <div 
              class="track-visualization" 
              on:contextmenu={(e) => handleContextMenu(e, track.id)}
            >
              <span class="track-label">{track.name}</span>
              
              <!-- このトラックのブロックを表示 -->
              {#each blocks.filter(block => block.trackId === track.id) as block (block.id)}
                <OscillatorBlock
                  id={block.id}
                  trackId={block.trackId}
                  startTime={block.startTime}
                  duration={block.duration}
                  frequency={block.frequency}
                  type={block.type}
                  isSelected={selectedBlockId === block.id}
                  on:update={updateBlock}
                  on:delete={deleteBlock}
                  on:select={selectBlock}
                />
              {/each}
            </div>
          </div>
        {/each}
      </div>
    </div>
  </div>

  <!-- 右クリックメニュー -->
  {#if contextMenu.visible}
    <div 
      class="context-menu"
      style="left: {contextMenu.x}px; top: {contextMenu.y}px;"
    >
      <div class="menu-item" on:click={addOscillator}>
        <span class="menu-icon"></span>
        <span>Add Oscillator</span>
      </div>
    </div>
  {/if}
</div>

<style>
  .timeline-container {
    display: flex;
    flex-direction: column;
    height: 100vh;
    background: #1a1a1a;
    color: #ffffff;
    font-family: 'Inter', sans-serif;
  }

  .timeline-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;
    background: #2a2a2a;
    border-bottom: 1px solid #3a3a3a;
  }

  .controls {
    display: flex;
    align-items: center;
    gap: 1rem;
  }

  .play-button, .stop-button {
    background: #4a4a4a;
    border: none;
    color: white;
    padding: 0.5rem 1rem;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1.2rem;
  }

  .play-button:hover, .stop-button:hover {
    background: #5a5a5a;
  }

  .time-display {
    font-family: monospace;
    font-size: 1.1rem;
  }

  .zoom-controls {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .zoom-controls button {
    background: #4a4a4a;
    border: none;
    color: white;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    cursor: pointer;
  }

  .timeline-main {
    display: flex;
    flex: 1;
    overflow: hidden;
  }

  .track-list {
    width: 250px;
    background: #2a2a2a;
    border-right: 1px solid #3a3a3a;
    overflow-y: auto;
  }

  .track-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;
    border-bottom: 1px solid #3a3a3a;
    font-weight: bold;
  }

  .add-track-btn {
    background: #4CAF50;
    border: none;
    color: white;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1.2rem;
  }

  .track-item {
    padding: 1rem;
    border-bottom: 1px solid #3a3a3a;
  }

  .track-info {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .track-name {
    font-weight: 500;
  }

  .remove-track-btn {
    background: #f44336;
    border: none;
    color: white;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    cursor: pointer;
  }

  .timeline-display {
    flex: 1;
    position: relative;
    overflow: auto;
  }

  .time-ruler {
    position: sticky;
    top: 0;
    height: 30px;
    background: #2a2a2a;
    border-bottom: 1px solid #3a3a3a;
    z-index: 10;
  }

  .time-mark {
    position: absolute;
    top: 5px;
    font-size: 0.8rem;
    color: #888;
    transform: translateX(-50%);
  }

  .playhead {
    position: absolute;
    top: 0;
    width: 2px;
    height: 100%;
    background: #ff4444;
    z-index: 20;
    pointer-events: none;
  }

  .tracks-display {
    position: relative;
    min-height: 100%;
  }

  .track-display {
    border-bottom: 1px solid #3a3a3a;
    position: relative;
  }

  .track-visualization {
    display: flex;
    align-items: center;
    padding: 0 1rem;
    height: 100%;
    position: relative;
    cursor: pointer;
  }

  .track-visualization:hover {
    background: rgba(255, 255, 255, 0.05);
  }

  .track-label {
    font-family: monospace;
    font-size: 0.9rem;
    color: #888;
    position: absolute;
    left: 1rem;
    z-index: 5;
  }

  /* 右クリックメニュー */
  .context-menu {
    position: fixed;
    background: #2a2a2a;
    border: 1px solid #3a3a3a;
    border-radius: 6px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
    z-index: 1000;
    min-width: 150px;
    padding: 0.5rem 0;
  }

  .menu-item {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    cursor: pointer;
    transition: background-color 0.2s;
    font-size: 0.9rem;
  }

  .menu-item:hover {
    background: #3a3a3a;
  }

  .menu-icon {
    font-size: 1rem;
  }
</style> 