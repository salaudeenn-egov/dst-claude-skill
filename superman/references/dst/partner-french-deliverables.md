# Translating partner deliverables into French (and the message that carries them)

Read this before producing a French version of any campaign report or data
workbook for a francophone tenant (Togo `tg`, Chad `ch`, and any Burundi /
Congo / Senegal partner surface), and before drafting the Slack / email message
that ships it.

First proven end-to-end on 2026-08-27 for Togo CPS 2026 Cycle 3, cumulative
Days 1-7 (20-26 August 2026).

---

## The deliverable set

A cumulative partner run produces three files, and the partner expects all
three translated as a set — translating only the Word report leaves the
annexes in English and the terminology inconsistent across the package:

| Source | French output |
|---|---|
| `... Report (Partner) — <date>.docx` | `Togo Rapport Cumulatif Jours 1-N (Partenaire) — ... (FR).docx` |
| `... Performance Data (Partner) — <date>.xlsx` | `Togo Donnees de Performance Cumulatives Jours 1-N (Partenaire) — ... (FR).xlsx` |
| `... CDD Sync Data — <date>.xlsx` | `Togo Donnees de Synchronisation DC Cumulatives Jours 1-N — ... (FR).xlsx` |

Keep filenames ASCII-safe (`Donnees`, `aout`) — they travel through Slack,
Google Drive and Windows paths. Accents inside the documents are fine.

---

## Terminology: use the pipeline's own francophone vocabulary

The canonical map lives in `dst/partner-feedback-form.md`. It is authoritative
because it was derived from what the pipeline actually emits for francophone
tenants, not from literal translation. Repeated here for the terms a report
hits, plus the ones settled during the Togo run:

| English | French |
|---|---|
| LGA | district (never "LGA" in a French deliverable) |
| ward | canton |
| health facility / HF | formation sanitaire / FS |
| SMC / SPAQ | CPS / SPAQ |
| ITN / LLIN / bed net | MILDA |
| CDD | **DC** / distributeur communautaire |
| Records / Treated / Not Treated | Enregistrements / Traités / Non traités |
| Coverage / Campaign Target | Couverture / Cible de la campagne |
| Ineligible / Referred / Refused | Non éligible / Référé / Refus |
| Died / Migrated / Absent | Décédé / Migré / Absent |
| Redose | Nouvelle dose |
| Never Synced / Days Synced | Jamais synchronisés / Jours synchronisés |
| HIGH / MODERATE / LOW / NOT REPORTED | ÉLEVÉ / MODÉRÉ / FAIBLE / NON RAPPORTÉ |
| Y / N day flags | O / N |

**Problem.** The first pass of the Togo Word report used **DBC**
(distributeur à base communautaire) for CDD and **Villages** for Wards. Both
are defensible textbook French and neither looked wrong on review — that is
exactly why this is worth recording.

**Approach.** The error surfaced only because the skill's own francophone map
was consulted before doing the Excel files, one step later than it should have
been. The map disagreed: the pipeline emits **DC** and **canton**.

**Root cause.** Translating from general French rather than from the project's
established vocabulary. Partners have been reading pipeline-generated French
for prior cycles, so an unfamiliar-but-correct synonym reads as a *different
metric*, not a stylistic variant.

**Fix / rule.** Open `dst/partner-feedback-form.md` and load the terminology
map BEFORE translating the first file, not after. Then check what the partner
has already received: if prior cycles shipped "CDD" verbatim, flag the switch
to the user rather than changing vocabulary mid-campaign on your own.

---

## fr-FR localisation of values

Applies to both Word and Excel:

- Thousands separator: `154,705` -> `154 705` using U+202F NARROW NO-BREAK
  SPACE. Not a plain space (line-breaks mid-number), not a period.
- Decimal comma and a space before the percent sign: `84.8%` -> `84,8 %`.
- Dates: `20 August 2026` -> `20 août 2026`; ISO cell values `2026-08-22` ->
  `22/08/2026`.
