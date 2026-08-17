# AGENTS.md — working notes for future sessions on this repo

This file is a handoff summary for whichever agent (or human) picks this
repo up next. It captures what's been analyzed, what's been decided, and
what's still open — so the next session doesn't have to re-derive the
whole workflow structure from scratch. Full detail lives in `README.md`;
this file is the condensed map plus how-to-work-with-this-file notes.

## What this repo is

- One file matters: `Complex Automations.json`, an n8n workflow export.
- It's **not one workflow** — it's 10 fully independent automations
  bundled on one canvas (188 nodes total, 178 executable + 10 sticky-note
  section headers). Verified via BFS over the workflow's `connections`
  graph starting from each trigger node: zero shared nodes between
  sections, node counts sum exactly to the file total.
- It's a marketing/agency-ops content-generation toolkit, not a full
  marketing platform. It generates copy, images, and emails; it does not
  do competitor research, budgeting, ad publishing, or lead
  qualification/closure. See the "Gap analysis" section of `README.md`
  for the detailed breakdown (recap below).

## How to work with this file efficiently

`Complex Automations.json` is ~500KB / 188 nodes — don't read it whole.
What worked well:

1. **Group nodes by trigger, not by position.** Sticky notes are visual
   labels only; the reliable way to find "which nodes belong to which
   automation" is BFS over `data['connections']` starting from each
   trigger node (`formTrigger`/`scheduleTrigger`/`googleDriveTrigger`).
   See the python snippet pattern used in this session — build an
   adjacency map from `connections`, BFS from each trigger, verify the
   union of all groups' node counts equals the total node count (it does:
   178). This is the fastest way to get an exact, verified node list per
   automation.
2. **Summarize nodes by type**, don't dump full JSON. For `code` nodes,
   the first 1-2 non-empty lines of `jsCode` are usually enough to tell
   what the node does without reading the whole script. For `googleSheets`
   nodes, `operation` + `documentId.cachedResultName` + `sheetName` tells
   you the data source. For `httpRequest`, `method` + `url` tells you the
   external service. For `gmail`, `sendTo` + `subject` tells you the
   recipient pattern (this is how the hardcoded-recipient gaps were
   found — worth re-checking after any "fix the gaps" pass).
3. The 10 groups, trigger node name → section (for quick node-name
   lookups without re-running BFS):
   - `📋 Launch Intake Form` → AI D2C Launch Kit PRO (21 nodes)
   - `📋 Property Listing Form` → AI Real Estate Marketing Kit (15 nodes)
   - `📋 New Client Form` + `⏰ Daily 10 AM: Kickoff Follow-Up?` + `⏰ Every
     Monday: Weekly Check-In` → AI Client Onboarding (26 nodes)
   - `📋 Launch Campaign Form1` + `⏰ Daily 10 AM: Email 2 Due?` + `⏰ Daily
     11 AM: Email 3 Due?` → AI Lead Gen Machine (25 nodes)
   - `📋 Influencer Brief1` → AI Influencer Factory (16 nodes)
   - `📋 Festival Campaign Form1` → AI Festival Campaign Generator PRO (25 nodes)
   - `New File in invoices/` + `Daily 8am trigger` → Invoice Digest (15 nodes)
   - `Every 60s` → AI First Outreach / compliance-gate (8 nodes)
   - `On form submission` (Proposal And Invoice) → Proposal & Invoice Autopilot (15 nodes)
   - `📋 What Do You Want to Learn?` → AI Learning Machine (12 nodes)

## Critical — unresolved as of this writing

**Hardcoded live secrets are still in the workflow JSON**, committed to
this now-public repo:
- 2 Google Gemini API keys, inline in `httpRequest` node URLs
  (`Gemini Vision API1` in Invoice Digest, `Gemini Generate SMS` in AI
  First Outreach).
- A Supabase `service_role` JWT (bypasses RLS), inline in 4 nodes in AI
  First Outreach (`Query New Leads`, `POST → compliance-gate`,
  `Update Lead Stage`).

**As of this session, these have not been rotated or fixed** — this has
been flagged to the user twice (in README and in chat) but no fix has been
applied yet, pending user confirmation to proceed. If picking this up:
check with the user whether rotation/fix has happened before assuming the
repo is still exposed. Do not reproduce the actual key values in any
future commit, doc, or chat output — describe by node name/location only.

## Analysis performed so far (chronological)

1. Full structural pass: node-type census, trigger inventory, credential
   inventory, per-automation flow tracing (data source → AI calls →
   output). Written into `README.md`.
