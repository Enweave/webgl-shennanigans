<script setup lang="ts">
// @ts-nocheck
import { onMounted, onUnmounted, ref, watch } from 'vue'

// Canvas background color (WebGL clear color) as a parameter
const props = withDefaults(defineProps<{ backgroundColor?: string; numberOfCharacters?: number }>(), {
  backgroundColor: '#000000', // matches previous hardcoded clearColor (0.05, 0.05, 0.08)
  numberOfCharacters: 1000,
})

function parseHexColorToRGBAFloats(str: string, fallback: [number, number, number, number]): [number, number, number, number] {
  if (typeof str !== 'string') return fallback
  const s = str.trim()
  if (!s.startsWith('#')) return fallback
  const h = s.slice(1)
  let r = 0, g = 0, b = 0, a = 255
  if (h.length === 3 || h.length === 4) {
    // #RGB or #RGBA
    const rr = h[0] + h[0]
    const gg = h[1] + h[1]
    const bb = h[2] + h[2]
    const aa = h.length === 4 ? h[3] + h[3] : 'ff'
    const rv = parseInt(rr, 16)
    const gv = parseInt(gg, 16)
    const bv = parseInt(bb, 16)
    const av = parseInt(aa, 16)
    if (Number.isFinite(rv) && Number.isFinite(gv) && Number.isFinite(bv) && Number.isFinite(av)) {
      r = rv; g = gv; b = bv; a = av
    } else return fallback
  } else if (h.length === 6 || h.length === 8) {
    // #RRGGBB or #RRGGBBAA
    const rr = h.slice(0, 2)
    const gg = h.slice(2, 4)
    const bb = h.slice(4, 6)
    const aa = h.length === 8 ? h.slice(6, 8) : 'ff'
    const rv = parseInt(rr, 16)
    const gv = parseInt(gg, 16)
    const bv = parseInt(bb, 16)
    const av = parseInt(aa, 16)
    if (Number.isFinite(rv) && Number.isFinite(gv) && Number.isFinite(bv) && Number.isFinite(av)) {
      r = rv; g = gv; b = bv; a = av
    } else return fallback
  } else {
    return fallback
  }
  return [r / 255, g / 255, b / 255, a / 255]
}

const defaultBG: [number, number, number, number] = [0.05, 0.05, 0.08, 1]
const bgColor = parseHexColorToRGBAFloats(props.backgroundColor, defaultBG)

// Frame rect in atlas (pixels)
interface Frame { x: number; y: number; width: number; height: number }

// Resolve asset URLs via Vite
const imageUrl = new URL('../assets/spritesheet.png', import.meta.url).toString()
const jsonUrl = new URL('../assets/spritesheet.json', import.meta.url).toString()

// Animation/config
const FrameTimeMs = 150
let currentN = Math.max(1, Math.floor(props.numberOfCharacters ?? 1000))
const speed = 0.01            // pixels per ms
const maxDistance = 100       // max distance from start before reversing
const canvasWidth = 900
const canvasHeight = 900

// Vue refs
const canvasRef = ref<HTMLCanvasElement | null>(null)
const fpsText = ref('')

// Sprite sheet frames (choose a few animation frames)
let spriteSheetOne: Frame[] = []
let texW = 1, texH = 1

// WebGL resources
let gl: WebGL2RenderingContext | null = null
let program: WebGLProgram
let vao: WebGLVertexArrayObject
let quadVbo: WebGLBuffer
let instVbo: WebGLBuffer
let uProjection: WebGLUniformLocation
let uAtlas: WebGLUniformLocation
let uTime: WebGLUniformLocation
let uFrameTime: WebGLUniformLocation
let uFrameCount: WebGLUniformLocation
let uUVRects0: WebGLUniformLocation
let texture: WebGLTexture

// Instance data
let startPos: Float32Array
let pos: Float32Array
let dir: Float32Array           // normalized direction (x,y)
let dist: Float32Array          // signed distance traveled along dir from start
let scale: Float32Array         // per-instance scale (w,h)
let frameOffset: Float32Array    // per-instance starting frame offset
let instanceData: Float32Array  // interleaved [tx,ty, sx,sy, frameOffset]

