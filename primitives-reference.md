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
