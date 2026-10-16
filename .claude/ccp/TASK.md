# Task-Format und Tier-Skala

Diese Datei ist die eine Quelle für alle Rollen und Commands der Pipeline.
Konfiguration: `.claude/ccp/pipeline.json`.

---

## Ein Task

Ein Task ist **eine Nahtstelle**, die genau ein Implementer in einem Branch umsetzt.

```markdown
---
id: 007
title: slugify() für Artikel-Titel
status: todo        # todo | doing | review | fix | done | blocked
tier: 2             # 1–5, siehe unten
runner:             # leer = pipeline.json; sonst claude | codex
blocked_by: []      # z. B. [005, 006]
attempts: 0         # Fehlversuche (rote Tests, Review CHANGES, Allowlist-Verstoß)
escalated: false    # hat der Planner schon einmal eine Rückfrage beantwortet?
---

## Ziel
Ein Satz im Imperativ. Braucht er ein „und“, sind es zwei Tasks.

## Warum
Zwei, drei Sätze: was das freischaltet, was bei falscher Umsetzung kaputtgeht.

## Kontext
- Festlegung: `docs/<datei>.md` § <Überschrift>
- Vorbild-Code: `src/<datei>` `function_name()`

## Schnittstelle
Signaturen, Datenformen, Konstanten **wörtlich** – nicht „sinnvolle API entwerfen“.
    # besteht schon, nicht ändern:
    # neu durch diesen Task:

## Dateien
Allowlist. Nur diese Pfade dürfen geändert oder angelegt werden; alles andere ist nur lesbar.
`*` bleibt in einem Verzeichnis, `**` geht beliebig tief, ein Verzeichnis endet mit `/`.
- `src/text/slug.py`
- `tests/test_slug.py`

## Nicht-Ziele
- Die Sache daneben, die man „gleich mit“ reparieren will.

## Fertig wenn
Nur Befehle, die mit exit 0 enden, oder ein Befehl und was er ausgeben muss.
- `./run_tests.sh`
- `grep -c 'def slugify' src/text/slug.py` gibt `1` aus
```

Später angehängte Abschnitte (nur lokal; auf GitHub als Kommentare):
`## Review <n>` (Befunde), `## Rückfrage` / `## Antwort` (Eskalation), `## Log` (eine Zeile pro Schritt).

### Status

```
todo ──> doing ──> review ──> done
  ^        │  ^       │
  │        v  │       v
  └───── fix ─┴──── fix          blocked = braucht dich
```

| Status | bedeutet | nächster Schritt |
|---|---|---|
| `todo` | bereit (wenn alle `blocked_by` erledigt sind) | `/ccp_implement` |
| `doing` | Implementer läuft oder Lauf wurde unterbrochen | `/ccp_implement` setzt fort |
| `review` | Tests grün, Diff in der Allowlist, Commit liegt auf `task/<id>` | `/ccp_review` |
| `fix` | Tests rot, Allowlist verletzt oder Review verlangt Änderungen | `/ccp_implement` (Nachbesserung) |
| `blocked` | Frage an dich | du antwortest, setzt `todo` |
| `done` | gemergt | – |

**Modus local:** Datei `tasks/<id>-<slug>.md`, Status im Frontmatter. Ideen stehen als
`- [ ] …` in `tasks/ideas.md`.
**Modus github:** Issue = Task (Abschnitte im Body, ohne Frontmatter). Labels:
`status:<status>`, `tier:<n>`, optional `runner:<name>`; Ideen sind Issues mit Label `idea`.
`blocked_by` steht als Zeile `Blocked by: #12, #13` im Body. `attempts` = Zahl der Kommentare
mit `<!-- ccp:fix -->` an Issue und PR; `escalated` = es gibt einen Kommentar mit
`<!-- ccp:answer -->`. `done` = PR gemergt, Issue geschlossen.

### Fehlversuche: hochstufen statt schleifen

Jedes Mal, wenn ein Task auf `fix` geht, steigt `attempts` um 1. Danach:
- `attempts == 2` → `tier` um 1 erhöhen (höchstens 5) – das nächste Modell ist stärker.
- `attempts >= 3` → `blocked` mit einer Zusammenfassung der drei Fehlschläge.

### Eskalation

Vier Dinge werden **gefragt, nie gelöst**: die Arbeit braucht eine Datei außerhalb der
Allowlist; der Task widerspricht einem Design-Dokument oder einer festen Entscheidung; die
Schnittstelle passt nicht zum Repo; ein „Fertig wenn“ lässt sich nicht ausführen.
Der Planner beantwortet eine Rückfrage **einmal** (`escalated: true`), die nächste geht an dich (`blocked`).

---

## Tier-Skala

Der Planner setzt `tier` danach, **wie viel Urteilsvermögen** die Aufgabe braucht – nicht
nach Zeilenzahl. Im Zweifel eine Stufe höher; das ist billiger als drei Fehlversuche.

| Tier | Art | Beispiele | Claude | Codex | lokal (später) |
|---|---|---|---|---|---|
| 1 trivial | mechanisch, eine Stelle, kein Entwurf | Tippfehler, Umbenennung, Konstante ändern, Doku-Absatz, Lint-Fix | haiku · high | luna · high | qwen-coder |
| 2 klein | eine Funktion nach vorhandenem Muster | Validierung ergänzen, Getter + Test, Formatierungsfunktion, Testfall nachziehen | sonnet · medium | sol · medium | qwen-coder |
| 3 standard | neue Einheit, eine Nahtstelle, mehrere Dateien | neuer Endpoint, Parser, Adapter nach Vorbild, CLI-Unterbefehl | sonnet · high | sol · medium | – |
| 4 schwer | modulübergreifend oder knifflig | Refactoring über Module, Caching, Nebenläufigkeit, Datenmigration | opus · medium | sol · high | – |
| 5 heikel | Architektur oder unklare Ursache | Sicherheitslogik, schwer reproduzierbarer Bug, neues Datenmodell | opus · high | astra · medium | – |

- **Runner:** Task-Feld `runner`, sonst `runner` aus `pipeline.json`. `auto` nimmt
  `auto_runner` (Standard: Codex für Tier 1–3, Claude für 4–5).
- **Codex-Modelle** stehen als Familie; `/ccp_implement` nimmt jeweils das neueste Modell der
  Familie aus `codex debug models`.
- **Reviewer:** `reviewer` in `pipeline.json`. `reviewer.runner` wählt Claude oder Codex (Standard:
  Codex – spart Claude-Limit); je Runner ein Modell für `default` und einzelne Tiers
  (Codex: astra · medium, Tier 5 astra · high; Claude: Opus · medium, Tier 4 high, Tier 5 xhigh).
  Bei Codex nie dasselbe Modell, das den Task umgesetzt hat – Ausnahme Tier 5: dort prüft
  immer das stärkste Modell (astra · high), auch wenn astra selbst umgesetzt hat.
- **lokal:** `"local": null` ist ein Platzhalter. Ein späterer Skill trägt dort einen Aufruf ein
  (z. B. `codex --oss` mit Ollama) und erweitert `/ccp_implement` um den Runner `local`.
