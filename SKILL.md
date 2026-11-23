---
name: autoresearch-genealogy
description: AI-assisted genealogy research using structured prompts, your own logged-in record-site sessions, web search, and a private research vault. Use when researching family history, expanding a family tree, resolving open genealogy questions, analyzing DNA matches, finding ancestors, or ordering vital records. Triggers on "genealogy", "family tree", "ancestors", "DNA relatives", "autoresearch", census records, obituaries, Find a Grave, FamilySearch, NUMIDENT, birth certificate.
---

# Autoresearch Genealogy

Structured AI-assisted genealogy research with source rigor. Every instrument below is described with the failure mode it has actually exhibited. An instrument behavior stated without a measurement note is a hypothesis — re-measure it in your own environment before you rely on it.

This method was developed over dozens of long research sessions that took one family's lines back nine generations and resolved a contested-parentage case (biological family identified and ranked from DNA + records) in about 27 hours of wall clock. The published unassisted pace for the same class of problem is "at least 26 hours of research over 3 to 4 months" (progenealogists.com/reference). What made the difference was not search volume; it was session-authenticated record APIs, positive-controlled negatives, and reading page images instead of OCR.

## Quick Start

```text
/autoresearch-genealogy 01          # Tree expansion
/autoresearch-genealogy 08          # Open question resolution
/autoresearch-genealogy dna-analysis
/autoresearch-genealogy records     # What to order, from whom, verified fees
```

## The vault

Keep all research in one private directory (the vault) that never gets published:

```text
<vault>/
  Family_Tree.md        # the tree, one claim per line, source on every claim
  Open_Questions.md     # current state of every open question
  Research_Log.md       # append a dated session entry every session
  Genetic_Profile.md    # DNA notes, match analyses
  Documents/            # private analyses (chmod 600)
  Documents/Evidence_YYYY_MM_DD/   # one subdir per session (chmod 700, files 600)
  prompts/              # the numbered research prompts (see Available Prompts)
  workflows/            # reusable procedures (intake, triage, OCR, oral history…)
```

The current state of every open question lives in the vault, not in this skill file. Before a session, read the latest `Research_Log.md` entry and the relevant open questions. (This skill once carried a "Current Research State" block that nobody updated; it was wrong for two and a half months. State belongs in the vault, where the work happens.)

## Hard rules (apply to every session)