let rafId = 0
let lastTime = 0
let isReady = false
// FPS tracking
let fpsLast = 0
let fpsFrames = 0

function makeOrtho(l:number, r:number, b:number, t:number, n:number, f:number) {
  const rl = r - l, tb = t - b, fn = f - n
  return new Float32Array([
    2/rl, 0,     0,       0,
    0,    2/tb,  0,       0,
    0,    0,    -2/fn,    0,
    -(r+l)/rl, -(t+b)/tb, -(f+n)/fn, 1,
  ])
}

function compile(gl: WebGL2RenderingContext, type: number, src: string) {
  const sh = gl.createShader(type)!
  gl.shaderSource(sh, src)
  gl.compileShader(sh)
  if (!gl.getShaderParameter(sh, gl.COMPILE_STATUS)) {
    const log = gl.getShaderInfoLog(sh)
    console.error(log, src)
    throw new Error('Shader compile failed')
  }
  return sh
}

function link(gl: WebGL2RenderingContext, vsSrc: string, fsSrc: string) {
  const vs = compile(gl, gl.VERTEX_SHADER, vsSrc)
  const fs = compile(gl, gl.FRAGMENT_SHADER, fsSrc)
  const prog = gl.createProgram()!
  gl.attachShader(prog, vs)
  gl.attachShader(prog, fs)
  gl.linkProgram(prog)
  if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
    const log = gl.getProgramInfoLog(prog)
    throw new Error('Program link failed: ' + log)
  }
  gl.deleteShader(vs)
  gl.deleteShader(fs)
  return prog
}

async function loadJSON(url: string) { const res = await fetch(url); return await res.json() }
function loadImage(url: string) { return new Promise<HTMLImageElement>((resolve, reject) => { const img = new Image(); img.onload = () => resolve(img); img.onerror = reject; img.src = url }) }

async function setupSpriteSheet() {
  const data = await loadJSON(jsonUrl)
  // Pick four walk frames
  const keys = Object.keys(data).filter(e=>e.includes("./assets/yutami/enemies/goblinpickaxerunning_56x32px_12fps/"));


  spriteSheetOne = []

  keys.forEach((k) => {
    const frame = data[k]
    spriteSheetOne.push({ x: frame.x, y: frame.y, width: frame.width, height: frame.height })
  })

  console.log(spriteSheetOne)
}

function createTexture(gl: WebGL2RenderingContext, img: HTMLImageElement) {
  const tex = gl.createTexture()!
  gl.bindTexture(gl.TEXTURE_2D, tex)
  gl.pixelStorei(gl.UNPACK_PREMULTIPLY_ALPHA_WEBGL, 0)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, img)
  gl.bindTexture(gl.TEXTURE_2D, null)
  return tex
}

