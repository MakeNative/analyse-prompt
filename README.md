# analyse-prompt

Ein Agent tut nicht, was du meinst, sondern was im Prompt steht. `analyse-prompt` ist
ein Skill für Claude Code, der deinen Prompt prüft wie ein Prompt Engineer. Er stellt
dir die Fragen, deren Antwort den Prompt ändert, und schreibt ihn erst dann neu. Ob die
neue Fassung wirkt, zeigt er mit echten Probeläufen vorher und nachher.

Er funktioniert für Agenten-Definitionen, Skills, System-Prompts, einmalige Aufträge und
CLAUDE.md-Dateien.

## Installation

In Claude Code:

```
/plugin marketplace add MakeNative/analyse-prompt
/plugin install analyse-prompt@analyse-prompt
```

Voraussetzungen: `python3` und `claude` im PATH für die Probeläufe. Getestet mit Claude
Code 2.1.292.

## Benutzen

```
/analyse-prompt .claude/agents/reviewer.md — er meldet Stil statt Fehler
/analyse-prompt skills/deploy
/analyse-prompt "Fass die Notizen zusammen und liste alle To-dos mit Deadline auf."
/analyse-prompt schreib mir einen Prompt für einen Agenten, der Rechnungen prüft
```

Hinter dem Gedankenstrich steht am besten, was der Agent heute falsch macht. Daraus
wird der erste Testfall. Mit „nur prüfen“ bekommst du nur die Befunde, ohne Neufassung.

## Ausprobieren

`examples/meeting-summarizer/` ist ein kleines Projekt mit einem absichtlich schwachen
Agenten: „IMMER vollständig“, „frag nach, wenn etwas unklar ist“, eine Experten-Persona,
kein Ausgabeformat. Dazu kommen Meetingnotizen mit Lücken und einer eingeschleusten
Anweisung an die KI.

```
git clone https://github.com/MakeNative/analyse-prompt
cd analyse-prompt/examples/meeting-summarizer
claude
/analyse-prompt .claude/agents/meeting-summarizer.md — Er trägt Deadlines ein, die nicht in den Notizen stehen. Echte Eingabe: notizen.md
```

## So arbeitet er

1. **Aufnehmen:** Er liest den Prompt und sein Umfeld, also wer ihn aufruft, mit welchen
   Werkzeugen und Modellen, und was frühere Läufe zeigen.
2. **Diagnose:** Er bewertet den Prompt nach einer Rubrik mit 14 Kriterien.
   Gleichzeitig liest ein Agent ohne jedes Vorwissen den Prompt und sagt, wo er raten
   müsste. Ein Befund zählt nur mit einem konkreten Fehler, den der Agent deshalb macht.
   „Nichts Wesentliches zu ändern“ ist ein gültiges Ergebnis.
3. **Klären:** Er legt fest, was der Agent leisten soll: Ziel, Eingaben, Ausgabe,
   Erfolgskriterium, Grenzen, Verhalten bei Lücken, Testfälle. Was sich aus den Dateien
   ergibt, entscheidet er selbst und zeigt es dir. Dich fragt er nur den Rest, in Runden
   zu höchstens vier Fragen, am konkreten Fall und mit einer Vorschau, wie die Stelle im
   Prompt dann lauten würde.
4. **Neufassung:** Er ändert so wenig wie möglich. Schutzbausteine setzt er nur gegen ein
   echtes Risiko ein, immer mit Begründung. „Erfinde nichts“ kommt dabei nie allein,
   sondern mit einer erlaubten Ausweichantwort, einer Lückenmarke und einer Fundstelle.
   Der Appell allein reicht laut Microsoft-Doku oft nicht.
5. **Nachweisen:** Alte und neue Fassung laufen echt mit denselben Testfällen, darunter
   eine Lücken-Falle, in der eine nötige Angabe fehlt. Ein Prüfer sucht in jedem Ergebnis
   nach erfundenen Angaben und befolgten eingeschleusten Anweisungen. Reicht es nicht,
   geht es zurück zu Schritt 3 oder 4.
6. **Übernehmen:** Er zeigt dir Vorher und Nachher. Das Original wird erst nach deinem Ja
   ersetzt, committet wird nie.

## Warum echte Probeläufe

