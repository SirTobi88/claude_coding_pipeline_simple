---
description: Idee in Tasks zerlegen (Planner). Ohne Argument alle offenen Ideen.
argument-hint: "[idee]"
---

# /ccp_plan

Lies `.claude/ccp/pipeline.json` (`mode`).

1. **Ideen sammeln**
   - Argument gesetzt (`$ARGUMENTS`): das ist die Idee. Lokal hängst du sie vorher als
     `- [ ] <idee>` an `tasks/ideas.md` an; auf GitHub legst du ein Issue mit Label `idea` an
     (`gh issue create --title "<idee>" --body "<idee>" --label idea`).
   - Kein Argument: lokal alle `- [ ]`-Zeilen aus `tasks/ideas.md`; GitHub
     `gh issue list --label idea --state open --json number,title,body`.
   - Keine Idee offen → melden, Ende.
2. **Planner starten**, je Idee einmal, nacheinander: Agent `planner` mit dem Prompt
   `Modus plan. mode=<mode>. Idee: <text>` (GitHub zusätzlich `Idee-Issue #<n>`).
3. **Melden:** je Idee die Rückgabe des Planners. Bei `FRAGE:` bleibt die Idee offen; lokal
   schreibst du die Frage eingerückt unter die Idee in `tasks/ideas.md`, auf GitHub als
   Kommentar ans Idee-Issue und das Label `status:blocked` dazu.