function initGL(canvas: HTMLCanvasElement) {
  gl = canvas.getContext('webgl2', { antialias: false, alpha: true }) as WebGL2RenderingContext
  if (!gl) throw new Error('WebGL2 not supported')

  // Fixed canvas size per requirements
  canvas.width = canvasWidth
  canvas.height = canvasHeight
  gl.viewport(0, 0, canvasWidth, canvasHeight)

  const vs = `#version 300 es\nprecision highp float;\nlayout(location=0) in vec2 aPos;\nlayout(location=1) in vec2 aUV;\nlayout(location=2) in vec2 iTranslate;\nlayout(location=3) in vec2 iScale;\nlayout(location=4) in float iFrameOffset;\nuniform mat4 uProjection;\nout vec2 vUV;\nflat out float vFrameOffset;\nvoid main(){\n  vec2 world = aPos * iScale + iTranslate;\n  gl_Position = uProjection * vec4(world, 0.0, 1.0);\n  vUV = aUV;\n  vFrameOffset = iFrameOffset;\n}`

  const fs = `#version 300 es\nprecision highp float;\nin vec2 vUV;\nflat in float vFrameOffset;\nuniform sampler2D uAtlas;\nuniform vec4 uUVRects[8]; // up to 8 frames supported\nuniform float uTime;\nuniform float uFrameTime;\nuniform int uFrameCount;\nout vec4 outColor;\nvoid main(){\n  float t = uTime / uFrameTime;\n  float idxF = mod(floor(t + vFrameOffset), float(uFrameCount));\n  int idx = int(idxF);\n  vec4 rect = uUVRects[idx];\n  vec2 uv = rect.xy + vUV * rect.zw;\n  outColor = texture(uAtlas, uv);\n}`

  program = link(gl, vs, fs)
  gl.useProgram(program)
  uProjection = gl.getUniformLocation(program, 'uProjection')!
  uAtlas = gl.getUniformLocation(program, 'uAtlas')!
  uTime = gl.getUniformLocation(program, 'uTime')!
  uFrameTime = gl.getUniformLocation(program, 'uFrameTime')!
  uFrameCount = gl.getUniformLocation(program, 'uFrameCount')!
  uUVRects0 = gl.getUniformLocation(program, 'uUVRects[0]')!

  // Quad geometry (unit quad centered at origin), with base UVs 0..1
  const quad = new Float32Array([
    // x,y,   u,v
    -0.5, -0.5, 0, 1,
     0.5, -0.5, 1, 1,
    -0.5,  0.5, 0, 0,
     0.5,  0.5, 1, 0,
  ])

  vao = gl.createVertexArray()!
  gl.bindVertexArray(vao)

  quadVbo = gl.createBuffer()!
  gl.bindBuffer(gl.ARRAY_BUFFER, quadVbo)
  gl.bufferData(gl.ARRAY_BUFFER, quad, gl.STATIC_DRAW)
  // aPos (loc 0)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 16, 0)
  // aUV (loc 1)
  gl.enableVertexAttribArray(1)
  gl.vertexAttribPointer(1, 2, gl.FLOAT, false, 16, 8)

  // Instance buffer [tx,ty,sx,sy,frameOffset]
  instVbo = gl.createBuffer()!
  gl.bindBuffer(gl.ARRAY_BUFFER, instVbo)
  gl.bufferData(gl.ARRAY_BUFFER, currentN * 5 * 4, gl.DYNAMIC_DRAW)
  const stride = 20 // bytes (5 floats)
  // iTranslate (loc 2)
  gl.enableVertexAttribArray(2)
  gl.vertexAttribPointer(2, 2, gl.FLOAT, false, stride, 0)
  gl.vertexAttribDivisor(2, 1)
  // iScale (loc 3)
  gl.enableVertexAttribArray(3)
  gl.vertexAttribPointer(3, 2, gl.FLOAT, false, stride, 8)
  gl.vertexAttribDivisor(3, 1)
  // iFrameOffset (loc 4)
  gl.enableVertexAttribArray(4)
  gl.vertexAttribPointer(4, 1, gl.FLOAT, false, stride, 16)
  gl.vertexAttribDivisor(4, 1)

  gl.bindVertexArray(null)
}

function initSprites() {
  const N = currentN
  startPos = new Float32Array(N * 2)
  pos = new Float32Array(N * 2)
  dir = new Float32Array(N * 2)
  dist = new Float32Array(N)
  scale = new Float32Array(N * 2)
  frameOffset = new Float32Array(N)
  instanceData = new Float32Array(N * 5)

  const minSize = 14
  const maxSize = 22
  const frameCount = Math.max(1, spriteSheetOne.length)

  for (let i = 0; i < N; i++) {
    const x = Math.random() * canvasWidth
    const y = Math.random() * canvasHeight
    startPos[i*2+0] = x
    startPos[i*2+1] = y
    pos[i*2+0] = x
    pos[i*2+1] = y

    // random normalized direction
    const ang = Math.random() * Math.PI * 2
    dir[i*2+0] = Math.cos(ang)
    dir[i*2+1] = Math.sin(ang)

    dist[i] = 0

    const s = minSize + Math.random() * (maxSize - minSize)
    scale[i*2+0] = s
    scale[i*2+1] = s

    // random starting frame offset in [0, frameCount)
    frameOffset[i] = Math.random() * frameCount
  }

  updateInstanceBuffer()
}

