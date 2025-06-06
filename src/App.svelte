<script lang="ts">
  import { onMount } from 'svelte';

  let videoEl: HTMLVideoElement;
  let canvasEl: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;
  let frameCount = 0;

  let amplitude = 12;
  let speed = 30;

  const rightEye = [33, 133];
  const leftEye = [362, 263];
  const nose = [1, 2, 98, 327];
  const mouth = [13, 14, 87, 317];

  onMount(() => {
    const faceMesh = new window.FaceMesh({
      locateFile: (file: string) =>
        `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`,
    });

    faceMesh.setOptions({
      maxNumFaces: 1,
      refineLandmarks: true,
      minDetectionConfidence: 0.5,
      minTrackingConfidence: 0.5,
    });

    faceMesh.onResults(onResults);

    const camera = new window.Camera(videoEl, {
      onFrame: async () => {
        await faceMesh.send({ image: videoEl });
        frameCount++;
      },
      width: 640,
      height: 480,
    });

    camera.start();
  });

  function onResults(results: any) {
    if (!canvasEl || !videoEl) return;
    if (!ctx) {
      ctx = canvasEl.getContext('2d', { willReadFrequently: true })!;
    }

    const width = videoEl.videoWidth;
    const height = videoEl.videoHeight;
    if (!width || !height) return;

    canvasEl.width = width;
    canvasEl.height = height;

    ctx.clearRect(0, 0, width, height);
    ctx.drawImage(results.image, 0, 0, width, height);

    if (results.multiFaceLandmarks.length > 0) {
      const landmarks = results.multiFaceLandmarks[0];

      const safeSpeed = Math.max(speed, 5);

      const dxEye = Math.sin(frameCount / safeSpeed) * amplitude;
      const dyEye = Math.cos(frameCount / (safeSpeed + 5)) * amplitude;
      const dxMouth = Math.sin(frameCount / (safeSpeed - 5)) * amplitude;
      const dyMouth = Math.cos(frameCount / (safeSpeed + 10)) * amplitude;
      const dxNose = Math.sin(frameCount / (safeSpeed + 15)) * amplitude * 0.5;
      const dyNose = Math.cos(frameCount / (safeSpeed + 20)) * amplitude * 0.5;

      drawMaskedPart(rightEye, dxEye, dyEye, landmarks);
drawMaskedPart(leftEye, -dxEye, dyEye, landmarks);
drawMaskedPart(nose, dxNose, dyNose, landmarks);
drawMaskedPart(mouth, dxMouth, dyMouth, landmarks);
    }
  }

  function drawMaskedPart(indices: number[], dx: number, dy: number, landmarks: any[]) {
  const region = getBoundingBox(landmarks, indices);
  const x = Math.round(region.x + dx);
  const y = Math.round(region.y + dy);

  if (region.w > 1 && region.h > 1 && Number.isFinite(x) && Number.isFinite(y)) {
    const tempCanvas = document.createElement('canvas');
    tempCanvas.width = region.w;
    tempCanvas.height = region.h;
    const tempCtx = tempCanvas.getContext('2d')!;

    // 映像からパーツを drawImage で切り取り
    tempCtx.drawImage(
      canvasEl,
      region.x,
      region.y,
      region.w,
      region.h,
      0,
      0,
      region.w,
      region.h
    );

    // 黒→透明のマスクを加える
    const gradient = tempCtx.createRadialGradient(
      region.w / 2,
      region.h / 2,
      region.w * 0.1,
      region.w / 2,
      region.h / 2,
      Math.max(region.w, region.h) / 2
    );
    gradient.addColorStop(0, 'rgba(0,0,0,0)');
    gradient.addColorStop(1, 'rgba(0,0,0,1)');

    tempCtx.globalCompositeOperation = 'destination-in';
    tempCtx.fillStyle = gradient;
    tempCtx.fillRect(0, 0, region.w, region.h);
    tempCtx.globalCompositeOperation = 'source-over';

    // 最終的に main canvas に描画（ズラした位置）
    ctx.drawImage(tempCanvas, x, y);
  }
}

  function getBoundingBox(landmarks: any[], indices: number[]) {
    const xs = indices.map(i => landmarks[i].x);
    const ys = indices.map(i => landmarks[i].y);

    const xMin = Math.min(...xs);
    const xMax = Math.max(...xs);
    const yMin = Math.min(...ys);
    const yMax = Math.max(...ys);

    const width = canvasEl.width;
    const height = canvasEl.height;

    const padding = 0.05;

    const boxX = Math.max(xMin - padding, 0);
    const boxY = Math.max(yMin - padding, 0);
    const boxW = Math.min(xMax - xMin + padding * 2, 1 - boxX);
    const boxH = Math.min(yMax - yMin + padding * 2, 1 - boxY);

    return {
      x: Math.floor(boxX * width),
      y: Math.floor(boxY * height),
      w: Math.max(~~(boxW * width), 4),
      h: Math.max(~~(boxH * height), 4),
    };
  }
</script>

<main>
  <video bind:this={videoEl} style="display: none;" />
  <canvas bind:this={canvasEl} />

  <div class="controls">
    <label>
      ズレ幅: {amplitude}px
      <input type="range" min="0" max="8" bind:value={amplitude} />
    </label>
    <label>
      ズレ速度: {speed}
      <input type="range" min="7" max="18" bind:value={speed} />
    </label>
  </div>
</main>

<style>
  canvas {
    width: 100vw;
    height: 100vh;
    display: block;
    background: black;
  }

  .controls {
    position: absolute;
    top: 1rem;
    left: 1rem;
    background: rgba(0, 0, 0, 0.6);
    padding: 1rem;
    border-radius: 0.5rem;
    color: white;
    font-size: 0.9rem;
  }

  .controls label {
    display: block;
    margin-bottom: 1rem;
  }

  input[type="range"] {
    width: 200px;
  }
</style>