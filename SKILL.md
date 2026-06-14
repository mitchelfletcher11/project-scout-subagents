---
name: project-scout-subagents
description: >
  Parallel subagent edition of project-scout. Use when the user wants to
  discover the latest resources before starting a project, wants to know
  what's new in the Claude/AI ecosystem, or asks about trending repos, new
  MCP servers, Claude Code plugins, project templates, or CLI tools. Runs all
  search sections in two rounds of parallel subagents instead of sequentially.
  Accepts an optional use-case hint (e.g. /project-scout-subagents HTML slides)
  but always conducts a contextual interview first.
allowed-tools: WebFetch WebSearch Bash Read Write Agent
arguments: [usecase]
---

# Project Scout (Subagent Edition)

You are helping the user discover the best resources before starting a new project. Work through the phases below in order.

---

## Phase 0 — Anthropic Skills Sync (always run first, silently)

Before anything else, check whether Anthropic has published new skills since the last run.

1. Fetch the current skill list from GitHub (authenticated):
   - Bash: `curl -sf -H "Authorization: Bearer $(cat ~/.claude/github-token 2>/dev/null)" "https://api.github.com/repos/anthropics/skills/contents/skills"` — parse the JSON array; extract the `name` field from each entry

2. Read the local snapshot:
   - Bash: `cat ~/.claude/anthropic-skills-snapshot.json 2>/dev/null || echo '{"lastChecked":"never","skills":[]}'`

3. Compare: find any skill names present in the GitHub list but NOT in the snapshot's `skills` array.

4. If new skills are found:
   - For each new skill, download its SKILL.md:
     `Bash: curl -sf "https://raw.githubusercontent.com/anthropics/skills/main/skills/{skill-name}/SKILL.md" -o ~/.claude/skills/{skill-name}/SKILL.md`
   - Update the snapshot file at `~/.claude/anthropic-skills-snapshot.json` with today's date and the full new skill list
   - **Notify the user** before Phase 1 with a banner:

```
🆕 New Anthropic skills detected since last project-scout run ({lastChecked}):
  • {skill-name} — {one-line description from its SKILL.md}
  ...
These have been installed globally to ~/.claude/skills/ and are now available.
```

5. If no new skills: skip the banner entirely and proceed silently to Phase 1.

---

## Phase 1 — Qualification Interview

Before running any searches, ask the user all of these questions **in a single message**. Do not ask technical questions (language, framework, open-source preference, etc.) — the user provides domain and goal context only; you derive technical terms from their answers.

If `$ARGUMENTS` was provided, use it to pre-fill question 1 as a starting point and ask the user to confirm or expand.

Ask:

1. **What is the main goal of this project?** What problem does it solve, or what does it create?
2. **Who is it for?** Describe the intended audience — yourself, a specific team, end-users, conference attendees, clients, etc.
3. **What does a successful outcome look like?** For example: a working demo, a polished shareable deliverable, an internal tool, a one-off script, something deployed publicly.
4. **In what context will it be used?** For example: live at an event, in a browser, shared via link, run locally, embedded in another product.
5. **What must it be capable of doing?** Describe required capabilities in plain language — e.g. "must look polished", "needs to be interactive", "must work offline", "needs to display data visually", "needs to pull in live information".
6. **Are there any constraints?** For example: must use a specific language or runtime, must run on specific hardware or architecture (ARM64, Raspberry Pi, 32-bit), limited RAM or storage, must avoid a build step, budget limits, must deploy on a specific platform, previous approach that failed and why.

After receiving answers:
- Synthesize a one-sentence description of what the user is building
- Derive three primary search terms from their answers — do not include language, framework, or platform names; do not ask the user for these:
  - `[DOMAIN]` — the unified domain label (e.g. `food-events`, `video-transcription`) — one concept, used for registry and API discovery
  - `[USECASE]` — the primary search angle within that domain (e.g. `food-event-discovery`, `speech-to-text-pipeline`)
  - `[USECASE2]` — a broader adjacent variant (e.g. `event-aggregation`, `audio-transcription`)
- Also extract three geographic and platform variables from the answers. These drive §10 (Geographic and Platform-Specific Sources), which runs in Round 2 alongside the other agents:
  - `[GEOGRAPHY]` — named country, region, or city if the user specified one (e.g. `Germany/NRW/Düsseldorf`, `Netherlands/Amsterdam`). Set to `None` if no specific geography mentioned.
  - `[TARGET_LANGUAGE]` — primary language of the target market (e.g. `German`, `French`, `Dutch`). Set to `English` if not specified or if English is the target.
  - `[CORE_PLATFORM]` — named platform that is central to the business or monitoring target (e.g. `Instagram`, `TikTok`, `Google Maps`). Set to `None` if no specific platform mentioned.
  - **Agent K (§10) runs whenever `[GEOGRAPHY] != None` OR `[CORE_PLATFORM] != None`.** Do not skip §10 when either variable is present — the English-language search terms cannot surface geographic or platform-specific sources on their own.
- **Confirm direction before searching** — show the user in a single message:
  - The one-sentence goal
  - All three derived search terms and what each will be used for ([DOMAIN] for registry and API discovery; [USECASE] and [USECASE2] for GitHub, community, package, and blog searches)
  - The three geographic/platform variables and whether §10 will run (state explicitly: "§10 Geographic Sources will run because [GEOGRAPHY] = X" or "§10 will not run — no geography or platform specified")
  - Whether the weighting between [USECASE] and [USECASE2] matches their intent (flag if one term will dominate results for the less important part of the project)
  - Ask: "Does this focus look right, or do you want to adjust before we search?"
- Wait for confirmation. If the user redirects, revise the terms and confirm again. Do not proceed to Phase 2 until confirmed.

---

## Phase 1.5 — Real-World Enumeration (to saturation)

**Purpose.** The point of the scout is to map the real world *before* designing our own build. Before the resource rounds (Phase 3) or any recommendation (Phase 4), exhaustively enumerate the real-world space: every existing solution to this problem, the use-cases practitioners actually use, and every instance of the core domain entity the project is about. **Enumeration comes first; synthesis and any "our build" recommendation come last and must reason against this inventory.** What already exists in the real world outranks what we could build — surface it first.

**Search ≠ enumeration.** Running a fixed set of queries and reporting hits is *sampling*, and it systematically under-covers (this is the failure mode §10 fell into when it ran only its predefined queries). Enumeration uses a different stopping rule: keep expanding categories and adjacent terms until a full pass surfaces nothing new. Stop on **saturation**, never on "I ran the planned queries."

**How to run it (subagent edition).** Saturation is iterative — each pass depends on what the previous one surfaced — so it does **not** parallelize cleanly across one-shot agents. Run it as a **single dedicated enumeration agent instructed to loop internally to saturation**, spawned and completed **before** the Round 1/Round 2 resource agents. Its classified inventory is then passed as context into the geographic (§10), community (§3), and resource agents so they extend it rather than rediscover it.

The enumeration agent's instructions:

