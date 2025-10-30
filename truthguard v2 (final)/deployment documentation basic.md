Chat gpt chat:
https://chatgpt.com/share/68cd154b-8b88-8003-bcea-fb7f1455fa50

Working deployment guide for TruthGuard AI

Nice — you got everything working end-to-end. Below I’ll turn that hard-earned knowledge into a careful, opinionated, step-by-step deployment tutorial that:

replicates the exact architecture you used (Vercel frontend, Railway/Render backend(s), Supabase DB),

points out the specific mistakes you encountered and how to avoid them, and

gives minimal, safe code/config snippets you should apply.

This is long and detailed by design (you asked for ≥1000 words). Read the “Fast action checklist” if you want the quick fixes, then go through the rest for full context.

At-a-glance architecture

GitHub monorepo truthguard-ai/ with subfolders:

frontend/ → Next.js app → Vercel

backend/ → FastAPI → Railway

ai-service/ → FastAPI + embeddings → Render (or Railway if you prefer)

database/ → SQL schema/seed → Supabase (managed Postgres + pgvector)

Fast action checklist (what to change now if something’s broken)

Backend CORS: Fix duplicate allow_origins keyword and set explicit allowed origins for your Vercel domains. (See corrected snippet below.)

Supabase Auth: Set “Site URL” and allowed redirect URLs to your Vercel domain(s) — otherwise email links will point to localhost.

Embedding mismatch: Ensure EMBEDDING_MODEL in ai-service matches the model used to create existing vectors. If you change model, re-embed or recreate the vector column.

Torch wheels on Render: Add PyTorch CPU wheel index or a compatible torch version in requirements.txt that Render can find.

Git push errors: git pull --rebase origin main then git push -u origin main.

Re-deploy rules:

If you change backend URL env var in Vercel: redeploy frontend.

If you change CORS or server code: redeploy backend (Railway).

If you change ai-service env / model: redeploy ai-service (Render).

Corrected backend CORS snippet (apply immediately)

You had duplicate allow_origins and then CORS failures in browser. Use a single call and explicit domains in production. Do not set "*" in production unless you absolutely must.

# backend/app/main.py — CORS middleware (fixed)
from fastapi.middleware.cors import CORSMiddleware

