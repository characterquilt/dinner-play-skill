---
name: running-dinner-campaigns
description: "Runs end-to-end dinner campaign pipelines for B2B GTM. Takes an idea and date, builds a target account list via web research, finds VP/C-level contacts via Apollo, fills and bounce-checks emails via Findymail, scores prospects, maps a trending restaurant in every US city, and outputs three XLSX deliverables. Use when running a dinner play, building dinner campaign target lists, finding executive dinner prospects, or planning micro-events for sales."
---

# Dinner Play — CharacterQuilt

End-to-end pipeline: **Idea → TAL → People → Score → Restaurants → Deliverables**

## End-to-End Workflow

### Inputs required

1. **An idea/thesis** — what kind of companies to target (e.g. "downstream broker networks", "franchise systems with 100+ locations")
2. **A date** — when the dinners will happen. If not given, default to N+4 weeks from today. **Dinners must be on Tuesday, Wednesday, or Thursday only.** If the calculated date falls on another day, snap to the nearest Wednesday.

### First step: Test a few queries, then confirm with the user

Before committing to a full run, do a quick test: run 2-3 small queries using Approach A (WebSearch agents) to see if the idea translates cleanly into search results. If results are noisy, recommend switching to Approach C.

Then present findings and ask the user to confirm ALL of the following:

1. **Number of target companies**: "How many companies do you want on the TAL? (e.g. 50, 100, 200)"
2. **Targets / Personas**: "Who should we find at each company? Default is C-level + VP marketers. Do you want different titles or seniorities?" (defaults: `person_seniorities: ["c_suite", "vp"]`, `person_titles: ["Marketing", "Partner Marketing", "Channel"]`)
3. **Discovery approach**: Based on your test queries, recommend A, B, C, or a combination. Explain what worked and what didn't. "I tested a few searches — Approach A/C gave better results because [reason]. I recommend [approach]. Sound good?"
4. **Dinner date**: "Are you looking to run these dinners on {nearest Tue/Wed/Thu to N+4 weeks}?" (must be Tue/Wed/Thu — snap to nearest Wednesday if needed)
5. **Idea interpretation**: Explain back how you will find companies — what archetypes/criteria you'll search for, what types of companies you WON'T look for

Only proceed after confirmation on all five points.

### Pipeline

```
Idea + Date
    │
    ├─► Section 5: Build TAL (Approach A, B, and/or C)
    │   └─► Validated domains with justifications
    │
    ├─► Section 2: Apollo Search + Enrich (US only)
    │   └─► VP/C-level marketers at each company
    │       (person_seniorities: ["vp", "c_suite", "director"])
    │       (person_titles: based on idea — e.g. "Marketing", "Partner Marketing")
    │       (person_locations: ["United States"])
    │
    ├─► Findymail: Fill missing emails + bounce check ALL emails
    │   └─► Email lookup via linkedin_url or name+domain
    │   └─► Bounce verification on every email (adds email_verified column)
    │
    ├─► Section 1: Score with GPT-5.4-mini
    │   └─► company_icp_fit + person_icp_fit scores
    │
    ├─► Restaurant mapping (structured output per city)
    │   └─► One restaurant per unique city (ALL cities, not just top N)
    │
    └─► Three output files (XLSX)
```

### Restaurant mapping

After Apollo gives you people with locations (city from their profile), deduplicate ALL unique cities and find one restaurant per city. Every city gets a restaurant — the strategy is to invite anyone who says yes and then fill the dinner with other people in that city.

Use GPT-5.4-mini with web search + structured output to map restaurants in parallel batches. This is faster and more consistent than spawning agents per city.

Restaurant criteria (unless user specifies otherwise):
- **Capacity**: Seats 6-10 comfortably for a group dinner
- **Vibe**: Not too loud, not a date spot — good for business conversation
- **Price**: $50-$100 per person for that city's market
- **Trending**: Recently opened or well-reviewed, not a chain

```python
from openai import OpenAI
from pydantic import BaseModel

class Restaurant(BaseModel):
    city: str
    state: str
    restaurant_name: str
    address: str
    price_range: str
    why_selected: str
    booking_url: str

class RestaurantBatch(BaseModel):
    restaurants: list[Restaurant]

client = OpenAI()

# Batch cities (10-15 per call) for efficiency
result = client.responses.parse(
    model="gpt-5.4-mini",
    reasoning={"effort": "low"},
    tools=[{"type": "web_search"}],
    input=[
        {"role": "system", "content": "You are a restaurant concierge. Find one restaurant per city for a business dinner."},
        {"role": "user", "content": f"""Find a restaurant in each of these cities for a business dinner of 6-10 people:
{city_list}

Requirements per restaurant:
- Seats 6-10 comfortably
- Not too loud — good for conversation
- Not a date spot — business appropriate
- ~$50-$100 per person
- Well-reviewed, not a chain
- Include booking URL (OpenTable/Resy preferred)"""},
    ],
    text_format=RestaurantBatch,
).output_parsed
```

Run one call per city with 20 parallel workers (ThreadPoolExecutor). Join `dinner_city`, `dinner_restaurant`, `dinner_address`, `dinner_date`, and `dinner_theme` columns to the people CSV by city.

