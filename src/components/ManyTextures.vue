<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import tex1 from './../assets/1.webp'
import tex2 from './../assets/2.webp'
import tex3 from './../assets/3.webp'



// Configurable number of sprites via URL ?n=10000 (default 2000)
function getCountFromQuery(defaultN = 2000) {
  const params = new URLSearchParams(window.location.search)
  const nStr = params.get('n')
  const n = nStr ? parseInt(nStr, 10) : defaultN
  return Number.isFinite(n) && n > 0 ? Math.min(n, 9000000) : defaultN
}

const canvasRef = ref<HTMLCanvasElement | null>(null)
const fpsText = ref('')
const statsText = ref('')
const N = getCountFromQuery()

let rafId = 0

onMounted(() => {
  const canvas = canvasRef.value!
  const ogGl = canvas.getContext('webgl2', { antialias: false, alpha: false }) as WebGL2RenderingContext | null
  if (!ogGl) {
    fpsText.value = 'WebGL2 not supported'
    return
  }

  const {gl, stats} = createLoggingContext(ogGl)



  // Resize canvas to display size * devicePixelRatio
  function resizeCanvasToDisplaySize() {
    const dpr = Math.min(window.devicePixelRatio || 1, 2)
    const width = Math.floor(canvas.clientWidth * dpr)
    const height = Math.floor(canvas.clientHeight * dpr)
    if (canvas.width !== width || canvas.height !== height) {
      canvas.width = width
      canvas.height = height
      gl.viewport(0, 0, width, height)
      // Update projection
      const proj = makeOrtho(0, width, height, 0, -1, 1)
      gl.useProgram(program)
      gl.uniformMatrix4fv(uProjection, false, proj)
    }
  }

  // Shaders for instanced quads (sprite batching)
  const vsSource = `#version 300 es
  precision highp float;
  layout(location=0) in vec2 aPosition;            // base quad pos (-0.5..0.5)
  layout(location=1) in vec2 aUV;                  // base quad uv
  layout(location=2) in vec2 iTranslate;           // per-instance position in pixels
  layout(location=3) in vec2 iScale;               // per-instance scale in pixels
  layout(location=4) in float iRotation;           // per-instance rotation in radians
  layout(location=5) in float iTexId;              // per-instance texture id (0,1,2)

  uniform mat4 uProjection;

  out vec2 vUV;
  flat out float vTexId;

  void main() {
    float c = cos(iRotation);
    float s = sin(iRotation);
    // rotate scaled vertex around origin, then translate
    vec2 local = aPosition * iScale;
    vec2 rotated = vec2(local.x * c - local.y * s, local.x * s + local.y * c);
    vec2 world = rotated + iTranslate;
    gl_Position = uProjection * vec4(world, 0.0, 1.0);
    vUV = aUV;
    vTexId = iTexId;
  }`

  const fsSource = `#version 300 es
  precision highp float;
  in vec2 vUV;
  flat in float vTexId;
  uniform sampler2D uTex0;
  uniform sampler2D uTex1;
  uniform sampler2D uTex2;
  out vec4 outColor;
  void main() {
    int id = int(vTexId + 0.5);
    vec4 c;
    if (id == 0) {
      c = texture(uTex0, vUV);
    } else if (id == 1) {
      c = texture(uTex1, vUV);
    } else {
      c = texture(uTex2, vUV);
    }
    outColor = c;
  }`

  function compileShader(type: number, source: string) {
    const sh = gl.createShader(type)!
    gl.shaderSource(sh, source)
    gl.compileShader(sh)
    if (!gl.getShaderParameter(sh, gl.COMPILE_STATUS)) {
      const info = gl.getShaderInfoLog(sh)
      gl.deleteShader(sh)
      throw new Error('Shader compile error: ' + info)
    }
    return sh
  }

  function createProgram(vs: WebGLShader, fs: WebGLShader) {
    const p = gl.createProgram()!
    gl.attachShader(p, vs)
    gl.attachShader(p, fs)
    gl.linkProgram(p)
    if (!gl.getProgramParameter(p, gl.LINK_STATUS)) {
      const info = gl.getProgramInfoLog(p)
      gl.deleteProgram(p)
      throw new Error('Program link error: ' + info)
    }
    return p
  }

  // Build program
  const vs = compileShader(gl.VERTEX_SHADER, vsSource)
  const fs = compileShader(gl.FRAGMENT_SHADER, fsSource)
  const program = createProgram(vs, fs)
  gl.useProgram(program)

  // Look up uniforms
  const uProjection = gl.getUniformLocation(program, 'uProjection')!
  const uTex0 = gl.getUniformLocation(program, 'uTex0')!
  const uTex1u = gl.getUniformLocation(program, 'uTex1')!
  const uTex2u = gl.getUniformLocation(program, 'uTex2')!
  // Bind sampler uniforms to texture units 0,1,2
  gl.uniform1i(uTex0, 0)
  gl.uniform1i(uTex1u, 1)
  gl.uniform1i(uTex2u, 2)

  // Base quad geometry (unit square centered at origin, size 1) -> but we will scale in pixels
  // We'll define quad from -0.5..0.5 in both axes
  const quad = new Float32Array([
    // x, y,   u, v
    -0.5, -0.5, 0, 1,
    0.5, -0.5, 1, 1,
    0.5,  0.5, 1, 0,
    -0.5,  0.5, 0, 0,
  ])
  const quadIndices = new Uint16Array([0,1,2, 0,2,3])

  // Create VAO
  const vao = gl.createVertexArray()!
  gl.bindVertexArray(vao)

  // Quad VBO
  const quadVBO = gl.createBuffer()!
  gl.bindBuffer(gl.ARRAY_BUFFER, quadVBO)
  gl.bufferData(gl.ARRAY_BUFFER, quad, gl.STATIC_DRAW)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 16, 0)       // aPosition
  gl.enableVertexAttribArray(1)
  gl.vertexAttribPointer(1, 2, gl.FLOAT, false, 16, 8)       // aUV

  // Index buffer
  const ebo = gl.createBuffer()!
  gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, ebo)
  gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, quadIndices, gl.STATIC_DRAW)

  // Instance attributes: translate(x,y), scale(x,y), rotation, texId
  const instanceStride = 6 * 4 // 6 floats
  const instanceData = new Float32Array(N * 6)
  const instanceVBO = gl.createBuffer()!
  gl.bindBuffer(gl.ARRAY_BUFFER, instanceVBO)
  gl.bufferData(gl.ARRAY_BUFFER, instanceData.byteLength, gl.DYNAMIC_DRAW)

  // attribute locations 2,3,4,5
  // iTranslate (vec2)
  gl.enableVertexAttribArray(2)
  gl.vertexAttribPointer(2, 2, gl.FLOAT, false, instanceStride, 0)
  gl.vertexAttribDivisor(2, 1)
  // iScale (vec2)
  gl.enableVertexAttribArray(3)
  gl.vertexAttribPointer(3, 2, gl.FLOAT, false, instanceStride, 8)
  gl.vertexAttribDivisor(3, 1)
  // iRotation (float)
  gl.enableVertexAttribArray(4)
  gl.vertexAttribPointer(4, 1, gl.FLOAT, false, instanceStride, 16)
  gl.vertexAttribDivisor(4, 1)
  // iTexId (float)
  gl.enableVertexAttribArray(5)
  gl.vertexAttribPointer(5, 1, gl.FLOAT, false, instanceStride, 20)
  gl.vertexAttribDivisor(5, 1)

  gl.bindVertexArray(null)

  // Load three textures: tex1, tex2, tex3
  const texture0 = gl.createTexture()!
  const texture1 = gl.createTexture()!
  const texture2 = gl.createTexture()!
  let loadedCount = 0
  function onLoaded() {
    loadedCount++
    if (loadedCount === 3) texReady = true
  }
  function loadInto(tex: WebGLTexture, url: string) {
    const img = new Image()
    img.src = url
    img.onload = () => {
      gl.bindTexture(gl.TEXTURE_2D, tex)
      gl.pixelStorei(gl.UNPACK_PREMULTIPLY_ALPHA_WEBGL, 1)
      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR)
      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
      gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, img)
      gl.generateMipmap(gl.TEXTURE_2D)
      gl.bindTexture(gl.TEXTURE_2D, null)
      onLoaded()
    }
  }
  loadInto(texture0, tex1)
  loadInto(texture1, tex2)
  loadInto(texture2, tex3)

  // Sprite state arrays for animation
  const pos = new Float32Array(N * 2)
  const scale = new Float32Array(N * 2)
  const rot = new Float32Array(N)
  const rotSpeed = new Float32Array(N)

  function rand(min: number, max: number) { return Math.random() * (max - min) + min }

  // Initialize after we know canvas size
  let texReady = false
  function initSprites() {
    const w = canvas.width
    const h = canvas.height
    const minSize = 16
    const maxSize = Math.max(24, Math.min(w, h) * 0.12)
    for (let i = 0; i < N; i++) {
      pos[i*2+0] = rand(0, w)
      pos[i*2+1] = rand(0, h)
      const s = rand(minSize, maxSize)
      const ar = 1.0 // assume square logo; using same scale for x/y
      scale[i*2+0] = s
      scale[i*2+1] = s / ar
      rot[i] = rand(0, Math.PI * 2)
      rotSpeed[i] = rand(-2.0, 2.0)
      // Pack initial tex id as random 0..2
      const base = i * 6
      instanceData[base+5] = Math.floor(Math.random() * 3)
    }
    // upload initial instance buffer
    updateInstanceBuffer()
  }

  function updateInstanceBuffer() {
    // pack: translate(x,y), scale(x,y), rotation, texId
    for (let i = 0; i < N; i++) {
      const base = i * 6
      instanceData[base+0] = pos[i*2+0]
      instanceData[base+1] = pos[i*2+1]
      instanceData[base+2] = scale[i*2+0]
      instanceData[base+3] = scale[i*2+1]
      instanceData[base+4] = rot[i]
      // base+5 already set to texId in init; leave unchanged here
    }
    gl.bindBuffer(gl.ARRAY_BUFFER, instanceVBO)
    gl.bufferSubData(gl.ARRAY_BUFFER, 0, instanceData)
  }

  // Simple ortho projection
  function makeOrtho(l:number, r:number, b:number, t:number, n:number, f:number) {
    const out = new Float32Array(16)
    const lr = 1 / (l - r)
    const bt = 1 / (b - t)
    const nf = 1 / (n - f)
    out[0] = -2 * lr
    out[5] = -2 * bt
    out[10] = 2 * nf
    out[12] = (l + r) * lr * 2
    out[13] = (t + b) * bt * 2
    out[14] = (f + n) * nf
    out[15] = 1
    return out
  }

  // GL state
  gl.clearColor(0.1, 0.1, 0.12, 1)
  gl.enable(gl.BLEND)
  gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)

  // Initial sizes
  resizeCanvasToDisplaySize()
  initSprites()

  // Stats
  let lastTime = performance.now()
  let frames = 0
  let acc = 0
  let lastFps = 0

  function render(time: number) {
    stats.reset()
    rafId = requestAnimationFrame(render)
    const dt = Math.min((time - lastTime) / 1000, 0.1)
    lastTime = time

    // animate rotations
    for (let i = 0; i < N; i++) {
      rot[i] += rotSpeed[i] * dt
    }

    // Update size if needed
    resizeCanvasToDisplaySize()

    // Upload instance data (only rotations changed, but repack for simplicity)
    for (let i = 0; i < N; i++) instanceData[i*6 + 4] = rot[i]
    gl.bindBuffer(gl.ARRAY_BUFFER, instanceVBO)
    gl.bufferSubData(gl.ARRAY_BUFFER, 0, instanceData)

    // Draw
    gl.clear(gl.COLOR_BUFFER_BIT)
    gl.useProgram(program)
    // Bind three textures to units 0,1,2
    gl.activeTexture(gl.TEXTURE0)
    gl.bindTexture(gl.TEXTURE_2D, texture0)
    gl.activeTexture(gl.TEXTURE1)
    gl.bindTexture(gl.TEXTURE_2D, texture1)
    gl.activeTexture(gl.TEXTURE2)
    gl.bindTexture(gl.TEXTURE_2D, texture2)

    gl.bindVertexArray(vao)

    // single draw call for all instances
    gl.drawElementsInstanced(gl.TRIANGLES, 6, gl.UNSIGNED_SHORT, 0, N)

    gl.bindVertexArray(null)

    // Stats update
    frames++
    acc += dt
    if (acc >= 0.5) {
      lastFps = Math.round(frames / acc)
      frames = 0
      acc = 0
      fpsText.value = `${lastFps} FPS`
      statsText.value = `${N} sprites | ${stats.drawCalls} draw call`
      // console.table(stats)
    }
  }

  window.addEventListener('resize', resizeCanvasToDisplaySize)

  // Start loop when texture ready to avoid flashing
  const texInterval = setInterval(() => {
    if (texReady) {
      clearInterval(texInterval)
      render(performance.now())
    }
  }, 16)

  onUnmounted(() => {
    cancelAnimationFrame(rafId)
    window.removeEventListener('resize', resizeCanvasToDisplaySize)
    gl.deleteBuffer(quadVBO)
    gl.deleteBuffer(instanceVBO)
    gl.deleteBuffer(ebo)
    gl.deleteVertexArray(vao)
    gl.deleteProgram(program)
    gl.deleteTexture(texture0)
    gl.deleteTexture(texture1)
    gl.deleteTexture(texture2)
  })
})


