<script lang="ts">
  import { onMount } from 'svelte';

  // タイムラインの状態管理
  let currentTime = $state(0);
  let duration = $state(60); // 60秒
  let isPlaying = $state(false);
  let zoom = $state(1); // ズームレベル
  let tracks = $state([
    { id: 1, name: 'Track 1', frequency: 440, type: 'sine' },
    { id: 2, name: 'Track 2', frequency: 880, type: 'triangle' }
  ]);

  // Web Audio API関連
  let audioContext: AudioContext | null = null;
  let oscillators: Map<number, OscillatorNode> = new Map();
  let gainNodes: Map<number, GainNode> = new Map();
  let masterGain: GainNode | null = null;

  // タイムラインの設定
  const pixelsPerSecond = 50 * zoom;
  const timelineHeight = 400;
  const trackHeight = 60;

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

  // オシレーター作成
  function createOscillator(trackId: number, frequency: number, type: OscillatorType) {
    if (!audioContext || !masterGain) return;

    // 既存のオシレーターを停止・削除
    stopOscillator(trackId);

    const oscillator = audioContext.createOscillator();
    const gainNode = audioContext.createGain();
    
    oscillator.frequency.setValueAtTime(frequency, audioContext.currentTime);
    oscillator.type = type;
    
    gainNode.gain.setValueAtTime(0.1, audioContext.currentTime); // 個別トラックのボリューム
    
    oscillator.connect(gainNode);
    gainNode.connect(masterGain);
    
    oscillators.set(trackId, oscillator);
    gainNodes.set(trackId, gainNode);
    
    oscillator.start();
  }

  // オシレーター停止
  function stopOscillator(trackId: number) {
    const oscillator = oscillators.get(trackId);
    const gainNode = gainNodes.get(trackId);
    
    if (oscillator) {
      oscillator.stop();
      oscillators.delete(trackId);
    }
    
    if (gainNode) {
      gainNodes.delete(trackId);
    }
  }

  // 全オシレーター停止
  function stopAllOscillators() {
    oscillators.forEach((oscillator, trackId) => {
      oscillator.stop();
    });
    oscillators.clear();
    gainNodes.clear();
  }

  // 全トラック再生開始
  function startAllOscillators() {
    if (!audioContext) {
      initAudioContext();
    }
    
    tracks.forEach(track => {
      createOscillator(track.id, track.frequency, track.type as OscillatorType);
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
    startAllOscillators();
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
    
    animationFrameId = requestAnimationFrame(animate);
  }

  function seekTo(event: MouseEvent) {
    const target = event.currentTarget as HTMLElement;
    const rect = target.getBoundingClientRect();
    const x = event.clientX - rect.left;
    currentTime = (x / pixelsPerSecond);
    currentTime = Math.max(0, Math.min(currentTime, duration));
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
    stopOscillator(trackId);
  }

  function updateTrackFrequency(trackId: number, frequency: number) {
    tracks = tracks.map(track => 
      track.id === trackId ? { ...track, frequency } : track
    );
    
    // 再生中なら即座に周波数を更新
    if (isPlaying) {
      const oscillator = oscillators.get(trackId);
      if (oscillator && audioContext) {
        oscillator.frequency.setValueAtTime(frequency, audioContext.currentTime);
      }
    }
  }

  function updateTrackType(trackId: number, type: string) {
    tracks = tracks.map(track => 
      track.id === trackId ? { ...track, type } : track
    );
    
    // 再生中なら即座に波形を更新
    if (isPlaying) {
      const oscillator = oscillators.get(trackId);
      if (oscillator) {
        oscillator.type = type as OscillatorType;
      }
    }
  }

  // 時間フォーマット
  function formatTime(seconds: number): string {
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  }

  onMount(() => {
    return () => {
      if (animationFrameId) {
        cancelAnimationFrame(animationFrameId);
      }
      stopAllOscillators();
      if (audioContext) {
        audioContext.close();
      }
    };
  });
</script>

<div class="timeline-container">
  <!-- ヘッダー -->
  <div class="timeline-header">
    <div class="controls">
      <button class="play-button" on:click={togglePlay}>
        {isPlaying ? '⏸' : '▶️'}
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
        <span>トラック</span>
        <button class="add-track-btn" on:click={addTrack}>+</button>
      </div>
      {#each tracks as track (track.id)}
        <div class="track-item">
          <div class="track-info">
            <span class="track-name">{track.name}</span>
            <button class="remove-track-btn" on:click={() => removeTrack(track.id)}>×</button>
          </div>
          <div class="track-controls">
                         <input 
               type="number" 
               value={track.frequency} 
               on:input={(e) => {
                 const target = e.target as HTMLInputElement;
                 updateTrackFrequency(track.id, parseInt(target.value));
               }}
               placeholder="周波数"
               min="20"
               max="20000"
             />
                         <select 
               value={track.type} 
               on:change={(e) => {
                 const target = e.target as HTMLSelectElement;
                 updateTrackType(track.id, target.value);
               }}
             >
              <option value="sine">正弦波</option>
              <option value="triangle">三角波</option>
              <option value="square">矩形波</option>
              <option value="sawtooth">ノコギリ波</option>
            </select>
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
      <div class="tracks-display" on:click={seekTo}>
        {#each tracks as track (track.id)}
          <div class="track-display" style="height: {trackHeight}px;">
            <!-- ここに周波数データの可視化を追加予定 -->
            <div class="track-visualization">
              <span class="frequency-label">{track.frequency}Hz</span>
            </div>
          </div>
        {/each}
      </div>
    </div>
  </div>
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
    margin-bottom: 0.5rem;
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

  .track-controls {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .track-controls input,
  .track-controls select {
    background: #3a3a3a;
    border: 1px solid #4a4a4a;
    color: white;
    padding: 0.25rem;
    border-radius: 4px;
    font-size: 0.9rem;
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
  }

  .frequency-label {
    font-family: monospace;
    font-size: 0.9rem;
    color: #888;
  }
</style> 