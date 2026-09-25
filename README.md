# KAMALA NEWS BACKEND

Flask + [crawl4ai](https://github.com/unclecode/crawl4ai) service that does
the actual crawling for the KAMALA NEWS frontend (`kamala-news.html`).

## Why a backend at all?

crawl4ai drives a real headless Chromium browser (via Playwright) to render
pages and turn them into clean markdown/links. That can't run inside a
browser tab â€” it needs a real OS process â€” so it lives here as a small API
the frontend calls, the same way `kamala-macro-backend` proxies FRED/Mistral/
Twelve Data for your other apps.

## Run it locally

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python -m playwright install --with-deps chromium
crawl4ai-setup      # one-time browser/cache setup, verifies the install
python app.py       # serves on http://localhost:5000
```

Quick check:

```bash
curl -s http://localhost:5000/api/health

curl -s -X POST http://localhost:5000/api/crawl \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.forexlive.com/"}' | head -c 500

curl -s -X POST http://localhost:5000/api/news-scan \
  -H "Content-Type: application/json" \
  -d '{"urls": ["https://www.forexlive.com/", "https://www.investing.com/news/forex-news"]}'
```

## Deploy to Render (Docker)

Regular Render buildpacks won't install Chromium's system libraries, so use
Render's **Docker** service type instead of the Python buildpack:

1. Push this `backend/` folder to a GitHub repo (or a subfolder of an
   existing one â€” set Render's "Root Directory" to `backend`).
2. Render dashboard â†’ New â†’ Web Service â†’ pick the repo.
3. Environment: **Docker** (Render will find the `Dockerfile` automatically).
4. Instance type: crawl4ai + Chromium wants real memory â€” the free tier's
   512MB will be tight; **Starter (1GB+)** is a safer floor.
5. Deploy. Render sets `PORT` itself; the Dockerfile's gunicorn command
   already binds `0.0.0.0:8000` and Render maps that automatically.
6. Once live, point the frontend's "Backend URL" field at
   `https://your-service.onrender.com`.

Same free-tier cold-start caveat as your other Render services: the first
request after idling will be slow while the container spins back up.

## Endpoints

| Method | Path             | Body                          | Returns                                   |
|--------|------------------|--------------------------------|--------------------------------------------|
| GET    | `/api/health`    | â€“                               | `{status, engine}`                          |
| POST   | `/api/crawl`     | `{"url": "..."}`                | title, description, full article markdown, headline links found on the page |
| POST   | `/api/news-scan` | `{"urls": ["...", "..."]}` (â‰¤12) | per-source list of headline links (title + url), crawled concurrently |

Both endpoints return `{"success": false, "error": "..."}` per-source/page on
failure rather than failing the whole request, except for malformed input
(missing/empty `url`/`urls`), which is a `400`.
