<template>
  <div class="app" :data-theme="theme">
    <!-- 背景效果 -->
    <div class="bg-grid"></div>
    <div class="bg-glow bg-glow-1"></div>
    <div class="bg-glow bg-glow-2"></div>
    <div class="bg-glow bg-glow-3"></div>

    <!-- Nav 导航栏 -->
    <nav class="nav">
      <div class="nav-content">
        <div class="nav-brand">
          <a href="#" class="nav-title" @click.prevent="scrollToTop">首页</a>
        </div>
        <div class="nav-right">
          <a href="#projects" class="nav-link">项目</a>
          <button
            class="theme-toggle"
            @click="toggleTheme"
            :title="theme === 'light' ? '切换到深色模式' : '切换到浅色模式'"
          >
            <span v-if="theme === 'light'">☾</span>
            <span v-else>☀</span>
          </button>
        </div>
      </div>
    </nav>

    <!-- Hero 区域 -->
    <section class="hero">
      <div class="hero-badge fade-in">
        <span class="badge-dot"></span>
        PORTFOLIO 2026
      </div>
      <h1 class="hero-title fade-in">Felix's space</h1>
      <p class="hero-tagline fade-in">欢迎访问我的项目集合，探索创意与技术的交汇</p>
      <div class="hero-actions fade-in">
        <a href="#projects" class="btn btn-primary">浏览项目</a>
        <a href="https://github.com/AxelWT" target="_blank" rel="noopener noreferrer" class="btn btn-outline">GitHub</a>
      </div>
    </section>

    <!-- 项目区域 -->
    <section id="projects" class="projects-section">
      <div class="section-header fade-in">
        <span class="section-number">01 / PROJECTS</span>
        <h2 class="section-title">项目集合</h2>
        <p class="section-desc">探索我的开源项目与技术实践</p>
      </div>
      <div class="projects-grid">
        <a
          v-for="project in projects"
          :key="project.name"
          :href="project.url"
          target="_blank"
          rel="noopener noreferrer"
          class="project-card fade-in"
        >
          <div class="project-icon">{{ project.icon }}</div>
          <h3 class="project-title">{{ project.name }}</h3>
          <p class="project-desc">{{ project.description }}</p>
        </a>
      </div>
    </section>

    <!-- Footer -->
    <footer class="footer">
      <p>&copy; {{ currentYear }} Felix's space</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { projects } from './data/projects.js'

const theme = ref('light')
const currentYear = computed(() => new Date().getFullYear())

onMounted(() => {
  const savedTheme = localStorage.getItem('theme')
  if (savedTheme) {
    theme.value = savedTheme
  } else if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
    theme.value = 'dark'
  }

  // IntersectionObserver 滚动淡入
  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          entry.target.classList.add('visible')
        }
      })
    },
    { threshold: 0.1 }
  )

  document.querySelectorAll('.fade-in').forEach((el) => {
    observer.observe(el)
  })
})

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
  localStorage.setItem('theme', theme.value)
}

function scrollToTop() {
  window.scrollTo({ top: 0, behavior: 'smooth' })
}
</script>

<style scoped>
/* 基础布局 */
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  position: relative;
  overflow-x: hidden;
  background: var(--bg-primary);
  transition: background 0.5s ease;
}

/* Nav 导航栏 */
.nav {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  padding: 1rem 3rem;
  background: linear-gradient(180deg, var(--bg-primary) 0%, transparent 100%);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  z-index: 1000;
}

.nav-content {
  max-width: 1200px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.nav-brand {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.nav-logo {
  font-size: 1.3rem;
}

.nav-title {
  font-family: 'Inter', sans-serif;
  font-size: 1rem;
  font-weight: 600;
  color: var(--text-primary);
  cursor: pointer;
  transition: color 0.2s ease;
}

.nav-title:hover {
  color: var(--accent-primary);
}

.nav-right {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.nav-link {
  font-family: 'Inter', sans-serif;
  font-size: 0.9rem;
  font-weight: 500;
  color: var(--text-secondary);
  transition: color 0.2s ease;
}

.nav-link:hover {
  color: var(--accent-primary);
}

.theme-toggle {
  background: var(--bg-secondary);
  border: 1px solid var(--border-subtle);
  border-radius: 50%;
  width: 36px;
  height: 36px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.1rem;
  color: var(--text-primary);
  transition: all 0.2s ease;
}

.theme-toggle:hover {
  border-color: var(--accent-primary);
  color: var(--accent-primary);
}

/* Hero 区域 */
.hero {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 8rem 2rem 4rem;
  text-align: center;
  position: relative;
  z-index: 1;
}

.hero-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1.25rem;
  border: 1px solid var(--border-subtle);
  border-radius: 100px;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.75rem;
  font-weight: 500;
  letter-spacing: 0.1em;
  color: var(--text-secondary);
  margin-bottom: 2rem;
}

.badge-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--accent-primary);
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}

