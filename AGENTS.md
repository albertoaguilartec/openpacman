# AGENTS.md

## What this is

Vanilla JS/HTML/CSS Pac-Man game. No build step, no bundler, no package manager, no tests.

## How to run

Open `src/index.html` in a browser, or serve `src/` with any static file server.

## Structure

All source lives under `src/`. Scripts are loaded via `<script>` tags in `src/index.html` — **order matters**:

1. `maze.js` (maze data)
2. `game.js` (game logic)
3. `render.js` (canvas rendering)
4. `main.js` (entry point, wires everything together)

Global scope only — no ES modules, no import/export.

## Conventions

- UI text is in Spanish.
- Canvas is fixed at 560×620 (`src/index.html:11`).

## Spec-driven workflow

The project follows a spec-driven development approach. Two skills are installed in `.agents/skills/`:

- **`spec`** — Design and write a spec before coding. Use it when starting a large feature. It guides you through scoping, clarifying questions, and producing a spec file saved to `specs/`. State starts as `Draft`.
- **`spec-impl`** — Implement an approved spec. Verifies the spec is in `Approved` state, creates a branch named `spec-NN-slug`, and implements the plan step by step with pauses for review.

**Flow:** `/spec <description>` → review and approve → `/spec-impl <NN-slug>`

Specs live in `specs/`. Config (e.g. `AutoCreateBranch`) goes in `specs/.spec-config.yml`.
