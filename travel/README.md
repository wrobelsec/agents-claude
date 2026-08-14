# Travel research agents

Two subagents that do live travel research on a single track each and return **raw sourced findings as tables** — never a narrative, never an itinerary. Synthesis is the orchestrator's job; these gather.

> **Required by the [`travel-agent` skill](https://github.com/wrobelsec/agent-skills-claude).** That skill dispatches these agents by name for every research track. Without them installed it has nothing to fan out to. If you want the skill, install these first.

They also work standalone for any research where sourcing discipline matters more than speed.

## The two agents

| | `travel-researcher` | `travel-scout` |
|---|---|---|
| **Model** | `sonnet` | `haiku` |
| **Tracks** | Air · Lodging · Food · Corridor/day trips · Entry, health &amp; climate · Points &amp; discounts · Law, scams &amp; stability · Fare rules | Ground transport · Experiences &amp; festivals · Community sentiment · Language |
| **Why this model** | Fine print is where money is lost and trouble starts — fare conditions, award restrictions, vaccination rules, criminal law, whether a ticket strands you. A confidently wrong answer is far worse than a slow one. | Mostly retrieval and list-building against named sources. Fast, cheap fan-out is the right trade. |
| **Tools** | `WebSearch`, `WebFetch`, `Read`, `ToolSearch` | same |

Neither has write access. Research agents have no reason to edit files.

### Why food is on the expensive model

It looks like a list-building track and it isn't. Any track that emits **links, addresses, or identifiers at volume** fails by *fabricating*, not by being slow. On the run these agents were built from, two of three food agents on the cheap tier produced sourcing failures — one invented a table of URLs and reused a single venue ID across two different restaurants, another returned map pins in the wrong prefecture. The careful-tier re-run hit the same bot-blocked site and **reported the block honestly instead of filling the gap.** That difference is worth the cost.

## What they enforce

Both definitions carry the same standing rules, which is the entire point of them being agents rather than prompt text:

- **Live fetches only.** No price, schedule, or opening hour from memory. Every row carries a source URL and the date accessed.
- **Links you may emit.** Never print a URL you didn't fetch or see in a live search result. **Never construct a shortened link** — real ones are opaque random strings, so a readable one is invented by definition. Give a verified street address instead. Scan your own output for duplicate IDs before returning, because that pattern is the signature of invention.
- **`UNVERIFIED` is a valid answer and a guess never is.** A short honest table beats a long padded one.
- **A blocked-source ladder.** Four failure modes that look alike need opposite responses — bot protection that won't clear on retry, rate limiting caused by sibling agents, JS-rendered pages that need a browser rather than a fetcher, and sites that are simply broken. Diagnose before retrying.
- **Don't substitute a weaker source class** for the one that defines the track. If the forums are unreachable, a blog's summary of them is not a degraded sentiment sweep — it's a different and near-worthless one.
- **API-first for commodity facts** — geocoding, daylight, holidays, weather, FX all have free keyless endpoints that can't be misread or invented. The list is in each definition.
- **Budget triage.** Briefs carry a search budget and ranked priorities; spend top-down and report partial rather than shallow.
- **Both currencies, always**, all-in rather than headline.
- **Names and addresses returned verbatim**, so the orchestrator's verification pass can resolve them. A paraphrased venue name cannot be checked and gets dropped.

### Why this lives in the agent, not the prompt

On the originating run, this preamble was pasted into roughly **37 separate dispatches**. The rules were only ever as reliable as remembering to include them — and the one dispatch that shipped without them is the one that fabricated its sources and had to be discarded and re-run twice. Moving them into the agent definition makes omission impossible.

## Install

```bash
cp -r travel ~/.claude/agents/
```

Or clone the whole repo to `~/.claude/agents`. Then start Claude Code and confirm `travel-researcher` and `travel-scout` appear in the available agent list.

**Subdirectories work** — discovery recurses, so `~/.claude/agents/travel/travel-scout.md` resolves as `travel-scout` with the model taken from its frontmatter. Verified by dispatch. If for some reason yours don't appear, move the two `.md` files up one level to `~/.claude/agents/` and restart; the definitions are unaffected by location.

## Requirements

- **Claude Code** with subagent support (`Agent` tool).
- **`WebSearch` and `WebFetch`** available in the session. The agents load these themselves via `ToolSearch` on start.
- **Browser tools** are optional but useful — JS-rendered pages (many booking engines, some official sites) return nothing to a plain fetcher.
- **No API keys required.** The keyless endpoints the agents lean on need no signup. Keyed APIs are entirely optional — see below.

---

## Optional: API keys

The agents work with no keys at all. Adding them fixes two specific classes of failure that scraping cannot: **flight fares that never resolve** because meta-search refuses to return multi-city results, and **stale venue data** — hours, closure days, permanently-closed businesses — which is the commonest way a food or excursion recommendation goes wrong.

Four are country-agnostic. Destination-specific ones (national transit programmes, tourism boards) are discovered per trip by the skill and don't belong in permanent config.

| API | Fixes | Free tier | Friction |
|---|---|---|---|
| **Amadeus Self-Service** | Flight and hotel offers — the open-jaw fare problem | Yes, test environment | **Test-tier data is partly cached and not guaranteed live.** Good for structure, confirm the number with the carrier |
| **Google Places (New)** | Venue hours, closure days, `business_status`, verified addresses, ratings with counts | Monthly credit | **Requires billing enabled on the Cloud project even to use the free credit** — a card on file |
| **Geoapify** *or* **LocationIQ** | Geocoding at ~thousands/day instead of ~1/sec | Yes | None meaningful |
| **Sherpa** | Entry and visa requirements as structured data | Developer tier, on request | Cross-check only — never the authority over a government page |

### The agents never hold a key

**The orchestrator makes every keyed call and passes results down.** Agents have no shell and no credentials, which is deliberate: a key passed into a subagent prompt lands in the session transcript, and there is no reason to put it there when the orchestrator can call on their behalf.

This runs in two phases, and the second is the more valuable one:

**Before dispatch — pre-fetch what is knowable in advance.** Fares for the routes in scope, entry requirements for the passport-and-destination pairs, coordinates for the cities and neighbourhoods, holidays, climate normals, the dated FX rate. Results go into the briefs with any caveat attached — free-tier flight data that is partly cached answers the structural question but not the exact fare, and an agent not told that will quote it as gospel.

**After return — verify what the agents found.** A restaurant discovered mid-research can't be pre-fetched, so it gets checked at synthesis: geocode every coordinate, run `business_status` on every venue, confirm hours and closure days. Doing it here rather than inside each agent is better on two counts — one consistent standard instead of twelve, and it is **the point where a fabricated row actually gets caught**, because a venue name that won't resolve is the tell.

The agents are briefed to cooperate with this: return names and addresses **exactly as the source gives them** so a lookup can resolve them, and never invent a coordinate or an opening time, since those are precisely the fields that get machine-checked.

### Setup

Get the credentials — steps are as documented by each provider; none of these has been tested here, since this repo ships no keys:

- **Amadeus** — register at `developers.amadeus.com`, create an app in the Self-Service workspace, copy the API Key and API Secret. Calls go to `test.api.amadeus.com`; production is a separate application.
- **Google Places** — at `console.cloud.google.com`: create or pick a project, enable **Places API (New)**, **enable billing**, then Credentials → Create API key. Restrict the key to the Places API.
- **Geoapify** — sign up at `myprojects.geoapify.com`, create a project, copy the key. (Or LocationIQ at `locationiq.com`.)
- **Sherpa** — request developer access at `developers.joinsherpa.com`.

Then add them to the `env` block of `~/.claude/settings.json`:

```json
{
  "env": {
    "AMADEUS_CLIENT_ID": "your-key",
    "AMADEUS_CLIENT_SECRET": "your-secret",
    "GOOGLE_PLACES_API_KEY": "your-key",
    "GEOAPIFY_API_KEY": "your-key",
    "SHERPA_API_KEY": "your-key"
  }
}
```

**That file is plain text on disk.** Same caveat as any local credential store. Set only the ones you want; every variable is independently optional.

### Verifying

Two checks worth keeping, because each maps to a real failure these fix:

- **Amadeus** — price a route you already know the answer to and compare. That tells you how far test-tier data sits from reality, which is the number you need before trusting it.
- **Places** — look up a business you know has closed and confirm `business_status` reports `CLOSED_PERMANENTLY`. That is the exact error this exists to prevent.

**Degradation is automatic.** A missing key is a quality reduction, never an error — the agent falls back to normal research and says in its findings that it did.

## Usage

Dispatch by name, in the background, one track per agent:

```
Agent(subagent_type: "travel-researcher", description: "Air: IAD→Tokyo", prompt: "<brief>")
Agent(subagent_type: "travel-scout",      description: "Ground transport",  prompt: "<brief>")
```

The model comes from the definition — don't pass `model` on the call.

### Usage tips

- **Cap concurrency at 4–6 agents.** The `WebSearch` budget is *shared across sibling agents*, and so is the session limit. A seventeen-agent fan-out exhausted the shared pool in about four minutes and lost fourteen of them. Dispatch in waves.
- **Give every brief an explicit search budget and a numbered priority list.** Agents respect a stated cap and invent their own when none is given.
- **Name the facts each agent owns**, where two tracks might both touch one. Without that, both research it shallowly and neither resolves it.
- **Expect `UNVERIFIED` cells and don't treat them as failure.** They mark where a source was blocked, which is information the reader needs.

## Related

- **[agent-skills-claude](https://github.com/wrobelsec/agent-skills-claude)** — the `travel-agent` skill these were built for.
- **[agents-claude](https://github.com/wrobelsec/agents-claude)** — this repo's root.
