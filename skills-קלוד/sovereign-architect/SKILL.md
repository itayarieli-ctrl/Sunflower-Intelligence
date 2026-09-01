---
name: sovereign-architect
description: Enterprise-grade execution protocol for Lovable projects — atomic build order, security fortress rules, component architecture standards, and pre-flight verification. Use when implementing features, refactoring, or debugging.
---

# Sovereign Architect — Execution Protocol

## Build order (atomic batches, to limit regression)
1. Backend core — tables, then RLS policies, then edge functions
2. Data layer — generated types, Zod schemas, React Query hooks
3. UI — isolated presentational components first
4. Integration — wire data to UI, then interactivity

## Component architecture
- Single responsibility: split any component doing fetching AND rendering AND complex state
- Composition over prop drilling beyond three levels
- Files over 150 lines: propose a refactor

## Defensive programming
- Validate every input with Zod: edge function params, form data, URL params
- Error boundaries around major feature blocks — never a white screen
- Every mutation shows loading, success, and human-readable error states

## Role-based response
- "slow" → audit re-renders, memoize, virtualize lists, hunt N+1 queries
- "save settings" → dedicated table, 1:1 to auth.users, RLS policies
- "make it professional" → spacing tokens, type scale, WCAG contrast

## Pre-flight checklist
- [ ] Stacks correctly at 320px
- [ ] Data persists to the database, not local state
- [ ] Console clean — zero warnings
- [ ] RLS policies active
