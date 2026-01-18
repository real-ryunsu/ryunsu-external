# Guide to Agentic Orchestration Architecture

> by Ryunsu Sung @ AWARE LAB

> A production-tested architecture for AI-assisted front-end engineering with minimal human intervention.

**Version:** 5.0  
**Last Updated:** January 2026  
**Audience:** Engineers building agentic development workflows

---

## Table of Contents

1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Architecture Diagram](#architecture-diagram)
4. [Agent Roles](#agent-roles)
5. [Knowledge System](#knowledge-system)
6. [Workflow Lifecycle](#workflow-lifecycle)
7. [Directory Structure](#directory-structure)
8. [Implementation Guide](#implementation-guide)
9. [Patterns & Anti-Patterns](#patterns--anti-patterns)
10. [Adaptation Examples](#adaptation-examples)

---

## Overview

### The Problem

AI agents can write code, but without structure they:
- Repeat the same mistakes across sessions
- Lose context between conversations
- Require constant human correction
- Can't coordinate on complex multi-part tasks

### The Solution

This architecture provides:

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│   ORCHESTRATOR          SUB-AGENTS           KNOWLEDGE             │
│   ─────────────         ──────────           ─────────             │
│   Coordinates           Execute              Accumulates           │
│   Delegates             Implement            Persists              │
│   Validates             Report               Improves              │
│                                                                    │
│   "What to do"          "How to do it"       "What we learned"     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

**Key insight:** Humans detect issues, agents correct them. Once detected, agents extract exact values from source data and implement fixes autonomously.

---

## Core Concepts

### Glossary

| Term | Definition |
|------|------------|
| **Orchestrator** | Main agent that coordinates work but doesn't implement code directly |
| **Sub-agent** | Delegated agent (via Task tool) that reads knowledge and executes tasks |
| **Wave** | A batch of related work processed together |
| **Knowledge** | Permanent files that accumulate learnings (PATTERNS.md, RULES.md) |
| **Cache** | Temporary cross-agent communication, cleared after each wave |
| **Flush** | Extract lessons from cache to knowledge, then clear cache |
| **Checkpoint** | A git commit marking stable state after a wave |

### The Five Principles

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  1. SINGLE ENTRY POINT                                              │
│     └─ One CLAUDE.md file tells agents where to start               │
│                                                                     │
│  2. DELEGATION OVER IMPLEMENTATION                                  │
│     └─ Orchestrator manages; sub-agents code                        │
│                                                                     │
│  3. RAW DATA EXTRACTION                                             │
│     └─ Extract exact values, don't interpret visually               │
│                                                                     │
│  4. CACHE → KNOWLEDGE LIFECYCLE                                     │
│     └─ Temporary findings become permanent patterns                 │
│                                                                     │
│  5. TRUST HIERARCHY                                                 │
│     └─ Sub-agents closest to source data are most trusted           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Diagram

### System Overview

```
                              HUMAN ENGINEER
                           (Exception Handler)
                                    │
                                    │ Identifies issues
                                    │ Approves decisions
                                    ▼
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│                         ORCHESTRATOR AGENT                            │
│                                                                       │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │
│   │  CLAUDE.md  │    │  STATE.md   │    │  CACHE.md   │               │
│   │  (entry)    │    │  (status)   │    │  (comms)    │               │
│   └─────────────┘    └─────────────┘    └─────────────┘               │
│                                                                       │
│   Responsibilities:                                                   │
│   • Check state on session start                                      │
│   • Delegate tasks to sub-agents                                      │
│   • Process sub-agent returns                                         │
│   • Run builds, sync, commit                                          │
│   • Flush cache after each wave                                       │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │   SUB-AGENT     │    │   SUB-AGENT     │    │   SUB-AGENT     │
   │   (Implement)   │    │   (Audit)       │    │   (Fix)         │
   └─────────────────┘    └─────────────────┘    └─────────────────┘
            │                       │                       │
            │  Reads:               │  Reads:               │  Reads:
            │  • RULES.md           │  • RULES.md           │  • RULES.md
            │  • PATTERNS.md        │  • PATTERNS.md        │  • PATTERNS.md
            │  • CACHE.md           │  • CACHE.md           │  • CACHE.md
            │                       │                       │
            │  Writes:              │  Writes:              │  Writes:
            │  • Source code        │  • Audit report       │  • Code fixes
            │  • CACHE.md           │  • CACHE.md           │  • CACHE.md
            │                       │                       │
            └───────────────────────┴───────────────────────┘
                                    │
                                    ▼
                        ┌─────────────────────┐
                        │      CACHE.md       │
                        │   (ephemeral)       │
                        └──────────┬──────────┘
                                   │
                                   │ FLUSH (after wave)
                                   ▼
                        ┌─────────────────────┐
                        │     KNOWLEDGE/      │
                        │   ┌─────────────┐   │
                        │   │ PATTERNS.md │   │
                        │   │ RULES.md    │   │
                        │   │ [domain].md │   │
                        │   └─────────────┘   │
                        │   (permanent)       │
                        └─────────────────────┘
```

### Information Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                         WAVE LIFECYCLE                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  START                                                               │
│    │                                                                 │
│    ▼                                                                 │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐    │
│  │  Check   │────▶│ Delegate │────▶│  Build   │────▶│  Flush   │    │
│  │  State   │     │  Tasks   │     │  Verify  │     │  Cache   │    │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘    │
│       │                │                │                │          │
│       ▼                ▼                ▼                ▼          │
│  STATE.md         Sub-agents        pnpm build      PATTERNS.md     │
│  CACHE.md         execute           passes?         RULES.md        │
│                                                                      │
│    │                                                                 │
│    ▼                                                                 │
│  CHECKPOINT (git commit)                                             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Agent Roles

### Orchestrator

The main agent that coordinates all work.

```
┌─────────────────────────────────────────────────────────────────────┐
│ ORCHESTRATOR                                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ READS           │ NEVER READS        │ TOOLS                        │
│ ────────────    │ ─────────────      │ ─────                        │
│ CLAUDE.md       │ Knowledge files    │ Task (delegation)            │
│ STATE.md        │ Source code        │ Bash (git, build)            │
│ CACHE.md        │                    │ TodoWrite                    │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ PROTOCOL:                                                           │
│                                                                     │
│   On Session Start:                                                 │
│   1. Read STATE.md → Current progress                               │
│   2. Check CACHE.md → Needs flush?                                  │
│   3. Run build → Passes?                                            │
│   4. Report to user                                                 │
│                                                                     │
│   On Task Request:                                                  │
│   1. Create todo list                                               │
│   2. Delegate via Task tool                                         │
│   3. Process sub-agent return                                       │
│   4. Run build, sync, commit                                        │
│                                                                     │
│   After Wave:                                                       │
│   1. Verify build                                                   │
│   2. Flush cache → knowledge                                        │
│   3. Update STATE.md                                                │
│   4. Commit checkpoint                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Sub-Agent

Delegated agents that read knowledge and execute specific tasks.

```
┌─────────────────────────────────────────────────────────────────────┐
│ SUB-AGENT                                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ INPUT CONTRACT (from orchestrator):                                 │
│ ───────────────────────────────────                                 │
│   Component: ButtonGroup                                            │
│   Action: implement | fix | audit                                   │
│   Source: [Figma node, API endpoint, spec file]                     │
│   Context: [additional details]                                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ PROTOCOL:                                                           │
│                                                                     │
│   1. Read Knowledge                                                 │
│      • RULES.md → What rules to follow                              │
│      • PATTERNS.md → Code patterns to use                           │
│      • CACHE.md → Recent findings                                   │
│                                                                     │
│   2. Execute Task                                                   │
│      • Extract raw data from source                                 │
│      • Apply patterns exactly                                       │
│      • Follow rules strictly                                        │
│                                                                     │
│   3. Report Findings                                                │
│      • Append to CACHE.md                                           │
│      • Return structured output                                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ OUTPUT FORMAT:                                                      │
│ ──────────────                                                      │
│   ## Sub-Agent Return: {component}                                  │
│                                                                     │
│   ### Code                                                          │
│   {files created/modified}                                          │
│                                                                     │
│   ### Findings                                                      │
│   | Category | Finding | Correction |                               │
│                                                                     │
│   ### Confidence                                                    │
│   {0-100}% - {reasoning}                                            │
│                                                                     │
│   ### Notes                                                         │
│   {context for orchestrator}                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Trust Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TRUST HIERARCHY                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   MOST TRUSTED ────────────────────────────────────── LEAST TRUSTED │
│                                                                     │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│   │ Raw Source  │    │  Sub-Agent  │    │ Orchestrator│             │
│   │   Data      │───▶│ Extraction  │───▶│ Secondhand  │             │
│   │ (Figma/API) │    │  (direct)   │    │   Info      │             │
│   └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                     │
│   RULE: Sub-agent extractions should be trusted unless:             │
│         1. Build fails with type error                              │
│         2. Visual comparison shows clear mismatch                   │
│         3. User reports issue with specific evidence                │
│                                                                     │
│   When uncertain: ASK USER to verify, don't assume                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Knowledge System

### Cache vs Knowledge

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│     DURING WAVE                           AFTER WAVE               │
│     ───────────                           ──────────               │
│                                                                    │
│     Agent A ──┐                           CACHE.md                 │
│               │                               │                    │
│     Agent B ──┼──▶ CACHE.md              ┌───┴───┐                 │
│               │    (append)              ▼       ▼                 │
│     Agent C ──┘                     PATTERNS  RULES                │
│                                        .md      .md                │
│                                                                    │
│                                        CACHE.md                    │
│                                        (cleared)                   │
│                                                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  CACHE.md (Ephemeral)          │  KNOWLEDGE/ (Permanent)           │
│  ─────────────────────         │  ──────────────────────           │
│  • Temporary findings          │  • Proven patterns                │
│  • Safe for agents to append   │  • Curated rules                  │
│  • Cleared after flush         │  • Accumulates over time          │
│  • Recent corrections          │  • Read by all agents             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Knowledge Files

| File | Purpose | Example Content |
|------|---------|-----------------|
| `PATTERNS.md` | Copy-paste code patterns | Component templates, CSS patterns |
| `RULES.md` | Implementation rules | Accessibility, naming conventions |
| `[domain].md` | Domain-specific mappings | ICONS.md, ENDPOINTS.md, SCHEMAS.md |

### Flush Protocol

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FLUSH PROTOCOL                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Review CACHE.md entries                                         │
│                                                                     │
│  2. For each finding, determine destination:                        │
│                                                                     │
│     ┌──────────────────┬───────────────────────────┐                │
│     │ Finding Type     │ Destination               │                │
│     ├──────────────────┼───────────────────────────┤                │
│     │ Code pattern     │ PATTERNS.md               │                │
│     │ Rule/principle   │ RULES.md                  │                │
│     │ Domain mapping   │ [domain].md               │                │
│     │ Extraction tip   │ EXTRACTION.md (if exists) │                │
│     └──────────────────┴───────────────────────────┘                │
│                                                                     │
│  3. Archive cache entries (optional summary table)                  │
│                                                                     │
│  4. Clear active findings section                                   │
│                                                                     │
│  5. Update STATE.md: Cache Status → CLEAN                           │
│                                                                     │
│  6. Commit: "flush: extract wave-N lessons"                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Workflow Lifecycle

### State Machine

```
┌─────────────────────────────────────────────────────────────────────┐
│                       STATE TRANSITIONS                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                          ┌─────────┐                                │
│                          │  START  │                                │
│                          └────┬────┘                                │
│                               │                                     │
│                               ▼                                     │
│                        ┌─────────────┐                              │
│                        │ Check State │                              │
│                        └──────┬──────┘                              │
│                               │                                     │
│                    ┌──────────┴──────────┐                          │
│                    │                     │                          │
│               Cache Dirty?          Cache Clean                     │
│                    │                     │                          │
│                    ▼                     │                          │
│               ┌─────────┐                │                          │
│               │  Flush  │                │                          │
│               └────┬────┘                │                          │
│                    │                     │                          │
│                    └──────────┬──────────┘                          │
│                               │                                     │
│                               ▼                                     │
│                          ┌─────────┐                                │
│               ┌─────────▶│  Ready  │◀─────────┐                     │
│               │          └────┬────┘          │                     │
│               │               │               │                     │
│               │               ▼               │                     │
│               │          ┌──────────┐         │                     │
│               │          │ Delegate │         │                     │
│               │          └────┬─────┘         │                     │
│               │               │               │                     │
│               │               ▼               │                     │
│               │          ┌──────────┐         │                     │
│               │          │ Process  │         │                     │
│               │          └────┬─────┘         │                     │
│               │               │               │                     │
│               │               ▼               │                     │
│               │          ┌──────────┐         │                     │
│               │          │  Build   │         │                     │
│               │          └────┬─────┘         │                     │
│               │               │               │                     │
│               │    Pass ──────┴────── Fail    │                     │
│               │      │                  │     │                     │
│               │      ▼                  ▼     │                     │
│               │  ┌───────┐         ┌───────┐  │                     │
│               │  │ Sync  │         │  Fix  │──┘                     │
│               │  └───┬───┘         └───────┘                        │
│               │      │                                              │
│               │      ▼                                              │
│               │  ┌────────┐                                         │
│               │  │ Commit │                                         │
│               │  └───┬────┘                                         │
│               │      │                                              │
│               │      ▼                                              │
│               │  Wave Complete?                                     │
│               │      │                                              │
│               │  Yes │  No ─────────────────────┐                   │
│               │      │                          │                   │
│               │      ▼                          │                   │
│               │  ┌─────────┐                    │                   │
│               └──│  Flush  │                    │                   │
│                  └─────────┘                    │                   │
│                                                 │                   │
│                  ┌──────────────────────────────┘                   │
│                  │                                                  │
│                  └──────▶ (next task in wave)                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Verification Loop

When issues are found after implementation:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      VERIFICATION LOOP                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   User Reports Issue                                                │
│         │                                                           │
│         ▼                                                           │
│   ┌───────────────────┐                                             │
│   │    Orchestrator   │                                             │
│   │  (same session)   │                                             │
│   └─────────┬─────────┘                                             │
│             │                                                       │
│             │  Continue with same sub-agent (session_id)            │
│             │                                                       │
│             ▼                                                       │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │                    VERIFICATION REQUEST                       │ │
│   │                                                               │ │
│   │  Issue: Badge not bottom-aligned                              │ │
│   │  Suspected Node: 71:1005                                      │ │
│   │                                                               │ │
│   │  Please:                                                      │ │
│   │  1. Re-fetch node with depth=100                              │ │
│   │  2. Show raw property values                                  │ │
│   │  3. Explain extraction logic                                  │ │
│   │  4. Confirm or correct                                        │ │
│   │                                                               │ │
│   └───────────────────────────────────────────────────────────────┘ │
│             │                                                       │
│             ▼                                                       │
│   ┌───────────────────┐                                             │
│   │    Sub-Agent      │                                             │
│   │  Re-extracts      │                                             │
│   │  Explains logic   │                                             │
│   └─────────┬─────────┘                                             │
│             │                                                       │
│             ▼                                                       │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │                   VERIFICATION RESPONSE                       │ │
│   │                                                               │ │
│   │  Raw Values: layoutMode=VERTICAL, alignItems=MAX              │ │
│   │  My Extraction: items-end justify-end                         │ │
│   │  Analysis: Missing self-end because parent...                 │ │
│   │  Correction: Add self-end to wrapper                          │ │
│   │                                                               │ │
│   └───────────────────────────────────────────────────────────────┘ │
│             │                                                       │
│             ▼                                                       │
│   Apply Fix → Update CACHE → Continue                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Directory Structure

### Recommended Layout

```
/project-root
│
├── CLAUDE.md                    ← Orchestrator entry point
├── ARCHITECTURE.md              ← This document (human reference)
│
├── .agent/                      ← Agent workspace
│   │
│   ├── README.md                ← Sub-agent routing guide
│   ├── STATE.md                 ← Current state, metrics, health
│   ├── CACHE.md                 ← Cross-agent cache (ephemeral)
│   │
│   ├── knowledge/               ← Permanent knowledge files
│   │   ├── PATTERNS.md          ← Code patterns (copy-paste ready)
│   │   ├── RULES.md             ← Implementation rules
│   │   └── [domain].md          ← Domain-specific (ICONS.md, API.md)
│   │
│   ├── prompts/                 ← Agent protocols
│   │   └── sub-agent.md         ← Sub-agent protocol & input contract
│   │
│   └── specs/                   ← Component/feature specifications
│       └── {component}.spec.md
│
├── src/                         ← Source code
│   └── components/
│
└── [build-tools]                ← package.json, tsconfig, etc.
```

### File Reference

| File | Owner | Purpose | Lifecycle |
|------|-------|---------|-----------|
| `CLAUDE.md` | Orchestrator | Entry point, glossary, protocol | Stable |
| `STATE.md` | Orchestrator | Progress, metrics, health | Updated per wave |
| `CACHE.md` | Sub-agents (write), Orchestrator (flush) | Cross-agent findings | Cleared after flush |
| `PATTERNS.md` | Sub-agents (read) | Code patterns | Accumulates |
| `RULES.md` | Sub-agents (read) | Decision rules | Accumulates |
| `sub-agent.md` | Sub-agents (read) | Protocol & contract | Stable |

---

## Implementation Guide

### Minimal Setup Checklist

```
[ ] CLAUDE.md at project root
    ├── Glossary of terms
    ├── Orchestrator protocol (session start, task handling, wave end)
    ├── File ownership table
    └── Quick commands

[ ] .agent/STATE.md
    ├── Current wave/progress
    ├── Cache status (CLEAN/DIRTY)
    └── Session history

[ ] .agent/CACHE.md
    ├── Active findings table
    ├── How to use (for agents)
    └── Flush protocol reference

[ ] .agent/knowledge/PATTERNS.md
    └── Initial code patterns

[ ] .agent/knowledge/RULES.md
    └── Initial implementation rules

[ ] .agent/prompts/sub-agent.md
    ├── Input contract (required/optional params)
    ├── Protocol (read → execute → report)
    ├── Output format
    ├── Error handling table
    └── Checklist before returning
```

### Validation Questions

A new agent should be able to answer:

| Question | Answer |
|----------|--------|
| Where do I start? | `CLAUDE.md` |
| What do terms mean? | Glossary in `CLAUDE.md` |
| How do I delegate? | Task tool with sub-agent.md protocol |
| What if something fails? | Error handling in sub-agent.md |
| Where do findings go? | Append to `CACHE.md` |
| When does cache clear? | After wave, via flush protocol |

---

## Patterns & Anti-Patterns

### Patterns (Do This)

```
┌─────────────────────────────────────────────────────────────────────┐
│                           PATTERNS                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Extract raw values from source data                              │
│    Sub-agent fetches JSON/AST, not screenshots                      │
│                                                                     │
│  ✓ Delegate with explicit contracts                                 │
│    Component: X, Action: Y, Source: Z                               │
│                                                                     │
│  ✓ Trust sub-agent extractions                                      │
│    They have direct source access                                   │
│                                                                     │
│  ✓ Flush cache after every wave                                     │
│    Lessons become permanent                                         │
│                                                                     │
│  ✓ Check CACHE before implementing                                  │
│    Avoid repeating recent mistakes                                  │
│                                                                     │
│  ✓ Use verification loop for issues                                 │
│    Continue conversation, don't restart                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Anti-Patterns (Don't Do This)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ANTI-PATTERNS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✗ Orchestrator reads knowledge files                               │
│    → Delegate to sub-agent instead                                  │
│                                                                     │
│  ✗ Sub-agent edits knowledge directly                               │
│    → Append to CACHE, orchestrator flushes                          │
│                                                                     │
│  ✗ Multiple entry points                                            │
│    → Single CLAUDE.md at root                                       │
│                                                                     │
│  ✗ Visual interpretation of designs                                 │
│    → Extract exact values from data source                          │
│                                                                     │
│  ✗ Orchestrator "corrects" sub-agent values                         │
│    → Trust extraction, verify with user if uncertain                │
│                                                                     │
│  ✗ Skip cache flush                                                 │
│    → Knowledge doesn't accumulate, lessons lost                     │
│                                                                     │
│  ✗ Missing input contract                                           │
│    → Sub-agents guess parameters, produce wrong results             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Intervention Curve

```
Human Intervention Over Time:

  ████████████████████  Early waves (high intervention)
  │
  ████████████          Middle waves (medium)
  │
  ████                  Late waves (low - exceptions only)
  │
  ──────────────────────────────────────────▶ Time

Why it decreases:
• Patterns accumulate in knowledge files
• Common mistakes are documented
• Agents learn from CACHE findings
• Rules cover edge cases
```

---

## Adaptation Examples

### For Design System (Figma → React)

```
.agent/knowledge/
├── PATTERNS.md      ← Component code patterns
├── RULES.md         ← Styling rules, accessibility
├── ICONS.md         ← Figma icon → library mapping
└── EXTRACTION.md    ← Figma JSON extraction protocol

Key additions:
• Figma API access in sub-agent.md
• Icon mapping workflow
• Design token extraction
```

### For API Integration

```
.agent/knowledge/
├── PATTERNS.md      ← API call patterns, error handling
├── RULES.md         ← Authentication, rate limits
├── ENDPOINTS.md     ← Endpoint documentation
└── SCHEMAS.md       ← Request/response schemas

Key additions:
• API credentials location
• Example curl commands
• Error response handling
```

### For Testing

```
.agent/knowledge/
├── PATTERNS.md      ← Test patterns (unit, integration, e2e)
├── RULES.md         ← What to test, coverage requirements
├── FIXTURES.md      ← Test data patterns
└── MOCKS.md         ← Mocking strategies

Key additions:
• Test runner commands
• CI integration notes
• Coverage thresholds
```

### For Full-Stack Features

```
.agent/knowledge/
├── PATTERNS.md      ← Frontend + backend patterns
├── RULES.md         ← Architecture decisions
├── DATABASE.md      ← Schema conventions
└── MIGRATIONS.md    ← Migration patterns

Key additions:
• Cross-layer coordination
• Transaction handling
• Rollback procedures
```

---

## Quick Reference

### CLAUDE.md Template

```markdown
# [Project] Orchestrator

> **You are in `/project`.** This is the workspace root.

## Glossary
| Term | Definition |
|------|------------|
| Wave | Batch of related work |
| ... | ... |

## Orchestrator Protocol

### On Session Start
1. Read .agent/STATE.md
2. Check cache status
3. Run build
4. Report to user

### On Task Request
1. Delegate to sub-agent
2. Process return
3. Build, sync, commit

### After Wave
1. Verify build
2. Flush cache
3. Update state
4. Commit

## File Ownership
| File | Owner | Purpose |
|------|-------|---------|
| ... | ... | ... |
```

### Sub-Agent Delegation Template

```markdown
## Sub-Agent Task

**Component:** [name]
**Action:** implement | fix | audit
**Source:** [figma node / api endpoint / spec file]
**Context:** [additional details]

Read protocol at: .agent/prompts/sub-agent.md
Read knowledge at: .agent/knowledge/
Append findings to: .agent/CACHE.md
Output code to: src/components/
```

---

## Conclusion

This architecture transforms AI agents from task executors to semi-autonomous engineers. The key principles:

1. **Clear entry points** — Agents know exactly where to start
2. **Explicit contracts** — Sub-agents know what parameters to expect
3. **Separated concerns** — Cache vs knowledge, orchestrator vs sub-agent
4. **Accumulating knowledge** — Every lesson becomes permanent
5. **Trust hierarchy** — Sub-agents closest to source are most trusted

The goal is not to eliminate human engineers but to **elevate their role** from implementation details to decision-making on genuinely ambiguous problems.

---

*Architecture v5.0 — Tested on 16-component design system implementation with 7 iterative waves.*
