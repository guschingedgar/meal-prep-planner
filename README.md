# Meal Prep — plan on the website, cook with the bot

Final project for **No Code Software Development** (NA10, Benjamin Tala) — Edgar Gusching, BSc AIDAMS, ESSEC × CentraleSupélec. Target: **Level 3**.

Planning a week of meals on a budget is tedious: you pick recipes, check what's already in the cupboard, merge ingredients into one shopping list, then still need help while cooking. Meal Prep automates the whole loop:

1. **Plan** on a website: budget, number of people, diet, cuisines, equipment, what's in the pantry, which lunches and dinners to cook. n8n builds a budget-constrained week with Spoonacular recipes chosen by an LLM (Groq), then a merged, aisle-sorted shopping list.
2. **Cook** with a Telegram bot: what's on today, the full recipe, a guided step-by-step cooking session with timers, meal swaps, the shopping list, and free-form cooking questions answered by an AI agent with memory.

**Live demo:** https://meal-prep-intake-1.vercel.app

## Architecture

<!-- Add the diagram as docs/architecture.png, then uncomment: ![Architecture](docs/architecture.png) -->

| Layer | Tool | Role |
|---|---|---|
| Frontend | Custom site on Vercel | Form, plan view, meal swaps, shopping list |
| Accounts | Supabase | Login, saved preferences, plan history |
| Logic | n8n (self-hosted, Hostinger VPS) | 7 workflows below |
| Data | Airtable | Shared memory between site and bot: requests, meals, shopping list, Telegram links and cooking sessions |
| Recipes | Spoonacular API | Recipe search, ingredients, nutrition, step-by-step instructions |
| AI | Groq (`openai/gpt-oss-120b`) | Picks the week's recipes, prices the shopping list, powers the Chef agent |
| Chat | Telegram | Cooking companion bot |

## The 7 workflows

### Stage 1 — Plan on the website (1–5)

| # | File | Trigger | What it does |
|---|---|---|---|
| 1 | `01-generate-plan.json` | Webhook `POST /meal-prep` | Validates the form, saves the request, replies at once with a plan reference (`MP-YYYYMMDD-XXXXX`), then searches Spoonacular, filters candidates by equipment and pantry, asks Groq to pick one recipe per slot within budget, validates the answer in code and saves every meal. Handles leftovers (dinner cooked in double → next day's lunch). |
| 2 | `02-plan-status.json` | Webhook `GET /meal-prep-statut?ref=` | Polled by the site: returns the finished plan, "in progress", or the failure reason. |
| 3 | `03-build-shopping-list.json` | Webhook `POST /meal-prep-valider` | When the week is validated: merges all ingredients **in code** (name normalisation, unit conversion to one unit per item), then asks Groq only for aisle, price and pantry match. |
| 4 | `04-read-shopping-list.json` | Webhook `GET /meal-prep-courses?ref=` | Returns the shopping list to the site. |
| 5 | `05-swap-meals.json` | Webhook `POST /meal-prep-remplacer` | Replaces the meals the user rejected with new recipes (never one already seen), keeping the rest of the week in mind. |

### Stage 2 — Cook with the Telegram bot (6–7)

| # | File | Trigger | What it does |
|---|---|---|---|
| 6 | `06-telegram-chef.json` | Telegram Trigger (message) | Parses intent and meal slot (Paris time), routes commands: `/today`, `/week`, `/recipe`, `/cook` (guided session: `next` / `back` / `repeat`), `/timer N`, `/list`, `/swap`, account linking via `/start MP-…`, and free-text questions to an AI Agent (Groq + 10-message memory). |
| 7 | `07-load-week.json` | Execute Workflow (sub-workflow) | Called by 6: resolves Telegram chat → account → latest ready plan → meals + current cooking session. Always returns exactly one item. |

## Course requirements

| Requirement | Where |
|---|---|
| Trigger | Webhooks (1–5), Telegram message trigger (6), sub-workflow trigger (7) |
| Router | `Route Command` and `Route Intent` Switch nodes (6); IF nodes throughout |
| Code node | Form validation, prompt building, plan parsing, ingredient merging (1, 3, 5), intent parsing (6) |
| API | Spoonacular (HTTP Request), Groq |
| Database | Airtable: 4 tables (requests, meals, shopping list, Telegram links) |
| AI node | `Chef Agent` with Groq chat model and window memory (6); Groq via HTTP for structured JSON (1, 3, 5) |
| Sub-workflow | 6 calls 7 (`Load Week`) |
| Error handling | Failures written back as `echec` with a reason the site displays; `onError: continue` + retries on API nodes; fallback recipes when the LLM answer is invalid |

## Run it yourself

1. Create the n8n credentials listed in `.env.example` (Airtable, Groq ×2, Telegram).
2. Import the 7 files from `workflows/` (n8n → *Import from file*). Import **7 before 6**, then re-select it in 6's `Load Week` node.
3. Replace `YOUR_SPOONACULAR_API_KEY` (workflows 1, 5, 6) and `YOUR_N8N_HOST` (workflow 6).
4. Re-link each Airtable / Groq / Telegram node to your credentials and activate the workflows.
5. Frontend: `frontend/` is a single static page (HTML + vanilla JS, Supabase JS from a CDN), no build step. In `index.html`, set the n8n webhook base URL and your Supabase URL + publishable key, then deploy the folder to Vercel (`vercel --prod` from `frontend/`).

## Screenshots

See `docs/screenshots/`.

## Limitations & next steps

- Groq free tier (30 requests/min, 1,000/day per organisation) caps how many plans can be generated.
- Spoonacular free quota limits recipe searches per day.
- Prices are LLM estimates, not live supermarket data.
- Planned: a redesign of the website UI.
