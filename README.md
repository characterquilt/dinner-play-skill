# Dinner Play Skill for Claude Code

A Claude Code skill that builds end-to-end dinner campaign pipelines: from an idea to a scored list of prospects with restaurant assignments in every city.

We used this playbook at [CharacterQuilt](https://characterquilt.com) (YC P26) to close 7 figures in ARR, double our ACV from 5 to 6 figures, and sign 2-3 deals per week — all without PR, LinkedIn ads, or anyone knowing our company.

## The idea

The hard part about getting customers is that nothing happens until someone says yes to something.

Most founders send 1,000 emails asking for demos. When they get 1-2 replies, they say "cold outbound doesn't work." The problem isn't outbound — it's that asking for a demo is like proposing marriage during appetizers. You're asking for way too much upfront.

What if instead of asking for a demo, you asked someone to a free dinner at a great restaurant with peers? That's a 2/10 on the difficulty scale. Almost everyone says yes.

Once you get people in a room, you've started a conversation. You've built a relationship. Now you're in the game — and there are a hundred ways to keep the conversation going.

## What this skill does

Give it an idea (e.g. "find Demandbase customers") and it runs the full pipeline:

1. **Builds a target account list** — discovers companies matching your idea using parallel web research agents
2. **Finds the right people** — Apollo search + enrichment for VP/C-level contacts at each company (US only)
3. **Fills + verifies emails** — Findymail lookup for missing emails, then bounce-checks every email
4. **Scores everyone** — company fit + person fit scores via GPT structured output
5. **Maps restaurants** — finds a trending, conversation-friendly restaurant in every city where a prospect lives ($50-100/pp, not a chain, not too loud)
6. **Outputs 3 XLSX files** — TAL, unified TAL, and scored people with dinner assignments

Every person gets: name, title, company, verified email, LinkedIn, city, scores, restaurant name + address, dinner date, and theme.

## How to use

### 1. Install as a Claude Code skill

```bash
# Copy SKILL.md into your Claude Code skills directory
mkdir -p ~/.claude/skills/dinner-play/
cp SKILL.md ~/.claude/skills/dinner-play/SKILL.md
```

### 2. Add your API keys

Open `~/.claude/skills/dinner-play/SKILL.md` and fill in the API Keys section near the top. You need:

| Service | What it does | Required? |
|---------|-------------|-----------|
| **OpenAI** | Scoring + restaurant mapping | Yes |
| **Apollo.io** | People search + enrichment | Yes |
| **Serper.dev** | Google search for domain/LinkedIn lookups | Yes |
| **Findymail** | Email lookup + bounce verification | Yes |
| **Apify** | LinkedIn profile/post scraping | Optional |
| **Webhound** | Autonomous web research (fallback for hard queries) | Optional |

### 3. Run it

In Claude Code:

```
/dinner-play

I need to find Demandbase customers
```

The skill will:
- Test a few search queries to see what works
- Ask you to confirm: number of companies, target personas, dinner date (Tue/Wed/Thu only), and theme
- Run the full pipeline autonomously
- Output 3 XLSX files with everything you need

### 4. What you get

| File | What's in it |
|------|-------------|
| `{campaign}-tal.xlsx` | Target account list (companies) |
| `{campaign}-tal-unified.xlsx` | Unified TAL across all discovery approaches |
| `{campaign}-people-scored.xlsx` | Scored people + dinner city + restaurant + date + theme |

The people file has a "Dinner Summary" tab showing every city, its restaurant, and how many verified prospects are there.

## The playbook (what to do with the output)

**Setup:**
- Book the restaurants (use the booking URLs in the output). Private dining is $1,500-3,000 for 10-12 people. Regular reservations are $1,000-1,500.
- Find 1-2 seed attendees from your network — response rates are higher when there are names on the invite.

**Outreach:**
- Use Smartlead (email) and HeyReach (LinkedIn) for invites.
- Simple 2-3 step sequences, 150-200 characters each.
- When someone accepts, use Sales Navigator to find 3 people they're connected to and ask them to forward the invite.

**Running the dinner:**
- Print 4x6 placecards with how people are connected and what they have in common.
- Send a "fun facts" email 3 days before ��� if they respond, 90% chance they show up.
- Bring swag/gifts.

**After:**
- Build custom demos for each attendee. You've earned trust — now you can pitch.
- For non-prospects, ask for referrals. They'll often return the favor.
- For every 1 person who attends, ~7 say "would love to come next time." Keep them on a list — these get 20-30% response rates on the next invite.

## Why dinners work

People say yes because:
- **WFH fatigue** — tired of Zoom, want to meet people IRL
- **FOMO** — want to know what peers are actually doing (not just LinkedIn bragging)
- **Career networking** — feel they should be doing more
- **The restaurant** — sometimes they just want to try a place they can't get into

The CIO or VP of Marketing isn't interested in your demo (yet), but will happily enjoy a restaurant she can't find the time to get to.

## Skill structure

This skill follows the [Claude Code skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices):

```
dinner-play/
├── SKILL.md                  # Main instructions (~200 lines, loaded when triggered)
├── primitives-reference.md   # API patterns for all 7 services (loaded as needed)
├── tal-building.md           # TAL discovery approaches A/B/C + validation (loaded as needed)
└── starter-script.md         # Complete Python pipeline script (loaded as needed)
```

**Design decisions:**
- **SKILL.md under 500 lines** — keeps context budget low. Claude reads reference files only when needed.
- **Third-person description** — required for skill discovery ("Runs end-to-end..." not "I run...")
- **Progress checklist** — Claude copies it and checks off steps as it goes.
- **Self-contained code snippets** — every API pattern is inline, no external script dependencies.
- **Conditional workflow** — tests Approach A first, recommends C only if noisy. Avoids offering too many options upfront.

**Customizing for your company:**
1. Change the `name` field to match your company (lowercase, hyphens only)
2. Adjust `person_titles` and `person_seniorities` defaults for your ICP
3. Modify restaurant criteria (price range, vibe) if your dinners are different
4. Add your own scoring logic in the starter script

## Built by

[CharacterQuilt](https://characterquilt.com) — YC P26. We build campaign infrastructure for B2B marketing teams.

Built with [Claude Code](https://claude.ai/code).