1. **Living people never leave the vault.** Describe them ("a maternal half-sister at 25%"), never name them, in anything public: sites, reports, posts, or notes that could be quoted. Before any publication, grep the outgoing text for every living person's name, with a positive control (a deceased name that must hit) proving the grep works. Read every hit before clearing it.
2. **No credentials, ever.** Record-site and DNA-site logins are the researcher's. The agent never types a password, never solves a CAPTCHA, never pulls from a password manager. When a session is dead, bring the sign-in page to the front and ask the human to sign in.
3. **Nothing outward without an explicit instruction.** Records requests, messages to DNA matches, emails, record orders, subscriptions: show the exact text or form first, and act only on an explicit "send" / "order" / "pay".
4. **A negative needs a positive control that sits outside the set you searched.** Three outcomes, never two: `found` / `genuinely absent (controlled)` / `could not measure (blocked, paged out, unindexed)`. Log all three in the Research_Log with the control named. Most false conclusions in genealogy are uncontrolled negatives.
5. **Read the image, not the OCR, before asserting what a record says.** Funeral-home record books, census sheets, newspaper pages, town-report tables: every one of these has produced a wrong or missing reading from OCR and a correct one from the page image.
6. **Confidence tiers on every claim:** Strong Signal (two independent sources, or one primary), Moderate Signal (one secondary source, or the family's account), Speculative (inference). A working identification stays Moderate until a record ties the person to the child.
7. **Session cookies and tokens never enter the vault, git, notes, or reports.** Cache them in one local, permission-locked place (mode 600) and nowhere else.
8. Vault style: source on every claim, negative results logged, no claim without a record id.

## Instruments, in the order to use them

### Tier 0: your own logged-in sessions, replayed as a local API client

The single highest-leverage pattern in this method: the big record sites run internal search APIs behind their web UI, and your browser is already authenticated to them. A small local CLI that replays your own session cookies against those internal endpoints turns a click-heavy website into a scriptable instrument — batch queries, post-filtering, and paging discipline that the UI makes impractical.

- Build the client on an HTTP library that can impersonate a real browser's TLS fingerprint. Plain HTTP clients with valid cookies get **403 bot-blocks** on major DNA/record sites (TLS fingerprinting); the same cookies through a browser-impersonating client succeed.
- Give the client a `check` command (session live / dead), typed exit codes (ok / bot-block / session-dead / empty-body), and a `rows` mode that post-filters results locally (see Tier 1: several query parameters are ranking hints, not filters).
- Cookies are read fresh from the browser profile on each run; when the session dies, the human signs in again in the browser and the next run picks it up. Never store the password.
- Browser automation (a CDP bridge into your real, logged-in browser) is the fallback for anything the API client cannot do — and the only route for sites that bot-wall plain fetchers.

### Tier 1: FamilySearch — what works and what silently lies

Measured behaviors (re-verify in your environment; platforms change):

| Fact | Measured behavior |
|---|---|
| Public platform API | Requires an OAuth bearer token the web session never exposes; cookie auth gets 401. The internal search service (`/service/search/hr/v2/personas`) is the working route with a web session. |
| Collection ids | The same record set can exist under multiple collection ids, and a wrong one returns "Results (0)" that looks like a negative. Verify every collection id against a known-positive query (a person you *know* is in it) before trusting a zero. (Example from practice: the US NUMIDENT collection's working id differed from the one most often cited, and the cited one returned clean zeros.) |
| **Deep-paging ceiling** | Past a certain offset (measured ≈1,100 rows), the search service silently returns page 0 again with HTTP 200. A "full scan" past the ceiling is a lie. Slice the query across parent-name, place, and date-range hints to cover the space in slices under the ceiling, and report rows beyond it as **unmeasured**, never absent. |
| Hint parameters | `motherSurname`, `spouseSurname`, `anyPlace`, `birthLikeDate` are **ranking hints, not filters** — a query can return tens of thousands of rows that ignore them. Post-filter locally on the parent/spouse strings. (Exception measured: in the 1950 US census collection, the birthplace parameter does act as a filter.) |
| Sparsely-indexed collections | Some collections (e.g., Québec parish registers) index almost no event dates, so any date-bounded query returns 0 rows — unmeasured, not negative. Sweep on surname alone, then post-filter on parents. Some record classes are simply not indexed past a cutoff year at all. |
| Results-page timing | The web results list can show "Results (0)" seconds after navigation for queries that return hundreds of thousands of rows a few seconds later. Wait for the count to settle, or use the API client. |
| **Tree merges** | Shared trees fuse same-name people (two men with the same name, born the same year, on different continents, were merged into one tree person with both men's sources attached). Never take a tree person's facts as one person: open each attached source and check its own parents and places. |
| Restricted films | Some digitized films are viewable only at an affiliated library/center. "Restricted" is a coverage fact, not a negative. |
| Garbled index spellings | Indexers mangle unfamiliar names (French names in US indexes especially). Search the consonant skeleton with a regex, never the exact string alone. |

### Tier 1: DNA match analysis (23andMe-shaped; the method ports to any match list)

- The "relatives in common" view on the profile page is the working triangulation endpoint; the equivalent parameter on the list endpoint may be silently ignored (identical output for every value). Verify an endpoint actually responds to its parameter before building on it.
- Sessions die on idle; check liveness first. Profile pages driven through browser automation should be batched small (measured: batches of 3, not 15 — larger batches time out).
- Write your conclusions back as private annotations on the match, so the next session inherits them.
- **Generation placement by ratio:** for a match M with no usable tree, compute share(M, relative) / share(subject, relative) across every relative both share. A geometric mean ≈ 0.5 puts M one generation below the subject (validated: 47 shared relatives, gm 0.57, median 0.54). This works when M has no surnames, no places, and no tree.
- **Relationship math** (one side): ~50% parent/child; ~25% grandparent, aunt/uncle, half-sibling; ~12.5% first cousin, great-aunt/uncle, half-aunt/uncle. A site's relationship label can be a generation off; the direct figure to the parent decides, not the label on the child.
- Re-measure key figures after every re-login and record the numbers; two identical readings 24 h apart are what make a number citable.

### Tier 1: newspapers

| Source | Recipe | Trap |
|---|---|---|
| Chronicling America (loc.gov) | Use the JSON API (`fo=json`) with state/date facets; full page text comes from the word-coordinates (alto_xml) service — the JSON page object has no `full_text`. Run from a real browser context if plain fetchers get 403/525. | The legacy search URL redirects; date coverage gaps for a region are a coverage limit, not absence — name a positive control (a page count in an adjacent window). |
| Regional newspaper archives (e.g., BAnQ for Québec) | For a narrow window, read every issue as page images and cite the document id. | OCR search misses names in lists (wedding guests, survivors); page images do not. |
| Local paper site search | Try the plain surname query's later pages, not just the phrase search. | Aggregators often surface only the obituary title; the paper's own archive has the text. |
| Library-proxy newspaper databases | Researcher signs in via their library; the agent works the parked, signed-in tab. | Third-party obituary mirrors get bot-walled (403); an index entry with no relatives named is a lead, not a result. |
| Funeral homes | Read the listing page and the article; some CMSs 500 on one article while the listing is 200; some SPA sites render empty templates to fetchers; aggregator copies vanish. | If the Wayback Machine has no capture, report "text not retrieved", never "no obituary". |

### Tier 1: vital and civil indexes

| Source | Recipe | Trap |
|---|---|---|
| NARA 1950 census (API) | The name+state+county search returns schedules with image URLs; read the sheet image. | The alternate query parameter returns enumeration districts only. Transcriptions lose household members — the image is the record. |
| NYC marriage index 1950–2017 (Solr) | Replay the endpoint directly with wildcard surname queries across all boroughs. | The export's GROOM_/BRIDE_ columns are **inverted** relative to the names they carry. |
| Reclaim The Records indexes (archive.org) | Read the **page images**; indexes are Soundex-ordered, then by given name. | The OCR text layer is unusable for names (a page containing a surname greps to 0). Know what does *not* exist before searching: e.g., no NYC birth index 1910–1965 in that corpus; some state death indexes end at a fixed year. |
| State digital archives (e.g., Washington) | Death, marriage, and divorce indexes at record level; read *all* rows for the name. | A same-name record can belong to a different person; resolve the spouse through NUMIDENT/SSDI before ordering a certificate. |
| SSDI mirrors | Free mirrors exist but are coverage-bounded (one common mirror is pre-2006 deaths only). | Anyone who died after the coverage window is absent by coverage, not by fact. |
| City directories (archive.org) | Download the full OCR text and read every row for the surname. | Some years are lending-restricted: "Item not available" = could not measure. |
| School yearbook sites | Full-text search over a school's run. | Class years outside the run are out of corpus, not absent. |
| Town reports / county histories (archive.org) | Landscape vital-stats tables: read visually. Download the text layer and grep with flexible whitespace (the OCR double-spaces). Positive-control before any negative. | OCR is garbage on these tables. |

### Tier 1: graves and funeral records

- **Find a Grave**: plain fetchers get rate-limited and captcha-walled; read it through your real browser session. Before any "no memorial" claim, run a cemetery-scoped search with a known memorial as the positive control. The **cemetery description** resolves renames (cemeteries are renamed when funeral homes are sold).
- **Funeral-home record collections** (university digital libraries): the viewer serves a small JPEG derivative that has produced multiple misreadings on a single record (wrong first name, wrong cemetery, wrong street, a survivor dropped). Every record also has a high-resolution master image (e.g., JP2) whose path is in the item's metadata — read the master.
- **Compiled genealogy sites** (parish-cited compilations, user trees): read them in your browser (they often block bots by robots or by behavior); treat every entry as a lead, then find each cited record yourself.

### Tier 2: web search

- Use more than one search provider, and probe the stack before trusting an empty result — individual providers go dead silently.
- A 403 on a plain page is a fetcher problem before it is a fact: escalate fetcher → text-extraction browser → your real browser session. A real 404/410 is gone; do not browser-retry it.
- Never construct a URL from a guess: read the sitemap (`/robots.txt` → `/sitemap.xml`) or the page's own links. A guessed slug 404s; the sitemap has the real path.

## Records to order: custodians, fees, rules

Verify every fee and form on the custodian's own page before ordering — and re-verify if your note is more than ~2 weeks old. Examples of the shape of this table (US-centric; build your own for your jurisdictions):

| Record | Custodian | Fee / rule (verify before use) |
|---|---|---|
| NYC pre-adoption (original) birth certificate | NYC health dept, form VR 214 | Small fee; notarized if mailed; adoptee-only via the usual vital-records portals; weeks of processing. The state form is the **wrong form** for five-borough births. |
| NYC marriage record 1950+ | NYC City Clerk | Records >50 years old are public; borough offices hold only recent decades. |
| SSA SS-5 (Social Security application) | SSA FOIA office (online portal or form SSA-711 by mail) | Flat fee per request. **Extreme Age Policy**: a number holder who would be >120 years old needs no proof of death, and parents' names are released. The SS-5 is the holder's own statement — independent of any death-certificate informant. |
| State birth/death certificates | State health dept | A cheap "computer certification" often omits parents' ages/birthplaces; order the photocopy/long form when the parents are the question. Deceased-registrant rules vary. |
| Pre-1927 county marriage licences | The county clerk, not the state | Digitized films of county licences may sit in the FamilySearch catalog. |
| Military personnel files (IDPF/OMPF) | National Personnel Records Center | Inquiry by mail or the online portal; expect months. |

Ordering is an outward action: draft the packet, show it, wait for "order". A packet handed to any agent must carry these rules verbatim.

## Session protocol (what produced nine generations in days, not months)

1. **State the objective and decompose it** into 2 to 6 deliverables, each with the evidence that would settle it. The decomposition is what keeps a 10-hour session honest.
2. **Instrument check before any search**: session-liveness checks for every authenticated instrument; browser bridge up. Dead session → the human signs in; keep working on instruments that need no login meanwhile.
3. **One evidence directory per session** (`Documents/Evidence_YYYY_MM_DD/<Topic>_SessionNN/`, private permissions), every data pull, page image, and extract saved there with the query that produced it.
4. **Parallel passes when the question fans out** (e.g., registers / deaths / newspapers / DNA / other-country records): a shared rules file (controls, three outcomes, living-people rule, no logins), each pass writes its own findings file, the main session merges and cites. Subagents get browser access only with an explicit protect-list of tabs.
5. **Research_Log entry every session**: objective; what was done, numbered, each with its source id; negatives with the control named; instrument notes; what was *not* done and by which rule; next records to get. That log is the only reason results stay reconstructible months later.
6. **Retractions are logged, not deleted**: superseded text stays, marked "withdrawn", with the reason. (Every long campaign accumulates confident misreads — a manifest misattributed to the wrong year and port, a haplogroup inferred from the wrong test. The log is where they go to be learned from.)
7. **Measure the clock** if you claim speed: first evidence file's timestamp to the commit that carries the finding. Report speed only against a *sourced* human baseline, labelled as their published expectation — never a number you made up.
8. **Free and direct sources first** (standing rule): exhaust free/direct-source routes — archives, libraries, the custodians themselves — before paywalls, trials, or new accounts. When a paid source is genuinely the only route, say so explicitly and let the human decide.

## DNA-driven adoption / contested-parentage search (validated method)

1. The ~50% one-side match is the parent on that side; the closest non-parent matches name the biological **family** (surname lists + grandparents' birthplaces).
2. Run relatives-in-common on each close target-side match; partition the in-common members into sub-clusters; find the couple each sub-cluster descends from. In the validating case, four of six sub-clusters converged on one couple.
3. Build **down** from that couple to people of the right age, in the right place, in the conception window — with a record per generation. Exclude by record (border cards, census, NUMIDENT, obituaries), never by plausibility.
4. Last mile for the **name**: the pre-adoption certificate, or a living relative's statement. DNA and records bound the family; only a document or a witness prints the given name. A witness statement is Moderate confidence; log its exact words and source.
5. **Identity bridge ≠ identity.** Proving that a documented person is the same person as a named individual in a record (a bridge) is a separate claim from proving parentage, and each needs its own evidence. More bad trees come from fusing these two claims than from any other single error.
6. **Surname anglicization and dit names**: surnames translate, compress, and get re-spelled by indexers — a French *dit* name may be indexed under either half, translated (Lévesque → Bishop, Boisvert → Greenwood are the classic documented examples), or garbled past recognition. Always search the phonetic skeleton, the translation, and both halves of a dit name.

## Search patterns that produced results

- **Search the children and siblings, not the person**: obituaries of deceased siblings and children name parents, spouses, and places that the person's own records omit.
- **Search the neighbours and the in-laws**: draft lists, subscription lists, and local-news notes place families beside each other; a relative's obituary often names the sibling you cannot find ("sister of … of Buffalo").
- **Border cards and passenger lists** for anyone who lived near an international line; read the image (occupation, spouse's name, stated purpose — and rejections).
- **Handwriting is evidence**: when an index reading of a name looks odd, verify it against the handwriting on the image letter by letter before building on it. One misread cursive name can cost a campaign weeks.
- **Retrieved vs. indexed — never conflate.** "No index hit" and "the record does not exist" are different claims; some record sets are retrievable only by browsing images in order.

## Publishing a tree (optional)

If you publish results (a family site, a report, a post):

- Deceased persons only. Hypothetical links are labelled hypothetical, low confidence, and kept out of the default view.
- Run the living-name sweep (Hard rule 1) on every outgoing change, with its positive control.
- Keep narrative and running logs off any page with a size or corpus budget; publish current-state summaries, keep the working log in the vault.
- Record what was published, when, and the version, in the Research_Log.

## Available prompts

Numbered prompt files live in the vault's `prompts/`; write them to fit your own tree. The set this method uses:

| # | Name | Purpose |
|---|------|---------|
| 01 | tree-expansion | Push every branch back using web research |
| 02 | cross-reference-audit | Find discrepancies between tree and sources |
| 03 | findagrave-sweep | Locate Find a Grave memorials |
| 04 | gedcom-completeness | Ensure GEDCOM matches vault |
| 05 | source-citation-audit | Verify two independent sources per person |
| 06 | unresolved-persons | Identify unnamed people in documents |
| 07 | timeline-gap-analysis | Find missing life event records |
| 08 | open-question-resolution | Attack every open research question |
| 09 | bygdebok-extraction | Extract from local history books |
| 10 | colonial-records-search | Pre-1800 American records |
| 11 | immigration-search | Passenger manifests, naturalization |
| 12 | dna-chromosome-analysis | Map genetic segments to ancestor lines |

Companion workflows live in `workflows/`: getting-started, new-ancestor-intake, document-triage, ocr-pipeline, oral-history-protocol, discrepancy-resolution, phase-planning.

## Skill self-improvement

This skill is a living document. After every research wave, append the lessons to a campaign-lessons log: which instrument lied and how, which search pattern produced, which trap cost time. Promote a lesson into the skill body only when it is about the *method* (an instrument behavior, a rule, a pattern) — never when it is about a *person*. Facts about people belong in the vault; the skill carries method only. That separation is exactly what makes the skill shareable.

## Ground-truth gate (mandatory)

Before asserting a stakes-bearing fact or taking any outward or irreversible action, verify against a **primary source fetched now** — never a cached or remembered value.

- Every ancestor fact (name, date, place, relationship) cites a specific record id (ark, document id, memorial number, file number). Never fabricate a person, a date, or a link.
- Every fee, form number, and custodian rule carries the date it was verified; re-read the custodian's page before any order.
- A "verified" note older than ~14 days is a hypothesis, not a fact (the decay rule).

Then: dry-run where possible, show the human exactly what will be sent, filed, or asserted, get explicit approval for any outward action, and write the verified fact back into its vault document with the date.
