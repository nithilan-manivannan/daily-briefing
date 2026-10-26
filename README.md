# daily-briefing

**A backend that syncs the school portal, Gmail, news, and Google Calendar into an AI generated daily briefing.**

daily-briefing is an Express backend for a newspaper style personal dashboard aimed at a school student. On a schedule it pulls data from four sources, an AI model summarizes everything into one coherent briefing, and a single page app renders the result like a small daily paper: today's agenda, the weather, school notices, a filtered inbox, world headlines, and notes. A GitHub Actions worker repeats the same pipeline against a shared MongoDB cluster every six hours, and a second workflow pings the Render deployment through the day so it stays warm.

## What it syncs

| Connector | Source | What it pulls |
|---|---|---|
| NPS scraper | School parent portal | Notices, circulars, homework, timetable data (Playwright driven) |
| Gmail | IMAP-less Gmail OAuth | Recent mail, with an AI classifier that labels marketing mail as drop |
| News | newsapi.org | World headlines for the day |
| Calendar | Google Calendar OAuth | Upcoming classes, exams, and events |

## How a briefing is made

1. A cron job (or a manual trigger from the UI) starts the sync service.
2. Each connector fetches its slice: portal pages are scraped with Playwright, Gmail and Calendar are read through OAuth, and headlines arrive from the news API.
3. The AI classifier drops marketing email so the inbox stays clean. Mail that survives gets summarized.
4. The summarizer arranges everything into one briefing narrative, weather included, and the result is stored in MongoDB.
5. The SPA loads it and renders the paper: masthead dateline, today strip, classes, exam strip, Saturday status, inbox, world, and notes.

Syncs run at 7:00 and 13:00 from the server's own scheduler. The rendered interface has six tabs: Today, Weather, School, Inbox, World, and Notes. Briefing text can carry jump links that scroll you straight to the underlying email, circular, or news item it refers to.

## API surface

All of these live under `/api` and are used by the front end:

| Endpoint | Purpose |
|---|---|
| `GET /health` | Server and database status |
| `GET /schedule/today` | Today's classes and exam strip |
| `GET /calendar/upcoming` | Upcoming calendar events |
| `GET /calendar/month` | Month view data |
| `GET /weather` | Current conditions for the brief |
| `GET /briefing` | The generated briefing narrative |
| `GET /items` | Synced items (mail, circulars, news) with filtering |
| `DELETE /items/:id` | Remove an item from the feed |
| `POST /sync/nps` `POST /sync/gmail` `POST /sync/news` `POST /sync/all` | Sync anything on demand |
| `GET /gmail/auth` `GET /gmail/callback` `GET /gmail/status` | Gmail OAuth walk through |
| `GET /calendar/auth` `GET /calendar/callback` | Calendar OAuth set up |
| `POST /calendar/extract` `POST /calendar/add` | Pull event text and turn it into calendar entries |
| `POST /summarize` | Summarize arbitrary text on demand |
| `GET /notes/list` `POST /notes/add` `DELETE /notes/:id` | Simple notes storage and AI parsing |
| `POST /notes/parse-image` | Turn a timetable photo into structured data |

## Deployment

| Piece | Where it runs |
|---|---|
| Server | Render free tier (`render.yaml` in the repo) |
| Data | MongoDB Atlas |
| Scheduled syncs | GitHub Actions every 6 hours via `sync-worker.js` |
| Keep alive | GitHub Actions pings the Render URL every 14 minutes, 05:00 to 23:59 UTC |

- `sync.yml` checks out the code, installs Playwright Chromium, runs the full sync and briefing generation inside Actions, using repository secrets for every credential.
- `keep-render-alive.yml` curls the site root every 14 minutes during waking hours (the cron runs from 05:00 to 23:59 UTC), so the free tier does not cold start when the paper is being read.
- A `Dockerfile` is included for any container host: Node 20, Playwright Chromium, `node server.js`.

## Local setup

Requirements: Node 20+, a MongoDB cluster, and accounts or APIs for each connector.

```bash
git clone https://github.com/nithilan-manivannan/daily-briefing
cd daily-briefing
npm install
cp .env.example .env
npm start          # node server.js
npm run dev        # node --watch server.js
```

Note: the `.env.example` covers the school portal basics. Gmail and Calendar also need OAuth client credentials in the same file.

| Variable | Purpose |
|---|---|
| `MONGODB_URI` | Connection string for the data store |
| `GROQ_API_KEY` | Summarization and classification brain |
| `NPS_PORTAL_URL` `NPS_USERNAME` `NPS_PASSWORD` | Parent portal login |
| `GMAIL_CLIENT_ID` `GMAIL_CLIENT_SECRET` `GMAIL_REDIRECT_URI` | Gmail OAuth |
| `GCAL_CLIENT_ID` `GCAL_CLIENT_SECRET` `GCAL_REDIRECT_URI` `GCAL_CALENDAR_ID` | Calendar OAuth |
| `NEWS_API_KEY` | Headlines |
| `PORT` | HTTP port (Render sets 10000) |

Lightweight helpers exist for running it Windows style in the background: `start-bg.ps1` and `stop.ps1`.

## Important note

The `.env.example` above does not include the OAuth variables. For a full sync, add `GMAIL_*` and `GCAL_*` values from your Google Cloud project as well.

## Repository structure

```
daily-briefing/
├── server.js                  # Express bootstrap, cron schedules, connectors
├── sync-worker.js             # one shot sync for GitHub Actions
├── src/
│   ├── config.js              # env to typed config with validation
│   ├── db.js                  # MongoDB connection
│   ├── connectors/            # nps, gmail, news, calendar
│   ├── services/              # sync, summarizer, briefing
│   ├── routes/api.js          # the REST surface above
│   └── utils/                 # migration helpers
├── public/                    # the SPA
├── index.html                 # dashboard shell
├── app.js · styles.css        # front end
├── debug.html · debug.js      # timetable setup page
├── render.yaml                # Render service definition
├── Dockerfile                 # container build
└── .github/workflows/         # sync every 6h, keep alive every 14 min
```

## Related project

[nps-ai-assistant](https://github.com/nithilan-manivannan/nps-ai-assistant) carries the earlier version of the same idea, built around GitHub Pages and an Actions driven pipeline. daily-briefing houses the live backend variant with OAuth driven Gmail and Calendar sync in one place.