allowed = [
    "http://localhost:3000",
    "http://localhost:3001",
    "https://truthguard-ai.vercel.app",
    "https://<your-vercel-preview>.vercel.app"  # add preview domains as needed
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


Why this fixes your issue: browsers perform a preflight OPTIONS request. If the server response lacks Access-Control-Allow-Origin (because CORS was misconfigured or crashed), the browser blocks the request. The SyntaxError you had (duplicate keyword) caused backend to crash or misbehave — fix that and CORS headers will be present.

Supabase Auth: fix signup link redirect to Vercel

You saw confirmation links redirecting to localhost or returning otp_expired. That happens when Supabase's Site URL or Redirect URLs are not set to your deployed frontend.

In Supabase dashboard → Project Settings → Authentication → URL Configuration:

Set Site URL to https://your-vercel-domain.vercel.app

Add Redirect URLs (exact):

https://your-vercel-domain.vercel.app

https://your-vercel-domain.vercel.app/* (if allowed)

Save.

If email templates contain {{ .SiteURL }} they will now use the correct domain. After updating, re-send verification email and test.

Environment variables: where to set them and what to set

Do not push .env files to the repo. Use each platform’s dashboard.

Frontend (Vercel Project frontend):

NEXT_PUBLIC_BACKEND_URL = https://truthguard-ai-production.up.railway.app (or whatever Railway URL is)

NEXT_PUBLIC_SUPABASE_URL = https://<your-sb>.supabase.co

NEXT_PUBLIC_SUPABASE_ANON_KEY = Supabase anon key

Backend (Railway service backend):

NEXT_PUBLIC_SUPABASE_URL = same

SUPABASE_SERVICE_ROLE_KEY = Supabase service role key (secret)

AI_SERVICE_URL = public URL of ai-service (Render/Railway)

NEXT_PUBLIC_BACKEND_URL (if used by other services)

JWT_SECRET etc.

AI service (Render / Railway service ai-service):

NEXT_PUBLIC_SUPABASE_URL

SUPABASE_SERVICE_ROLE_KEY

EMBEDDING_MODEL = e.g. sentence-transformers/all-mpnet-base-v2 (choose one)

API keys used by ai-service (SERPER_API_KEY, OPENROUTER_API_KEY, ...)

Important: When you change a NEXT_PUBLIC_* var used by frontend, redeploy the frontend. When you change backend CORS or server code, redeploy the backend.

Torch + GPU woes (Render): how to avoid build failures

You hit pip errors No matching distribution found for torch==2.3.0+cpu and later fixed by using versions Render could fetch.

Two options:

Use a stable cpu version that Render knows about, and include the extra index for PyTorch CPU wheels in requirements.txt or the service build command.

Or avoid building heavy ML libs on Render: use a managed ai-host with GPU or host the heavy inference locally/elsewhere.

If you keep torch in requirements.txt, add this at top of your requirements.txt (or instruct Render to use the extra index in the build command):

--find-links https://download.pytorch.org/whl/cpu/torch_stable.html
torch==2.8.0+cpu
torchvision==0.23.0+cpu
...


Render needs either CPU wheels accessible via download.pytorch.org or direct wheel URLs.

Vector dimension mismatch — cause and minimal fix

You saw errors:

different vector dimensions 768 and 384
expected 768 dimensions, not 384


Cause: your knowledge_base.embedding vectors in Postgres were created with one model (embedding dimension 384, e.g. all-MiniLM-L6-v2), then ai-service switched to all-mpnet-base-v2 (768 dims) — pgvector rejects mixing dims.

Minimal options to fix:

Make ai-service use the original model (quick): set EMBEDDING_MODEL in ai-service env to whichever model produced the existing vectors (probably all-MiniLM-L6-v2) and redeploy ai-service. That avoids touching the DB.

Migrate the DB to 768 dims (if you prefer new model):

Create a new column: embedding_768 vector(768) NULL

Recompute embeddings for all articles using the new model via a script or the ai-service add-article path (reindex).

After all rows have new embeddings, drop old column and rename — or create a new knowledge_base table and re-insert records (simpler).

Do not attempt to reshape or coerce vectors — re-embed.

SQL pattern (minimal, illustrative):

-- Add new column (create vector extension and adjust name if needed)
ALTER TABLE knowledge_base ADD COLUMN embedding_768 vector(768);
-- Then use your ai-service to POST each article -> add_article will fill embedding_768 (you may modify add_article to write embedding_768)
-- After done:
ALTER TABLE knowledge_base DROP COLUMN embedding;
ALTER TABLE knowledge_base RENAME COLUMN embedding_768 TO embedding;


(You will need to modify ai-service insert statement to write to embedding_768 while migrating.)

How to test step-by-step (order matters)

Local smoke tests:

curl http://localhost:8000/ (backend root)

curl http://localhost:8001/health (ai-service)

curl -X OPTIONS https://your-backend-url/api/v1/claims/submit (simulate preflight)

Supabase auth test: create a test user via frontend and click confirmation link — confirm it goes to Vercel URL.

Cross-service test: submit a claim from the frontend, check backend logs for POST /api/v1/claims/submit, check backend calls to ai-service /analyze and ai-service /add-article.

Vector test: create a test article flow that triggers add-article. If you see different vector dimensions stop and pick the correct model or re-embed.

Git gotchas you ran into (how to avoid)

remote origin already exists: use git remote set-url origin <url> to change remote.

rejected ... fetch first: do git pull --rebase origin main (resolve conflicts) then git push -u origin main.

Do not commit node_modules, .next, or .env — include them in .gitignore.

CRLF warnings: run git config core.autocrlf true on Windows or set consistent line endings.

Where to redeploy and when

Change only backend code or CORS → redeploy the backend (Railway).

Change ai-service env (model, keys) → redeploy ai-service (Render).

Change frontend env or UI code → redeploy frontend (Vercel).

Change database rows or schema → update Supabase (SQL editor) — no redeploy needed but services may rely on new schema.

If you change a URL env var used by the frontend (NEXT_PUBLIC_BACKEND_URL), you must redeploy Vercel so the new env is baked into the client.

Minimal code changes you likely need (recap)

Fix CORS (remove duplicate keyword) — backend main.py (snippet above).

Set EMBEDDING_MODEL consistently across environments and choose whether to re-embed or switch ai-service back to original model.

Add PyTorch --find-links to requirements.txt or lock to a Render-available version.

Set Supabase Site URL & Redirect URLs to Vercel domain.

Final testing checklist (before announcing “it’s live”)

Backend /api/v1/health returns healthy (AI + DB healthy).

ai-service /health is healthy and logs show embeddings being computed without errors.

Frontend (Vercel) can successfully call backend endpoints (no CORS errors in Browser console).

Supabase sign-up confirmation links lead to Vercel domain and create sessions correctly.

Vector-related operations (add-article / analyze) run without dimension errors.

All secrets are stored in dashboards, not in repo.

If you like, I can:

produce the exact requirements.txt tweaks you should use for Render (I saw what versions you installed successfully — I can generate the precise snippet to avoid the earlier torch/torchvision mismatch),

provide a one-file migration script to re-embed all rows (safe, iterative),

or craft the exact git commands to safely force sync your local with remote (with conflict resolution steps).

Tell me which one you want next and I’ll give it — minimal, tested, copy-paste ready.