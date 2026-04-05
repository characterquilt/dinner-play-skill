---
name: dinner-play-characterquilt
description: "End-to-end dinner play pipeline for CharacterQuilt. Takes an idea + date, builds a target account list, finds VP/C-level marketers via Apollo, scores them, maps each city to a trending restaurant, and outputs three deliverables (TAL XLSX, unified TAL, scored people with dinners). Use when the user wants to run a dinner play, build a dinner campaign, find dinner targets, or plan executive dinners."
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

## Primitives Reference

The sections below document the five core primitives used by the pipeline above.

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

## 1. LLM Structured Output + Web Search

**Model:** `gpt-5.4-mini` | **Reasoning:** `{"effort": "low"}` | **Web search:** always on | **Parallelism:** 15-60 ThreadPoolExecutor workers

### Pattern A: Pydantic + web search (preferred)

```python
from openai import OpenAI
from pydantic import BaseModel

class CompanyAttributes(BaseModel):
    industry: str
    icp_fit_score: int
    icp_fit_rationale: str
    is_b2b: bool
    company_domain: str

client = OpenAI()
result = client.responses.parse(
    model="gpt-5.4-mini",
    reasoning={"effort": "low"},
    tools=[{"type": "web_search"}],
    input=[
        {"role": "system", "content": "You are a sales intelligence analyst."},
        {"role": "user", "content": f"Research {domain} for buying signals."},
    ],
    text_format=CompanyAttributes,
).output_parsed
```


### Pattern B: Raw JSON schema

```python
SCHEMA = {
    "type": "json_schema",
    "name": "CompanyAttributes",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "industry": {"type": "string"},
            "icp_fit_score": {"type": "integer"},
            "rationale": {"type": "string"},
        },
        "required": ["industry", "icp_fit_score", "rationale"],
        "additionalProperties": False,
    },
}

response = client.responses.create(
    model="gpt-5.4-mini",
    reasoning={"effort": "low"},
    tools=[{"type": "web_search"}],
    input=[...],
    text={"format": SCHEMA},
)
result = json.loads(response.output_text)
```

### Parallel execution with checkpointing

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=20) as executor:
    futures = {executor.submit(process_row, row): i for i, row in enumerate(rows)}
    for future in as_completed(futures):
        results.append(future.result())
        if len(results) % 25 == 0:  # checkpoint every 25
            write_csv(output_path, results)
```

---

## 2. Apollo — People Search + Enrichment

**Auth:** `x-api-key` header with `APOLLO_API_KEY`

### People Search

```python
import requests

resp = requests.post(
    "https://api.apollo.io/api/v1/mixed_people/api_search",
    headers={"x-api-key": APOLLO_API_KEY, "Content-Type": "application/json"},
    json={
        "q_organization_domains_list": ["stripe.com", "ramp.com"],  # max 100 per call
        "person_titles": ["Marketing", "Demand Gen", "Growth"],
        "include_similar_titles": True,
        "person_seniorities": ["vp", "director", "head", "c_suite"],
        "person_locations": ["San Francisco"],           # cities or countries only
        "organization_num_employees_ranges": ["51,200", "201,1000"],
        "contact_email_status": ["verified"],
        "page": 1,
        "per_page": 100,
    },
)
people = resp.json()["people"]
```

**Extracted fields:** `apollo_id`, `first_name`, `last_name`, `title`, `headline`, `seniority`, `linkedin_url`, `email`, `email_status`, `city`, `state`, `country`, `organization`, `domain`, `industry`, `estimated_num_employees`, `twitter_url`, `github_url`, `departments`, `functions`

### Bulk Enrichment

```python
resp = requests.post(
    "https://api.apollo.io/api/v1/people/bulk_match",
    headers={"x-api-key": APOLLO_API_KEY, "Content-Type": "application/json"},
    params={"reveal_personal_emails": "false", "reveal_phone_number": "false"},
    json={
        "details": [                          # batches of 10
            {"id": "apollo_id_here"},         # best: direct Apollo ID
            {"linkedin_url": "https://..."},  # or: LinkedIn URL
            {"email": "jane@stripe.com"},     # or: email
            {"first_name": "Jane", "last_name": "Doe", "domain": "stripe.com"},  # or: name+domain
        ]
    },
)
matches = resp.json()["matches"]
credits = resp.json().get("credits_consumed", 0)
```

**Hard rules:**
- Search alone gives obfuscated data — MUST run enrich after search
- Full pipeline: Apollo search → Apollo enrich → Findymail email fill → Findymail bounce check
- Max 100 domains per search call, max 3,000 total results
- Rate limit: 429 → sleep 60s and retry

---

## 3. Apify — LinkedIn Profiles + Posts

**Auth:** `ApifyClient(token=APIFY_API_TOKEN)` with round-robin token rotation for multiple tokens.

### LinkedIn Profile Scraper

```python
from apify_client import ApifyClient

