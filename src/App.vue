<template>
  <div class="app" :data-theme="theme" :class="{ 'no-webgl': !webgl }">
    <!-- LAYER 0: 海面（WebGL shader，失败时回退到静态渐变） -->
    <canvas ref="seaCanvas" class="sea"></canvas>
    <div class="sea-fallback" aria-hidden="true"></div>

    <!-- LAYER 200: 四角 chrome -->
    <header class="chrome chrome-tl">
      <span class="brand">≈&nbsp;FELIX'S SPACE</span>
    </header>

    <nav class="chrome chrome-tr">
      <a class="nav-link" href="#projects" @click.prevent="openProjects">Projects</a>
      <a
        class="nav-link"
        href="https://github.com/AxelWT"
        target="_blank"
        rel="noopener noreferrer"
      >GitHub&nbsp;↗</a>
    </nav>

    <p class="chrome chrome-bc hint">MOVE — THE SEA FOLLOWS</p>

    <footer class="chrome chrome-bl">
      <p>© {{ currentYear }} FELIX'S SPACE</p>
      <p class="dim">SOMEWHERE AT SEA · ALL TIDES CALM</p>
    </footer>

    <div class="chrome chrome-br">
      <button class="ctl" @click="toggleMotion">
        {{ motionOn ? 'Motion On' : 'Motion Off' }}
      </button>
      <button class="ctl" @click="toggleTheme">
        {{ theme === 'day' ? 'Night ☾' : 'Day ☀' }}
      </button>
    </div>

    <!-- LAYER 100: hero 主文案 -->
    <main class="hero">
      <p class="kicker">FELIX'S SPACE — PORTFOLIO 2026</p>
      <h1 class="line">
        欢迎来我的海，所有<a class="dotted" href="#projects" @click.prevent="openProjects">项目</a>都是岛屿，<br />
        随潮汐涨落，漂流而立。
      </h1>
      <p class="sub">BUILT IN SPARE TIME, SHAPED BY CURIOSITY.</p>
    </main>

    <!-- LAYER 300: 项目浮层 -->
    <div
      class="overlay"
      :class="{ 'is-open': overlayOpen, 'is-closing': overlayClosing }"
      @click.self="closeProjects"
    >
      <section class="panel" role="dialog" aria-label="项目列表">
        <div class="panel-head">
          <div>
            <h2 class="panel-title">Project Log</h2>
            <p class="panel-meta">{{ projects.length }} ENTRIES · OPEN SEA</p>
          </div>
          <button class="close" aria-label="关闭项目列表" @click="closeProjects">✕</button>
        </div>
        <ul class="entries">
          <li v-for="(project, i) in projects" :key="project.name">
            <a
              class="entry"
              :href="project.url"
              target="_blank"
              rel="noopener noreferrer"
            >
              <span class="idx">{{ String(i + 1).padStart(2, '0') }}</span>
              <span class="name">{{ project.name }}</span>
              <span class="desc">{{ project.description }}</span>
              <span class="arr" aria-hidden="true">↗</span>
            </a>
          </li>
        </ul>
      </section>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import { projects } from './data/projects.js'

/* ------------------------------------------------------------------ */
/* 海面 shader：高度场光线步进 + 域扭曲噪声波浪 + 距离雾                */
/* 调色板刻意压低饱和度——银灰昼海 / 墨蓝夜海                           */
/* ------------------------------------------------------------------ */

const VERT = `
attribute vec2 aPos;
void main() { gl_Position = vec4(aPos, 0.0, 1.0); }
`