### Dinner theme

Generate a one-sentence theme based on the idea — this is the topic of discussion for the dinner. Example: for Demandbase customers, the theme might be "discuss the future of ABM and how AI will and won't affect it." The theme is the same across all dinners in a campaign. Ask the user to confirm or tweak it before finalizing.

### Output deliverables (three XLSX files)

| File | Contents | Key columns |
|------|----------|-------------|
| `{campaign}-tal.xlsx` | Target account list (companies only) | domain, company_name, employee_count, archetype, fit_justification, confidence |
| `{campaign}-tal-unified.xlsx` | Unified TAL across all discovery approaches (A + B deduplicated) | Same as above + `discovery_source` (archetype / lookalike / both) |
| `{campaign}-people-scored.xlsx` | Scored people with dinner assignments | first_name, last_name, title, company, email, linkedin_url, city, company_icp_fit, person_icp_fit, dinner_city, dinner_restaurant, dinner_address, dinner_date, dinner_theme |

---

## Progress Checklist

Copy this checklist and track progress:

```
Campaign Progress:
- [ ] Confirm: idea, company count, personas, date (Tue/Wed/Thu), theme
- [ ] Build TAL (Approach A/B/C) → companies_unified.csv
- [ ] Apollo Search (US only) → people_search.csv
- [ ] Apollo Enrich → people_enriched.csv
- [ ] Findymail: fill missing emails
- [ ] Findymail: bounce check ALL emails → people_verified.csv
- [ ] Score: company_icp_fit + person_icp_fit
- [ ] Restaurant mapping: one per unique city
- [ ] Build 3 XLSX deliverables
- [ ] Deliver output files to team
```

---

## Reference Files

| File | Contents |
|------|----------|
| [primitives-reference.md](primitives-reference.md) | API patterns for OpenAI, Apollo, Apify, Serper, Findymail (code snippets + auth + rate limits) |
| [tal-building.md](tal-building.md) | How to build target account lists from an idea (Approach A/B/C, validation, iteration) |
| [starter-script.md](starter-script.md) | Complete self-contained Python script for the post-TAL pipeline |

---

## Prerequisites: API Keys

Before running this skill, you need API keys for the following services. Paste your keys into this section — the skill reads them directly from here.

| Service | Sign up | What it does | Required? |
|---------|---------|-------------|-----------|
| **OpenAI** | [platform.openai.com](https://platform.openai.com) | Structured output scoring + restaurant mapping (GPT-5.4-mini) | Yes |
| **Apollo.io** | [apollo.io](https://www.apollo.io) | People search + enrichment (find VP/C-level contacts at target companies) | Yes |
| **Serper.dev** | [serper.dev](https://serper.dev) | Google search API for domain lookups + LinkedIn searches | Yes |
| **Findymail** | [findymail.com](https://www.findymail.com) | Email lookup + bounce verification | Yes |
| **Apify** | [apify.com](https://www.apify.com) | LinkedIn profile + post scraping (optional — for deep enrichment) | Optional |
| **Anthropic** | [console.anthropic.com](https://console.anthropic.com) | Claude API (used by Claude Code itself) | Already have it |
| **Webhound** | [webhound.ai](https://www.webhound.ai) | Autonomous web research for TAL building (Approach C fallback only) | Optional |

```
OPENAI_API_KEY=<your-openai-api-key>
APOLLO_API_KEY=<your-apollo-api-key>
SERPER_API_KEY=<your-serper-api-key>
FINDYMAIL_API_KEY=<your-findymail-api-key>
APIFY_API_TOKEN=<your-apify-api-token>
APIFY_API_TOKEN_2=<your-apify-api-token-2-optional>
ANTHROPIC_API_KEY=<your-anthropic-api-key>
WEBHOUND_API_KEY=<your-webhound-api-key-optional>
```

---

## Quick Reference

| Primitive | Model / Service | Endpoint | Rate Limit |
|-----------|----------------|----------|------------|
| LLM scoring | `gpt-5.4-mini` | OpenAI Responses API | 15-60 parallel workers |
| People search | Apollo | `POST /api/v1/mixed_people/api_search` | 100 domains/call, 429→60s sleep |
| People enrich | Apollo | `POST /api/v1/people/bulk_match` | Batches of 10 |
| Email lookup | Findymail | `POST /api/search/mail` | 20 parallel workers |
| Email verify | Findymail | `POST /api/verify` | 20 parallel workers |
| LinkedIn profiles | Apify `supreme_coder/linkedin-profile-scraper` | Apify Actor API | 60 workers, token rotation |
| LinkedIn posts | Apify `LQQIXN9Othf8f7R5n` | Apify Actor API | 60 workers, token rotation |
| Domain/LinkedIn search | Serper | `POST google.serper.dev/search` | 50 calls/sec |
| Company news | Serper | `POST google.serper.dev/news` | 50 calls/sec |
| TAL discovery (fallback) | Webhound | `POST api.webhound.ai/api/v1/extractions` | 10 concurrent, 1/sec |