- Times: `06:40` -> `06h40`.
- Do NOT touch numeric Excel cells. They are stored as numbers with a number
  format, so Excel renders separators per the reader's locale automatically.
  Only *string* values need localising — in these workbooks that is the
  `Coverage %` column (a string like `76.3%`) and `Sync Dates`.

---

## Word: the python-docx hyperlink trap

**Problem.** After a first clean-looking run, three strings in the output were
doubled:

```
CDD Sync Data ↗Données de synchronisation des DC ↗
12 formations sanitaires ... au total. Liste complète dans le fichier Excel de performance ↗Full list in performance Excel ↗
```

The obvious read is a bad dictionary entry or a double-apply of the
translation pass. Both are wrong.

**Approach.** Dumped the paragraph's parts instead of its text:

```python
p.text     # 'Showing 12 LOW / Low Activity facilities of 86 total. Full list in performance Excel ↗'
p.runs     # ['Showing 12 LOW / Low Activity facilities of 86 total. ', '']
p.hyperlinks[0].text   # 'Full list in performance Excel ↗'
```

**Root cause.** In python-docx (confirmed 1.2.0) `paragraph.text` INCLUDES
hyperlink text, but `paragraph.runs` EXCLUDES it — hyperlink runs live under a
`w:hyperlink` element, not directly under `w:p`. So the common idiom "match on
`p.text`, write the translation into `runs[0]`, blank the rest" writes the
whole translated sentence into the plain-text run while the hyperlink run
keeps its English text, and the two concatenate.

**Fix.** Translate run *groups* separately, never the paragraph as a whole:

```python
def walk(container):
    for p in container.paragraphs:
        if not p.text.strip():
            continue
        apply(p.runs, "text")                        # plain runs only
        for h in getattr(p, "hyperlinks", []):
            apply(h.runs, "link")                    # each hyperlink's runs
```

Consequences for the dictionary: key the plain part and the link text as
SEPARATE entries (`"Showing 12 ... of 86 total."` and
`"Full list in performance Excel ↗"`), and match on the stripped string while
restoring the original leading/trailing whitespace, because the plain run ends
with a significant trailing space.

Other Word mechanics that worked:

- Writing into `runs[0]` and blanking the remaining runs preserves character
  formatting, which is why the output keeps the source styling exactly.
- Walk tables recursively (`cell.paragraphs`, nested tables) and walk
  `section.header` / `section.footer` / `first_page_*` explicitly — the footer
  disclaimer is easy to miss and it names the extract timestamp.
- Have the translator PRINT every string it could not translate. That log is
  the completeness check: after a good run the only entries left should be
  proper nouns (district and facility names such as `amou`, `usp oga`,
  `cms bon secours`), which are intentionally kept in the source language.

---

## Excel: check before renaming sheet tabs

Renaming a sheet silently breaks anything that references it by name. Run this
pre-check on every workbook and only proceed if all four come back empty:

```python
wb.defined_names                       # defined names
ws.conditional_formatting._cf_rules    # CF rules keyed on English literals
[c for row in ws.iter_rows() for c in row if c.hyperlink]
# plus: grep the cells for formulas referencing other sheet names
```

For the Togo pair all four were empty, so tab renaming was safe. Freeze panes
(`D3`) and autofilter refs (`$A$2:$Z$2`) are sheet-local and survive a rename.

Sheet-name mapping used; accented tab names are valid and worth using so tabs
match the `Statut` values inside them:

`ALL FACILITIES` -> `TOUTES LES FS`, `LOW` -> `FAIBLE`,
`MODERATE` -> `MODÉRÉ`, `HIGH` -> `ÉLEVÉ`, `NO TARGET` -> `SANS CIBLE`,
`LOW ACTIVITY` -> `FAIBLE ACTIVITÉ`, `NOT REPORTED` -> `NON RAPPORTÉ`,
`SUMMARY` -> `SYNTHÈSE`, `NEVER SYNCED` -> `JAMAIS SYNCHRONISÉ`,
`LOW SYNCED` -> `FAIBLE SYNCHRO`. District tabs stay as-is (proper nouns).

Translate: sheet tabs, the title band above the header row, the header row,
status/flag vocabulary, percentage strings, ISO dates. Leave alone: facility
names, district names, canton lists, usernames, user IDs.