const FRAG = `
#ifdef GL_FRAGMENT_PRECISION_HIGH
precision highp float;
#else
precision mediump float;
#endif

uniform vec2 uRes;
uniform float uTime;
uniform vec2 uMouse;
uniform float uNight;

float hash21(vec2 p) {
  p = fract(p * vec2(127.1, 311.7));
  p += dot(p, p + 34.45);
  return fract(p.x * p.y);
}

float vnoise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0);
  float a = hash21(i);
  float b = hash21(i + vec2(1.0, 0.0));
  float c = hash21(i + vec2(0.0, 1.0));
  float d = hash21(i + vec2(1.0, 1.0));
  return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}

float fbm(vec2 p) {
  float v = 0.0;
  float a = 0.5;
  for (int i = 0; i < 3; i++) {
    v += a * vnoise(p);
    p = mat2(1.6, 1.2, -1.2, 1.6) * p;
    a *= 0.5;
  }
  return v;
}

float waveHeight(vec2 p, float t) {
  vec2 w = vec2(fbm(p * 0.32 + t * 0.05), fbm(p * 0.32 - t * 0.04));
  vec2 q = p + (w - 0.5) * 2.4;
  float h = 0.0;
  float amp = 0.40;
  float freq = 0.50;
  float spd = 0.80;
  for (int i = 0; i < 4; i++) {
    float an = 0.7 + float(i) * 2.0;
    vec2 dir = vec2(cos(an), sin(an));
    float ph = dot(q, dir) * freq + t * spd;
    float wv = 1.0 - abs(sin(ph));
    h += wv * wv * amp;
    freq *= 2.1;
    amp *= 0.42;
    spd *= 1.18;
  }
  h += (sin(dot(p, vec2(0.35, 0.42)) * 0.55 - t * 0.42) * 0.5 + 0.5) * 0.26;
  return h;
}

vec3 skyColor(vec3 rd, vec3 sunDir) {
  float y = max(rd.y, 0.0);
  vec3 zen = mix(vec3(0.58, 0.63, 0.68), vec3(0.045, 0.065, 0.090), uNight);
  vec3 hor = mix(vec3(0.80, 0.81, 0.80), vec3(0.140, 0.170, 0.210), uNight);
  vec3 col = mix(hor, zen, smoothstep(0.0, 0.55, y));
  float s = max(dot(rd, sunDir), 0.0);
  vec3 sunC = mix(vec3(1.00, 0.96, 0.88), vec3(0.72, 0.78, 0.86), uNight);
  col += sunC * pow(s, 18.0) * 0.25;
  col += sunC * pow(s, 280.0) * 1.1;
  return col;
}

void main() {
  vec2 uv = (gl_FragCoord.xy * 2.0 - uRes) / uRes.y;
  float t = uTime * 0.6;

  // 相机缓慢向前漂移，鼠标提供轻微视差
  vec3 ro = vec3(0.0, 2.4, uTime * 1.4);
  vec3 rd = normalize(vec3(
    uv.x * 0.72 + uMouse.x * 0.20,
    uv.y * 0.72 + 0.10 + uMouse.y * 0.08,
    -1.0
  ));
  vec3 sunDir = normalize(vec3(0.32, 0.20, -1.0));

  vec3 col;

  if (rd.y > -0.015) {
    col = skyColor(rd, sunDir);
  } else {
    float tm = 0.0;
    float tPrev = 0.0;
    float dPrev = ro.y - waveHeight(ro.xz, t);
    float hitT = -1.0;
    for (int i = 0; i < 56; i++) {
      vec3 pos = ro + rd * tm;
      float d = pos.y - waveHeight(pos.xz, t);
      if (d < 0.002 + tm * 0.001) {
        hitT = mix(tPrev, tm, clamp(dPrev / max(dPrev - d, 1e-4), 0.0, 1.0));
        break;
      }
      tPrev = tm;
      dPrev = d;
      tm += max(d * 0.6, 0.02 + tm * 0.004);
      if (tm > 80.0) break;
    }

    if (hitT < 0.0) {
      // 掠射未命中——直接融入地平线雾色
      col = skyColor(normalize(vec3(rd.x, 0.02, rd.z)), sunDir);
    } else {
      vec3 pos = ro + rd * hitT;
      float eps = 0.10 + hitT * 0.03;
      float hc = waveHeight(pos.xz, t);
      float hx = waveHeight(pos.xz + vec2(eps, 0.0), t);
      float hz = waveHeight(pos.xz + vec2(0.0, eps), t);
      vec3 n = normalize(vec3(hc - hx, eps, hc - hz));
      n = normalize(mix(n, vec3(0.0, 1.0, 0.0), clamp(hitT * 0.012, 0.0, 0.7)));

      float fres = 0.04 + 0.96 * pow(1.0 - max(dot(n, -rd), 0.0), 5.0);
      vec3 refl = skyColor(reflect(rd, n), sunDir);

      float crest = clamp(hc * 1.35, 0.0, 1.0);
      vec3 deep    = mix(vec3(0.140, 0.200, 0.230), vec3(0.012, 0.025, 0.040), uNight);
      vec3 shallow = mix(vec3(0.380, 0.440, 0.460), vec3(0.050, 0.080, 0.110), uNight);

      col = mix(mix(deep, shallow, crest), refl, fres);

      float spec = pow(max(dot(reflect(rd, n), sunDir), 0.0), 240.0);
      col += mix(vec3(1.00, 0.95, 0.85), vec3(0.75, 0.82, 0.92), uNight) * spec * 1.2;

      vec3 fogC = skyColor(normalize(vec3(rd.x, 0.02, rd.z)), sunDir);
      float fog = 1.0 - exp(-hitT * 0.035);
      col = mix(col, fogC, fog);
    }
  }

  float r2 = dot(uv, uv);
  col *= 1.0 - 0.15 * r2;
  col += (hash21(gl_FragCoord.xy + fract(uTime)) - 0.5) * (2.0 / 255.0);

  gl_FragColor = vec4(col, 1.0);
}
`