function createLoggingContext(gl: WebGL2RenderingContext) {
  const stats = {
    drawCalls: 0,
    textureBinds: 0,
    bufferBinds: 0,
    uniformUpdates: 0,
    reset() {
      this.drawCalls = 0;
      this.textureBinds = 0;
      this.bufferBinds = 0;
      this.uniformUpdates = 0;
    },
  };

  const original = Object.create(gl);

  // Wrap draw calls
  gl.drawArrays = new Proxy(gl.drawArrays, {
    apply(target, thisArg, args) {
      stats.drawCalls++;
      return Reflect.apply(target, thisArg, args);
    },
  });

  gl.bindTexture = new Proxy(gl.bindTexture, {
    apply(target, thisArg, args) {
      stats.textureBinds++;
      return Reflect.apply(target, thisArg, args);
    },
  });

  gl.bindBuffer = new Proxy(gl.bindBuffer, {
    apply(target, thisArg, args) {
      stats.bufferBinds++;
      return Reflect.apply(target, thisArg, args);
    },
  });

  gl.uniform1f = new Proxy(gl.uniform1f, {
    apply(target, thisArg, args) {
      stats.uniformUpdates++;
      return Reflect.apply(target, thisArg, args);
    },
  });
  gl.uniform2f = new Proxy(gl.uniform2f, {
    apply(target, thisArg, args) {
      stats.uniformUpdates++;
      return Reflect.apply(target, thisArg, args);
    },
  });

  return { gl, stats };
}

</script>

<template>
  <div class="wrap">
    <div class="hud">
      <div>{{ fpsText }}</div>
      <div>{{ statsText }}</div>
      <div class="hint">Use ?n=NUMBER to change sprite count</div>
    </div>
    <canvas ref="canvasRef" class="glcanvas"></canvas>
  </div>
</template>

<style scoped>
.wrap {
  position: fixed;
  inset: 0;
  display: grid;
}
.glcanvas {
  width: 100%;
  height: 100%;
  display: block;
}
.hud {
  position: absolute;
  top: 8px;
  left: 8px;
  background: rgba(0,0,0,0.4);
  color: #e0e0e0;
  padding: 8px 10px;
  border-radius: 6px;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
  font-size: 12px;
  line-height: 16px;
  pointer-events: none;
}
.hint {
  opacity: 0.7;
}
</style>
