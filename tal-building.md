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