function updateInstanceBuffer() {
  const N = currentN
  for (let i = 0; i < N; i++) {
    const base = i*5
    instanceData[base+0] = pos[i*2+0]
    instanceData[base+1] = pos[i*2+1]
    instanceData[base+2] = scale[i*2+0]
    instanceData[base+3] = scale[i*2+1]
    instanceData[base+4] = frameOffset[i]
  }
  gl!.bindBuffer(gl!.ARRAY_BUFFER, instVbo)
  gl!.bufferSubData(gl!.ARRAY_BUFFER, 0, instanceData.subarray(0, N*5))
}

function render(now: number) {
  const g = gl!
  if (!lastTime) lastTime = now
  const dt = Math.min(50, now - lastTime)
  lastTime = now

  // FPS update (smoothed over ~250ms)
  if (!fpsLast) fpsLast = now
  fpsFrames++
  const elapsed = now - fpsLast
  if (elapsed >= 250) {
    const fps = (fpsFrames * 1000) / elapsed
    fpsText.value = fps.toFixed(1) + ' FPS'
    fpsFrames = 0
    fpsLast = now
  }

  // Update positions: bounded random walk around startPos within a circle of radius maxDistance
  const N = currentN
  const step = speed * dt
  const maxR = maxDistance
  const maxR2 = maxR * maxR
  const turnPerMs = 0.004 // radians per ms (max random turn rate)
  const jitter = turnPerMs * dt
  for (let i = 0; i < N; i++) {
    // rotate current direction by a small random angle
    const a = (Math.random() * 2 - 1) * jitter
    const c = Math.cos(a), s = Math.sin(a)
    const dx0 = dir[i*2+0], dy0 = dir[i*2+1]
    let dx = dx0 * c - dy0 * s
    let dy = dx0 * s + dy0 * c
    // normalize to avoid drift
    const len = Math.hypot(dx, dy) || 1
    dx /= len; dy /= len

    // tentative move
    let nx = pos[i*2+0] + dx * step
    let ny = pos[i*2+1] + dy * step

    // vector from origin
    const ox = nx - startPos[i*2+0]
    const oy = ny - startPos[i*2+1]
    const r2 = ox*ox + oy*oy
    if (r2 > maxR2) {
      // clamp to boundary and reflect direction, with a slight inward bias to avoid sliding
      const r = Math.sqrt(r2) || 1
      const nX = ox / r
      const nY = oy / r
      nx = startPos[i*2+0] + nX * maxR
      ny = startPos[i*2+1] + nY * maxR
      const dot = dx * nX + dy * nY
      dx = dx - 2.0 * dot * nX
      dy = dy - 2.0 * dot * nY
      // inward bias
      dx += -nX * 0.2
      dy += -nY * 0.2
      const len2 = Math.hypot(dx, dy) || 1
      dx /= len2; dy /= len2
    }

    // write back
    dir[i*2+0] = dx
    dir[i*2+1] = dy
    pos[i*2+0] = nx
    pos[i*2+1] = ny
    dist[i] = Math.min(maxR, Math.hypot(nx - startPos[i*2+0], ny - startPos[i*2+1]))
  }
  updateInstanceBuffer()

  // Clear and draw
  g.enable(g.BLEND)
  g.blendFuncSeparate(g.SRC_ALPHA, g.ONE_MINUS_SRC_ALPHA, g.ONE, g.ONE_MINUS_SRC_ALPHA)
  g.clearColor(bgColor[0], bgColor[1], bgColor[2], bgColor[3])
  g.clear(g.COLOR_BUFFER_BIT)

  g.useProgram(program)
  g.bindVertexArray(vao)
  g.activeTexture(g.TEXTURE0)
  g.bindTexture(g.TEXTURE_2D, texture)
  g.uniform1i(uAtlas, 0)
  g.uniformMatrix4fv(uProjection, false, makeOrtho(0, canvasWidth, canvasHeight, 0, -1, 1))
  g.uniform1f(uTime, now)
  g.drawArraysInstanced(g.TRIANGLE_STRIP, 0, 4, currentN)
  g.bindVertexArray(null)

  rafId = requestAnimationFrame(render)
}