Header rows differ between the two workbooks — the performance workbook has a
title band in row 1 and headers in row 2; the CDD sync workbook has headers in
row 1. Parameterise `header_row` rather than assuming.

---

## Surface source defects, do not silently repair them

Two generator bugs were visible in the source and were carried through
verbatim into the French version, then reported to the user:

- **Report section 5.1b "Sync Status by LGA" is column-misaligned.** Rows read
  `1 | 1 | Ogou | 2 | 0 | 0 | 0 | 2` under an 8-column header, i.e. the values
  shift one column right. A translation pass must not paper over this, or the
  bug ships to partners looking intentional and never gets fixed upstream.
- **`ogou` / `Ogou` case collision** in the CDD sync workbook produces a sheet
  literally named `ogou1`. Same district, two casings, so the generator
  de-duplicates the tab name with a numeric suffix. Also visible as a two-CDD
  `Ogou` row in the summary alongside the real 305-CDD `ogou`.

Both belong in the pipeline's boundary-name normalisation, not in the
translation layer.

---

## The accompanying partner message

Conventions confirmed with the user on 2026-08-27:

- **French, and it ships the feedback form alongside the report.** Send the FR
  form to a francophone audience.
- **No figures and no data quality.** Per the PARTNER POLICY in
  `dst/partner-feedback-form.md`: DQ is never shared with partners, and
  partners do not receive the short Slack summary. The message is a pointer to
  the report, not a summary of it.
- **Tag the named accountable readers, never `@channel` / everyone.** The
  channel still sees the message; tagging only controls notifications. Naming
  two people creates ownership of the response, a broad ping diffuses it, and
  on an external partner channel a sudden widening of the tag on a routine
  cumulative report reads as escalation ("is something wrong?"). Widen only by
  NAME, and only when a specific stakeholder asked for the cumulative or the
  report is the final record of the cycle. If broad reach is wanted for the
  feedback form, send a separate short message about the form a day or two
  later instead of widening the tag on the report.

Template (Days 1-N cumulative):

```
Bonjour @<partner 1> @<partner 2>,

Veuillez trouver ci-dessous le rapport cumulatif d'analyse des données pour les
Jours 1 à N (<date début> au <date fin>), couvrant l'ensemble de la campagne
CPS <année> Cycle <n>. Il présente la performance globale de chaque formation
sanitaire ainsi que le statut de synchronisation des distributeurs
communautaires (DC) sur toute la durée de la campagne.

Rapport cumulatif — Jours 1 à N : <lien>

Vous trouverez également en pièces jointes les données détaillées : la
performance par formation sanitaire et la synchronisation des DC.

Nous y joignons enfin le formulaire de retour d'information. Vos commentaires
sur le contenu et le format de ces rapports nous seront très utiles pour les
améliorer.

Formulaire de retour : <lien FR>

Restant à votre disposition pour tout complément d'information.

Merci
```

The daily variant is the same message with "rapport quotidien ... pour le Jour
N" and without the cumulative framing.

---

## Verification checklist before sending

1. Re-open each output and dump every paragraph, table cell, and the
   header/footer; confirm no doubled strings (the hyperlink trap) and no
   English left except proper nouns.
2. Confirm the untranslated-strings log contains only district and facility
   names.
3. Spot-check one synced CDD row: day flags must read `O`/`N`, `Sync Dates`
   must be `DD/MM/YYYY`, `Statut` must be the French band.
4. Confirm sheet tabs and the `Statut` values inside them use the same words.
5. Confirm no DQ columns or sections leaked into the partner copies (the
   partner performance workbook is the copy WITHOUT the 6 DQ columns — see
   `dst/automation-and-structure.md`, partner report DQ-strip).

Working scripts for the 2026-08-27 run were one-off translators driven by an
explicit source-string dictionary (deliberately: a dictionary fails loudly on
an unrecognised string, whereas a machine-translation pass silently mangles
facility names and numbers). Keys are exact source strings, so a change to the
generator's wording requires a dictionary update — that is the intended
failure mode, since the untranslated log tells you immediately.
