# claude coding pipeline simple (ccp)

> Die schlanke Schwester von
> [claude_coding_pipeline](https://github.com/SirTobi88/claude_coding_pipeline). Gleiche
> Grundlogik (Planner → Implementer → Reviewer, Tasks mit Allowlist und prüfbarem „Fertig wenn“),
> aber ohne Hilfsskripte, Hooks, Bot-Account und Branch-Protection-Pflicht. Wer GitHub als
> erzwungenes Regelwerk mit Reviewer-Bot und Zeitplan will, nimmt die große Variante.

Eine schlanke Pipeline für Claude Code: ein **Planner** zerlegt Ideen in kleine Tasks und
schätzt ihre Schwierigkeit, ein **Implementer** – Claude oder Codex, Modell passend zur
Schwierigkeit – setzt jeden Task in einem eigenen Worktree um, ein **Reviewer** prüft ihn,
und bei OK wird gemergt.

Nur Markdown und eine JSON-Datei: drei Agents, sechs Commands, keine Skripte, kein Python.
Zwei Modi: **local** (Tasks als Dateien, Merge im lokalen Repo) und **github** (Issues, PRs,
Merge auf GitHub – funktioniert auch mit GitHub Free).

```
Idee ──/ccp_plan──> Tasks (mit Tier) ──/ccp_implement──> Worktree + Tests ──/ccp_review──> Merge
                                           ^                                   │
                                           └────────── fix (max. 3, dann du) ──┘
```

## Rollen

| Rolle | Datei | Modell |
|---|---|---|
| Planner | `.claude/agents/planner.md` | Opus |
| Implementer | `.claude/agents/implementer.md` | frei – nach Tier (Claude oder Codex) |
| Reviewer | `.claude/agents/reviewer.md` | Opus, bei Tier 5 Fable |

Rückfragen vom Implementer beantwortet der Planner **einmal**; die nächste geht an dich.

## Tier-Skala

Der Planner setzt je Task ein Tier. Das Modell steht nur in `.claude/ccp/pipeline.json` –
neue Modelle trägst du dort ein, nirgends sonst.

| Tier | Art | Beispiel | Claude | Codex |
|---|---|---|---|---|
| 1 | trivial | Umbenennung, Konstante, Doku | haiku · low | luna · low |
| 2 | klein | Funktion + Test nach Muster | sonnet · medium | terra · medium |
| 3 | standard | neuer Endpoint, Parser, Adapter | sonnet · high | sol · medium |
| 4 | schwer | Refactoring über Module, Nebenläufigkeit | opus · high | sol · high |
| 5 | heikel | Sicherheitslogik, unklarer Bug, Datenmodell | fable · high | astra · high |

`"runner": "auto"` nimmt Codex für Tier 1–3 und Claude für 4–5 (`auto_runner`). Ein Task kann
per `runner:` übersteuern. Nach zwei Fehlversuchen steigt das Tier um eins, nach drei wird der
Task an dich übergeben. Details und Beispiele: [.claude/ccp/TASK.md](.claude/ccp/TASK.md).

Lokale Modelle (z. B. qwen-coder über `codex --oss` + Ollama) sind vorgesehen (`"local": null`
je Tier), aber noch nicht eingebaut – das kommt später als eigener Skill.

## Commands

| Command | macht |
|---|---|
| `/ccp_setup local\|github` | richtet das Repo ein (Testbefehl, Ordner bzw. Labels, CI, optional Branch Protection) |
| `/ccp_plan ["idee"]` | Idee → Tasks; ohne Argument alle offenen Ideen |
| `/ccp_implement <id>` | Task umsetzen lassen, prüfen (Allowlist, Tests), übergeben |
| `/ccp_review <id>` | Review; bei OK mergen, sonst zurück auf `fix` |
| `/ccp_next` | genau einen sinnvollen Schritt ausführen |
| `/ccp_status` | Überblick und **Braucht dich** |

**Vollautomatisch:** `/loop /ccp_next`. Jeder Durchlauf erledigt einen Schritt (review → fix →
todo → Ideen planen). Blockierte Tasks werden übersprungen und im Status gelistet; ist nichts
mehr zu tun, beendet sich der Loop.

## Einbauen in ein Projekt

1. In das Projekt kopieren: `.claude/agents/`, `.claude/commands/`, `.claude/ccp/`,
   `.claude/settings.json` (gibt es schon eine, die `permissions` zusammenführen), `CLAUDE.md`
   (als Vorlage, falls keine da ist). Für den github-Modus zusätzlich `.github/`.
2. In Claude Code im Projekt: `/ccp_setup local` oder `/ccp_setup github`.
3. `/ccp_plan "was du willst"`, dann `/ccp_next` von Hand oder `/loop /ccp_next`.

### Voraussetzungen

- Claude Code, git. Für Codex: `codex` CLI, angemeldet.
- github-Modus: `gh`, angemeldet. Kein zweiter Account nötig.

### GitHub Free

| | öffentliches Repo | privates Repo |
|---|---|---|
| Issues, PRs, Merge über `gh` | ✅ | ✅ |
| Actions (`ci.yml`) | ✅ | ✅ 2.000 Min./Monat |
| Branch Protection (Pflicht-Check `ci`) | ✅ | ❌ (ab Pro) |

Die Pipeline verlässt sich nie auf Branch Protection: Sie merged erst, wenn der Reviewer OK
gesagt hat, der Testbefehl lokal grün ist und alle Checks des PRs grün sind. Branch Protection
ist nur ein zusätzliches Netz dafür, dass auch sonst niemand roten Code merged.
Ohne Actions: `"ci": false` – dann zählt nur der lokale Testlauf.

## Antworten, wenn die Pipeline fragt

- **local:** in `tasks/<id>-….md` die Antwort unter `## Antwort` schreiben, `status: todo` setzen.
- **github:** Kommentar ins Issue, Label `status:blocked` durch `status:todo` ersetzen.

Anhalten: Loop stoppen. Laufende Agents beenden ihren Schritt; ein unterbrochener Task steht
auf `doing` und wird beim nächsten `/ccp_next` fortgesetzt.

## Dateien

```
.claude/
  agents/      planner.md, implementer.md, reviewer.md
  commands/    ccp_setup, ccp_plan, ccp_implement, ccp_review, ccp_next, ccp_status
  ccp/         pipeline.json (Konfiguration), TASK.md (Task-Format + Tier-Skala)
  settings.json  Freigaben für den unbeaufsichtigten Loop
.github/       workflows/ci.yml, ISSUE_TEMPLATE/task.md   (nur github-Modus)
CLAUDE.md      Vorlage für Projektregeln
```

Im Projekt entstehen zur Laufzeit `tasks/` (local) und `.worktrees/` (beides in `.gitignore`).