client = ApifyClient(token=APIFY_API_TOKEN)
run = client.actor("supreme_coder/linkedin-profile-scraper").call(
    run_input={
        "urls": [linkedin_url],
        "findContacts": {"contactCompassToken": ""},
    },
    timeout_secs=120,
)
items = list(client.dataset(run["defaultDatasetId"]).iterate_items())
profile = items[0]  # first_name, last_name, headline, positions, education, location, summary, companyName, jobTitle
```

### LinkedIn Posts Scraper

```python
run = client.actor("LQQIXN9Othf8f7R5n").call(
    run_input={
        "username": linkedin_username,   # extracted from URL via regex
        "page_number": 1,
        "limit": 3,
    },
    timeout_secs=300,
)
items = list(client.dataset(run["defaultDatasetId"]).iterate_items())
# Each post: text, url, date, stats.total_reactions (likes), stats.comments, stats.reposts
```

### Token rotation pool

```python
class ApifyClientPool:
    """Round-robin across multiple Apify tokens. Auto-marks exhausted tokens."""

    @classmethod
    def from_env(cls):
        tokens = [os.environ.get(k) for k in ["APIFY_API_TOKEN", "APIFY_API_TOKEN_2", "APIFY_API_TOKEN_3"] if os.environ.get(k)]
        return cls(tokens) if tokens else None

    def get_client(self) -> ApifyClient:
        # round-robin, skip exhausted tokens
        ...
```

**Parallelism:** 60 workers default. Both profile + posts scrapers run concurrently.

---

## 4. Serper.dev — Domain + LinkedIn Searches

**Auth:** `X-API-KEY` header with `SERPER_API_KEY` | **Endpoint:** `POST https://google.serper.dev/search` (or `/news`)

### LinkedIn people search

```python
resp = requests.post(
    "https://google.serper.dev/search",
    headers={"X-API-KEY": SERPER_API_KEY, "Content-Type": "application/json"},
    json={"q": f"{first_name} {title} {org} site:linkedin.com/in", "num": 5},
)
for result in resp.json().get("organic", []):
    if re.search(r"linkedin\.com/in/[^/]+/?", result["link"]):
        return result["link"]  # validate first_name appears in slug
```

### LinkedIn company page search

```python
resp = requests.post(
    "https://google.serper.dev/search",
    headers={"X-API-KEY": SERPER_API_KEY, "Content-Type": "application/json"},
    json={"q": f"site:linkedin.com/company {domain}", "num": 5},
)
# filter for URLs containing linkedin.com/company/
```

### Company domain lookup

```python
resp = requests.post(
    "https://google.serper.dev/search",
    headers={"X-API-KEY": SERPER_API_KEY, "Content-Type": "application/json"},
    json={"q": f"{company_name} official website", "num": 5},
)
# filter out: linkedin.com, facebook.com, twitter.com, wikipedia.org, crunchbase.com, glassdoor.com, g2.com, etc.
```

### Company news (last 3 months)

```python
resp = requests.post(
    "https://google.serper.dev/news",
    headers={"X-API-KEY": SERPER_API_KEY, "Content-Type": "application/json"},
    json={"q": f"{company_name} company", "num": 5, "tbs": "qdr:m3"},
)
news = [{"title": n["title"], "link": n["link"], "snippet": n["snippet"], "date": n["date"]}
        for n in resp.json().get("news", [])]
```

**Rate limiting:** Token bucket, 50 calls/sec, 50 parallel workers.


---

## 5. Findymail — Email Lookup + Bounce Check

**Auth:** `Authorization: Bearer {FINDYMAIL_API_KEY}` | **Base URL:** `https://app.findymail.com/api`

Two steps: (1) find missing emails, (2) bounce check ALL emails.

### Email lookup (for people missing email after Apollo enrich)

```python
import requests

FINDYMAIL_API_KEY = "<your-findymail-api-key>"
headers = {"Authorization": f"Bearer {FINDYMAIL_API_KEY}", "Content-Type": "application/json"}

# Method 1: Via LinkedIn URL (preferred — more accurate)
resp = requests.post("https://app.findymail.com/api/search/mail", headers=headers,
                     json={"linkedin_url": linkedin_url}, timeout=30)
email = resp.json().get("contact", {}).get("email")

# Method 2: Via name + domain (fallback when no LinkedIn URL)
resp = requests.post("https://app.findymail.com/api/search/mail", headers=headers,
                     json={"first_name": first_name, "last_name": last_name, "domain": domain}, timeout=30)
email = resp.json().get("contact", {}).get("email")
```

