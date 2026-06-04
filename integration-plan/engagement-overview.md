[engagement-overview.md](https://github.com/user-attachments/files/28590754/engagement-overview.md)
# DetQ — Consulting Engagement Overview

**What You're Getting, How It's Delivered, What It Costs Your Team**
*Engagement Reference v1.0*

---

## What DetQ Is, In One Paragraph

DetQ is a query layer that sits on top of your existing corporate bond database. Your analysts type plain English questions — "top 10 tightest spread deals in Q3 rated BBB by Moody's" — and DetQ returns a structured, formatted table in seconds. No Excel. No manual filtering. No analyst time spent on data retrieval. The same query, run tomorrow or next month, returns the same output. That's the product.

---

## The Problem DetQ Solves

Most credit and analytics teams run on a workflow that looks like this:

1. Analyst or client asks a data question
2. Analyst opens Excel or a proprietary data terminal
3. Analyst filters, cross-references, and formats manually
4. Analyst returns results — anywhere from 5 minutes to 45 minutes later

At the junior end, this is 40–60% of an analyst's day. At the senior end, it's constant interruption. For client-facing teams, it's a bottleneck on responsiveness.

DetQ eliminates step 2, 3, and 4. The question goes in; the table comes out.

---

## What Makes DetQ Different From a Chatbot or Generic AI Tool

This question comes up in every evaluation. The answer matters:

**A chatbot reasons and approximates.** If you ask a chatbot "show me tight spread deals," it will decide what "tight" means, pick a threshold, and return results filtered on that assumption — without telling you it made the assumption.

**DetQ enforces and guards.** If you ask DetQ "show me tight spread deals," it returns all deals sorted by spread ascending and lets you see the data. It never invents a threshold you didn't state. If you say "show me deals with spread under 80 bps," it applies that filter exactly as stated.

The distinction is not cosmetic. In a financial context, a system that invents assumptions is a system that can give you wrong answers confidently. DetQ's architecture makes that structurally impossible — the deterministic guard rules enforce it at the code level, not the prompt level.

---

## Engagement Scope

A standard DetQ engagement covers:

| Deliverable | Description |
|---|---|
| Schema Discovery | Inventory of your bond database(s), field mapping to DetQ's normalized vocabulary |
| Guard Rule Configuration | Adaptation of the 10 core guard rules to your firm's specific field naming, flag conventions, and domain vocabulary |
| Integration & Deployment | Installation and configuration of the DetQ pipeline on your infrastructure |
| UI Configuration | Setup of the query interface with your firm's branding and preferred quick-search chips |
| Analyst Validation | Structured UAT session with your analyst team — 30–50 test queries across query types |
| Documentation | Firm-specific schema map, configuration file, and analyst user guide |
| Hypercare Period | 30-day post-deployment support window for query logic refinements |

---

## What We Need From You

DetQ is designed to minimize the burden on your team. The integration requires:

| Your input | Time required |
|---|---|
| Database schema export (column names + types) | 1–2 hours (one-time) |
| Sample of 20–30 real analyst queries | 1–2 hours (one-time) |
| IT access: database read permissions for DetQ service | IT ticket |
| IT access: deployment environment (server or cloud instance) | IT ticket |
| 2 analyst sessions for UAT (validation testing) | 2 × 2 hours |
| Stakeholder review of UAT results | 1 hour |

Total estimated time from your team: **8–12 hours across 4–6 weeks.**

We handle everything else.

---

## What We Do Not Touch

DetQ is a read-only query layer. The engagement explicitly excludes:

- Any modification to your existing database
- Any new data ingestion or enrichment
- Any changes to your data governance or access controls
- Any write operations of any kind

DetQ queries your data. It does not change it.

---

## Deployment Options

| Option | Description | Best for |
|---|---|---|
| On-premises | DetQ runs on your infrastructure. No data leaves your environment. | Air-gapped or regulated environments |
| Private cloud | DetQ deployed on your cloud tenant (AWS, Azure, GCP). | Firms with existing cloud infrastructure |
| Hosted (evaluation only) | DetQ deployed on a sandboxed instance with synthetic data. | Initial evaluation before full integration |

The local LLM option (Ollama + phi-3) supports full on-premises deployment with zero data egress. API-based LLM alternatives are available for firms without GPU infrastructure.

---

## Phased Delivery Summary

The engagement runs in four phases across 6–8 weeks. Full detail in `phased-delivery-plan.md`.

| Phase | Duration | Key Output |
|---|---|---|
| Phase 1: Discovery | Week 1–2 | Schema map, query sample analysis, risk register |
| Phase 2: Configuration | Week 3–4 | Mapped field configuration, guard rule adaptation, dev deployment |
| Phase 3: Validation | Week 5–6 | Analyst UAT, query logic refinements, sign-off |
| Phase 4: Deployment | Week 7–8 | Production deployment, documentation, hypercare begins |

---

## ROI at a Glance

For a team of 6 analysts running 15 queries per day at an average of 20 minutes per query, DetQ recovers approximately **600 analyst-hours per year** — or the equivalent of one full-time analyst position dedicated entirely to data retrieval.

The interactive ROI model is in `/financial-model/roi-calculator.xlsx`. Enter your team's actual numbers to generate a firm-specific impact projection.

---

## Frequently Asked Questions

**Do we need to replace our existing data infrastructure?**
No. DetQ queries what you already have. Your database, your data, your infrastructure — DetQ adds a query layer on top.

**What if our database schema is different from the reference?**
That's expected and fully accounted for. The schema mapping process in Phase 1 documents every difference, and the field mapping configuration in `criteria.js` handles the translation. See `schema-mapping-guide.md` for the full process.

**What if our analysts use different terminology?**
The extraction prompt is configurable. During Phase 2, we review your sample analyst queries and adapt the AI extraction vocabulary to match how your team actually talks about data.

**Can we extend DetQ to support additional markets or asset classes?**
Yes. The pipeline is market-agnostic — the market module defines the database, table, field map, and aggregation logic. Adding a new market (HY, CLO, CMBS) requires a new market module and schema mapping, not changes to the core pipeline.

**What's the long-term support model?**
After the hypercare period, DetQ's deterministic architecture means it requires minimal ongoing maintenance. Guard rules and field mappings are configuration files — updates don't require pipeline changes. New query patterns identified post-deployment can be addressed through guard rule additions without touching core logic.

---

*Full delivery timeline in `phased-delivery-plan.md`.*
*ROI model in `/financial-model/roi-calculator.xlsx`.*
