# Copilot Instructions for regulate-app

## Project Overview
- This is a Vue 3 application scaffolded with Vite. The main entry point is `src/main.js`, which mounts the root component `App.vue`.
- Uses Vue Single File Components (SFCs) with `<script setup>` syntax. See [Vue SFC docs](https://vuejs.org/guide/scaling-up/sfc.html) for details.
- All source code lives in the `src/` directory. Components are in `src/components/`.
- Static assets are in `public/` and `src/assets/`.

## Key Files & Structure
- `src/App.vue`: Root component, sets up main layout and routing (if used).
- `src/main.js`: App bootstrap, creates Vue app and mounts it.
- `src/components/HelloWorld.vue`: Example component, demonstrates SFC and props usage.
- `vite.config.js`: Vite build configuration.
- `style.css`: Global styles.

## Developer Workflows
- **Development server:**
  - Run `npm install` to install dependencies.
  - Start dev server: `npm run dev` (served by Vite, hot reload enabled).
- **Build for production:**
  - Run `npm run build` (outputs to `dist/`).
- **Preview production build:**
  - Run `npm run preview`.
- **No test setup is present by default.**

## Project Conventions
- Use `<script setup>` in all new Vue SFCs for concise composition API usage.
- Prefer placing reusable UI in `src/components/`.
- Use relative imports within `src/`.
- Global styles go in `src/style.css`.
- Static files (e.g., favicon) go in `public/`.

## Integration & External Dependencies
- Vite is used for fast development and optimized builds.
- No custom service boundaries or advanced state management by default. Add Vuex/Pinia or router as needed.
- No backend/API integration is present in the starter; add as needed.

## Examples
- See `src/components/HelloWorld.vue` for prop usage and SFC patterns.
- See `vite.config.js` for build customization.

---
For more, see the [Vue 3 docs](https://vuejs.org/) and [Vite docs](https://vitejs.dev/).
