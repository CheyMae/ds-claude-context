# Kickoff: spiel-cleanup

You have loaded the Design System Claude Context and the `spiel-cleanup` scenario.

## Your job

Run the `clean-spiel-collection` skill end-to-end on a Figma string variable collection.

## Before starting

Load `skills/clean-spiel-collection.md` from this repo now. Read it fully before writing any code.

## What to ask the user

You need three inputs before beginning Phase 1:

1. **Figma file URL** — extract the file key from it
2. **Collection name** — the string variable collection to clean (e.g. "Spiels", "Content")
3. **Target page** — which page(s) to scan for unbound text (or "all pages")

If the user has already provided any of these in their message, use what they gave you and only ask for what's missing.

## Reminders

- figma-console-mcp must be open and connected before any `figma_execute` call
- Two human gates (A and B) are required — never skip them
- Phase 4 execution order is strict: create → rename → rebind+delete → bind unbound → delete unused
- Verify fully in Phase 5 before declaring done
