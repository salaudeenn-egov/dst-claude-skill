# Partner feedback form for the Campaign Data Analysis Report (2026-08-27)

Built a Google Form to ask report partners what they actually use and what is
missing, in English and French. Two live forms, created programmatically.

Useful mainly for the **auth findings** (which credential types can and cannot
create a Form), the **francophone terminology map**, and two **product facts
about the partner report** that were only established by designing this.

---

## Product facts established while designing it

Both came from the user correcting a draft, so they are policy, not inference:

1. **Data quality is NEVER shared with partners.** Not as a section, not as a
   summary line, not as an optional question. The partner report already strips
   DQ sections 3.2 and 3.5 and ships a DQ-stripped Excel
   (`report.py::_upload_partner_excel`), and the form must not offer it back.
   A first draft asked "would you like data quality included?" - wrong: it is
   not a choice to offer.
2. **Partners do not receive the short Slack summary today.** They get the
   report only. So "the Slack summary is enough for me" was an invalid answer
   option - it described a channel that does not exist for them. Reframed as a
   forward-looking question ("would a SHORT summary also be useful?").

Attribution changed twice. Organisation and state were removed, then in the
revision round **Name and Organisation came back as REQUIRED questions in Part 1
and the optional contact question was removed**. So responses are now fully
attributed and not anonymous - good for follow-up, and worth remembering when
reading critical answers.

---

## Which credentials can create a Google Form

Tested in order, all three results verified against live API responses:

| Credential | Result |
|---|---|
| **API key** | **Impossible.** `401 UNAUTHENTICATED` - "API keys are not supported by this API. Expected OAuth2 access token or other authentication credentials that assert a principal." A key identifies a PROJECT; creating a form writes into someone's Drive, so Google demands an identity. No key will ever work |
| **Service account** (`dst-report@digit-health-75ed0`) | Blocked at `403 SERVICE_DISABLED` (Forms API not enabled). Even once enabled it is the WRONG tool: the form would be owned by the service account, responses land in ITS Drive, and ownership cannot be handed to a person. Service accounts also commonly hit `storageQuotaExceeded` creating Drive files |
| **OAuth Desktop client** | **Worked.** `InstalledAppFlow.run_local_server(port=0)`, token cached, form owned by whoever approves the consent screen |
| **Apps Script** | Also works and is simpler for one-off use - `FormApp.create()` runs AS the user, so no client, no API to enable, no ownership problem. Script kept at `D:\DST\partner_feedback_form.gs` |

**Choose Desktop app, not Web application**, when creating the OAuth client:
the installed-app flow binds a temporary localhost listener and needs no
registered redirect URI. Web application forces exact redirect URIs and
`redirect_uri_mismatch` debugging for no benefit.

**Gotcha - two different GCP projects.** The service account lives in
`digit-health-75ed0`; the OAuth client the user created lives in
`teak-clarity-501004-v8`. So the "enable the API" link printed in the service
account's error names the WRONG project for the OAuth path. Always read the
project id out of the error text.

**`forms.body` carries no identity.** The cached token has an empty `account`
field and no `id_token`, so after creating a form you CANNOT tell which account
owns it - check Drive or the form's avatar. Do not add `openid`/`email` scopes
just for this; it is a one-time question with a five-second UI answer.

---

## Forms API v1 limits (not script bugs)

- **Cannot set the post-submit confirmation message.** Print it for the user to
  paste into Settings -> Presentation.
- **Cannot attach a response spreadsheet.** Responses tab -> Link to Sheets, in
  the UI. (Apps Script CAN do this: `form.setDestination()`.)
- `forms().create()` accepts **only** `info.title`. Everything else -
  description included - goes through `batchUpdate`.

**Structural notes**
- A grid is a `questionGroupItem`: `grid.columns.options` for the columns,
  `questions[].rowQuestion.title` for the rows. Required is set PER ROW, not on
  the item.
- "Other" free text is an option entry `{"isOther": true}`, not a flag.
- **Delete items in DESCENDING index order.** Deleting a low index shifts every
  higher one down, so ascending deletion removes the wrong questions. This bit
  during the session and was caught only by reading the form back.
- **Read the form back after every change.** `batchUpdate` reports success
  without telling you what the form now looks like; a `forms().get()` diff is
  the only real verification.

---

## Francophone terminology map

Taken from what the pipeline itself emits for francophone tenants, not
translated literally. Verified by grep of `pipeline/` - it uses **district**,
never LGA, for those tenants.

| English | French / francophone tenant |
|---|---|
| LGA | **district** |
| ward | **canton** (Togo sub-district) |
| SMC | **CPS** (chimioprevention du paludisme saisonnier). SPAQ is the DRUG, not the campaign - do not label a campaign "SMC / SPAQ" |
| ITN / LLIN, bed net | **MILDA** (moustiquaire impregnee longue duree) |
| CDD | **DC** / distributeur communautaire |
| health facility | **formation sanitaire** |

Reusable beyond this form - any Togo or Chad partner-facing text should use
these, and "LGA" in a French deliverable is a tell that it was translated
from the Nigeria template without review.

---

## Form design decisions worth reusing

