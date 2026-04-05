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
