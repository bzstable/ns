# Fixing Vercel NOT_FOUND (404) — Fix, Cause, and How to Avoid It

This doc explains the NOT_FOUND error you hit when deploying a static HTML site to Vercel, and how we fixed it.

---

## 1. The fix (what we changed)

We did two things:

### A. Use a `public` folder and point output there

- **Created** `public/` and **moved** `index.html` into `public/index.html`.
- **Set** in `vercel.json`:
  - `"outputDirectory": "public"` — so Vercel serves **only** the contents of `public/` as the site root.
  - `"buildCommand": null` — no build step; static files are served as-is.
  - `"framework": null` — explicitly “Other” (static), so Vercel doesn’t treat the project as Next/React/etc.

### B. Keep rewrites for SPA-style routing

- **Kept** the rewrite so every path serves the same HTML:
  - `"rewrites": [ { "source": "/(.*)", "destination": "/index.html" } ]`
- That way `/`, `/anything`, or a refresh on any path still returns your app (no 404 from the server).

**What you need to do:**

1. Commit and push: `public/index.html`, updated `vercel.json`, and (if you prefer a single source of truth) you can **remove** the old root `index.html` so the only entry point is `public/index.html`.
2. Redeploy on Vercel.

After that, the deployment URL (e.g. `https://your-project.vercel.app`) should serve your Valentine’s page instead of NOT_FOUND.

---

## 2. Root cause: what was going wrong

### What the code/setup was doing

- You had a **static** site: a single `index.html` (and maybe `vercel.json`) at the **repo root**.
- In `vercel.json` we had set `"outputDirectory": "."` (root) and `"buildCommand": null`.

So we intended: “Don’t run a build; serve files from the project root.”

### What Vercel actually does for “Other” (static) projects

