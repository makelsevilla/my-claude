---
name: session-handoff
description: >
  Recognize and respond to session management shorthand commands typed by the user.
  Trigger this skill whenever the user types any of these commands: /handoff, /recap,
  /decisions, or /todo. These commands are used to summarize, extract, or
  package the current conversation for handoff, review, or continuation in a new session.
  Use this skill any time a message consists of or starts with a forward-slash command
  related to session management, even if no other explanation is given.
---

# Session Handoff Skill

This skill defines a set of shorthand commands for managing and summarizing chat sessions.
These commands are designed to be **domain-agnostic** and usable by any team member,
regardless of what topic or project the session is about.

---

## Commands

### `/handoff`
Generate a structured summary suitable for pasting into a new session to continue where this one left off.

**Output format:**
- **Goal** — What the session was trying to accomplish
- **Key Decisions** — Important choices made and the reasoning behind them
- **Current State** — Where things stand right now
- **Open Questions** — Unresolved items to revisit next session
- **Constraints** — Any rules, preferences, requirements, or context mentioned
- **Next Steps** — Suggested starting point for the next session

Keep it concise but complete enough to fully re-prime a new session without re-reading the full conversation.

---

### `/recap`
A brief, high-level summary of what happened this session.

**Output format:**
3–5 bullet points covering the main topics discussed and outcomes reached. No deep detail — just enough to remember what this session was about.

---

### `/decisions`
Extract only the decisions made during this session.

**Output format:**
A numbered list. For each decision: what was decided, and briefly why (if mentioned).

---

### `/todo`
Extract all open action items, follow-ups, or unresolved tasks mentioned during the session.

**Output format:**
A checklist. Each item should be actionable and self-contained — readable without needing the full conversation for context.

---

## Notes for Claude

- These commands may appear alone on a line (e.g., the user sends just `/handoff`) or at the start of a message.
- Respond only with the formatted output — no preamble like "Sure! Here's your handoff summary."
- Adapt the output to the actual conversation content. If a section has nothing to report (e.g., no open questions), write "None noted" rather than omitting the section.
- Keep language neutral and domain-agnostic. Avoid assuming the session was technical, creative, or otherwise — reflect what actually happened.
