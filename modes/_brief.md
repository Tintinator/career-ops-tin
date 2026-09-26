# Tin La — Triage Brief

<!-- ============================================================
     THIS FILE IS YOURS. It is USER LAYER — never auto-updated by
     `node update-system.mjs`.
     ============================================================ -->

## Identity
Backend Software Engineer — 5 yrs, closer to high SWE2 than Senior. Targeting high SWE2–low Senior level (not Staff+, not new-grad); JDs stating 3-5 yrs are a better fit than 5+ yrs. Based in Houston, TX, actively targeting relocation to NYC, Seattle, or the Bay Area (open to relocation within the US only). US citizen/authorized, no sponsorship needed. US-only search — non-US roles are out of scope. Backend/infra/distributed-systems focus — not looking for AI/ML roles.

## Target Archetypes
| # | Archetype | What they buy (your proof) |
|---|-----------|----------------------------|
| 1 | **Backend Software Engineer** | REST API ownership, microservice migrations, access control (Cox Automotive, Samsung) |
| 2 | **Infrastructure / Platform Engineer** | Owned emergency shutdown + NDU features on a storage system; node health monitoring, cluster recovery |
| 3 | **Site Reliability Engineer** | Primary on-call for the two highest-risk features in a storage cluster; 326 days continuous uptime for a partner customer |

<!-- Analog archetypes: Distributed Systems Engineer, Storage/Systems Engineer -->

## Proof Points (use exact metrics in matching)
- Prevented 8x16 cluster data loss by building an automatic emergency shutdown feature + real-time node health monitoring
- Achieved zero-downtime cluster updates (NDU), resulting in 326 days of continuous uptime for a partner customer
- Increased cluster fault tolerance, validated with 93 days of I/O uptime under resilience testing
- Reduced engineer onboarding time by 50% via updated Confluence guide

## Comp Strategy
| Target | Requirement |
|--------|-------------|
| ~$150K | Base case, any location (open to relocation) |
| $180K+ | Higher intensity / on-call heavy acceptable |

**Hard floor: $140K. Below that, FAIL regardless of other signals.**

## Location Scoring
- Non-US role → **hard DQ** (see below), regardless of remote policy
- On-site/hybrid in **NYC, Seattle, or Bay Area** → **5.0** (target relocation markets)
- Fully remote (US) / async-first → **5.0**
- Regular hybrid or on-site, local (Houston, no move) → **5.0**
- On-site/hybrid requiring relocation to a non-target US metro → **3.0**
- High travel (>25%) → **deduct 0.5–1.0**

## Hard DQ Criteria — instant FAIL (< 3.0)
- Role based outside the United States (US-only search)
- People-management role (Engineering Manager, Director, VP, Head of) — IC track only, regardless of comp
- Staff, Principal, or above — too senior for target level (high SWE2–low Senior only)
- New-grad / entry-level role — too junior for target level
- AI/ML-focused role (AI Engineer, ML Engineer, Applied AI/ML, AI Research, LLM/NLP roles) — not looking for AI/ML work, even at an AI-industry company, if the role itself isn't backend/infra
- Primary hands-on skill outside backend/infra/distributed systems (e.g. pure frontend, mobile-only, embedded)
- Stated comp ceiling below $140K
- Requires an active security clearance Tin does not hold

## Quick Scoring Guide

Bands are relative to `triage_threshold` (`config/profile.yml → pipeline.triage_threshold`,
default **3.5**), matching the verdict table in `modes/triage.md` — so a score at or
above the threshold is PASS, and only the band below it is MARGINAL.

| Score | Verdict | What it means |
|-------|---------|---------------|
| ≥ threshold (default 3.5) | **PASS** | Clears the bar — strong archetype + comp + location, gaps bridgeable |
| 3.0 – (threshold − 0.1) | **MARGINAL** | Borderline — shown to user as one line |
| < 3.0 | **FAIL** | Does not clear the bar — filtered |

## Soft Red Flags (−0.5 each, additive)
- Role emphasizes greenfield product work over reliability/infra ownership
- No on-call/incident-response component at all (mismatch with track record)

**Level-fit note (not a deduction):** a JD stating "3-5 years" is a better level match than one stating "5+ years" — Tin is closer to a high SWE2 than a Senior. Don't penalize "5+ yrs" postings, just don't score them as a tighter level match than a 3-5 yr posting.

## Priority Override List — always return PASS regardless of score
(none yet)