### Bounce verification (run on ALL emails — Apollo-found and Findymail-found)

```python
resp = requests.post("https://app.findymail.com/api/verify", headers=headers,
                     json={"email": email}, timeout=30)
data = resp.json()
verified = data.get("verified", False)  # True = safe to send
provider = data.get("provider", "")     # e.g. "Google", "Microsoft"
```

Run with 20 parallel workers (ThreadPoolExecutor). Adds `email_verified` (bool) and `email_provider` columns.

**Rate limits:** 429 → exponential backoff (2^attempt seconds). 402 = out of credits, 423 = subscription paused.

---

## 6. Building Target Account Lists from an Idea

Given an idea/thesis (e.g. "companies with downstream broker networks that need co-branded marketing"), this is how to build a TAL of company domains suitable for Apollo people search.

**Do NOT start with Apollo company search.** Apollo is for finding *people* at companies you already know. The TAL-building step is upstream: you need to discover which companies match the thesis first.

There are three independent discovery approaches. Use one or more depending on what you have.

### Choosing between Approach A and C

Approach A (Claude Code agents with WebSearch) works well for ideas with clear structural archetypes. Approach C (Webhound) works better when the idea is qualitative, requires web technology lookups, or is hard to decompose into search queries. **To decide:** run a few test queries with Approach A first. If the initial results are noisy or off-target, switch to Approach C.

### Approach A: From scratch (archetype-based discovery)

Use when you have an idea/thesis but no seed companies.

**1. Translate the idea into archetype patterns.** Break into structural patterns, not keywords:

```
Example for "downstream networks":
1. Franchise / multi-location (corporate → franchisees)
2. Channel / partner ecosystems (vendor → resellers/dealers/agents)
3. Financial services networks (hub → advisors/brokers/credit unions)
4. Higher-ed / multi-campus operators
5. Insurance distribution (carrier/MGA → independent agents)
```

**2. Spawn parallel Claude Code agents** (one per archetype). Each uses WebSearch + WebFetch to find and verify companies. **Do NOT use GPT-5.4-mini here** — that's for scoring/enrichment later (Section 1). Discovery needs Claude's reasoning + web tools.

```
Agent prompt per archetype:

"Find 20-30 companies matching this archetype: {archetype_description}

ICP criteria:
{icp_criteria_text}

For EACH company, use WebSearch to find it and WebFetch on its website to verify:
- It has a real domain
- It has verifiable downstream entities with marketing intent
- It fits the structural pattern (not just 'has a partner program')

Prefer non-obvious companies (not FAANG, not direct competitors of seed companies).
Be brutally honest about fit — if unsure, exclude.

Write results to {output_csv} with columns:
company_name, domain, employee_count_estimate, downstream_entity_type,
downstream_count_estimate, marketing_intent_evidence, why_non_obvious, confidence"
```

Run many agents in parallel — not just one per archetype, but multiple angles per archetype. For "franchise / multi-location", run separate agents for QSR franchises, fitness franchises, home services, education, etc. The goal is breadth through parallelism. 5-15 agents is typical. Each produces a CSV fragment. Consolidate and deduplicate after.

### Approach B: From seed companies (lookalike expansion)

Use when you already have a TAL and want to expand it. The key insight: match the *distribution model*, not the product vertical. "Non-obvious" = same archetype, different vertical. Insurance broker network → supplements via practitioners. Franchise QSR → franchise tutoring.

Spawn a Claude Code agent per seed company:

```
Agent prompt per seed:

"Seed company: {seed_name} ({seed_domain})
Their model: {seed_downstream_description}

Find 5-10 companies that have the SAME distribution model but in DIFFERENT verticals.
Example: if seed is cyber insurance via brokers, find other products distributed
via independent agents/brokers (life insurance, supplements via practitioners, etc.)

Do NOT return:
- Direct competitors in the same vertical
- Companies already on this list: {existing_domains}

Use WebSearch and WebFetch to verify each company exists and has the distribution
model you claim. Write results to {output_csv} with columns:
seed_company, lookalike_company, domain, employee_count_estimate,
downstream_entity_type, fit_strength, non_obvious_rating, honest_fit_assessment"
```

### Approach C: Webhound extraction (when A is too hard)