- **One required open question, capped at ONE change** ("if we can change only
  one thing, what should it be?"). Open wishlists produce twenty items and no
  priority; forcing a single answer gives a ranking for free.
  NOTE: this question was later REMOVED at the user's request along with the
  rest of Part 4 - see the revision round below for what that costs. The
  principle still holds for the next form.
- **The section-by-section usage grid is the highest-value question.** It tells
  you what to cut, with evidence. Include a "**I did not know it was there**"
  column: a section nobody uses because nobody noticed it is a presentation
  problem, and the fix is different from removing it.
- **Grid rows must use the report's OWN headings** so respondents can find what
  they are rating. An early draft said "the 10 facilities where fewest teams
  sent data", which is plainer English but matches nothing the reader has seen;
  it became "Top 10 facilities with the lowest CDD sync rate", mirroring heading
  5.2. In the revision round those three sync rows (5.1 / 5.1b / 5.2) were then
  MERGED into one - rating three slices of one section separately is noise. Use
  the report's language, but at the granularity the reader thinks in.
- **Ask for exact times, not time bands.** "Morning / Afternoon / Evening" does
  not convert into a `report_times` cell. A free-text "what exact time do you
  want each report?" plus "how many per day" does - and it must ask for the
  **TIMEZONE explicitly** (the first version only said "your own local time",
  which still leaves "11am" unusable). Slots are configured in UTC.
- **Plain language for the audience.** The first draft used "denominators",
  "granularity", "synchronisation", "interpret". Rewritten to short sentences
  and common words after the user pointed out the audience.
- **If everything is required, give required free-text an escape phrase**
  ("if everything is clear, write 'all clear'"). A mandatory open box with
  nothing to say produces an abandoned form or a stray character.
- **Keep the intro honest.** After making everything required, the description
  still read "only the last question needs an answer" and three questions still
  said "you can leave this empty". Required-ness and helper text drift apart on
  every edit; re-read them together.

---

## Artifacts

| File | Purpose |
|---|---|
| `D:\DST\create_partner_form.py` | English form, Forms API + OAuth |
| `D:\DST\create_partner_form_fr.py` | French form (Togo), separate form so responses do not mix |
| `D:\DST\partner_feedback_form.gs` | Apps Script alternative - runs as the user, no API to enable |
| `D:\DST\.form_token.json` | cached refresh token. **Treat as a credential**, never commit |

Both live forms: 4 parts, 13 questions, 11x5 grid, everything required except
the contact question.

KNOWN GAP: the French form was written **without accents** (`donnees`,
`reponses`) to avoid encoding corruption through the shell, after this session
lost characters twice to escaping. Readable but unpolished for a partner-facing
document - fix in the editor or re-run with accents.

Also open: `create_partner_form.py` drifted behind the live English form (the
questions removed on request, the CDD wording, the summary and timing questions,
the required flags were all applied via the API directly). Only matters if the
form is ever regenerated from the script.

---

## Revision round (same session) - what the user changed, and why it matters

The first draft was reviewed line by line and reshaped. The edits are worth
recording because most of them encode facts about the report and the programme,
not just wording preferences.

**The report's real name is the "Campaign Data Analysis Report".** Not "the
campaign report". Both forms are titled after it, and the usage question names
it explicitly. Took three passes to land ("remove campaign" then "campaifn data
analsysis"), so use the full name in any partner-facing text.

**"Programme Issues and Resolutions" is MISLEADING to partners.** The user's
words: the section actually surfaces DATA issues - health facilities performing
low, low CDD sync - not programme problems and their resolutions. The grid row
now reads "The data issues we point out - for example health facilities
performing low, or low CDD sync". Worth considering whether the report HEADING
should change too; the form now describes the section more accurately than the
report does.

**Campaign list corrections** (domain facts, not preferences):
  - "SMC / SPAQ" -> **SMC**. SPAQ is the drug, not the campaign; naming both
    conflates the two.
  - added **Polio**
  - added **Co-delivery** = more than one medicine or product delivered
    together. The parenthetical gloss was then removed - the term is understood
    by this audience and does not need explaining.
  - FR equivalent is **Co-administration**.

**Merges, because the report does not separate them in the reader's mind:**
  - "Results for each LGA" + "for each health facility" -> ONE row
  - the THREE CDD sync rows (summary / by LGA / lowest-sync facilities) -> ONE
    row. Rating three slices of the same section separately is noise.
  - grid dropped 11 rows -> **8**, which matters: the grid is the question most
    likely to make someone abandon the form.

**"The Excel file" -> "Detailed report in Excel".** Names the artifact by what
it IS, not by its file type.

**Removed from the missing-data options:** "How the target numbers were worked
out" and "Charts showing more days". **Added:** "A chart showing predictions for
the coming days" - forecasting, which is a genuinely new capability request
rather than a presentation tweak.

**The exact-time question must also ask for the TIMEZONE.** Slots are configured
in UTC; a partner writing "11am and 5pm" without a timezone is unusable.

**Part 4 was removed entirely** - the ONE-thing question, "anything else", and
the contact question. Name and Organisation moved INTO Part 1 as required
questions instead.

  COST, stated at the time and worth restating if this form is reused: removing
  the "if we can change only ONE thing" question removes the only place a
  partner can raise something the option lists did not anticipate, and the only
  thing that produces a PRIORITY ordering. The remaining open box is "is
  anything unclear?", which surfaces confusion, not wants.

Final shape of both forms: **3 parts, 12 questions, all required, 8-row grid.**

---

## Process lesson: rebuild vs patch

Twelve interleaved edits across two forms is where index-shift errors happen, so
that round was done as a **wholesale rebuild** - delete every item descending,
recreate in order from one declarative list. Much safer than twelve positional
patches.

That is only safe BEFORE the form is shared: a rebuild replaces every item id,
which breaks the column mapping of any existing response. Once a form has
responses, patch individual items only.

Subsequent smaller rounds (title, one question) were done as targeted
`updateItem` patches, which is the right tool at that size.

**Script drift is the standing risk here.** `D:\DST\update_partner_forms.py`
holds the rebuild definition, but the last three rounds went straight through
the API, so it is behind the live forms. Only matters if a form is regenerated
from it - but that is exactly when it would bite.
