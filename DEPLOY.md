# Deploying FinFlo: backend on Render, frontend on Vercel

```
Phone / browser ──► Vercel (frontend, static files)
                       │  config.js says where the backend is
                       ▼
                    Render (backend, Node/Express) ──► Claude API (your key)
```

- **Backend** (`finflo-backend-deploy.zip`): a small Express server. It keeps your Anthropic API key
  secret and calls Claude for document OCR and F2F auto-fill. Host it on **Render**.
- **Frontend** (`finflo-frontend-deploy.zip`): the React/Vite app, built to static files. Host it on **Vercel**.

Order: **backend first** (you need its URL), then frontend, then lock the backend to the frontend's URL.

You need:
- a GitHub account
- a Render account (render.com)
- a Vercel account (vercel.com)
- your Anthropic API key

Sign in to Render and Vercel with GitHub; it makes connecting repos one click.

---

## Step 0: Put each part in its own GitHub repo

Render deploys only from a Git repo, so create **two private repos**, e.g. `finflo-backend` and
`finflo-frontend`.

For each zip:
1. Unzip it.
2. Open a terminal in the unzipped folder.
3. Run (example for the backend):

```bash
git init
git add .
git commit -m "FinFlo backend"
git branch -M main
git remote add origin https://github.com/<your-username>/finflo-backend.git
git push -u origin main
```

Repeat in the frontend folder with the `finflo-frontend` repo.

The zips contain **no** `.env`, `node_modules` or `dist`, and `.gitignore` keeps them out. **Never commit
your API key.**

> Alternative: one repo holding both folders also works. Set **Root Directory** to `backend` on
> Render and to `frontend` on Vercel.

---

## Step 1: Backend on Render

### Option A: Blueprint (uses the included `render.yaml`)
1. Render dashboard → **New +** → **Blueprint**.
2. Connect the `finflo-backend` repo, then click **Apply**.
3. Render asks for two values:
   - `ANTHROPIC_API_KEY`: your key (`sk-ant-...`).
   - `ALLOWED_ORIGIN`: leave **empty** for now. You fill it in during Step 3.
4. Wait for the deploy to finish ("Live").

### Option B: Manual Web Service
1. Render dashboard → **New +** → **Web Service**, then connect the `finflo-backend` repo.
2. Settings:

| Setting | Value |
|---|---|
| Runtime | Node |
| Region | Singapore (closest to India) |
| Branch | main |
| Root Directory | *(empty; or `backend` if you used one repo)* |
| Build Command | `npm ci --omit=dev` |
| Start Command | `npm start` |
| Instance type | Free (testing) or Starter (no sleep) |
| Health Check Path | `/` |

3. **Environment** → add:
   - `ANTHROPIC_API_KEY` = your key
   - `NODE_VERSION` = `22`
4. Do **not** set `PORT` or `HOST`. Render provides `PORT`, and the server listens on all
   interfaces automatically when it runs on Render.
5. Click **Create Web Service**.

### Check it
Open `https://<your-service>.onrender.com/` in a browser. You should see:

```json
{"status":"ok","service":"finflo-ocr-backend"}
```

Copy this URL (no trailing slash), e.g. `https://finflo-backend-abcd.onrender.com`.

> **Free plan note:** the service sleeps after ~15 minutes idle. The first request afterwards takes
> 30–60 seconds, so the first OCR / F2F call can look slow or time out. Open the backend URL once
> before a demo to wake it, or use the Starter plan.

---

## Step 2: Frontend on Vercel

### 2a. Point the frontend at the backend
In the frontend repo, edit **`public/config.js`**:

```js
window.__FINFLO_API_BASE__ = "https://finflo-backend-abcd.onrender.com";
```

Use your Render URL, with no trailing slash. Then commit and push:

```bash
git commit -am "Point frontend at Render backend"
git push
```

(Leave it `""` only for local development.)

### 2b. Import the project
1. Vercel dashboard → **Add New…** → **Project**, then import the `finflo-frontend` repo.
2. Vercel reads the included `vercel.json`. Check these settings:

| Setting | Value |
|---|---|
| Framework Preset | Vite |
| Root Directory | *(empty; or `frontend` if you used one repo)* |
| Install Command | `npm ci` |
| Build Command | `npm run build` |
| Output Directory | `dist` |

3. No environment variables are needed on Vercel. The key lives only on Render.
4. Click **Deploy**. You get a URL like `https://finflo-frontend.vercel.app`.

`vercel.json` also does two things:
- It stops `config.js` from being cached, so changing the backend URL takes effect immediately after
  a redeploy.
- It allows the microphone, camera and location, which the F2F and photo screens need.

> Without GitHub: install the Vercel CLI and run `npx vercel --prod` inside the frontend folder.
> Render still needs the backend in a Git repo.

---

## Step 3: Lock the backend to your frontend (CORS)

1. Render → your backend service → **Environment**.
2. Set `ALLOWED_ORIGIN` = your Vercel URL, e.g. `https://finflo-frontend.vercel.app`. Use the exact
   origin, with no trailing slash and no path.
   - To allow more than one origin (for example a custom domain), separate them with commas:
     `https://finflo-frontend.vercel.app,https://app.yourdomain.com`
3. Save. Render redeploys automatically.

Until you set this, the backend accepts requests from any website. Anyone who finds the URL could use
your API key's credits, so don't skip this step.

> Vercel **preview** deployments (one per branch or PR) have different URLs and are blocked once this
> is set. Add them to the list if you need them.

---

## Step 4: Test the live app

On a phone or PC, open the Vercel URL. Both sites are HTTPS, so the microphone, camera and GPS work
without certificate warnings.

1. **App:** the home screen loads, and the logo and name show "LSQ Finance".
2. **New LAP Application → document upload:** upload a PAN or Aadhaar image. Fields fill in, which
   proves the OCR call through Render works.
3. **Sanction Pending → a case → F2F:** record or load the sample and tap **Stop & auto-fill**. The form
   fills in, which proves the Claude F2F extraction works.
4. **Android:** the Back button goes back inside the app.

### If something fails
| Symptom | Cause / fix |
|---|---|
| OCR/auto-fill error, browser console shows **CORS** | `ALLOWED_ORIGIN` doesn't exactly match the Vercel URL (check https, no trailing `/`). The Render logs show `CORS: blocked request from …`, which names the exact origin to add. |
| Calls go to `/api/...` on vercel.app and return 404 | `public/config.js` is still `""`. Set the Render URL, push, and redeploy. |
| First call very slow / times out | The free Render instance was asleep. Retry, or wake it by opening the backend URL first. |
| `401` / "invalid x-api-key" in Render logs | Wrong or missing `ANTHROPIC_API_KEY` on Render. |
| `429` rate limited | Your Anthropic account's rate limit was hit. Wait and retry. |
| Mic/camera blocked | Use the HTTPS Vercel URL (not http) and allow the permission in the browser's site settings. |
| Vercel build fails on `npm ci` | Make sure `package-lock.json` was committed. |

---

## Updating later
- **Code changes:** `git push` to either repo. Render and Vercel each redeploy on their own.
- **Changing the backend URL:** edit `public/config.js` in the frontend, then push. No code change is
  needed.
- **Rotating the API key:** change `ANTHROPIC_API_KEY` in Render → Environment. Nothing changes on the
  frontend.

## Alternative: frontend on Render too
Render → **New +** → **Static Site**, then connect the frontend repo:
- Build Command: `npm ci && npm run build`
- Publish Directory: `dist`

Set `config.js` the same way, and put this Static Site's URL in `ALLOWED_ORIGIN`.