Use when the idea is qualitative, requires technology lookups, or Approach A produces noisy results. Webhound runs autonomous web research agents that search, verify, and return structured company data.

**Auth:** `Authorization: Bearer {WEBHOUND_API_KEY}` | **Base URL:** `https://api.webhound.ai/api/v1`

**Models:** `"flash"` (fast, default) or `"pro"` (most capable). These are Webhound's internal models — use `"flash"` unless the query is very complex.

**Simplest flow — auto_start:**

```python
import requests, time

WEBHOUND_BASE = "https://api.webhound.ai/api/v1"
headers = {"Authorization": f"Bearer {WEBHOUND_API_KEY}", "Content-Type": "application/json"}

# 1. Create + auto-start extraction
resp = requests.post(f"{WEBHOUND_BASE}/extractions", headers=headers, json={
    "prompt": f"Find {n} companies that meet ALL of the following criteria:\n{idea_criteria_text}",
    "max_results": n,
    "credit_limit": 10.00,  # USD budget cap
    "model": "flash",
    "auto_start": True,
    "schema": {
        "entity": {
            "name": "Company",
            "criteria": ["Is it a legitimate company?", "Does it have a publicly discoverable domain?"],
            "description": "Companies matching the criteria"
        },
        "reasoning": "Return a concise reason for inclusion.",
        "attributes": [
            {"name": "company_name", "type": "string", "is_primary": True,
             "description": "Full company name", "standard_format": "Full company name"},
            {"name": "domain", "type": "string",
             "description": "Root domain (no path)", "standard_format": "example.com"},
            {"name": "reasoning", "type": "string",
             "description": "One-sentence rationale", "standard_format": "Plain text"},
        ]
    },
    "search_plan": "Search trusted sources (company sites, press, investor pages) and verify criteria from multiple sources.",
    "completion_criteria": f"Stop when {n} companies are found that clearly meet the criteria with domains.",
})
session_id = resp.json()["session_id"]

# 2. Poll until complete (typically 2-10 min)
while True:
    status_data = requests.get(f"{WEBHOUND_BASE}/extractions/{session_id}/status", headers=headers).json()
    s = status_data["status"]
    items_so_far = status_data["progress"]["items_extracted"]
    cost = status_data["costs"]["total_cost"]
    print(f"Status: {s}, Items: {items_so_far}, Cost: ${cost:.2f}")
    if s in ("completed", "paused"):  # paused = budget exhausted
        break
    time.sleep(3)

# 3. Export as CSV directly (easiest)
csv_resp = requests.get(f"{WEBHOUND_BASE}/extractions/{session_id}/export",
                        headers=headers, params={"format": "csv"})
with open("webhound_results.csv", "w") as f:
    f.write(csv_resp.text)

# OR fetch items programmatically (paginated, 100 per page)
items = requests.get(f"{WEBHOUND_BASE}/extractions/{session_id}/items",
                     headers=headers, params={"limit": 100, "offset": 0}).json()["items"]
# Each item: {"data": {"company_name": {"value": "..."}, "domain": {"value": "..."}, ...}}
```

**Pause → resume with new guidance** (if results are going off track):

```python
# Pause
requests.post(f"{WEBHOUND_BASE}/extractions/{session_id}/pause", headers=headers)

# Resume with updated direction
requests.post(f"{WEBHOUND_BASE}/extractions/{session_id}/start", headers=headers,
              json={"message": "Skip healthcare companies, focus only on fintech"})
```

**Cost reference:** ~$0.20 for 10 results, ~$5-15 for 1,000+ results. Flash model is ~6x cheaper than Pro.

**Rate limits:** 10 concurrent sessions, 1 creation/sec.

**When to use Webhound over Approach A:**
- The idea is qualitative ("companies that struggle with brand compliance across distributed teams")
- You need technology-based lookups (e.g. "companies using Salesforce + HubSpot")
- Archetype decomposition doesn't produce clean search vectors
- You tried a few Approach A agents and results were noisy


### Validation (runs on output of any approach)

Spawn a separate Claude Code agent that reads the consolidated candidate list and tries to disqualify each company. This catches ~30% of candidates.

```
Agent prompt for validation:

"Read {candidates_csv}. For EACH company, use WebSearch and WebFetch to verify:

1. DOWNSTREAM ENTITIES: Does this company actually have the claimed downstream
   entities? Verify the number.
2. MARKETING INTENT: Do the downstream entities actively do marketing or want
   marketing done for them? (If they're passive inventory/listings, FAIL)
3. GOVERNANCE/SCALE NEED: Does the hub need to produce branded template variants
   at scale? (If it's one-off creative, FAIL)

Common failure patterns — auto-FAIL these:
- Holding company roll-ups where acquired entities keep independent brands
- 'Has a partner program' ≠ downstream network (weak signal)
- Platform/marketplace models where platform isn't producing marketing
- Passive downstream entities without marketing function

Be brutally honest. If you're unsure, FAIL the company.
Write validated results to {output_csv} adding columns:
passes (bool), failure_reason, downstream_verified, marketing_intent_verified,
governance_need_verified"
```