From [Vercel’s Configure a Build](https://vercel.com/docs/deployments/configure-a-build#output-directory):

- For **Framework Preset = “Other”** (no framework):
  - **Output directory** is chosen as:
    - `public` **if** a `public` directory exists,
    - otherwise `.` (project root).
- **Only the contents of that output directory** are deployed and served. Anything outside that directory is **not** part of the deployment.

So:

- If Vercel (or dashboard settings) ended up using an output directory that **doesn’t** contain your `index.html`, then for every request (including `/`) there is **no file to serve** → **NOT_FOUND**.
- That can happen if:
  1. **Output directory was wrong** — e.g. overridden in the dashboard to `dist` or `build` (empty for a static-only repo), or
  2. **`outputDirectory: "."` was ambiguous or overridden** — so the “served root” wasn’t where `index.html` lived, or
  3. **Root Directory** in Vercel is set to a subfolder that doesn’t have `index.html` (or has a different structure).

In all those cases the **condition that triggers NOT_FOUND** is: **the request URL is resolved to a path in the deployment output where no file exists** (and no rewrite sends it to `index.html`).

### Misconception / oversight

- **Misconception:** “If I put `outputDirectory: '.'` and my file is at root, Vercel will serve it.”
- **Reality:** For “Other” projects, Vercel’s **default** is “use `public` if present, else root”. Explicitly setting `outputDirectory` to `"."` can still interact badly with:
  - How Vercel resolves the project root (e.g. with a Root Directory set), and
  - The fact that many presets expect a **subfolder** (e.g. `public`, `dist`, `out`) as the thing that gets deployed.
- **Oversight:** We didn’t align with Vercel’s **recommended** pattern for static “Other” projects: put static assets in `public/` and set the output directory to `public`. Doing that removes ambiguity and matches the platform’s defaults.

So the fix is: **put the app in `public/` and set `outputDirectory` to `"public"`**, and (optionally) set `framework` to `null` and `buildCommand` to `null` so no build or framework logic runs.

---

## 3. Why this error exists and the right mental model

### Why NOT_FOUND exists

- **HTTP and CDN semantics:** The server is supposed to return “404 Not Found” when the requested resource doesn’t exist. So NOT_FOUND is the correct response when the path doesn’t match any file (or rewrite) in the deployment.
- **Security / predictability:** Serving “whatever is at root” by default for every path could expose non-public files. Vercel only serves a **well-defined output directory**, so you explicitly control what’s public.

So the error is “correct” behavior: it’s protecting users from getting the wrong (or missing) resource and you from accidentally exposing the wrong directory.

### Correct mental model

1. **Deployment = one output tree**
   - A Vercel deployment has a single **output directory** (e.g. `public`, or `dist`, or `.`).
   - Only files **inside** that directory are deployed and served.
   - If `index.html` is not under that directory, `/` has nothing to serve → NOT_FOUND.

2. **Request → path resolution**
   - For a request like `GET /` or `GET /foo`:
     - Vercel first checks **rewrites** (e.g. “everything → /index.html”).
     - If no rewrite matches, it looks for a **file** at that path inside the output directory.
   - If there is no matching rewrite and no file at that path → NOT_FOUND.

3. **Static “Other” projects**
   - No build step.
   - Output directory = “what gets deployed.”
   - Best practice: put all static files in `public/` and set `outputDirectory` to `"public"` so the “deployment root” is unambiguous.

So: **NOT_FOUND = “for this URL, the deployment has no file (and no rewrite sent it to a valid resource).”** Fix it by making sure the output directory contains your entry file and/or rewrites send all routes to that file.

---

## 4. What to watch for next time

### Warning signs

- **404 on the deployment URL** right after “successful” deploy → output directory likely doesn’t contain `index.html` or is wrong.
- **Repo has only HTML/JS/CSS** but you see a “Build” step in the dashboard → check that Build Command is empty (or `null` in `vercel.json`) and Framework is “Other.”
- **Output Directory** in Vercel is something like `dist` or `build` but you never run a build → deployment is serving an empty (or wrong) folder.

### Similar mistakes

- Using **Root Directory** so that the app root is a subfolder, but still assuming `index.html` at repo root is served.
- Adding a **framework** (e.g. Next.js) by mistake: then Vercel expects a build and a framework-specific output (e.g. `.next`), and your plain `index.html` is ignored.
- **Typo or case:** `Index.html` vs `index.html` — some environments are case-sensitive; the doc mentions ensuring the filename is correct.

### Code smells / patterns

- Single static `index.html` at repo root **without** a clear `vercel.json` that sets `outputDirectory` (and optionally `framework: null`, `buildCommand: null`).
- No rewrite for `/(.*)` → `/index.html` when the app is a **single-page app** (one HTML file, client-side routing). Without it, refreshing on `/some-path` can 404.

---

## 5. Other valid approaches and trade-offs

| Approach | Trade-off |
|--------|-----------|
| **`public/` + `outputDirectory: "public"`** (what we did) | Matches Vercel’s default for “Other.” Clear separation: only `public/` is deployed. Slight change: entry file lives in `public/index.html`. |
| **Keep files at repo root, `outputDirectory: "."`** | Can work if Framework is “Other,” Build Command is empty, and Root Directory is not used. More fragile: dashboard overrides or future defaults might switch to `public` and then root files wouldn’t be served. |
| **Root Directory in dashboard** | If the app lives in a subfolder (e.g. `apps/sudha`), set Root Directory to that folder so `vercel.json` and `public/` (or root) are relative to it. Same ideas apply inside that root. |
| **Build step that copies HTML into `dist`** | e.g. `"buildCommand": "mkdir -p dist && cp public/index.html dist/"`, `outputDirectory: "dist"`. Works but adds complexity; unnecessary for a single static file. |

Recommendation: for a static-only site, use **`public/` + `outputDirectory: "public"`** and rewrites for SPA fallback. It’s the most reliable and matches Vercel’s design.

---

## Summary

- **Fix:** Put `index.html` in `public/`, set `outputDirectory: "public"`, `buildCommand: null`, `framework: null`, and keep the rewrite to `/index.html`. Redeploy.
- **Cause:** NOT_FOUND happens when the deployment’s output directory doesn’t contain a file (or rewrite) for the requested URL; our setup was ambiguous so the served root didn’t reliably contain `index.html`.
- **Concept:** Deployment = one output tree; NOT_FOUND = no file and no rewrite for that path.
- **Future:** Always align output directory with where your entry file lives; for static “Other” projects, prefer `public/` and explicit `vercel.json`.