onMounted(async () => {
  const canvas = canvasRef.value as HTMLCanvasElement
  initGL(canvas)
  await setupSpriteSheet()
  const img = await loadImage(imageUrl)
  texW = img.naturalWidth || img.width
  texH = img.naturalHeight || img.height
  texture = createTexture(gl!, img)

  // Set initial uniforms
  gl!.useProgram(program)
  gl!.uniformMatrix4fv(uProjection, false, makeOrtho(0, canvasWidth, canvasHeight, 0, -1, 1))
  // Upload UV rects for frames and timing
  const frameCount = Math.min(8, spriteSheetOne.length)
  const uvArray = new Float32Array(frameCount * 4)
  for (let i = 0; i < frameCount; i++) {
    const fr = spriteSheetOne[i]
    uvArray[i*4+0] = fr.x / texW
    uvArray[i*4+1] = fr.y / texH
    uvArray[i*4+2] = fr.width / texW
    uvArray[i*4+3] = fr.height / texH
  }
  gl!.uniform1i(uAtlas, 0)
  gl!.uniform1f(uFrameTime, FrameTimeMs)
  gl!.uniform1i(uFrameCount, frameCount)
  gl!.uniform4fv(uUVRects0, uvArray)

  initSprites()
  lastTime = performance.now()
  isReady = true
  rafId = requestAnimationFrame(render)
})

onUnmounted(() => {
  if (rafId) cancelAnimationFrame(rafId)
  if (!gl) return
  gl.deleteBuffer(quadVbo)
  gl.deleteBuffer(instVbo)
  gl.deleteTexture(texture)
  gl.deleteVertexArray(vao)
  gl.deleteProgram(program)
  gl = null
})
// Apply change in number of instances at runtime
function applyCharacterCount(nRaw: number | undefined) {
  if (!gl) return
  const n = Math.max(1, Math.floor(nRaw ?? 1))
  if (n === currentN) return
  currentN = n
  // Resize instance buffer storage to new capacity (5 floats per instance)
  gl.bindBuffer(gl.ARRAY_BUFFER, instVbo)
  gl.bufferData(gl.ARRAY_BUFFER, currentN * 5 * 4, gl.DYNAMIC_DRAW)
  // Reinitialize per-instance arrays and data
  initSprites()
}

watch(() => props.numberOfCharacters, (val) => {
  if (!isReady) return
  applyCharacterCount(val)
})

</script>

<template>
  <div class="wrapper">
    <div class="hud"><span>{{ fpsText }}</span></div>
    <canvas
      ref="canvasRef"
      :width="canvasWidth"
      :height="canvasHeight"
      :style="{ width: canvasWidth + 'px', height: canvasHeight + 'px' }"
    ></canvas>
  </div>
</template>

<style scoped lang="sass">
.wrapper
  width: 100vw
  height: 100vh
  display: flex
  justify-content: center
  align-items: center
  background: #131313
  position: relative

.hud
  width: 100%
  position: fixed
  top: 12px
  left: 12px
  z-index: 10
  color: #9effa1
  background: rgb(19, 19, 19)
  font: 12px/1.2 ui-monospace, SFMono-Regular, Menlo, Consolas, monospace
  padding: 6px 8px
  border-radius: 4px
  letter-spacing: 0.5px
  pointer-events: none
  user-select: none

canvas
  box-shadow: 0 10px 30px rgba(0,0,0,0.3)
  border-radius: 8px
</style>