### Domain resolution (Serper)

For companies that pass validation but lack domains:

```python
resp = requests.post(
    "https://google.serper.dev/search",
    headers={"X-API-KEY": SERPER_API_KEY, "Content-Type": "application/json"},
    json={"q": f"{company_name} official website", "num": 5},
)
# Filter out linkedin.com, facebook.com, wikipedia.org, crunchbase.com, etc.
```

### Feed validated domains into Apollo people search

Once you have a clean domain list, use Apollo (Section 2) to find people:

```python
resp = requests.post(
    "https://api.apollo.io/api/v1/mixed_people/api_search",
    headers={"x-api-key": APOLLO_API_KEY, "Content-Type": "application/json"},
    json={
        "q_organization_domains_list": validated_domains[:100],  # max 100
        "person_titles": ["Marketing", "Partner Marketing", "Channel"],
        "person_seniorities": ["vp", "director", "head", "c_suite"],
        "person_locations": ["United States"],  # US only for dinner feasibility
        "per_page": 100,
    },
)
```

Then run Apollo bulk enrich (Section 2) to get full profiles.

### Output format

Final TAL CSV columns:

| Column | Source |
|--------|--------|
| `domain` | Serper / LLM discovery |
| `company_name` | LLM discovery |
| `employee_count` | LLM / Apollo |
| `downstream_entity_type` | LLM discovery |
| `downstream_count` | LLM discovery |
| `marketing_intent_evidence` | LLM validation |
| `archetype` | LLM classification (e.g. "franchise", "channel", "financial_network") |
| `tier` | LLM classification (1A toolkit / 1B central exec) |
| `confidence` | LLM validation (high/medium) |
| `fit_justification` | LLM validation |

### Iteration patterns that work

1. **Audit → Remove → Replace**: After first pass, remove weak fits (30% typical), then run new searches with tighter criteria informed by what you learned.
2. **Cross-vertical expansion**: Once you find a strong pattern in one vertical, search for the same structural pattern in adjacent verticals.
3. **Consolidation beats expansion**: 100 companies with 3-part justifications > 500 companies with vibes. The validation pass is more valuable than the discovery pass.
4. **Employee size filtering**: Apply whatever size constraints the ICP specifies. Don't hardcode limits — let the idea/thesis define the range.

### What fails and how to recover

| Failure | Why | Recovery |
|---------|-----|----------|
| Holding company roll-ups | Acquired entities keep independent brands | Check if there's a single marketing org governing all brands |
| "Has a partner program" | Weak signal; partner portal ≠ downstream marketing scale | Verify the hub *produces* marketing for partners, not just enables |
| Passive downstream entities | No marketing intent = not ICP | Reclassify as Tier 3 (centralized marketing) |
| Platform/marketplace models | Platform connects buyers/sellers but doesn't produce marketing | Only qualifies if platform team produces co-branded assets for sellers |
| Direct competitors in list | Not "non-obvious" | Replace with same-model, different-vertical lookalikes |

---

## Quick Reference

| Primitive | Model / Service | Endpoint | Rate Limit |
|-----------|----------------|----------|------------|
| LLM scoring | `gpt-5.4-mini` | OpenAI Responses API | 15-60 parallel workers |
| People search | Apollo | `POST /api/v1/mixed_people/api_search` | 100 domains/call, 429→60s sleep |
| People enrich | Apollo | `POST /api/v1/people/bulk_match` | Batches of 10 |
| LinkedIn profiles | Apify `supreme_coder/linkedin-profile-scraper` | Apify Actor API | 60 workers, token rotation |
| LinkedIn posts | Apify `LQQIXN9Othf8f7R5n` | Apify Actor API | 60 workers, token rotation |
| LinkedIn people | Serper | `POST google.serper.dev/search` | 50 calls/sec |
| LinkedIn companies | Serper | `POST google.serper.dev/search` | 50 calls/sec |
| Company domains | Serper | `POST google.serper.dev/search` | 50 calls/sec |
| Company news | Serper | `POST google.serper.dev/news` | 50 calls/sec |

---

## Full Pipeline Starter Script

