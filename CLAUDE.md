# <Projektname> – Projektregeln

<!-- EDIT: ein, zwei Sätze, was das Projekt ist, Stack, Zielplattformen. Jeder Agent liest
diese Datei bei jedem Lauf – halte sie kurz: Verweise, Regeln, Entscheidungen. Details
gehören in die Dokumente, auf die sie zeigt. -->

## Regel null: die Design-Dokumente gelten

Vor dem Ändern eines Systems sein Design-Dokument lesen. Würde eine Änderung einer
dokumentierten Entscheidung widersprechen: **sagen und fragen**, nicht still abweichen. Ändert
sich eine Entscheidung wirklich, das Dokument in derselben Änderung nachziehen.

| Bereich | Dokument |
|---|---|
| Task-Format, Tier-Skala | `.claude/ccp/TASK.md` |
| *(deine Design-Dokumente)* | |

## Feste Entscheidungen

<!-- EDIT: was man leicht aus Versehen verletzt und nicht ohne Rückfrage geändert werden darf.
Je ein, zwei Zeilen. -->

- *(Entscheidung)*

## Arbeitsweise

- Codeänderungen laufen über Tasks der ccp-Pipeline (`/ccp_plan`, `/ccp_next`). Ein Task ist
  der Auftrag: **Dateien** ist eine Allowlist, **Nicht-Ziele** bleiben unberührt, **Fertig wenn**
  wird ausgeführt, nicht behauptet.
- Niemand beurteilt seine eigene Arbeit: der Reviewer ist ein anderer Agent als der Implementer,
  und gemergt wird nur nach seinem OK und grünen Tests.
- Parallele Arbeit nur in Worktrees (`.worktrees/<id>`), nie im Hauptverzeichnis.
- Commits im Stil `feat(bereich): …`, `fix(bereich): …`, `docs(…)`, `test(…)`, `chore(…)`.
- Testbefehl: `test_command` in `.claude/ccp/pipeline.json`. Sein Exit-Code ist das Ergebnis.