.hero-title {
  font-family: 'Playfair Display', serif;
  font-size: clamp(3rem, 8vw, 5rem);
  font-weight: 700;
  line-height: 1.1;
  background: var(--gradient-1);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  margin-bottom: 1.25rem;
}

.hero-tagline {
  font-family: 'Inter', sans-serif;
  font-size: 1.15rem;
  color: var(--text-secondary);
  max-width: 600px;
  line-height: 1.7;
  margin-bottom: 2.5rem;
}

.hero-actions {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  justify-content: center;
}

.btn {
  display: inline-block;
  padding: 0.8rem 2rem;
  border-radius: 10px;
  font-family: 'Inter', sans-serif;
  font-size: 0.95rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  text-decoration: none;
}

.btn-primary {
  background: var(--gradient-1);
  color: #ffffff;
  border: none;
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 30px rgba(0, 212, 170, 0.25);
}

.btn-outline {
  background: transparent;
  color: var(--text-primary);
  border: 1px solid var(--border-subtle);
}

.btn-outline:hover {
  border-color: var(--accent-primary);
  color: var(--accent-primary);
  transform: translateY(-2px);
}

/* 项目区域 */
.projects-section {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem 2rem 6rem;
  position: relative;
  z-index: 1;
}

.section-header {
  margin-bottom: 3rem;
}

.section-number {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.8rem;
  font-weight: 500;
  letter-spacing: 0.15em;
  color: var(--accent-primary);
  display: block;
  margin-bottom: 0.75rem;
}

.section-title {
  font-family: 'Playfair Display', serif;
  font-size: clamp(2rem, 4vw, 3rem);
  font-weight: 700;
  color: var(--text-primary);
  margin-bottom: 0.5rem;
}

.section-desc {
  font-family: 'Inter', sans-serif;
  font-size: 1.05rem;
  color: var(--text-secondary);
}

.projects-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1.5rem;
}

.project-card {
  background: var(--bg-secondary);
  border: 1px solid var(--border-subtle);
  border-radius: 20px;
  padding: 2rem;
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 1.5rem;
  cursor: pointer;
  transition: all 0.3s ease;
  text-decoration: none;
}

.project-card:hover {
  transform: translateY(-4px);
  border-color: var(--accent-primary);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.1);
}

[data-theme="dark"] .project-card:hover {
  box-shadow: 0 12px 40px rgba(0, 212, 170, 0.08);
}

.project-icon {
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg-tertiary);
  border-radius: 12px;
  font-size: 1.5rem;
}

.project-title {
  font-family: 'Playfair Display', serif;
  font-size: 1.15rem;
  font-weight: 600;
  color: var(--text-primary);
}

.project-desc {
  font-family: 'Inter', sans-serif;
  font-size: 0.9rem;
  color: var(--text-secondary);
  line-height: 1.6;
  margin: 0;
}

/* Footer */
.footer {
  text-align: center;
  padding: 2rem;
  border-top: 1px solid var(--border-subtle);
  color: var(--text-muted);
  font-family: 'Inter', sans-serif;
  font-size: 0.85rem;
  margin-top: auto;
  position: relative;
  z-index: 1;
}

/* 响应式布局 */
@media (max-width: 768px) {
  .nav {
    padding: 1rem 1.5rem;
  }

  .nav-link {
    display: none;
  }

  .hero {
    padding: 7rem 1.5rem 3rem;
  }

  .projects-section {
    padding: 2rem 1.5rem 4rem;
  }

  .projects-grid {
    grid-template-columns: 1fr;
  }
}
</style>