/* ------------------------------------------------------------------ */
/* 状态                                                                */
/* ------------------------------------------------------------------ */

const seaCanvas = ref(null)
const webgl = ref(true)
const overlayOpen = ref(false)
const overlayClosing = ref(false)
const currentYear = computed(() => new Date().getFullYear())

const savedTheme = localStorage.getItem('theme')
const theme = ref(
  savedTheme === 'night' || savedTheme === 'dark' ? 'night'
  : savedTheme === 'day' || savedTheme === 'light' ? 'day'
  : window.matchMedia('(prefers-color-scheme: dark)').matches ? 'night'
  : 'day'
)

const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches
const savedMotion = localStorage.getItem('motion')
const motionOn = ref(savedMotion ? savedMotion === 'on' : !prefersReduced)

/* ------------------------------------------------------------------ */
/* WebGL 渲染循环                                                       */
/* ------------------------------------------------------------------ */

const QUALITY = [0.75, 0.55, 0.4]
let gl = null
let program = null
const uni = {}
let elapsed = motionOn.value ? 0 : 42
let nightMix = theme.value === 'night' ? 1 : 0
const mouse = { x: 0, y: 0 }
const mouseTarget = { x: 0, y: 0 }
let qualityIdx = 0
let rafId = 0
let running = false
let last = 0
let emaMs = 16
let frames = 0
let lastStatic = 0
let readyShown = false
let closeTimer = 0

function compile(type, src) {
  const s = gl.createShader(type)
  gl.shaderSource(s, src)
  gl.compileShader(s)
  if (!gl.getShaderParameter(s, gl.COMPILE_STATUS)) {
    console.error(gl.getShaderInfoLog(s))
    return null
  }
  return s
}

function initSea() {
  const canvas = seaCanvas.value
  gl = canvas.getContext('webgl', {
    antialias: false,
    depth: false,
    stencil: false,
    alpha: false,
  })
  if (!gl) {
    webgl.value = false
    return
  }
  const vs = compile(gl.VERTEX_SHADER, VERT)
  const fs = compile(gl.FRAGMENT_SHADER, FRAG)
  if (!vs || !fs) {
    webgl.value = false
    return
  }
  program = gl.createProgram()
  gl.attachShader(program, vs)
  gl.attachShader(program, fs)
  gl.linkProgram(program)
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    webgl.value = false
    return
  }
  gl.useProgram(program)

  const buf = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buf)
  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1, -1, 3, -1, -1, 3]), gl.STATIC_DRAW)
  const loc = gl.getAttribLocation(program, 'aPos')
  gl.enableVertexAttribArray(loc)
  gl.vertexAttribPointer(loc, 2, gl.FLOAT, false, 0, 0)

  uni.res = gl.getUniformLocation(program, 'uRes')
  uni.time = gl.getUniformLocation(program, 'uTime')
  uni.mouse = gl.getUniformLocation(program, 'uMouse')
  uni.night = gl.getUniformLocation(program, 'uNight')

  canvas.addEventListener('webglcontextlost', (e) => {
    e.preventDefault()
    stopLoop()
    webgl.value = false
  })

  resize()
  renderFrame()
}

function resize() {
  if (!gl) return
  const canvas = seaCanvas.value
  const dpr = Math.min(window.devicePixelRatio || 1, 1.75)
  let w = Math.round(canvas.clientWidth * dpr * QUALITY[qualityIdx])
  let h = Math.round(canvas.clientHeight * dpr * QUALITY[qualityIdx])
  const m = Math.max(w, h)
  if (m > 2048) {
    w = Math.round((w * 2048) / m)
    h = Math.round((h * 2048) / m)
  }
  if (canvas.width !== w || canvas.height !== h) {
    canvas.width = w
    canvas.height = h
    gl.viewport(0, 0, w, h)
  }
}

