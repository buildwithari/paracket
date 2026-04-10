# Paracket

An AI-powered brand voice marketing system. It analyzes how a brand communicates, then generates and schedules platform-native social media content that sounds like them.

## What it does

1. **Brand Voice Analysis** - Scrapes YouTube transcripts, company blogs, and optionally Reddit to build a multi-dimensional voice profile (tone, formality, vocabulary patterns, humor style, sentence structure, and more) using GPT-4o.
2. **Trend Discovery** - Monitors Hacker News, Product Hunt, and Dev.to simultaneously to surface relevant topics for the brand.
3. **Content Generation** - Produces a master message grounded in the voice profile, then adapts it for Twitter/X (280 chars), Mastodon (500 chars), and Reddit (conversational format).
4. **Scheduling** - Posts are saved as JSON, committed to the repo, and published at the scheduled time via a GitHub Actions workflow that runs every 5 minutes. No dedicated server needed.

## Background

Most social media tools either handle scheduling without generating content, or generate content that reads generically and has no connection to how a brand actually writes. Paracket tries to solve both at once.

The core idea is that a brand's voice is learnable from its existing public content. Given enough samples of how a company writes on YouTube, their blog, and Reddit, GPT-4o can extract a structured profile covering things like formality level, sentence rhythm, humor style, use of jargon, and how they frame topics. That profile then drives content generation, so posts aren't just topically relevant but actually sound like the brand.

The workflow is intentionally sequential. You pick a company, collect data, run analysis, then generate content against a trending topic. Nothing is posted without you reviewing and approving it first. The automation only kicks in at the scheduling step, where GitHub Actions checks every 5 minutes for posts that are due.

**Brand voice profile** - GPT-4o extracts 10+ characteristics from collected content samples. The prompt engineering went through many iterations to get past surface-level tone labels and capture subtler qualities like how technical the vocabulary is, whether the writing is first or third person, and how the brand typically opens a post.

**Blog discovery** - Finding a company's RSS feed is harder than it sounds. Early versions used pattern matching against common URL structures, which failed often. Now GPT-4o suggests likely blog URLs based on the company name, and the scraper validates them. Success rate improved significantly.

**Trend sources** - Originally Reddit was the primary trend source. During development, the Reddit account got banned due to request patterns triggering fraud detection. Rather than just fixing Reddit, the system was rebuilt to pull from Hacker News, Product Hunt, and Dev.to simultaneously. This also made it more resilient: if one source has nothing relevant for a given brand, the others fill the gap.

**Scheduling without a server** - Streamlit Cloud does not support background tasks. The solution was to use GitHub Actions as a cron runner. The Streamlit UI writes scheduled posts as JSON files to the repo, and Actions picks them up at the right time, posts them, and writes the status back. It is a bit unconventional but keeps infrastructure costs at zero.

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

# Optional - required only if using Reddit as a data source
REDDIT_CLIENT_ID = "..."
REDDIT_CLIENT_SECRET = "..."
REDDIT_USER_AGENT = "paracket/1.0"

# Optional - required only for automated posting
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
3. Add your API keys under **Advanced settings > Secrets**.
4. For scheduled posting, add the same secrets as GitHub Actions repository secrets and enable the workflow in `.github/workflows/post-scheduler.yml`.

See `streamlit_app/DEPLOYMENT.md` for detailed instructions.

## Getting API keys

| Service | Where to get it |
|---|---|
| OpenAI | platform.openai.com/api-keys |
| YouTube Data API v3 | console.cloud.google.com, APIs & Services, Credentials |
| Reddit | reddit.com/prefs/apps, Create App (type: script) |
| Twitter/X | developer.twitter.com, Projects & Apps |
| Mastodon | Your instance Settings, Development, New Application |

## Cost

| Component | Cost |
|---|---|
| Streamlit Cloud hosting | Free |
| GitHub Actions scheduling | Free (within limits) |
| YouTube Data API | Free (10k quota units/day) |
| Reddit API | Free |
| OpenAI GPT-4o | ~$0.02 per brand analysis, ~$0.01 per generation run |

A full run (scrape + analyze + generate for 3 platforms) costs roughly $0.04-0.08.

## How it was built

Paracket was originally prototyped in n8n as a visual workflow, then ported to Python/Streamlit to make it accessible without requiring workflow automation knowledge. Streamlit Cloud has no background task support, so GitHub Actions handles scheduling as an external cron runner, with JSON files in the repo as shared state between the UI and the scheduler.

The brand voice analysis went through many prompt engineering iterations. The goal was to get past sentiment classification and capture the qualities that make communication feel authentic: rhythm, register, humor style, topic framing.

See `PORTFOLIO_CASE_STUDY.md` for the full development story.

## Security

- Never commit `.streamlit/secrets.toml` - it is gitignored by default.
- Rotate API keys if accidentally exposed.
- All generated content requires user review before scheduling.

---

Built with Streamlit and OpenAI GPT-4o. Part of INFO7375 at Northeastern University.
