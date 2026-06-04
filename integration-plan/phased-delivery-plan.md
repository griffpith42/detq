[phased-delivery-plan.md](https://github.com/user-attachments/files/28590787/phased-delivery-plan.md)
# DetQ — Phased Delivery Plan

**Consulting Engagement Timeline & Milestone Tracker**
*Integration Reference v1.0*

---

## Engagement Overview

Total duration: **6–8 weeks**
Team required from client: 1 technical contact + 2 analyst validators
Deliverables: configured DetQ instance, schema documentation, analyst guide, 30-day hypercare

---

## Phase 1: Discovery (Weeks 1–2)

**Objective:** Fully understand the client's database structure, analyst workflows, and query patterns before any configuration begins.

### Activities

| Activity | Owner | Output |
|---|---|---|
| Kickoff call — stakeholders, scope, timeline confirmation | Both | Signed scope document |
| Database schema export and review | Client IT + DetQ lead | Schema inventory spreadsheet |
| Sample query collection — 20–30 real analyst queries | Client analysts | Query sample document |
| Query pattern analysis — identify field coverage gaps | DetQ lead | Gap analysis report |
| Infrastructure review — deployment environment, DB access | Client IT + DetQ lead | Infrastructure checklist |
| Risk register — known complexity areas, edge cases | DetQ lead | Risk register document |

### Milestone: Phase 1 Sign-Off
Client stakeholder reviews and approves:
- Schema inventory (all target fields identified and mapped)
- Query sample (representative of actual analyst use cases)
- Infrastructure checklist (deployment path confirmed)
- Risk register (no blocking issues)

**Gate:** Phase 2 does not begin until Phase 1 sign-off is complete.

---

## Phase 2: Configuration (Weeks 3–4)

**Objective:** Build and configure the DetQ instance for the client's specific database, schema, and query vocabulary.

### Activities

| Activity | Owner | Output |
|---|---|---|
| Field mapping — complete `fieldMap` object in `criteria.js` | DetQ lead | Configured field map |
| Guard rule adaptation — review all 10 rules against client schema | DetQ lead | Adapted guard rule config |
| Stop token review — identify client-specific terms to protect | DetQ + client analyst | Updated stop token list |
| Extraction prompt adaptation — align vocabulary to client query style | DetQ lead | Adapted prompt file |
| Database connection setup — read-only service account | Client IT | Working DB connection |
| Development deployment — DetQ running against dev/staging database | DetQ lead | Dev instance URL |
| Initial smoke test — 5–10 queries against dev instance | DetQ lead | Smoke test results |

### Configuration Checklist

Before Phase 2 is complete, confirm:

- [ ] All schema fields from Phase 1 are mapped in `fieldMap`
- [ ] Date field(s) confirmed and fallback logic configured
- [ ] Spread field type confirmed (numeric vs. string — `parseNumLoose()` if string)
- [ ] Boolean flag storage format confirmed (0/1, true/false, or Y/N)
- [ ] Deal grouping key confirmed for IG/HY
- [ ] Deal grouping key confirmed for ABS (if applicable)
- [ ] Stop token list reviewed against client's domain vocabulary
- [ ] Top N default confirmed (default: 10)
- [ ] Market routing confirmed (which markets are in scope)
- [ ] Dev deployment accessible to DetQ lead and client technical contact

### Milestone: Phase 2 Sign-Off
Client technical contact confirms:
- Dev instance is accessible
- Smoke test queries return plausible results
- No blocking configuration issues

**Gate:** Phase 3 does not begin until Phase 2 sign-off is complete.

---

## Phase 3: Validation (Weeks 5–6)

**Objective:** Systematically validate DetQ's query output against analyst expectations, identify and resolve edge cases.

### Analyst UAT Session Structure

Two sessions, each approximately 2 hours, with 2 analyst validators.

**Session 1 — Core Query Patterns (Week 5)**

| Query category | # of test queries | Focus |
|---|---|---|
| Date range queries | 5 | Date field mapping, quarter/month parsing |
| Ranking queries | 5 | Sort field, Top N enforcement |
| Rating filter queries | 5 | Rating field, agency guard |
| Sector filter queries | 3 | Sector field, sanity guard |
| Deal flag queries | 5 | Boolean flags, stop token protection |
| Numeric filter queries | 5 | Numeric intent gate, explicit vs. inferred |
| Compound queries | 7 | Multiple simultaneous filters |

**Session 2 — Edge Cases & Refinements (Week 6)**

Queries from Session 1 that returned unexpected results. Known failure mode categories from the guard rule reference. Analyst-nominated edge cases — queries they use regularly that weren't in the standard set.

### Issue Classification

| Severity | Description | Response |
|---|---|---|
| P1 — Blocking | Query returns incorrect results with no workaround | Fix before Phase 4 |
| P2 — Significant | Query returns unexpected results but correct data reachable | Fix before Phase 4 |
| P3 — Minor | Cosmetic or formatting issue | Fix in hypercare period |
| P4 — Enhancement | Desired feature not in current scope | Added to roadmap |

**Target for Phase 3 completion:** Zero P1 issues. Zero P2 issues. P3 issues documented and scheduled.

### Milestone: Phase 3 Sign-Off
Both analyst validators confirm:
- All queries from the standard test set return correct results
- No P1 or P2 issues remain open
- System behavior is consistent and predictable

**Gate:** Phase 4 does not begin until both analysts sign off on UAT.

---

## Phase 4: Production Deployment (Weeks 7–8)

**Objective:** Deploy DetQ to the production environment, deliver all documentation, begin hypercare.

### Activities

| Activity | Owner | Output |
|---|---|---|
| Production database connection — read-only service account | Client IT | Production connection confirmed |
| Production deployment — DetQ on production infrastructure | DetQ lead | Production instance live |
| Production smoke test — 10 queries against live data | DetQ lead + client | Production test sign-off |
| UI configuration — branding, quick-search chips, help tab | DetQ lead | Final UI delivered |
| Documentation delivery — schema map, config file, user guide | DetQ lead | Documentation package |
| Analyst onboarding session — 1-hour walkthrough | DetQ lead + analysts | Analysts able to use independently |
| Hypercare start — 30-day support window begins | DetQ lead | Support channel confirmed |

### Documentation Package Delivered at Phase 4

| Document | Contents |
|---|---|
| Schema map | Final `fieldMap` configuration, field inventory, grouping keys |
| Configuration reference | All guard rule settings, stop token list, Top N configuration |
| Analyst user guide | How to write queries, what DetQ can and cannot do, example queries |
| Admin guide | Deployment configuration, environment variables, DB connection settings |
| Test results archive | Full UAT session results from Phase 3, issue log, resolutions |

### Hypercare Period (Days 1–30 Post-Deployment)

During the 30-day hypercare window:
- Response to P1/P2 issues within 1 business day
- Weekly check-in call with client stakeholder
- Guard rule refinements and prompt adjustments available at no additional cost
- Query pattern additions (new stop tokens, new deal flags) available at no additional cost

Issues beyond hypercare scope: new market modules, schema changes due to database migration, UI feature additions.

---

## Milestone Summary

| Milestone | Target Date | Sign-Off Required From |
|---|---|---|
| Phase 1 Sign-Off | End of Week 2 | Client stakeholder |
| Phase 2 Sign-Off | End of Week 4 | Client technical contact |
| Phase 3 Sign-Off | End of Week 6 | Both analyst validators |
| Production Deployment | End of Week 8 | Client stakeholder |
| Hypercare End | Day 30 post-deployment | — |

---

## Risk Register Template

Use this template during Phase 1 discovery to document and track engagement risks.

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Database access delayed by IT procurement | Medium | High — delays Phase 2 start | Initiate IT ticket in Week 1 |
| Spread field stored as non-numeric string | Medium | Medium — requires `parseNumLoose()` | Confirm in schema review |
| Deal grouping key not unique | Low | High — produces duplicate deals | Verify uniqueness in schema review |
| Analyst query vocabulary differs significantly from reference | Medium | Low — addressed in prompt adaptation | Review query sample carefully |
| Production environment differs from dev | Low | Medium — re-test in Phase 4 | Run full smoke test on production |
| Rating data stored across multiple tables | Low | High — requires JOIN logic in SQL generation | Flag in schema review |

---

*For engagement scope and FAQ, see `engagement-overview.md`.*
*For field mapping instructions, see `/framework/schema-mapping-guide.md`.*
