# Space Index

个人门户主页 — 一个基于 Vue 3 + Vite 构建的项目展示页面。

## 功能特性

- 项目卡片网格展示，支持响应式布局（3列 / 2列 / 1列）
- 深色 / 浅色主题切换，自动保存用户偏好
- 滚动淡入动画（IntersectionObserver）
- 自定义字体（Playfair Display、JetBrains Mono、Inter）

## 技术栈

- [Vue 3](https://vuejs.org/)（Composition API + `<script setup>`）
- [Vite 6](https://vite.dev/)

## 快速开始

```bash
# 安装依赖
npm install

# 启动开发服务器（默认 http://localhost:3000）
npm run dev

# 构建生产版本
npm run build

# 预览构建产物
npm run preview
```

## 项目结构

```
space-index/
├── index.html          # 入口 HTML
├── vite.config.js      # Vite 配置
├── package.json
├── public/             # 静态资源
│   └── vite.svg
├── src/
│   ├── main.js         # 应用入口
│   ├── App.vue         # 根组件
│   ├── assets/
│   │   └── style.css   # 全局样式与 CSS 变量
│   └── data/
│       └── projects.js # 项目数据
└── main.py             # PyCharm 默认脚本（与前端无关）
```

## 添加项目

编辑 `src/data/projects.js`，按格式添加新条目：

```js
{
  name: '项目名称',
  description: '项目描述',
  url: 'https://example.com',
  icon: '🚀'
}
```