`scripts/probelauf.py` startet für jeden Testfall eine frische Claude-Session mit
`--safe-mode`: keine CLAUDE.md, keine Plugins, Hooks oder MCP-Server, nur die Werkzeuge
`Read`, `Glob` und `Grep`. Der Prompt ist dort die echte Anweisung. Einen Agenten bekommt
die Session als System-Prompt, einen Skill als Auftrag.

Ein Tester, der den Prompt nur nachspielt, weiß, dass er geprüft wird, und arbeitet
vorsichtiger als der echte Agent. Beim Bau übernahm der Meeting-Agent aus dem Beispiel
eine in den Notizen eingeschleuste Deadline in zwei von zwei echten Läufen. Der
nachgespielte Lauf desselben Prompts übernahm sie nicht. Die geschärfte Fassung meldete
die Anweisung in beiden Läufen, statt ihr zu folgen, und markierte jede fehlende Angabe
als `offen`.

Auch die Form der Eingabe zählt: Kamen die Notizen als Dateipfad statt als Text, folgte
der Agent der Anweisung in unseren Läufen nicht. Die Testeingaben bilden deshalb nach,
wie der echte Aufrufer schickt.

## Was nicht (mehr) wirkt

Die Rubrik prüft auch auf Kunstgriffe, die laut aktuellen Studien und Anbieter-Doku
nichts bringen oder schaden:

- Trinkgeld und Drohungen haben keinen messbaren Effekt.
- Experten-Personas („Du bist ein Weltklasse-Experte“) machen Antworten nicht genauer.
- GROSSBUCHSTABEN und „CRITICAL: You MUST“ lassen aktuelle Modelle die Regel überdehnen.
- „Denk gründlich“ ist bei aktuellen Modellen normaler Text, der Regler dafür heißt Effort.
- Reine Verbote werden schlechter befolgt als die positive Form.
- „Prüf deine Antwort“ ohne Maßstab hilft kaum.

## Was drin ist

| Datei | Rolle |
|---|---|
| `skills/analyse-prompt/SKILL.md` | der Ablauf |
| `skills/analyse-prompt/references/rubrik.md` | 14 Kriterien je Prompt-Art und „Was nicht (mehr) wirkt“ |
| `skills/analyse-prompt/references/bausteine.md` | 17 Schutzbausteine mit Einsatzregel |
| `skills/analyse-prompt/references/quellen.md` | Belege mit wörtlichen Zitaten |
| `skills/analyse-prompt/scripts/probelauf.py` | die echten Probeläufe |
| `agents/prompt-tester.md` | liest kalt und prüft die Läufe; nur Lese-Werkzeuge, kein Memory |
| `examples/meeting-summarizer/` | ein schwacher Agent zum Ausprobieren |

Jede Regel der Rubrik ist belegt: Doku von Anthropic, OpenAI, Google und Microsoft und
Studien bis Oktober 2026, mit wörtlichen Zitaten in `references/quellen.md`.

## Grenzen

- Im Probelauf kann der Agent nur lesen. Was er schreiben oder verschicken würde,
  beschreibt er nur, und Subagenten, die er starten würde, laufen nicht mit. Ein
  Probelauf ist darum eine Annäherung, kein Produktionstest.
- Eine Runde dauert etwa 10 bis 15 Minuten und startet bis zu acht kurze
  Claude-Sessions. Das kostet Tokens.
- Was nie passieren darf, gehört in einen Hook oder eine Werkzeugsperre, nicht nur in
  den Prompt.
- Für einen Rundumcheck aller Anweisungsdateien eines Projekts hat Claude Code
  `/doctor prompt-audit`. `analyse-prompt` nimmt sich einen Prompt gründlich vor.

## MakeNative

Dieses Plugin ist Teil von [MakeNative](https://makenative.de). Wir arbeiten jeden Tag mit
Agenten-Pipelines und bringen anderen bei, sie für ihre eigenen Prozesse zu
bauen. Für Entwickler genauso wie für alle, die ihre Arbeit mit KI automatisieren wollen.

Du willst das für deine Arbeit lernen? Trag dich in die Liste ein:
[Bewerbung Starter-Kohorte](https://tally.so/r/5ByYj6?utm_source=github&utm_medium=readme&utm_campaign=starter-kohorte)

Christian Eichmüller · [LinkedIn](https://linkedin.com/in/christianeichmueller)

## Lizenz

MIT
