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

## Critical — partially resolved, action still needed from the user

**The code-side fix is done.** All 5 nodes that used to hardcode secrets
now reference n8n stored credentials instead:
- `Gemini Vision API1` and `Gemini Generate SMS` → `genericAuthType:
  httpQueryAuth`, pointing at the existing `Query Auth account` credential
  (id `SCFIE3udZwAl5Kbz`) already used by the other 16 Gemini nodes. No
  new credential needed for these two.
- `Query New Leads`, `POST → compliance-gate`, `Update Lead Stage` →
  `genericAuthType: httpCustomAuth`, pointing at a **new** credential
  named `Supabase Service Role (Custom Auth)` that does **not exist yet**
  in the user's n8n instance (n8n's Custom Auth credential type was used
  because Supabase needs both `apikey` and `Authorization` headers set to
  the same JWT, and a single `httpHeaderAuth` credential only carries one
  header). The node JSON has a placeholder credential ID
  (`REPLACE_WITH_REAL_CREDENTIAL_ID`) that will show as "credential not
  found" in n8n until the user creates the real credential and re-selects
  it on all 3 nodes — this is expected and documented in the README
  security section.
- Verified by grep after editing: zero occurrences of either leaked
  Gemini key or the JWT's payload signature remain anywhere in
  `Complex Automations.json`.

**What's still NOT done — cannot be done from this repo alone:**
1. The actual key **rotation** — Google Cloud Console for the 2 Gemini
   keys, Supabase Project Settings → API for the `service_role` key. Code
   changes don't invalidate a key; the old values are still live until the
   user revokes them.
2. Checking Supabase logs for unexpected access since the repo went
   public — the user needs to do this, no log access from here.
3. Creating the new `Supabase Service Role (Custom Auth)` credential in
   the user's live n8n instance and re-linking it on the 3 nodes (can't
   be done from a git repo — n8n credentials are stored server-side in
   the n8n instance, not in the exported workflow JSON).
4. Confirming whether the existing `Query Auth account` credential's
   stored value is itself one of the leaked keys (if whoever built the 2
   rogue nodes copy-pasted from it) — if so it needs rotating too, not
   just the 2 that were visibly hardcoded. Nobody has confirmed this
   either way yet.
5. **Git history still contains the old secret values** in the pre-fix
   commits (`88f970a`, `d63387d`, `9e29030` on this branch, plus whatever
   was on `main` before). Removing them from the current file does not
   remove them from history. Scrubbing history (`git filter-repo` +
   force-push) is a separate, more invasive action that has NOT been
   done and needs the user's explicit go-ahead before attempting, since
   it rewrites shared history and could break other clones/PRs.

If picking this up: don't assume the secrets are safe just because the
code fix landed — confirm with the user whether steps 1-5 above have
happened. Do not reproduce the actual key values in any future commit,
doc, or chat output — describe by node name/location only.

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

## Status: one implementation change made; gap-analysis roadmap otherwise unbuilt

`Complex Automations.json` **has** been modified once: the 5
hardcoded-secret nodes were switched to credential references (see above)
— a minimal, targeted text-level edit (not a full re-serialize) so the
diff stayed reviewable (`47 insertions(+), 27 deletions(-)`, only touching
those 5 nodes). Everything else — the marketing-agent gap analysis
(budgeting, competitor analysis, rich media input, Apollo, ads-platform,
lead qualification/closure) — is still a roadmap, not implemented.

## Open decisions the user hasn't made yet

- Whether/when to actually rotate the exposed keys and create the new
  Supabase credential in their n8n instance (the workflow-JSON side is
  fixed; the account-side actions in the list above are still pending).
- Whether to scrub git history of the old secret values (destructive,
  needs explicit confirmation before attempting).
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
