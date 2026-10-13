---
description: Genau einen sinnvollen Pipeline-Schritt ausführen. Mit /loop /ccp_next läuft die Pipeline durch.
---

# /ccp_next

Lies `.claude/ccp/pipeline.json` (`mode`) und sieh dir den Zustand an:
- **local:** Frontmatter aller `tasks/*.md`, offene Zeilen in `tasks/ideas.md`.
- **github:** `gh issue list --state open --json number,title,labels,body --limit 200`.

Wähle den **ersten** passenden Schritt, führe ihn aus und hör danach auf:

1. Ein Task in `review` → `/ccp_review <id>`
2. Ein Task in `fix` → `/ccp_implement <id>`
3. Ein Task in `doing` (Lauf wurde unterbrochen) → `/ccp_implement <id>`
4. Ein Task in `todo`, dessen `blocked_by` alle `done` bzw. geschlossen sind → `/ccp_implement <id>`
5. Eine offene Idee ohne `status:blocked` bzw. ohne offene Frage → `/ccp_plan`
6. Sonst: nichts zu tun.

Bei mehreren Kandidaten innerhalb einer Stufe die kleinste ID. Führe den Schritt aus, indem du
den jeweiligen Command (`.claude/commands/ccp_*.md`) liest und ihm folgst.

**Nur eine Pipeline-Session gleichzeitig.** `doing` bedeutet „unterbrochen“ nur, wenn keine
andere Session an dem Task arbeitet.

## Abschluss

Eine Zeile: was getan wurde und mit welchem Ergebnis. Dazu, falls vorhanden, **Braucht dich:**
alle `blocked`-Tasks und blockierten Ideen mit ihrer Frage.

## In /loop

Läufst du als `/loop /ccp_next` (selbst getaktet):
- Schritt ausgeführt → nächster Durchlauf nach 60 Sekunden.
- Nichts zu tun, aber Tasks sind `blocked` → nächster Durchlauf nach 30 Minuten (vielleicht
  hast du dann geantwortet).
- Nichts zu tun und nichts blockiert → Loop beenden und melden „Pipeline leer“.