function renderFrame() {
  if (!gl) return
  gl.uniform2f(uni.res, seaCanvas.value.width, seaCanvas.value.height)
  gl.uniform1f(uni.time, elapsed)
  gl.uniform2f(uni.mouse, mouse.x, mouse.y)
  gl.uniform1f(uni.night, nightMix)
  gl.drawArrays(gl.TRIANGLES, 0, 3)
  if (!readyShown) {
    readyShown = true
    seaCanvas.value.classList.add('shader-ready')
  }
}

function tick(now) {
  if (!running) return
  const dtMs = now - last
  last = now
  elapsed += Math.min(dtMs / 1000, 0.05)
  mouse.x += (mouseTarget.x - mouse.x) * 0.045
  mouse.y += (mouseTarget.y - mouse.y) * 0.045
  const target = theme.value === 'night' ? 1 : 0
  nightMix += (target - nightMix) * 0.05
  renderFrame()

  // 帧耗时滑窗平均，过载时逐级降分辨率
  emaMs += (dtMs - emaMs) * 0.04
  frames++
  if (frames > 120 && emaMs > 27 && qualityIdx < QUALITY.length - 1) {
    qualityIdx++
    resize()
    emaMs = 16
    frames = 0
  }
  rafId = requestAnimationFrame(tick)
}

function startLoop() {
  if (running || !webgl.value) return
  running = true
  last = performance.now()
  rafId = requestAnimationFrame(tick)
}

function stopLoop() {
  running = false
  cancelAnimationFrame(rafId)
}

// 动效关闭时，指针 / 主题变化仍以低频重绘单帧
function requestStatic() {
  if (running || !webgl.value) return
  const now = performance.now()
  if (now - lastStatic < 120) return
  lastStatic = now
  requestAnimationFrame(renderFrame)
}

/* ------------------------------------------------------------------ */
/* 交互                                                                */
/* ------------------------------------------------------------------ */

function onResize() {
  resize()
  requestStatic()
}

function onPointerMove(e) {
  mouseTarget.x = (e.clientX / window.innerWidth) * 2 - 1
  mouseTarget.y = -((e.clientY / window.innerHeight) * 2 - 1)
  requestStatic()
}

function onVisibility() {
  if (document.hidden) stopLoop()
  else if (motionOn.value) startLoop()
}

function onKeyDown(e) {
  if (e.key === 'Escape' && overlayOpen.value) closeProjects()
}

function toggleTheme() {
  theme.value = theme.value === 'day' ? 'night' : 'day'
  localStorage.setItem('theme', theme.value)
  if (!running) {
    nightMix = theme.value === 'night' ? 1 : 0
    requestAnimationFrame(renderFrame)
  }
}

function toggleMotion() {
  motionOn.value = !motionOn.value
  localStorage.setItem('motion', motionOn.value ? 'on' : 'off')
  if (motionOn.value) startLoop()
  else stopLoop()
}

function openProjects() {
  overlayOpen.value = true
}

function closeProjects() {
  if (!overlayOpen.value || overlayClosing.value) return
  overlayClosing.value = true
  closeTimer = setTimeout(() => {
    overlayOpen.value = false
    overlayClosing.value = false
  }, 350)
}

onMounted(() => {
  initSea()
  if (motionOn.value) startLoop()
  window.addEventListener('resize', onResize)
  window.addEventListener('pointermove', onPointerMove)
  window.addEventListener('keydown', onKeyDown)
  document.addEventListener('visibilitychange', onVisibility)
})

onBeforeUnmount(() => {
  stopLoop()
  clearTimeout(closeTimer)
  window.removeEventListener('resize', onResize)
  window.removeEventListener('pointermove', onPointerMove)
  window.removeEventListener('keydown', onKeyDown)
  document.removeEventListener('visibilitychange', onVisibility)
})
</script>

<style scoped>
/* ------------------------------------------------------------------ */
/* LAYER 0: 海面                                                        */
/* ------------------------------------------------------------------ */

.sea {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  height: 100dvh;
  z-index: 0;
  opacity: 0;
  transition: opacity 1.2s ease;
}

.sea.shader-ready {
  opacity: 1;
}

.no-webgl .sea {
  display: none;
}

