---
description: "Start the Big Lake API and UI for local development on Windows. Sets up virtual environments, installs dependencies, and launches both servers."
agent: "agent"
tools: ["run_in_terminal"]
---

Start the Big Lake API and UI for local development on Windows (PowerShell).

## API (Flask on port 5000)

1. If `api/.venv/` does not exist, create it: `python -m venv .venv`
2. Activate the venv: `& .venv\Scripts\Activate.ps1`
3. Install dependencies: `pip install -r requirements.txt`
4. If `api/.env` does not exist, copy from `.env.example`
5. Start the API: `python run.py`

Run these commands from the `api/` workspace folder.

## UI (Vite on port 5173)

1. If `ui/node_modules/` does not exist, run `npm install`
2. Start the dev server: `npm run dev`

Run these commands from the `ui/` workspace folder. Use a **separate terminal** from the API.

## Important

- Do NOT set `ENV` in `api/.env` — leave it unset so the default admin/admin login works locally.
- The UI expects the API at `http://localhost:5000` (configured in `ui/.env`).
- Run each server in its own terminal so both stay alive.
- Wait for the API to start before confirming the UI is ready.