2. Security scan: found the hardcoded secrets above.
3. Verified specific claims by grepping the JSON rather than assuming:
   - Confirmed Lead-Gen's 3 outreach emails and Client Onboarding's
     welcome email are hardcoded to fixed test addresses
     (`teamlancemart@gmail.com`, `admin@reoclaw.com`) rather than the
     real contact/client email — these automations do not currently
     deliver to real recipients as configured.
   - Confirmed "Apollo integration" is a static Google Sheet
     (`apollo-contacts-export`) manually populated from a CSV export —
     no live Apollo API call exists anywhere in the file.
   - Confirmed a "Marketing Budget (Monthly)" field exists on the D2C
     form and *is* used — but only to look up a hardcoded channel-mix
     percentage table in a code node, not for real budget
     tracking/allocation/spend.
   - Confirmed the Real Estate form's "Reference Image" file-upload field
     is captured but never referenced by any downstream node — it's a
     dead input; no image-conditioned generation happens anywhere.
   - Confirmed zero ad-platform (Meta/Google/LinkedIn Ads) nodes exist
     anywhere in the file — "campaign" output is always content emailed
     to a human, never something actually published/targeted/spent.
   - Confirmed zero MCP nodes/servers are used anywhere — this is a
     plain node/HTTP-based automation stack, not an MCP-orchestrated
     agent workflow.
4. Marketing-agent gap analysis (budgeting, competitor analysis, rich
   media input, Apollo, ads-platform integration, lead
   qualification/closure) — full detail in `README.md`'s "Gap analysis:
   this vs. a real marketing agent" section. One-line summary of each:
   - **Budgeting**: cosmetic tier→% lookup only; needs a real ledger
     (Supabase table), spend-sync from ad platforms, and a pre-publish
     budget gate.
   - **Competitor analysis**: pure LLM hallucination from form text;
     needs a SERP API + Meta Ad Library + scraping step feeding real data
     into the "Market Intelligence" agents.
   - **Rich media input**: text-only forms, one dead file field; needs
     real file-upload fields feeding Gemini's multimodal input for
     image-conditioned generation, plus a real video-gen API for the
     "reel video prompts" the D2C form currently only stubs as text.
   - **Apollo**: static CSV-in-a-Sheet; needs live Apollo People Search
     API calls with pagination/credit handling + email verification.
   - **Ads-platform integration**: doesn't exist; needs Meta/Google/
     LinkedIn Ads API integration, a Campaign Builder step, and — this is
     a hard requirement, not optional — a human-approval gate before any
     real spend, since the workflow currently has zero human-in-the-loop
     steps anywhere.
   - **Lead qualification/closure**: binary replied/not-replied only;
     needs AI-based lead scoring, a real CRM (HubSpot/Pipedrive/Zoho)
     with proper pipeline stages, meeting booking, and closure/revenue
     attribution back to the budget ledger.
   - Cross-cutting: move shared state off Sheets onto Supabase/Postgres,
     fix secrets before adding more, add a reusable human-approval
     pattern, de-duplicate the repeated 4-agent pipeline (D2C/Festival)
     into an `Execute Workflow` sub-workflow, add retry/backoff, add
     cost/credit visibility.
   - Full API/credential inventory table is in `README.md`.

## Status: analysis only, no implementation yet

Nothing in `Complex Automations.json` itself has been modified this
session — only `README.md` and this file. No new nodes, credentials, or
API integrations have been added to the workflow. The gap analysis is a
roadmap, not a changelog.

## Open decisions the user hasn't made yet

- Whether/when to fix the hardcoded-secret issue (rotation + moving to
  proper n8n credentials).
- Which of the 6 gap areas (budgeting / competitor analysis / rich media
  input / Apollo / ads-platform / lead qualification) to build first.
  README suggests budgeting + ads-platform as a natural coupled pair
  (you need the ledger before you can safely auto-publish spend) but this
  is a suggestion, not a decision.
- Which CRM (HubSpot vs Pipedrive vs Zoho) and which ad platform (Meta vs
  Google vs LinkedIn) to wire up first — each needs its own app
  registration/approval, so this gates real implementation start.
- Whether an MCP-based/agentic redesign is wanted at all, vs. continuing
  to extend the current fixed-pipeline node-by-node style.

## Conventions to keep if you extend this

- Don't reproduce secret values anywhere (chat, commits, docs) — describe
  by node name/location, and point at where to find/rotate them.
- When quoting a "fact" about the workflow (a hardcoded value, a missing
  integration, a field that's unused), verify by grepping/parsing the
  JSON directly rather than inferring from node names — several of the
  most useful findings this session (dead file field, hardcoded test
  recipients, cosmetic budget field) were only caught by actually reading
  the node parameters/code, not by their names alone.
- Push to branch `claude/n8n-marketing-workflow-analysis-lf4tvb` in
  `adityamathur20/lerneanLabs-marketingAgent` unless told otherwise.