.sea-fallback {
  display: none;
  position: fixed;
  inset: 0;
  z-index: 0;
  background: linear-gradient(180deg, #6a7480 0%, #a9b2b4 52%, #3d4d54 53%, #263339 100%);
}

.no-webgl .sea-fallback {
  display: block;
}

[data-theme='night'].sea-fallback,
[data-theme='night'] .sea-fallback {
  background: linear-gradient(180deg, #0b1016 0%, #131a22 52%, #060a0f 53%, #04070b 100%);
}

/* ------------------------------------------------------------------ */
/* LAYER 200: 四角 chrome                                               */
/* ------------------------------------------------------------------ */

.chrome {
  position: fixed;
  z-index: 200;
  font-family: var(--mono);
  font-size: 12px;
  font-weight: 500;
  line-height: 1.7;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: var(--ink-dim);
  text-shadow: 0 1px 12px rgba(10, 18, 24, 0.4);
}

.chrome-tl {
  top: var(--inset);
  left: var(--inset);
}

.brand {
  color: var(--ink);
}

.chrome-tr {
  top: var(--inset);
  right: var(--inset);
  display: flex;
  gap: 2em;
}

.chrome-bl {
  bottom: var(--inset);
  left: var(--inset);
}

.chrome-bl .dim {
  color: var(--ink-faint);
  font-size: 10px;
  letter-spacing: 0.2em;
}

.chrome-br {
  bottom: var(--inset);
  right: var(--inset);
  display: flex;
  gap: 1.6em;
}

.hint {
  bottom: var(--inset);
  left: 50%;
  transform: translateX(-50%);
  font-size: 10px;
  letter-spacing: 0.34em;
  color: var(--ink-faint);
  white-space: nowrap;
}

@media (hover: none), (max-width: 680px) {
  .hint {
    display: none;
  }
}

.nav-link,
.ctl {
  position: relative;
  color: var(--ink-dim);
  transition: color 0.25s ease, text-shadow 0.3s ease;
  padding: 4px 0;
}

.nav-link:hover,
.ctl:hover {
  color: var(--ink);
  text-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

.nav-link::after {
  content: '✶';
  position: absolute;
  top: -0.75em;
  right: -0.7em;
  font-size: 10px;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.nav-link:hover::after {
  opacity: 0.9;
  animation: star-spin 0.85s linear infinite;
}

@keyframes star-spin {
  to {
    transform: rotate(360deg);
  }
}

/* ------------------------------------------------------------------ */
/* LAYER 100: hero 主文案                                               */
/* ------------------------------------------------------------------ */

.hero {
  position: relative;
  z-index: 100;
  height: 100vh;
  height: 100dvh;
  display: grid;
  place-content: center;
  text-align: center;
  padding: 0 6vw 6vh;
  text-shadow: 0 1px 16px rgba(10, 18, 24, 0.4);
}

.kicker {
  font-size: 11px;
  font-weight: 500;
  letter-spacing: 0.3em;
  text-transform: uppercase;
  color: var(--ink-faint);
  margin-bottom: 2.4em;
}

.line {
  font-family: var(--serif);
  font-style: italic;
  font-weight: 400;
  font-size: clamp(24px, 3vw, 42px);
  line-height: 1.4;
  color: rgba(255, 255, 255, 0.96);
  text-wrap: balance;
}

.sub {
  margin-top: 2.2em;
  font-size: 12px;
  font-weight: 400;
  letter-spacing: 0.22em;
  color: var(--ink-faint);
}

@media (max-width: 640px) {
  .line br {
    display: none;
  }
}

.dotted {
  position: relative;
  text-decoration: underline dotted;
  text-decoration-thickness: 0.04em;
  text-underline-offset: 0.14em;
  transition: text-shadow 0.3s ease;
}

.dotted:hover {
  text-decoration: none;
  text-shadow: 0 0 8px rgba(255, 255, 255, 0.9);
}

.dotted::after {
  content: '✶';
  position: absolute;
  left: 50%;
  bottom: -1.3em;
  width: 1em;
  font-size: 0.42em;
  text-shadow: none;
  transform: translateX(-50%);
  opacity: 0;
  transition: opacity 0.2s ease;
}

.dotted:hover::after {
  opacity: 0.9;
  animation: star-spin 0.85s linear infinite;
}

@keyframes rise-in {
  from {
    opacity: 0;
    transform: translateY(16px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

.kicker {
  animation: rise-in 1s var(--ease) 0.25s backwards;
}

.line {
  animation: rise-in 1.1s var(--ease) 0.45s backwards;
}

.sub {
  animation: rise-in 1.1s var(--ease) 0.7s backwards;
}

/* ------------------------------------------------------------------ */
/* LAYER 300: 项目浮层                                                  */
/* ------------------------------------------------------------------ */

.overlay {
  position: fixed;
  inset: 0;
  z-index: 300;
  display: grid;
  place-items: center;
  padding: calc(var(--inset) + 10px);
  background: rgba(6, 10, 14, 0.32);
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.35s var(--ease);
}

.overlay.is-open {
  opacity: 1;
  pointer-events: auto;
}

.overlay.is-closing {
  opacity: 0;
  pointer-events: none;
}

.panel {
  width: min(880px, 100%);
  max-height: min(74vh, 700px);
  display: flex;
  flex-direction: column;
  background: var(--panel-bg);
  -webkit-backdrop-filter: blur(18px) saturate(1.15);
  backdrop-filter: blur(18px) saturate(1.15);
  border: 1px solid var(--panel-border);
  box-shadow: 0 30px 80px rgba(0, 0, 0, 0.5);
  opacity: 0;
  transform: translateY(16px);
  transition: opacity 0.35s var(--ease), transform 0.35s var(--ease);
}

.overlay.is-open .panel {
  opacity: 1;
  transform: none;
}

.panel-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  padding: 26px 30px 14px;
  border-bottom: 1px solid var(--hairline);
}

.panel-title {
  font-size: 13px;
  font-weight: 600;
  letter-spacing: 0.3em;
  text-transform: uppercase;
  color: var(--ink);
}

.panel-meta {
  margin-top: 8px;
  font-size: 11px;
  letter-spacing: 0.14em;
  color: var(--ink-faint);
}

.close {
  font-size: 15px;
  color: var(--ink-dim);
  padding: 2px 4px;
  transition: color 0.2s ease, transform 0.25s ease;
}

.close:hover {
  color: #fff;
  transform: rotate(90deg);
}

.entries {
  list-style: none;
  overflow-y: auto;
  padding: 10px 18px 22px;
  scrollbar-width: thin;
  scrollbar-color: rgba(255, 255, 255, 0.22) transparent;
}

.entries::-webkit-scrollbar {
  width: 8px;
}

.entries::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.18);
}

.entry {
  display: grid;
  grid-template-columns: 2.6rem minmax(0, 1fr) auto 1.3rem;
  align-items: baseline;
  gap: 0 1.2rem;
  padding: 14px 12px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);
  transition: background 0.2s ease;
}

li:last-child .entry {
  border-bottom: none;
}

.entry:hover {
  background: rgba(255, 255, 255, 0.05);
}

.idx {
  font-size: 11px;
  letter-spacing: 0.1em;
  color: var(--ink-faint);
}

.name {
  font-family: var(--serif);
  font-style: italic;
  font-weight: 400;
  font-size: 19px;
  color: var(--ink);
  transition: text-shadow 0.3s ease;
}

.entry:hover .name {
  text-shadow: 0 0 12px rgba(255, 255, 255, 0.45);
}

.desc {
  font-size: 11.5px;
  letter-spacing: 0.08em;
  color: var(--ink-dim);
}

.arr {
  justify-self: end;
  font-size: 13px;
  color: var(--ink-faint);
  transition: transform 0.25s ease, color 0.25s ease;
}

.entry:hover .arr {
  color: var(--ink);
  transform: translate(2px, -2px);
}

@media (max-width: 640px) {
  .entry {
    grid-template-columns: 1.9rem minmax(0, 1fr) 1.2rem;
    grid-template-areas:
      'idx name arr'
      'idx desc .';
    row-gap: 4px;
  }
  .idx { grid-area: idx; }
  .name { grid-area: name; }
  .desc { grid-area: desc; }
  .arr { grid-area: arr; }
  .chrome-tr { gap: 1.2em; }
}

/* ------------------------------------------------------------------ */
/* 无障碍：减少动效                                                      */
/* ------------------------------------------------------------------ */

@media (prefers-reduced-motion: reduce) {
  .kicker,
  .line,
  .sub {
    animation: none;
  }
  .nav-link:hover::after,
  .dotted:hover::after {
    animation: none;
  }
  .sea {
    transition: none;
  }
}
</style>
