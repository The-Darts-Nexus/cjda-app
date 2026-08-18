# 14. Handover Improvement Backlog

## Purpose

The current handover is sufficient for understanding the CJDA business rules, feature set, and development roadmap. However, it is not yet a complete developer handover.

The items below should be progressively documented to make this file the single source of truth for future development sessions and AI-assisted development.

---

## 14.1 Repository Structure Documentation

Future update required:

Document the application structure, including:

```text
src/
components/
pages/
services/
hooks/
utils/
```

For each major file document:

- Purpose
- Key responsibilities
- Dependencies

Example:

```text
season-log.ts
Responsible for standings generation and season calculations.

night-setup.ts
Responsible for block entry, calculations and submission.

supabase.ts
Centralized Supabase client and database access layer.
```

Reason:

A future developer or AI session should be able to locate functionality immediately without reverse-engineering the repository.

---

## 14.2 Core Application Flow

Future update required:

Document:

```text
User Login
→ Home Page
→ Night Setup
→ Submit Night
→ Save Results
→ Update Standings
→ Update Season Log
```

Include:

- Page relationships
- Data flow
- Submission workflow

Reason:

Understanding the application flow is critical when debugging calculations and data synchronisation issues.

---

## 14.3 Core Database Tables

Future update required:

Document all primary tables and their purpose.

Minimum tables expected:

```text
players
seasons
night_types
nights
blocks
block_players
results
accolades
competitions
competition_entries
competition_results
attendance_events
attendance_records
```

For each table capture:

- Table purpose
- Key fields
- Relationships
- Any important constraints

Reason:

Database structure is currently tribal knowledge and not captured in the handover.

---

## 14.4 Database Relationship Map

Future update required:

Create a simple relationship diagram.

Example:

```text
Season
 └─ Nights
      └─ Blocks
           └─ Results
                └─ Accolades

Players
 └─ Results
 └─ Accolades
 └─ Attendance
```

Reason:

Will simplify troubleshooting and future feature development.

---

## 14.5 Critical Functions

Future update required:

Document all major functions.

Examples:

```text
calcPlayerSeasonStats()

Purpose:
Calculates Season Log standings.

Uses:
- results
- blocks
- accolades

Returns:
- season points
- selection points
- attendance totals
```

```text
submitNight()

Purpose:
Commits a completed night to Supabase.

Creates:
- blocks
- block_players
- results
- accolades
```

Document:

- Inputs
- Outputs
- Side effects

Reason:

Prevents accidental changes to critical calculation logic.

---

## 14.6 Critical Business Assumptions

Future update required:

Document all assumptions that must never change without approval.

Known examples:

```text
League Night and Competition Night cannot occur on the same date.

DIDO Playoff rounds never award season points.

DIDO Playoff rounds never award selection points.

Competition accolades always contribute to player statistics.
```

Reason:

These rules drive multiple calculation engines throughout the application.

---

## 14.7 Architectural Decision Log

Future update required:

Maintain a running decision register.

Example:

### 2026-08-14

Decision:
Separate Season Date Range and Selection Date Range.

Reason:
Both rankings operate independently and can overlap.

Impact:
seasons table requires separate date fields.

Future entries should follow the same structure:

- Date
- Decision
- Reason
- Impact

Reason:

Provides historical context for future development decisions.

---

## 14.8 Known Issues Register

Future update required:

Track unresolved production issues.

Template:

```text
Issue:
Description

Impact:
Low / Medium / High

Workaround:
Current workaround

Status:
Open / In Progress / Resolved
```

Reason:

Avoid losing knowledge between development sessions.

---

## 14.9 Lessons Learned

Future update required:

Record significant technical discoveries.

Example:

### RLS Permissions Incident

Problem:
Data appeared empty throughout the application.

Cause:
Missing Supabase grants.

Resolution:
Explicit GRANT statements added for all required roles.

Lesson:
Check permissions before debugging frontend logic.

Reason:

Can save hours of repeated troubleshooting.

---

## 14.10 Domain Glossary

Future update required:

Document all CJDA terminology.

Examples:

### Season Points

Used for:
- Prize Giving Standings

Formula:
(GW × 2) + Legs

### Selection Points

Used for:
- National Team Selection

Formula:
(GW × 2) + Legs

### DIDO

Competition format consisting of:

- Block Phase
- Playoff Phase

Only Block Phase contributes match points.

### GDF Calendar

Attendance tracking mechanism for federation events.

Reason:

Allows future developers and AI assistants to immediately understand the domain language used throughout the project.

---

## 14.11 Target State of Handover

Goal:

This document should eventually contain enough information that a developer can:

- Understand CJDA business rules.
- Understand database architecture.
- Understand repository structure.
- Understand data flow.
- Understand historical decisions.
- Understand known issues.
- Resume development after months away from the project.
- Assist with bug fixing without requiring additional onboarding.

Current assessment:

Business Documentation: 9/10
Technical Documentation: 6/10

Target:

Business Documentation: 10/10
Technical Documentation: 10/10