This is a complete, self-contained Python script that runs the post-TAL pipeline (Apollo → Findymail → Score → Restaurants → XLSX). The TAL discovery step (Section 6) uses Claude Code agents and runs before this script.

```python
#!/usr/bin/env python3
"""
Dinner Play Pipeline — runs after TAL is built.
Input: phase1/companies_unified.csv (from TAL discovery agents)
Output: 3 XLSX files in output/
"""
import csv, os, json, time, threading, requests
from concurrent.futures import ThreadPoolExecutor, as_completed
from collections import Counter
from urllib.parse import urlparse
from datetime import datetime, timedelta
from openai import OpenAI
from pydantic import BaseModel
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

# ===== CONFIG (edit these per campaign) =====
CAMPAIGN_NAME = "cq.CHANGEME.YYYY-MM-DD"
IDEA = "CHANGEME — one sentence describing the target"
THEME = "CHANGEME — dinner discussion topic"
DINNER_DATE = "YYYY-MM-DD"  # Must be Tue/Wed/Thu
TITLES = ["Marketing", "Demand Gen", "ABM", "Growth", "Digital Marketing"]
SENIORITIES = ["c_suite", "vp"]

# ===== API KEYS =====
OPENAI_API_KEY = "<your-openai-api-key>"
APOLLO_API_KEY = "<your-apollo-api-key>"
FINDYMAIL_API_KEY = "<your-findymail-api-key>"
SERPER_API_KEY = "<your-serper-api-key>"

BASE = f"campaigns/{CAMPAIGN_NAME}"
os.makedirs(f"{BASE}/phase1", exist_ok=True)
os.makedirs(f"{BASE}/phase2", exist_ok=True)
os.makedirs(f"{BASE}/output", exist_ok=True)

# ===== STEP 1: Load TAL =====
print("=" * 60)
print("STEP 1: Load TAL")
tal = []
with open(f"{BASE}/phase1/companies_unified.csv", "r") as f:
    for row in csv.DictReader(f):
        tal.append(row)
domains = [r["domain"] for r in tal if r.get("domain")]
print(f"  {len(tal)} companies, {len(domains)} domains")

# ===== STEP 2: Apollo Search (US only) =====
print("=" * 60)
print("STEP 2: Apollo Search (US only)")
all_people = []
for i in range(0, len(domains), 100):
    batch = domains[i:i+100]
    page = 1
    while True:
        resp = requests.post("https://api.apollo.io/api/v1/mixed_people/api_search",
            headers={"x-api-key": APOLLO_API_KEY, "Content-Type": "application/json"},
            json={"q_organization_domains_list": batch, "person_titles": TITLES,
                  "include_similar_titles": True, "person_seniorities": SENIORITIES,
                  "person_locations": ["United States"], "page": page, "per_page": 100})
        if resp.status_code == 429: time.sleep(60); continue
        if resp.status_code != 200: break
        data = resp.json(); people = data.get("people", [])
        if not people: break
        all_people.extend(people)
        if len(all_people) >= data.get("total_entries", 0) or page >= 10: break
        page += 1; time.sleep(0.3)
print(f"  {len(all_people)} people found")

# ===== STEP 3: Apollo Enrich =====
print("=" * 60)
print("STEP 3: Apollo Enrich")
enriched = []; credits = 0
search_rows = []
for p in all_people:
    org = p.get("organization", {}) or {}
    search_rows.append({"first_name": p.get("first_name",""), "last_name": p.get("last_name",""),
        "title": p.get("title",""), "organization": org.get("name",""),
        "domain": org.get("primary_domain",""), "apollo_id": p.get("id",""),
        "linkedin_url": p.get("linkedin_url",""), "email": p.get("email",""),
        "city": "", "state": "", "country": "", "seniority": ""})

for i in range(0, len(search_rows), 10):
    batch = search_rows[i:i+10]
    details = [{"id": p["apollo_id"]} if p.get("apollo_id") else
               {"first_name": p["first_name"], "last_name": p["last_name"], "domain": p["domain"]}
               for p in batch]
    resp = requests.post("https://api.apollo.io/api/v1/people/bulk_match",
        headers={"x-api-key": APOLLO_API_KEY, "Content-Type": "application/json"},
        params={"reveal_personal_emails": "false", "reveal_phone_number": "false"},
        json={"details": details}, timeout=30)
    if resp.status_code == 429: time.sleep(60); continue
    if resp.status_code != 200:
        if "insufficient credits" in resp.text.lower(): print(f"  OUT OF CREDITS at {len(enriched)}"); break
        for p in batch: enriched.append(p); continue
    data = resp.json(); credits += data.get("credits_consumed", 0)
    for j, m in enumerate(data.get("matches", [])):
        if m is None: enriched.append(search_rows[i+j]); continue
        org = m.get("organization", {}) or {}
        enriched.append({"first_name": m.get("first_name",""), "last_name": m.get("last_name",""),
            "title": m.get("title",""), "seniority": m.get("seniority",""),
            "organization": org.get("name",""), "domain": org.get("primary_domain",""),
            "email": m.get("email",""), "email_status": m.get("email_status",""),
            "linkedin_url": m.get("linkedin_url",""), "city": m.get("city",""),
            "state": m.get("state",""), "country": m.get("country",""),
            "apollo_id": m.get("id","")})
    if (i//10+1) % 20 == 0: print(f"  {len(enriched)}/{len(search_rows)} enriched, {credits} credits")
    time.sleep(0.2)
print(f"  {len(enriched)} enriched, {credits} credits")

# ===== STEP 4: Findymail fill + bounce =====
print("=" * 60)
print("STEP 4: Findymail email fill + bounce check")
fm_h = {"Authorization": f"Bearer {FINDYMAIL_API_KEY}", "Content-Type": "application/json"}

# Fill missing emails
missing = [p for p in enriched if not p.get("email")]
found = 0
for p in missing:
    payload = ({"linkedin_url": p["linkedin_url"]} if p.get("linkedin_url") else
               {"first_name": p["first_name"], "last_name": p["last_name"], "domain": p["domain"]}
               if p.get("first_name") and p.get("domain") else None)
    if not payload: continue
    try:
        r = requests.post("https://app.findymail.com/api/search/mail", headers=fm_h, json=payload, timeout=30)
        if r.status_code == 200:
            email = r.json().get("contact", {}).get("email")
            if email: p["email"] = email; found += 1
    except: pass
    time.sleep(0.1)
print(f"  Found {found} new emails")

# Bounce check ALL
with_email = [p for p in enriched if p.get("email")]
print(f"  Bounce checking {len(with_email)} emails...")
_lock = threading.Lock(); _verified = 0
def verify_one(person):
    global _verified
    try:
        r = requests.post("https://app.findymail.com/api/verify", headers=fm_h, json={"email": person["email"]}, timeout=30)
        if r.status_code == 200:
            d = r.json(); person["email_verified"] = str(d.get("verified", False))
            person["email_provider"] = d.get("provider", "")
            with _lock:
                if d.get("verified"): _verified += 1
            return
    except: pass
    person["email_verified"] = ""; person["email_provider"] = ""

with ThreadPoolExecutor(max_workers=20) as ex:
    list(ex.map(verify_one, with_email))
for p in enriched:
    if not p.get("email"): p["email_verified"] = ""; p["email_provider"] = ""
print(f"  {sum(1 for p in enriched if p.get('email'))} with email, {_verified} verified")

# ===== STEP 5: Restaurant mapping =====
print("=" * 60)
print("STEP 5: Restaurant mapping")
cities = {}
for p in enriched:
    if p.get("city") and p.get("email"):
        key = f"{p['city']}, {p['state']}"
        if key not in cities: cities[key] = {"city": p["city"], "state": p["state"]}

class Restaurant(BaseModel):
    restaurant_name: str; address: str; price_range: str; why_selected: str; booking_url: str

oai = OpenAI(api_key=OPENAI_API_KEY)
restaurants = {}
def find_restaurant(ck, ci):
    try:
        result = oai.responses.parse(model="gpt-5.4-mini", reasoning={"effort": "low"},
            tools=[{"type": "web_search"}],
            input=[{"role": "system", "content": "You are a restaurant concierge."},
                   {"role": "user", "content": f"Find a restaurant in {ci['city']}, {ci['state']} for a business dinner of 6-10. Not too loud, not a date spot, $50-$100/pp, well-reviewed, not a chain. Include booking URL."}],
            text_format=Restaurant).output_parsed
        return ck, {"city": ci["city"], "state": ci["state"], "restaurant_name": result.restaurant_name,
                    "address": result.address, "price_range": result.price_range,
                    "why_selected": result.why_selected, "booking_url": result.booking_url}
    except Exception as e:
        return ck, {"city": ci["city"], "state": ci["state"], "restaurant_name": "TBD",
                    "address": "", "price_range": "", "why_selected": str(e)[:100], "booking_url": ""}

done = 0
with ThreadPoolExecutor(max_workers=20) as ex:
    futs = {ex.submit(find_restaurant, k, v): k for k, v in cities.items()}
    for f in as_completed(futs):
        ck, res = f.result(); restaurants[ck] = res; done += 1
        if done % 30 == 0: print(f"  {done}/{len(cities)}")
print(f"  {len(restaurants)} restaurants found")

# ===== STEP 6: Build XLSX =====
print("=" * 60)
print("STEP 6: Build XLSX")
companies_map = {r["domain"]: r for r in tal}

def style_header(ws):
    fill = PatternFill(start_color="1F2937", end_color="1F2937", fill_type="solid")
    font = Font(color="FFFFFF", bold=True, size=11)
    for cell in ws[1]: cell.fill = fill; cell.font = font; cell.alignment = Alignment(horizontal="left")
    for col in ws.columns:
        ml = max(len(str(c.value or "")) for c in col)
        ws.column_dimensions[col[0].column_letter].width = min(ml + 4, 50)

# TAL
wb1 = Workbook(); ws1 = wb1.active; ws1.title = "Target Account List"
h = ["company_name","domain","verification_source","discovery_source"]; ws1.append(h)
for r in tal: ws1.append([r.get(k,"") for k in h])
style_header(ws1); wb1.save(f"{BASE}/output/{CAMPAIGN_NAME}-tal.xlsx")

# Unified TAL
wb2 = Workbook(); ws2 = wb2.active; ws2.title = "Unified TAL"; ws2.append(h)
for r in tal: ws2.append([r.get(k,"") for k in h])
style_header(ws2); wb2.save(f"{BASE}/output/{CAMPAIGN_NAME}-tal-unified.xlsx")

# People Scored
wb3 = Workbook(); ws3 = wb3.active; ws3.title = "People Scored"
ph = ["first_name","last_name","title","seniority","organization","domain","email","email_verified",
      "email_provider","linkedin_url","city","state","company_icp_fit","person_icp_fit",
      "dinner_city","dinner_restaurant","dinner_address","dinner_date","dinner_theme"]
ws3.append(ph); scored = 0
for p in enriched:
    if not p.get("email"): continue
    co = companies_map.get(p.get("domain",""), {})
    sources = co.get("discovery_source","")
    sc = len(sources.split("; ")) if sources else 0
    company_score = min(60 + sc * 10, 95)
    sen = p.get("seniority",""); title = (p.get("title","") or "").lower()
    person_score = 90 if sen == "c_suite" else 80 if sen == "vp" else 50
    if any(kw in title for kw in ["abm","demand gen","growth","revenue","marketing ops","marketing automation"]):
        person_score = min(person_score + 10, 95)
    ck = f"{p.get('city','')}, {p.get('state','')}"; rest = restaurants.get(ck, {})
    ws3.append([p.get("first_name",""), p.get("last_name",""), p.get("title",""), sen,
        p.get("organization",""), p.get("domain",""), p.get("email",""),
        p.get("email_verified",""), p.get("email_provider",""), p.get("linkedin_url",""),
        p.get("city",""), p.get("state",""), company_score, person_score,
        p.get("city",""), rest.get("restaurant_name",""), rest.get("address",""),
        DINNER_DATE, THEME])
    scored += 1
style_header(ws3)

# Dinner Summary sheet
ws_sum = wb3.create_sheet("Dinner Summary")
ws_sum.append(["City","State","Restaurant","Address","Price Range","Date","People Count",
               "Verified Emails","Theme"])
cc = Counter(f"{p.get('city','')}, {p.get('state','')}" for p in enriched if p.get("email") and p.get("city"))
cv = Counter(f"{p.get('city','')}, {p.get('state','')}" for p in enriched if p.get("email_verified")=="True" and p.get("city"))
for ck, count in cc.most_common():
    rest = restaurants.get(ck, {})
    ws_sum.append([rest.get("city",""), rest.get("state",""), rest.get("restaurant_name",""),
                   rest.get("address",""), rest.get("price_range",""), DINNER_DATE, count,
                   cv.get(ck, 0), THEME])
style_header(ws_sum)
wb3.save(f"{BASE}/output/{CAMPAIGN_NAME}-people-scored.xlsx")

print(f"\n{'='*60}")
print(f"DONE: {CAMPAIGN_NAME}")
print(f"  TAL: {len(tal)} companies")
print(f"  People: {scored} with email ({_verified} verified)")
print(f"  Cities: {len(restaurants)} with restaurants")
print(f"  Theme: {THEME}")
print(f"  Date: {DINNER_DATE}")
```

To use: (1) run TAL discovery agents to produce `phase1/companies_unified.csv`, (2) edit the CONFIG block at the top, (3) run the script. Everything else is automatic.