**Step 1 — Define the enumeration target and scope boundary.** From Phase 1, name the class of thing to enumerate in one sentence (e.g. "platforms where people post local-service requests in Germany"). Write an explicit **in-scope test** (what counts as one instance) and a short **exclusion list**. Without this boundary the loop cannot terminate.

**Step 2 — Derive the category space.** Break the target into its natural sub-types (e.g. general classifieds, trade/Handwerker boards, freelance/project boards, care/tutoring, informal digital locations). Treat the list as provisional — it grows during the loop.

**Step 3 — Saturation loop.** For each category, search in **every relevant language** (`[TARGET_LANGUAGE]` + English at minimum). For every instance found, record and **classify it as you go** on the dimensions that decide usability for the goal — derived from Phase 1 (e.g. accessible-vs-closed, has the needed data, free-vs-paid, legal/ToS risk, hardware/runtime fit). After each pass, **expand** with any new adjacent categories the results revealed, then run again. Continue until a full pass yields **no new instances and no new categories**.

**Step 4 — Return the classified inventory.** (a) Full inventory — every instance, its category, its usability classification; (b) ranked shortlist of the highest-value *usable* instances; (c) explicit "excluded / not usable — why" list; (d) an explicit **saturation status** (`reached` only if the agent's last full pass surfaced nothing new) and a list of **categories still likely incomplete** with reasons. This is a **required input to Phase 4**.

**Verified saturation — enforced by the orchestrator, not the agent.** A single enumeration agent tends to *declare* saturation while a whole category is still unmined (e.g. "largely saturated" while an entire syndication network was never searched). So the orchestrator must verify, not trust: when the enumeration agent returns status `not reached` (or names incomplete categories), **spawn a follow-up enumeration agent scoped to exactly those categories**, instructed to return only NEW instances + its own saturation status. Repeat until a follow-up pass returns **zero new instances**. Only then is the inventory saturated. If you stop before that for time/scope reasons, state plainly what remains unmined — never relabel a partial inventory as complete.

**Guard.** Keep it domain-scoped (Step 1's boundary) so it terminates; saturation is the bound, not a query count. If the target is a pure build-tooling question with no real-world entity or existing-solution space (e.g. "an HTML slide template"), state that in one line and skip to Phase 2 — but do not skip merely because enumeration is laborious.

---

## Phase 2 — Credential Setup (silent, runs before spawning subagents)

Load stored API keys via Bash before spawning any subagents:

```bash
GITHUB_TOKEN=$(cat ~/.claude/github-token 2>/dev/null)
LIBRARIES_KEY=$(cat ~/.claude/libraries-io-key 2>/dev/null)
SO_KEY=$(cat ~/.claude/stackoverflow-key 2>/dev/null)
```

These three keys are **optional** — the skill works without them, just with stricter rate limits
and a couple of data sources skipped. Each source falls back gracefully when its key is absent.

**Surface the option once (zero-state rule — never silently no-op).** On a run where one or more
key files are missing, tell the user *once* that they exist and are optional, with where to get
each, then continue without blocking:
- `~/.claude/github-token` — higher GitHub API limits + trending-repo data. Create at <https://github.com/settings/tokens> (no scopes needed for public read; `public_repo` is plenty).
- `~/.claude/libraries-io-key` — package health/version data across ecosystems. Free key at <https://libraries.io/account>.
- `~/.claude/stackoverflow-key` — higher Stack Exchange API quota. Free key at <https://stackapps.com/apps/oauth/register>.

If the user wants any of them, write the pasted value to the named file with `chmod 600`. If they
decline (or don't respond), proceed — do not ask again on subsequent runs within the session.

Also pre-scan the locally installed skills directory and extract each skill's name and description. This runs in the orchestrator so Agent A does not need to read 30+ SKILL.md files inside the subagent:

```bash
LOCAL_SKILLS_CONTEXT=$(for s in $(ls ~/.claude/skills/ 2>/dev/null); do
  desc=$(grep -m1 "^description:" ~/.claude/skills/$s/SKILL.md 2>/dev/null | sed 's/^description: *//')
  [ -n "$desc" ] && echo "- $s: $desc"
done)
```

Pass `LOCAL_SKILLS_CONTEXT` as additional context in the Agent A prompt. Agent A should use this list for §4 Step 1 (local skills assessment) and skip re-scanning the directory.

---

## Phase 2.5 — Community Detection + Blog Discovery (executed by Agent C in Round 1)

The instructions in this section are executed by Agent C in Phase 3 Round 1. They are preserved here so the subagent can read and follow them.

---

### Community Detection

Do not assume which platforms the community for this domain uses. Discover it fresh each run — no predetermined platform list.

Run these discovery queries in parallel:

```bash
# 1. Primary community platform
WebSearch "[USECASE] community forum practitioners ask questions [CURRENT_YEAR]"

# 2. Dedicated Q&A community
WebSearch "[USECASE] community Q&A help discussion [CURRENT_YEAR]"

# 3. Social and informal communities
WebSearch "[USECASE] reddit discord slack linkedin community practitioners [CURRENT_YEAR]"
```

From the results, extract every platform, forum, community, and discussion venue that surfaces. Record as **`DISCOVERED_PLATFORMS`** — a list of platform names, URLs, and types. Do not limit to known platform types. If an unfamiliar platform appears repeatedly in results, include it.

For each platform found, note its type — this determines search priority in §3:
- **Vendor community** (a platform's own official forum) — highest signal; practitioners with deep product knowledge post here
- **Dedicated Q&A site** (a Stack Exchange site or equivalent dedicated to this domain) — high signal; voted answers surface best solutions
- **Community forum or subreddit** — high signal for honest practitioner opinions and real-world experience
- **Social platform** (X.com, LinkedIn, Discord, Slack archives, YouTube) — variable signal; useful for recent announcements and practitioner demonstrations
- **Other** — record whatever was found; assess signal when reading results

Record `DISCOVERED_PLATFORMS` for use in §3. Do not filter or discard any platform at this stage.

---

### Blog Discovery

Run in parallel with Community Detection.

**Step 1 — Find curated lists:**

```bash
WebSearch "top [USECASE] blogs [CURRENT_YEAR]"
```

Identify the top 3 curated list articles in the results (e.g. "Best DevOps Blogs 2025", "Top Python Blogs 2025"). These lists are maintained by the community and give 10–20 filtered blogs in a single fetch — far more efficient than discovering individual blogs through search.

WebFetch each of the top 3 curated list URLs:
```
WebFetch [curated list URL 1]
WebFetch [curated list URL 2]
WebFetch [curated list URL 3]
```

Extract: blog name, URL, specialization. Filter to blogs relevant to the specific domain and use case from Phase 1. Prioritise blogs that cover release notes or breaking changes — these are the most valuable for practitioners keeping up with a platform.

**Step 2 — Direct practitioner search:**

```bash
WebSearch "[USECASE] blog best practices [CURRENT_YEAR]"
```

This surfaces individual posts from smaller blogs that may not appear on curated lists. Extract the domain of each result (not the post URL, the root domain) as a candidate blog.

Record the combined, deduplicated list as **`BLOG_LIST`** — used in §8.

---

### Structured Data API Discovery

Run in parallel with Community Detection and Blog Discovery.

Search for domain-specific structured data APIs beyond the hardcoded baseline (GitHub API, PyPI, npm, Libraries.io):

```bash
WebSearch "authoritative API [DOMAIN] registry programmatic data [CURRENT_YEAR]"
WebSearch "[DOMAIN] developer API structured data registry packages [CURRENT_YEAR]"
```

(`[DOMAIN]` is the unified domain label derived in Phase 1 — e.g. `food-events`, `video-transcription`.)

For each discovered API candidate:
1. **Assess:** does it provide structured, queryable data — packages, releases, downloads, health scores, version history, or similar metrics?
2. **If yes:** fetch its documentation to extract the base URL, query format, and authentication requirements
3. **Record** as `DISCOVERED_APIS` — name, base URL, query format, authentication method, what data it provides

---

## Phase 3 — Parallel Execution via Subagents

Execute all search sections using the Agent tool in two rounds. Spawn all agents within a round in a single message — this is what enables true parallelism.

### Context to pass to every subagent

Every subagent prompt must include:
- **Use case summary:** the one-sentence summary from Phase 1
- **Search terms:** [USECASE] and [USECASE2]
- **Domain:** [DOMAIN]
- **Credentials:** `GITHUB_TOKEN=$(cat ~/.claude/github-token 2>/dev/null)` · `LIBRARIES_KEY=$(cat ~/.claude/libraries-io-key 2>/dev/null)` · `SO_KEY=$(cat ~/.claude/stackoverflow-key 2>/dev/null)`
- **Instruction:** "Read `~/.claude/skills/project-scout-subagents/SKILL.md`. Execute the section(s) specified below only, following the instructions there exactly. Substitute [USECASE], [USECASE2], [DOMAIN], and any other variables with the values provided in this prompt. Return all findings in full — do not summarise or truncate."

---

### Round 1 — spawn all three in a single Agent tool message

**Agent A — Claude Code Plugins + Skills (§4)**
Execute §4 (Claude Code Plugins & Skills) from the SKILL.md.
**LOCAL_SKILLS_CONTEXT (pre-scanned in Phase 2):** [pass LOCAL_SKILLS_CONTEXT here]. Use this for §4 Step 1 — do not re-scan `~/.claude/skills/`. Proceed directly to §4 Step 2 (online search).
Return: locally installed skills assessment + online plugins/skills table.

**Agent B — GitHub Repos (§1)**
Execute §1 (GitHub Repos) from the SKILL.md in full — including README reads for the top 5 repos and Issues checks for the top 5 repos.
Return:
1. Full repos table (all results, sorted by stars)
2. README notes per repo (architecture decisions, key dependencies, gotchas, performance notes)
3. Issues findings per repo (current breakage, dependency changes, health signals)
4. **[KEY-PACKAGES]** — the 3–5 most critical packages identified from repo analysis, each tagged with its ecosystem (e.g. `scrapling [Python]`, `n8n-nodes-base [npm]`, `axios [npm]`). This list is required for Round 2.

**Agent C — Community Detection + Blog Discovery (Phase 2.5)**
Execute Phase 2.5 from the SKILL.md — Community Detection, Blog Discovery, and Structured Data API Discovery — in full.
Return:
1. **DISCOVERED_PLATFORMS** — platform names, URLs, and types
2. **BLOG_LIST** — blog names and URLs
3. **DISCOVERED_APIS** — name, base URL, query format, authentication method

---

### Round 2 — after Round 1 completes, spawn all seven in a single Agent tool message

Collect [KEY-PACKAGES] from Agent B and DISCOVERED_PLATFORMS + BLOG_LIST + DISCOVERED_APIS from Agent C. Pass the relevant outputs as additional context in each agent prompt below.

**Agent D — Package Health (§2)**
Execute §2 (Package Health) using [KEY-PACKAGES] from Agent B.
Return: package health table.

**Agent E — Platform Release Currency (§9)**
Execute §9 (Platform Release Currency) using [KEY-PACKAGES] from Agent B and DISCOVERED_APIS from Agent C as additional registry sources.
Return: release currency findings per dependency, Known Breakage signals.

**Agent F — MCP Servers (§5)**
Execute §5 (MCP Servers & Connectors).
Return: runtime-usable and dev-time MCP server findings.

**Agent G — Templates (§6)**
Execute §6 (Project Templates & CLAUDE.md Patterns).
Return: template repos table.

**Agent H — CLI Tools (§7)**
Execute §7 (CLI Tools).
Return: CLI tools table with runtime/dev-time/build-time tags.

**Agent I — Community Insights (§3)**
Execute §3 (Community Insights) using DISCOVERED_PLATFORMS from Agent C.
Return: all relevant results per platform, Watch out for flags.

**Agent J — Blogs (§8)**
Execute §8 (Blogs) using BLOG_LIST from Agent C.
Return: blogs table and relevant recent posts found per blog.

**Agent K — Geographic and Platform-Specific Sources (§10)**
Execute §10 only if `[GEOGRAPHY] != None` OR `[CORE_PLATFORM] != None`. Pass [GEOGRAPHY], [TARGET_LANGUAGE], and [CORE_PLATFORM] as variables.
Return: geographic directories, local job boards, language-specific communities, platform-specific monitoring tools, and local trade publications — all assessed for ARM64/hardware compatibility and API availability.
If both variables are `None`: skip this agent entirely.

---

After both rounds complete, proceed to Phase 4 with all agent results in hand.

---

## Phase 3 — Sections

The sections below are the instructions each subagent reads and executes. Variable substitutions are provided in each subagent's prompt.

---

### 1. GitHub Repos (authenticated REST API — exact star counts, no rate limit)

Use Bash+curl to pass the Authorization header (WebFetch cannot set headers):

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]&sort=stars&order=desc&per_page=30"

curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE2]&sort=stars&order=desc&per_page=30"
```

Run both queries and merge results, deduplicating by repo name. Parse the JSON `items` array:
- `full_name` → repo name
- `stargazers_count` → exact star count
- `language` → primary language
- `description` → description
- `html_url` → URL
- `pushed_at` → last commit date — flag repos not updated in >12 months as potentially stale

Show all results sorted by stars descending. Flag anything with 1k+ stars prominently.

**After identifying the top 5 repos by stars, read their READMEs:**

```
WebFetch https://raw.githubusercontent.com/{owner}/{repo}/main/README.md
```

If `main` returns a 404, try `master`. From each README extract and record internally (for use in Phase 4 synthesis):
- Architecture decisions made (e.g. how they structured routes, what they used for async, how they called the API)
- Key dependencies actually used — note the ecosystem for each (`[Python]` or `[npm]`) so they can be correctly tagged in KEY-PACKAGES
- Any documented gotchas, known issues, or limitations sections
- Any performance notes or scaling warnings

Do not dump the full README into the output — use these notes only in Phase 4.

**After reading READMEs, check open Issues on the top 5 repos:**

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/issues?state=open&per_page=20&sort=updated"
```

Search issue titles and bodies for keywords related to the use case (platform names, feature areas, error types). Record internally:
- Any issues describing current breakage or known incompatibility
- Any issues flagging a dependency or API change that broke functionality
- Total open issue count and most recent activity as a health signal

Do not dump all issues into the output — use these notes only for the Known Breakage section in Phase 4.

**Derive KEY-PACKAGES:** From all dependencies identified across the top 5 READMEs, select the 3–5 most critical packages — the ones without which the core use case cannot function. Tag each with its ecosystem: `package-name [Python]` or `package-name [npm]`. Return these as `[KEY-PACKAGES]` for Round 2.

---

### 2. Package Health (PyPI + npm + Libraries.io)

For each package in [KEY-PACKAGES], check its registry based on the ecosystem tag:

**Python packages** (`[Python]` tag) — fetch from PyPI:
```
WebFetch https://pypi.org/pypi/{package}/json
```
Extract: `info.version`, `info.summary`, `urls[0].upload_time` (last release date). Flag packages with no release in >12 months.

**npm packages** (`[npm]` tag):
```
WebFetch https://registry.npmjs.org/{package}
```
Extract: `dist-tags.latest` (current version), `time[latest]` (last release date), `description`. Flag packages with no release in >12 months.

**Libraries.io SourceRank** — health score across all ecosystems (0–30 scale):
```bash
curl -s "https://libraries.io/api/search?q=[USECASE]&api_key=$LIBRARIES_KEY&per_page=30&sort=rank"
```
Extract per result: `name`, `rank` (SourceRank), `stars`, `latest_release_published_at`, `normalized_licenses`, `platform`.
- Rank ≥20: healthy and active
- Rank 10–19: acceptable
- Rank <10 or no release in >18 months: flag as risky

---

### 3. Community Insights (uses DISCOVERED_PLATFORMS from Phase 2.5)

Search every platform in `DISCOVERED_PLATFORMS`. Run all searches in parallel. Do not skip any platform — quality filtering happens when reading results, not before searching.

**For each platform in DISCOVERED_PLATFORMS:**

If the platform has a searchable domain:
```bash
WebSearch "[USECASE] site:[PLATFORM_DOMAIN]"
```

For dedicated Q&A sites on the Stack Exchange network, use the authenticated API for higher signal:
```bash
curl -s "https://api.stackexchange.com/2.3/search?order=desc&sort=votes&intitle=[USECASE]&site=[SITE_ID]&key=$SO_KEY&pagesize=5"
```

For platforms without a directly searchable domain (Discord, Slack archives, YouTube channels):
```bash
WebSearch "[USECASE] [PLATFORM_NAME] [CURRENT_YEAR]"
```

**From each platform, extract:**
- All relevant threads, posts, videos, or results — show title and URL
- Any result describing a known issue, limitation, or workaround — flag as "Watch out for"
- Any result where a practitioner describes a real-world solution — note as high-signal

**When presenting results:** vendor communities and dedicated Q&A sites first, then community forums, then social platforms. Within each tier, sort by recency. Omit a platform from the output only if the search returned genuinely zero relevant results.

---

### 4. Claude Code Plugins & Skills

**Step 1 — Scan locally installed skills:**

```bash
ls ~/.claude/skills/
```

For each directory found, read its SKILL.md:
```bash
cat ~/.claude/skills/{skill-name}/SKILL.md
```

Extract the `name` and `description` fields from the frontmatter. Assess relevance to the current use case from Phase 1. A skill is relevant if its description covers the same functional area, even partially. Record all relevant local skills — they are used in Phase 4 for comparison against online findings. Do not stop or narrow the search based on what is found here.

**Step 2 — Online search (always runs regardless of Step 1 findings):**

- WebFetch `https://raw.githubusercontent.com/anthropics/claude-plugins-official/main/registry.json` — official plugin registry; filter for entries relevant to the use case
- WebSearch `site:github.com anthropics claude plugins [USECASE]` — community plugins
- Report: name, star count if available, what capability it adds, install command

---

### 5. MCP Servers & Connectors (authenticated GitHub API for exact star counts)

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+mcp+server&sort=stars&order=desc&per_page=30"
```
- WebFetch `https://github.com/modelcontextprotocol/servers` — official MCP servers repo; note any matching the use case
- Collect all results — applicability filtering happens in Phase 4

---

### 6. Project Templates & CLAUDE.md Patterns

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+CLAUDE.md+in:path&sort=stars&order=desc&per_page=30"
```
- WebSearch `[USECASE] starter template boilerplate claude code github [CURRENT_YEAR]`
- Report: repo name, exact stars, stack, URL, notable CLAUDE.md patterns

---

### 7. CLI Tools

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+cli&sort=stars&order=desc&per_page=30"
```
- WebFetch `https://registry.npmjs.org/-/v1/search?text=[USECASE]+cli&size=25`
- WebFetch `https://pypi.org/search/?q=[USECASE]+cli`
- WebSearch `best cli tools [USECASE] [CURRENT_YEAR] github awesome`
- Report: tool name, exact stars or weekly downloads, what it does, install command

---

### 8. Blogs (uses `BLOG_LIST` from Phase 2.5)

For all blogs in `BLOG_LIST`, check for recent relevant content:

```bash
WebSearch "[BLOG_NAME] [USECASE]"
```

Or if the blog has a known URL pattern, WebFetch it directly.

Extract per blog: name, URL, specialization, most relevant recent post title if found. Flag blogs that cover release notes or breaking changes — these are the highest-value ongoing resource for practitioners keeping up with a platform.

Report in output as a table. Include the curated list source that surfaced them so the user can explore the full list beyond the top entries.

---

### 9. Platform Release Currency

For **each key dependency** identified in §2 — every library, framework, or platform the project relies on — run Steps 1–3 below separately.

The goal is not just "what's new" — it is "what changed recently that affects this use case, and what is currently broken."

**Step 1 — Determine the current version of each key dependency:**

For versioned platforms with named releases (WordPress, Shopify, etc.): identify the current release name and date from official docs or community sources.

For libraries and frameworks: check the latest version from the package registry or GitHub releases for each dependency:

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/releases?per_page=3"
```

Record the current version per dependency. Use these versions throughout the remaining search to tag guidance as current-era vs. potentially outdated.

**Step 2 — Find the best changelog source for each dependency:**

Official changelogs (GitHub releases, CHANGELOG.md, official docs) are the baseline for each. For commercial platforms, also search for the best third-party practitioner aggregator:

```bash
WebSearch "[DEPENDENCY NAME] release notes [CURRENT_YEAR] changelog"
WebSearch "best [DEPENDENCY NAME] release coverage blog [CURRENT_YEAR]"
```

Every major platform has a small number of practitioners who cover every release in depth — find them via the blog discovery step in Phase 2.5. For libraries without formal changelogs, the GitHub Issues tracker is often the most accurate release notes source. Check it specifically.

**Step 3 — Fetch the last 2–3 releases from the best source found for each dependency:**

WebFetch the changelog or aggregator URL per dependency. Extract and record internally:
- Breaking changes relevant to the use case
- New native capabilities that may replace custom solutions being planned
- Deprecated features the project might accidentally rely on

**Step 4 — For packages used with a development server (Flask, FastAPI, Django, Express, etc.):**

If a key library involves authentication, OAuth, or any flow that blocks waiting for a callback, check specifically for integration issues with the target development server:

```bash
WebSearch "[AUTH PACKAGE] [FRAMEWORK] development server reload [CURRENT_YEAR]"
WebSearch "[AUTH PACKAGE] [FRAMEWORK] blocking hot reload issue"
```

Common failure mode: OAuth flows that call `run_local_server()` or equivalent block the process. When a framework's `--reload` / watch mode restarts the process mid-flow, the token is never written and the flow silently fails. Flag any such findings in the Known Breakage section.

**Step 5 — For libraries that interact with live external services:**

If a key library scrapes or calls a live platform, check specifically:

```bash
WebSearch "[LIBRARY NAME] [PLATFORM NAME] broken [CURRENT_YEAR]"
WebSearch "[LIBRARY NAME] [PLATFORM NAME] not working issue"
```

Also check the library's GitHub Issues (already fetched in Section 1) for open issues mentioning the live platform.

Record all findings for Phase 4 synthesis and the Known Breakage output section.

---

### 10. Geographic and Platform-Specific Sources

**Trigger:** Run this section only when `[GEOGRAPHY] != None` OR `[CORE_PLATFORM] != None`. If both are `None`, skip entirely.

This section exists because §1–§9 use English-language search terms derived from the use case. Those terms cannot surface:
- Platforms and communities dominant in a non-English target geography
- Business directories and registries specific to the target country
- Job boards and professional networks where the target market operates
- Monitoring and analytics tools specific to a named platform ([CORE_PLATFORM])
- Trade publications and industry associations in the target geography

**Method — enumerate, do not sample.** The searches listed in each sub-section below are a *starting set*, not the whole job. Apply the Phase 1.5 saturation rule: keep expanding categories and adjacent terms (in `[TARGET_LANGUAGE]` and English) until a full pass surfaces no new sources, classifying each as you find it. Extend the Phase 1.5 inventory passed in as context — do not stop at the example queries, and do not rediscover what the enumeration agent already found.

Run all applicable sub-sections in parallel.

---

**Sub-section A — Geographic business directories and registries** (run if `[GEOGRAPHY] != None`)

Search for the primary business directories and official registries in `[GEOGRAPHY]`:

```bash
WebSearch "business directory [GEOGRAPHY] restaurant hotel API programmatic [CURRENT_YEAR]"
WebSearch "official company registry [GEOGRAPHY] API access [CURRENT_YEAR]"
WebSearch "[GEOGRAPHY] yellow pages business listings API scraper python [CURRENT_YEAR]"
```

For each directory or registry found:
1. Check whether an official API exists — fetch its developer docs if available
2. If no official API, check for Apify actors or Python scrapers on GitHub
3. Note: name, base URL, what data it provides (name, address, phone, website, social links), API availability, cost, ARM64 compatibility, GDPR status for the target country

---

**Sub-section B — Geographic job boards as intent signal sources** (run if `[GEOGRAPHY] != None`)

Job postings for marketing, social media, or content roles at businesses in the target geography are explicit demand signals. Search for the dominant job boards in the target country:

```bash
WebSearch "job board [GEOGRAPHY] API scraper python social media marketing [CURRENT_YEAR]"
WebSearch "largest job site [GEOGRAPHY] programmatic access [CURRENT_YEAR]"
WebSearch "[GEOGRAPHY] job postings API official government employment [CURRENT_YEAR]"
```

For each job board found:
1. Prioritise any official government/public job board first — these often have free public APIs
2. Check for Apify actors for private job boards
3. Note: name, URL, API availability, filter parameters (location radius, keyword, date), cost, ARM64 compatibility

---

**Sub-section C — Language-specific communities and social platforms** (run if `[TARGET_LANGUAGE] != English`)

Repeat the community discovery searches from Phase 2.5 in `[TARGET_LANGUAGE]`:

```bash
WebSearch "[USECASE] community forum practitioners [TARGET_LANGUAGE] [CURRENT_YEAR]"
WebSearch "[USECASE] hilfe fragen diskussion [TARGET_LANGUAGE] [CURRENT_YEAR]"  # adapt verb to language
WebSearch "[USECASE] [GEOGRAPHY] reddit forum social media community [CURRENT_YEAR]"
```

Also search for Reddit communities in the target language:
```bash
WebSearch "subreddit [TARGET_LANGUAGE] [GEOGRAPHY] [USECASE] community"
```

For each platform or community found, note type (forum, subreddit, professional network, trade group), size if available, and whether it is searchable via API or requires manual monitoring.

**Blog discovery in `[TARGET_LANGUAGE]`:**

Run blog discovery searches in the target language in parallel with the community searches above:

```bash
WebSearch "[USECASE] blog [TARGET_LANGUAGE] [CURRENT_YEAR]"
WebSearch "beste [USECASE] blogs [TARGET_LANGUAGE] [CURRENT_YEAR]"
WebSearch "[USECASE] Ratgeber Tipps [GEOGRAPHY] [CURRENT_YEAR]"  # adapt nouns to language
```

For each blog or publication found, record: name, URL, language, specialization, and whether it covers pain-point content (e.g. why a business needs a social media presence) that could inform outreach messaging. Add to `BLOG_LIST` from Phase 2.5 — these complement the English-language list, not replace it.

---

**Sub-section D — Platform-specific monitoring tools** (run if `[CORE_PLATFORM] != None`)

The English-language search terms do not surface tools designed specifically for `[CORE_PLATFORM]`. Run targeted searches:

```bash
WebSearch "[CORE_PLATFORM] account monitoring inactive dormant detection tool [CURRENT_YEAR]"
WebSearch "[CORE_PLATFORM] geographic business presence audit API python [CURRENT_YEAR]"
WebSearch "[CORE_PLATFORM] private API python library [CURRENT_YEAR] github"
WebSearch "detect [CORE_PLATFORM] missing presence local business tool [CURRENT_YEAR]"
WebSearch site:github.com "[CORE_PLATFORM] monitoring python"
```

For each tool found:
1. Assess: can it detect dormant or absent accounts for businesses in a geographic area?
2. Check: official API vs private API vs web scraping — note ToS risk for each
3. Note: ARM64 compatibility, installation method, rate limits, ban risk

---

**Sub-section E — Local trade publications and industry associations** (run if `[GEOGRAPHY] != None`)

```bash
WebSearch "[GEOGRAPHY] [DOMAIN] trade publication magazine online [CURRENT_YEAR]"
WebSearch "[GEOGRAPHY] hospitality restaurant hotel industry association [CURRENT_YEAR]"
WebSearch "[GEOGRAPHY] [DOMAIN] news website trade [TARGET_LANGUAGE] [CURRENT_YEAR]"
```

For each publication or association found:
1. Check whether it covers the target use case (marketing, social media, digital presence)
2. Note: whether articles are in the target language, whether the site is scrapable (RSS feed, HTML), whether it publishes pain-point content that could inform outreach messaging
3. Flag any association that runs events where target businesses self-identify as needing help

---

**Sub-section F — Language-specific and geography-specific GitHub repositories** (run if `[GEOGRAPHY] != None` OR `[TARGET_LANGUAGE] != English`)

§1 sorts global results by stars descending. Regional and language-specific tools rarely accumulate global stars even when widely used within the target geography. Run targeted searches that surface these:

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+[GEOGRAPHY]&sort=updated&order=desc&per_page=20"

curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+[GEOGRAPHY]&sort=stars&order=desc&per_page=20"
```

Also search the web for local developer projects:
```bash
WebSearch "github [USECASE] [GEOGRAPHY] open source [CURRENT_YEAR]"
WebSearch "[USECASE] [TARGET_LANGUAGE] open source projekt github [CURRENT_YEAR]"
```

For each repo found that does NOT already appear in §1's results:
- Record: full_name, stars, last push date, description
- Flag if it solves a problem specific to the target geography (e.g. a German job board API client, a national business registry scraper)
- Note: ARM64 compatibility and whether a pip or npm install is available

---

**Return from §10:**

1. **Geographic directories table**: name, API availability, cost, what data it provides, ARM64 status, GDPR note
2. **Job boards table**: name, access method, filter parameters available, cost
3. **Language-specific communities**: platform, language, size, access method
4. **Platform-specific monitoring tools**: tool name, what it detects, API type, ToS risk, ARM64 status
5. **Trade publications and associations**: name, URL, language, relevance to use case, scrapability
6. **Summary: best new signal sources found** — ranked by signal quality and cost, with recommended polling frequency

---

## Phase 4 — Applicability Filter + Synthesis (runs after all Round 2 agents complete)

This phase reasons over all gathered results before producing output. Do not skip it.

**Consume the Phase 1.5 enumeration first.** Before any "what should we build" reasoning, present the real-world inventory from Phase 1.5: what already exists, what practitioners actually use, and the ranked usable instances. The Build Spectrum and the recommendation are framed as a response to that inventory — "given everything that already exists, here is whether and how to build" — never as a build designed in isolation. If the enumeration surfaced an existing solution that already meets the goal, say so plainly before proposing a custom build.

### Runtime Architecture Derivation

Before filtering or synthesising anything, derive the runtime architecture from what the search actually found. Identify the language and deployment pattern that dominates the top GitHub repos and community results. For example: if the top repos are Python Flask apps, the runtime is "local Python server"; if they are n8n workflow files, the runtime is "self-hosted Node workflow engine"; if results are evenly split, note that explicitly. Record this as `[DERIVED_ARCHITECTURE]` and use it throughout the rest of Phase 4. Do not carry forward any architecture assumed or noted in Phase 1.

### Applicability Filter

Using `[DERIVED_ARCHITECTURE]`, evaluate every MCP server, plugin, and CLI tool found:

**MCP servers** — ask for each: "Can this server be called at runtime from inside a [Flask app / Node server / browser / etc.]?" MCP servers are invoked by Claude Code during development sessions — they are NOT importable libraries. Tag each result:
- `dev-time` — useful during Claude Code development sessions only (most MCP servers fall here)
- `runtime` — exposes an HTTP or SDK interface the app itself can call

Only show `runtime`-tagged MCP servers in the MCP section of output. Move `dev-time` ones to a separate "Useful during development" note, or omit if none are relevant.

**CLI tools** — tag each:
- `build-time` — runs during setup/installation only
- `dev-time` — useful during development but not shipped
- `runtime` — called by the app itself (e.g. `ffmpeg` as a subprocess)

**Plugins** — same logic: is this callable from within the app, or only from a Claude Code session?

### Known Breakage Compilation

Collect all breakage signals found across Section 1 (GitHub Issues from Agent B), Section 9 (Platform Release Currency from Agent E), and Section 3 (Community Insights from Agent I). For each signal:
- Name the specific library or platform affected
- Describe what is broken or changed in plain language
- Link to the source (GitHub issue, release note, community thread)

Only include confirmed breakage or instability — not general limitations or design tradeoffs.

---

### Hardware and Architecture Compatibility Check

If Phase 1 mentioned a specific hardware architecture or resource constraint (ARM64, Raspberry Pi, x86_32, limited RAM, limited storage), run a targeted compatibility check for each top recommended package before writing Design Decisions:

```bash
WebSearch "[PACKAGE] ARM64" (or relevant architecture)
WebSearch "[PACKAGE] low memory out of memory"
```

Flag any package with known issues on the stated architecture or under the stated resource constraints.

---

### Local vs Online Skills Comparison

If Section 4 found both locally installed skills AND online skills/plugins covering the same functional area, produce a comparison for each overlapping pair:

- **What the local skill does** — based on its SKILL.md description
- **What the online alternative does** — based on registry or repo description
- **Pros of the local skill** — already installed, known behavior, tailored to this project's existing patterns
- **Cons of the local skill** — may be narrower in scope, not updated, or missing capabilities the online version has
- **Recommendation** — which to use and why, or whether combining both makes sense

If there is no overlap between local and online findings, omit the comparison entirely.

### Orchestration Layer Evaluation

Before committing to a hand-rolled implementation, evaluate whether a dedicated tool would handle the project's orchestration needs better than custom code.

From the Phase 1 answers, identify what type of orchestration this project requires — for example: event-triggered workflow, scheduled batch pipeline, API-to-API automation, conversational bot routing, data transformation pipeline. Do not assume a fixed category; derive it from what the project actually does.

Then search:

```bash
WebSearch "[orchestration type] tools [CURRENT_YEAR]"
WebSearch "[orchestration type] open source self-hosted alternatives [CURRENT_YEAR]"
```

For each candidate tool surfaced:
- Assess whether it covers the specific orchestration needs of this project better than custom code would
- Check compatibility with `[DERIVED_ARCHITECTURE]` and any hardware constraints stated in Phase 1 (ARM64, RAM limits, etc.)
- Note the adoption cost: does it require a separate service to run, a different language or runtime, a learning curve?

If a dedicated tool would handle the orchestration layer better than custom code for this project, include it as a Design Decision with a concrete recommendation. If custom code is still the right call, state that explicitly and briefly — do not silently skip the evaluation.

---

### README Pattern Extraction

Using the README notes gathered by Agent B, identify 2–3 patterns from real implementations that are directly portable to this project. Note gotchas that are not obvious from the library documentation.

### Tech Stack Recommendation

> **Scope note:** This section compares specific frameworks and runtimes within the custom code path (e.g. Python + Scrapling vs. Node.js + Playwright). The Build Spectrum section handles the higher-level question of whether custom code is the right approach at all — do not repeat that analysis here.

Based on all gathered findings — GitHub repo languages and stars, community platform dominance, package health, orchestration layer evaluation, and any constraints stated in Phase 1 question 6 (hardware, language, platform, budget) — identify 2–3 candidate approaches. If Phase 1 question 6 specified a required language or platform, exclude approaches that violate it before presenting candidates. For each remaining candidate:
- Name the approach (e.g. Python with Scrapling + Twilio, n8n self-hosted, Node.js with Playwright)
- State what the findings support or argue against it — reference specific repos, star counts, community signal, or package health scores
- Note any known breakage or compatibility issues for this approach given the user's stated constraints

Then state a clear recommendation: which approach to use and why, grounded in specific findings. This becomes the "Stack" passed to Phase 5. Do not carry forward any tech direction assumed before searching — the recommendation must emerge from Phase 3–4 findings only.

### Reusable internal method skills

If the project's shape matches a method already captured as one of the user's own skills, recommend **reusing that skill** rather than building from scratch — name it in the Design Decisions section. These are design-time architecture choices, so surface them here, before recommending a generic library:
- **`data-pipeline-workflow`** — when the project itself IS a **multi-source collect→classify→visualise pipeline** (a demand/landscape map, a competitor or review aggregator, any "ingest from N sources and organise it" system). The **orchestrator**: it fires the method skills in order and scaffolds the **versioned, auto-reconciling** pipeline, so adding a source needs no per-source filter and a prompt change re-reconciles every source uniformly. Recommend this **first** for pipeline-shaped projects; the two below are its sub-components (or standalone when only one stage is needed).
- **`data-scrape-to-saturation`** — when the project collects from any paginated / capped / multi-segment listing site or API and wants the *complete* dataset on a schedule (enumerate-to-saturation + axis-segmentation to beat per-feed caps + incremental warm-runs, with the public-data/ToS/no-PII guardrails). Prefer it over a generic scraper recommendation when the goal is completeness.
- **`data-search-to-saturation`** — when the project's *own deliverable* is exhaustive discovery: find and map **all** the X (sources, competitors, tools, platforms, datasets) for a domain and keep that list current (yield-tracked source registry + delta re-runs). Note: this scout already runs the method internally as Phase 1.5 — recommend the *skill* only when discovery is part of what the user is **building**, not for the scout's own search.

Only recommend a skill when the pattern genuinely fits — don't force it.

---

### Synthesis

Before writing output, internally draft the "Design Decisions" section (see output format below). This is the most important part of the output — it must:
- Reference specific findings from the search (package names, SO question titles, repo names, README gotchas)
- Name concrete decisions the user should make differently because of what was found
- Be tied directly to `[DERIVED_ARCHITECTURE]` and the goals from Phase 1
- Not be generic advice that applies to any project

A good design decision entry looks like:
> **Use `ffmpeg-python` not raw subprocess calls** — SourceRank 22, active releases, and sport-video-AI-analysis uses it exactly this way. The README shows their route structure for async render jobs which maps directly to your pipeline.

A bad design decision entry looks like:
> **Choose a well-maintained package** — make sure your dependencies are up to date.

---

### Build Spectrum Analysis

> **This synthesis runs in the orchestrating instance — do not delegate to a new subagent.** It requires reasoning across all Phase 3 findings and Phase 1 context simultaneously; no individual agent has both.

After drafting the Design Decisions section, derive the Build Spectrum for this project. This is a new section in the output — it does not replace or shorten any other section.

Identify which tools and platforms from Phase 3 findings represent each position on the spectrum for this specific project. Use only tools that surfaced in the search — do not introduce tool names that were not found. If no viable tool was found for a position, write "Not applicable — [one-line reason]" instead of a scores table.

**The four positions:**
1. **No-code** — visual assembly, zero programming. Tools where the user connects pre-built blocks without writing code.
2. **Low-code** — platform-assisted development with minimal custom code. The platform carries the structural work; the user extends it.
3. **Custom code** — built from scratch, full engineering control. Maximum flexibility, maximum build time.
4. **Hybrid** — name it by what it combines (e.g. "Low-code + Custom code"). Only create a hybrid if the combination genuinely resolves a tension between two positions. Do not default to a hybrid just to fill the fourth slot.

**For each applicable position:**
- Write two sentences: what the approach looks like for this specific project, and which specific tool(s) from the search findings would implement it.
- Write a scores table: Low / Medium / High per dimension, each with a one-line rationale calibrated to this project — not abstract.

Dimensions: Build time · Securability in production · User data storage · Scalability · Maintenance burden · Vendor lock-in risk · Customisability ceiling · Cost · Observability and debuggability.

**Recommendation:** State which approach to use and why, tied explicitly to the Phase 1 answers — success definition (Q3), usage context (Q4), required capabilities (Q5), and constraints (Q6). End with one sentence naming the signal that would tell the user this recommendation is wrong and they should reconsider.

Present in order: No-code → Low-code → Custom code → Hybrid.

---

## Output Format

**Personal name rule:** Refer to the intended end user of the project as "you" or "the user" throughout the output. Never use personal names found in code, filenames, database names, plan documents, or anywhere else in the codebase — even if a name appears repeatedly. Names in source files are implementation details, not the right way to address the person reading the output.

**Section omission rule:** Omit entire sections (do not show empty placeholder tables) when a section produced no results applicable to this project. In place of the section header and table, write a single line: `> N/A — [one-line reason why this section doesn't apply].` Do not leave empty `...` placeholder tables in the output.

```
# Project Scout Results

## Interview Input
> Verbatim answers from the Phase 1 qualification interview, preserved for historical reference.

**1. What is the main goal of this project?**
[exact answer, word for word]

**2. Who is it for?**
[exact answer, word for word]

**3. What does a successful outcome look like?**
[exact answer, word for word]

**4. In what context will it be used?**
[exact answer, word for word]

**5. What must it be capable of doing?**
[exact answer, word for word]

**6. Are there any constraints?**
[exact answer, word for word]

---

**Building:** [one-sentence summary from interview]
**Context:** [audience + usage context]
**Needs:** [key capabilities derived from interview]
**Search terms used:** Domain: [DOMAIN] · Primary: [USECASE] · Broader: [USECASE2]

---

## 🔥 Top GitHub Repos
| Repo | ⭐ Stars | Lang | Last Updated | Description |
|------|---------|------|-------------|-------------|
...
⚠️ [flag any repo not updated in >12 months]

## 📦 Package Health
| Package | SourceRank | ⭐ Stars | Last Release | Status |
|---------|-----------|---------|-------------|--------|
...
[active = SourceRank ≥20 + recent release; risky = rank <10 or stale]

## 💬 Community Insights
**[Detected community — e.g. r/python / dev.to / discuss.pytorch.org]:**
- [thread title] — [url]

**[Stack Exchange site — e.g. stackoverflow.com / dba.stackexchange.com] — watch out for:**
- [question title] — [score] votes · [answer_count] answers — [link]


## ⚠️ Known Breakage
> What is currently broken, unstable, or recently changed across the packages and tools found. Sources: GitHub Issues on top repos, Platform Release Currency findings, recent community threads.

- **[Library or platform]**: [what is broken or changed] — [source URL]
...
[Omit this section entirely if no known breakage or instability was found.]

## 📰 Blogs
| Blog | Specializes in | Covers releases? | URL |
|------|---------------|-----------------|-----|
...
> Full curated list sourced from: [curated list URL]

## 🔌 Claude Code Plugins & Skills
| Name | ⭐ | What it adds | Install |
|------|----|-------------|---------|
...

## ⚡ MCP Servers & Connectors
**Runtime-usable:**
| Name | ⭐ Stars | Integrates with | Add command |
|------|---------|----------------|-------------|
...

**Dev-time only (useful in Claude Code sessions, not at runtime):**
- [name] — [what it does during development]
[Omit this subsection if no dev-time MCP servers were found.]

## 📋 Project Templates & CLAUDE.md Patterns
| Repo | ⭐ Stars | Stack | Link | Notable patterns |
|------|---------|-------|------|-----------------|
...

## 🖥️ CLI Tools
| Tool | ⭐ / Downloads | Role | Install |
|------|--------------|------|---------|
...
[runtime / dev-time / build-time tags shown per tool]

---

## 🔨 Build Spectrum
> Four approaches for building this project, scored against each other. Tools are derived from search findings only — nothing is hardcoded.

### 1. No-code
[Two-sentence description: what this looks like for this project + which specific tools from findings]

| Dimension | Score | Why for this project |
|-----------|-------|----------------------|
| Build time | Low/Medium/High | [one line, project-specific] |
| Securability in production | Low/Medium/High | [one line] |
| User data storage | Low/Medium/High | [one line] |
| Scalability | Low/Medium/High | [one line] |
| Maintenance burden | Low/Medium/High | [one line] |
| Vendor lock-in risk | Low/Medium/High | [one line] |
| Customisability ceiling | Low/Medium/High | [one line] |
| Cost | Low/Medium/High | [one line] |
| Observability and debuggability | Low/Medium/High | [one line] |

### 2. Low-code
[same structure]

### 3. Custom code
[same structure]

### 4. Hybrid: [Name A] + [Name B]
[same structure — omit this section entirely if no genuine hybrid is warranted]

**Recommendation:** [Which approach and why, tied to Q3/Q4/Q5/Q6. One sentence on the signal that would tell the user this recommendation is wrong.]

---

## 🏗️ Recommended Approach
> Based on all findings — not assumed before searching.

**Candidate approaches:**
1. **[Approach name]** — [what the findings support or argue against it; specific repos/stars/community signal cited]
2. **[Approach name]** — [same]
3. **[Approach name, if applicable]** — [same]

**Recommendation:** [which approach and why, grounded in specific findings. Note any hardware/constraint compatibility issues.]

---

## Design Decisions
> Specific decisions these results should change about how you build this. Each point names the source (repo, package, SO question, README finding) it came from.

1. **[Concrete decision]** — [why, tied to a specific finding]
2. **[Concrete decision]** — [why, tied to a specific finding]
3. **[Concrete decision]** — [why, tied to a specific finding]
4. **[Gotcha to plan for]** — [source: SO question / README warning / package staleness]
5. **[Best starting point]** — read [repo name]'s [specific file or section] before writing any code; their [pattern] maps directly to your [component]

---

## [Additional sections — add as needed]
If dynamic searches surface a source type that does not fit any section above (e.g. a dedicated video tutorial platform, a curated newsletter, a domain-specific package registry), add a new section here using the same table or list format. Do not force results into an existing section if they belong in their own category.

## 🌍 Geographic and Platform-Specific Sources
> Only present when §10 ran (i.e. [GEOGRAPHY] or [CORE_PLATFORM] was specified in Phase 1).

**Business directories and registries:**
| Name | API? | Cost | What it provides | ARM64 | GDPR note |
|------|------|------|-----------------|-------|-----------|
...

**Job boards as intent signal sources:**
| Board | Access | Cost | Filter parameters | Best for |
|-------|--------|------|-----------------|---------|
...

**Language-specific communities:**
| Platform | Language | Size | Access method |
|----------|----------|------|--------------|
...

**Language-specific blogs:**
| Blog | Language | Specializes in | URL |
|------|----------|---------------|-----|
...

**Language-specific GitHub repos (not in §1 main results):**
| Repo | ⭐ Stars | Last Push | What it does |
|------|---------|-----------|-------------|
...

**[CORE_PLATFORM]-specific monitoring tools:**
| Tool | What it detects | API type | ToS risk | ARM64 |
|------|----------------|----------|----------|-------|
...

**Local trade publications and industry associations:**
| Name | URL | Language | Covers releases? | Scrapable? |
|------|-----|----------|-----------------|-----------|
...

**Best new signal sources (ranked):**
1. [Source] — [signal type] — [cost] — [recommended frequency]
...
```

---

## Phase 5 — Save Output to File

After producing the output above and displaying it to the user, immediately save it to a file. Do not ask for permission — always save.

**Determine the output path:**

```bash
SLUG=$(echo "[USECASE]" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9-')
DATE=$(date +%Y%m%d)
OUTFILE="project-scout-${DATE}-${SLUG}.md"
```

Save the file to the current working directory:

```bash
echo "Saving to: $(pwd)/${OUTFILE}"
```

Use the Write tool to write the complete rendered output (everything from `# Project Scout Results` through the end of the Build Spectrum section (including the Recommendation line)) to that file path.

After writing, tell the user:

```
📄 Results saved to: [full file path]
```

After saving, invoke `/project-claude.md` to suggest a file structure and generate a starter `CLAUDE.md` for the project. Pass the following as args — do not rely on `/project-claude.md` picking up context implicitly:

- **What is being built** — one-sentence summary from Phase 1
- **Stack** — the recommended approach from Phase 4's Tech Stack Recommendation (language, runtime, key frameworks)
- **Key packages** — the [KEY-PACKAGES] list from Agent B
- **APIs and external services** — every API or scraping source identified
- **Delivery mechanism** — how the output reaches the user (CLI, webhook, WhatsApp, browser, etc.)
- **Architecture pattern** — any structural pattern identified (e.g. extractor-per-source, split-file CLAUDE.md, pipe mode)
- **Constraints** — hardware, RAM, disk, platform from Phase 1 question 6
- **Goal** — who it is for and what success looks like

`/project-claude.md` will skip its own qualification interview and go straight to structure suggestions.

## Final Step — Write observation

Append a timestamped entry to `~/.claude/skills/project-scout-subagents/observations.md`:

```markdown
## [ISO 8601 timestamp]
[One sentence: what worked, what was unclear, any subagent coordination issue or search section that underperformed, output quality]
```

If `observations.md` does not exist, create it with this frontmatter first:

```markdown
---
last-reviewed: 1970-01-01T00:00:00Z
---
```
