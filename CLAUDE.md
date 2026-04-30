# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ekluvya Assessment is an adaptive AI-powered exam preparation tool for Indian competitive exams (JEE and NEET). It generates questions using Groq LLM (llama-3.3-70b-versatile), adapts difficulty based on student proficiency, and produces AI-streamed diagnostic reports.

## Commands

### Server (Express + MongoDB) — run from `server/`
```bash
npm run dev     # start with nodemon (hot reload)
npm start       # production start
```

### Client (React + Vite + Tailwind) — run from `client/`
```bash
npm run dev     # Vite dev server on :5173 (proxies /api to :5000)
npm run build   # production build to client/dist
```

Both must run simultaneously for local development. No test framework is configured.

## Environment Variables

Server requires a `.env` file in `server/` with:
- `MONGODB_URI` — MongoDB connection string (required)
- `GROQ_API_KEY` — Groq API key (format: `gsk_...`). Without it, questions and reports fall back to static/hardcoded content
- `GROQ_MODEL` — optional, defaults to `llama-3.3-70b-versatile`
- `CLIENT_URL` — optional, defaults to `http://localhost:5173`
- `PORT` — optional, defaults to 5000

## Architecture

**Monorepo with two independent npm projects** (`client/` and `server/`), no shared workspace config.

### Server (`server/`)
- **Express + Mongoose** (CommonJS `require`)
- Three route groups mounted at `/api/assessment`, `/api/questions`, `/api/report`
- `controllers/questionsController.js` — the core adaptive engine. Two endpoints:
  - `POST /generate` — initial batch of questions for setup (AI with static fallback)
  - `POST /next` — single adaptive question during test, difficulty derived from proficiency level (0-100 mapped to easy/medium/hard)
- `controllers/reportController.js` — streams AI diagnostic report via **SSE** (`text/event-stream`), with a hardcoded fallback if Groq fails
- `lib/groqClient.js` — Groq SDK wrapper with connectivity check
- `lib/buildPrompt.js` — constructs the report analysis prompt
- `data/questions.js` — static question pool used as fallback when AI is unavailable
- Single Mongoose model: `Assessment` (stores exam results with per-question answer data)

### Client (`client/`)
- **React 18 + React Router v6 + Tailwind CSS** (Vite, ESM)
- `context/AssessmentContext.jsx` — central state via `useReducer`. Tracks exam config, question bank, answers, proficiency level, and timer. All test state flows through dispatch actions.
- Page flow: `Landing` → `Setup` (config + question generation) → `Test` (adaptive quiz) → `Report` (SSE-streamed AI analysis)
- `hooks/useStream.js` — EventSource hook that consumes the SSE report stream, splits output into insights and study plan sections at the `---PLAN---` marker
- `lib/api.js` — axios client with `/api` baseURL (Vite proxies to server)
- Vite dev server proxies `/api` requests to `http://localhost:5000`

### Key Data Flow
1. Setup page calls `POST /api/questions/generate` to get initial question batch
2. During test, after each answer, client may call `POST /api/questions/next` with current proficiency level, used IDs, and weak/strong topics to get an adaptive follow-up question
3. On test completion, client calls `POST /api/assessment/save`, receives an assessment ID
4. Report page opens `GET /api/report/:id/stream` as an EventSource for streamed AI analysis

## gstack

**Web browsing:** Always use the `/browse` skill from gstack for all web browsing. Never use `mcp__claude-in-chrome__*` tools.

### Available Skills
`/office-hours`, `/plan-ceo-review`, `/plan-eng-review`, `/plan-design-review`, `/design-consultation`, `/design-shotgun`, `/design-html`, `/review`, `/ship`, `/land-and-deploy`, `/canary`, `/benchmark`, `/browse`, `/connect-chrome`, `/qa`, `/qa-only`, `/design-review`, `/setup-browser-cookies`, `/setup-deploy`, `/retro`, `/investigate`, `/document-release`, `/codex`, `/cso`, `/autoplan`, `/plan-devex-review`, `/devex-review`, `/careful`, `/freeze`, `/guard`, `/unfreeze`, `/gstack-upgrade`, `/learn`
