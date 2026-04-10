# Paracket

**AI-powered brand voice marketing system** — analyze how a brand communicates, then automatically generate and schedule platform-native social media content that sounds like them.

## What it does

1. **Brand Voice Analysis** — Scrapes YouTube transcripts, company blogs, and optionally Reddit to build a multi-dimensional voice profile (tone, formality, vocabulary patterns, humor style, sentence structure, and more) using GPT-4o.
2. **Trend Discovery** — Monitors Hacker News, Product Hunt, and Dev.to simultaneously to surface relevant topics for the brand.
3. **Content Generation** — Produces a master message grounded in the voice profile, then adapts it for Twitter/X (280 chars), Mastodon (500 chars), and Reddit (conversational format).
4. **Scheduling** — Posts are saved as JSON, committed to the repo, and published at the scheduled time via a GitHub Actions workflow that runs every 5 minutes — no dedicated server needed.

## Architecture

```
paracket/
├── streamlit_app/
│   ├── app.py                      # Home page + quick setup
│   ├── pages/
│   │   ├── 1_Brand_Analysis.py     # Data collection + voice profiling
│   │   ├── 2_Content_Generator.py  # Trend fetch + post generation
│   │   ├── 4_Scheduled_Posts.py    # Schedule, edit, cancel posts
│   │   └── 5_History.py            # Past analyses
│   ├── modules/
│   │   ├── brand_voice_analyzer.py # GPT-4o analysis + generation
│   │   ├── ai_blog_finder.py       # GPT-4o powered blog URL discovery
│   │   ├── youtube_scraper.py      # YouTube Data API + transcripts
│   │   ├── blog_scraper.py         # RSS + BeautifulSoup scraping
│   │   ├── reddit_scraper.py       # PRAW-based Reddit scraping
│   │   ├── hackernews_scraper.py   # HN trending topics
│   │   ├── producthunt_scraper.py  # Product Hunt trending
│   │   ├── devto_scraper.py        # Dev.to trending
│   │   └── social_poster.py        # Twitter, Mastodon, Reddit posting
│   ├── scheduler.py                # Run by GitHub Actions to post on schedule
│   ├── data/                       # Saved brand voice profiles (gitignored)
│   └── .streamlit/
│       └── secrets.toml.template   # API key template
└── .github/workflows/
    └── post-scheduler.yml          # Runs scheduler.py every 5 minutes
```

## Setup

### Prerequisites

- Python 3.9+
- API keys for: OpenAI (GPT-4o), YouTube Data API v3, Reddit (optional), Twitter/X (optional), Mastodon (optional)

### Local development

```bash
cd streamlit_app

python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

cp .streamlit/secrets.toml.template .streamlit/secrets.toml
# Edit secrets.toml and add your API keys

streamlit run app.py
```

App runs at `http://localhost:8501`.

### secrets.toml

```toml
OPENAI_API_KEY = "..."
YOUTUBE_API_KEY = "..."

# Optional — required only if using Reddit as a data source
REDDIT_CLIENT_ID = "..."
REDDIT_CLIENT_SECRET = "..."
REDDIT_USER_AGENT = "paracket/1.0"

# Optional — required only for automated posting
TWITTER_API_KEY = "..."
TWITTER_API_SECRET = "..."
TWITTER_ACCESS_TOKEN = "..."
TWITTER_ACCESS_TOKEN_SECRET = "..."

MASTODON_ACCESS_TOKEN = "..."
MASTODON_API_BASE_URL = "https://mastodon.social"
```

### Streamlit Cloud deployment

1. Push this repo to GitHub.
2. Go to [share.streamlit.io](https://share.streamlit.io), connect the repo, and set the main file to `streamlit_app/app.py`.
3. Add your API keys under **Advanced settings → Secrets**.
4. For scheduled posting, add the same secrets as GitHub Actions repository secrets and enable the workflow in `.github/workflows/post-scheduler.yml`.

See `streamlit_app/DEPLOYMENT.md` for detailed instructions.

## Getting API keys

| Service | Where to get it |
|---|---|
| OpenAI | platform.openai.com/api-keys |
| YouTube Data API v3 | console.cloud.google.com → APIs & Services → Credentials |
| Reddit | reddit.com/prefs/apps → Create App (type: script) |
| Twitter/X | developer.twitter.com → Projects & Apps |
| Mastodon | Your instance's Settings → Development → New Application |

## Cost

| Component | Cost |
|---|---|
| Streamlit Cloud hosting | Free |
| GitHub Actions scheduling | Free (within limits) |
| YouTube Data API | Free (10k quota units/day) |
| Reddit API | Free |
| OpenAI GPT-4o | ~$0.02 per brand analysis, ~$0.01 per generation run |

A typical full workflow (scrape + analyze + generate for 3 platforms) costs roughly $0.04–0.08.

## How it was built

Paracket was originally prototyped in n8n as a visual workflow, then ported to Python/Streamlit to make it accessible without requiring workflow automation knowledge. The scheduling problem — Streamlit Cloud has no background task support — was solved by using GitHub Actions as an external cron runner, with JSON files in the repo as shared state between the UI and the scheduler.

The brand voice analysis went through many prompt engineering iterations to move beyond sentiment classification toward capturing the intangible qualities that make communication feel authentic: rhythm, register, humor style, topic framing.

See `PORTFOLIO_CASE_STUDY.md` for the full development story.

## Security

- Never commit `.streamlit/secrets.toml` — it's gitignored by default.
- Rotate API keys if accidentally exposed.
- All generated content requires user review before scheduling.

---

Built with Streamlit and OpenAI GPT-4o. Part of INFO7375 at Northeastern University.
