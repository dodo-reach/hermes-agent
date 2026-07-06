# DPA escalation — the EU/UK Article 77 complaint path

The killer feature of the GDPR pipeline: when a data broker fails to honour an Article 17
erasure request within the 30-day Article 12(3) window, the subject can file an Article 77
complaint with their national supervisory authority. CCPA has no equivalent.

This guide covers the operational steps an EU/UK subject (or their agent) follows to escalate.

## The decision: when to escalate

`pdd.py next` surfaces the escalation automatically when **all** of these hold:

- The subject's residency is an EU member state or the UK (`dossier.is_eu_residency()` returns true).
- The broker has `gdpr_scope: true` (brokers flagged as unlikely to honour Art. 17 are not
  escalated — `escalate` against them is still legal but rarely productive).
- The case is in `submitted` or `awaiting_processing` state (the broker was notified and
  has had its chance).
- 35+ days have elapsed since the Art. 17 was filed (Art. 12(3)'s 30 days + a 5-day grace
  for time-zone and clock-skew tolerance). The constant is `DPA_ESCALATION_THRESHOLD_DAYS`
  in `scripts/autopilot.py`; it can be raised to 90 if you want to wait for the full
  maximum extension window (see below).
- The subject has not already filed a DPA complaint for this broker
  (`dossier.preferences.dpa_complaint_filed_<broker_id>` is unset).

If all five hold, `next_actions` adds a `dpa_escalate` (or `dpa_escalate_generic`) action
to the output queue with the rendered command. The agent executes the command, which
produces a complaint file at `subjects/<subject_id>/drafts/dpa_complaint_<dpa>_<broker>.txt`.

### Note on the 35 vs 90 day threshold

Article 12(3) GDPR gives the controller:

- **30 days minimum** (one month from receipt — the standard response window).
- **+60 days optional extension** (two further months for complex requests, but the
  broker MUST notify the extension within the first month).

So the realistic response window is **30 to 90 days**. The skill surfaces escalation at
**35 days** — that's the common case where the broker does not exercise the extension
clause. If the broker explicitly invoked the 2-month extension, you can raise the
threshold to 90 (`DPA_ESCALATION_THRESHOLD_DAYS = 90` in `scripts/autopilot.py`) to wait
for the full maximum window before filing. The skill's `dpa_escalate` action `why` text
reminds you of the 90-day ceiling when it surfaces the escalation.

## The procedure (per step)

### 1. Render the complaint

```bash
python3 scripts/pdd.py escalate <subject_id> <broker_id> \
    --request-date 2026-05-27 \
    --request-channel PEC
```

- `--request-date`: the date the Art. 17 was sent (default: lookup in dossier preferences).
- `--request-channel`: how the Art. 17 was sent (email / PEC / web form / post). The complaint
  cites this so the DPA knows how the broker received the prior request.
- The complaint is rendered in the DPA's working language automatically:
  Italian for Garante, French for CNIL, German for BfDI, English for ICO + the generic fallback.
- The output JSON includes the web-form URL, email, and (for Italy) the PEC address.

### 2. Review the draft

Open `subjects/<subject_id>/drafts/dpa_complaint_<dpa>_<broker>.txt`. Check for:

- Correct spelling of your name and contact email.
- Correct address (rendered from `identity.current_address`).
- Correct broker name and the date you sent the prior Art. 17.
- The subject-matter is the right broker (the render takes the first `broker` argument).

### 3. Gather attachments

The DPA will require:

1. **A copy of the Art. 17 request you sent to the broker.** Screenshot of the sent email
   (including headers), or a copy of the PEC receipt, or a postal receipt, depending on the
   channel you used.
2. **Proof of receipt.** Most brokers acknowledge Art. 17 receipts; if not, the postmark /
   PEC delivery receipt is the next-best evidence.
3. **Any broker response.** If they replied (refusal, partial erasure, "we don't have your
   data"), include it. If they did not respond, the complaint notes this and asks the DPA to
   treat the silence as non-compliance.
4. **A copy of your photo-bearing ID.** Most DPAs require this for the first complaint
   (anti-fraud). Send a **copy**, never the original.

### 4. File with the DPA

| DPA       | Country | Language | Channel                                              | Web form                                          |
|-----------|---------|----------|------------------------------------------------------|---------------------------------------------------|
| Garante   | IT      | it       | **PEC preferred** (protocollo@pec.gpdp.it) or web form | https://www.garanteprivacy.it/diritti/come-agire-per-tutelare-i-tuoi-dati-personali/reclamo |
| CNIL      | FR      | fr       | Online form (preferred) or post (Service des plaintes, 3 Place de Fontenoy, 75007 Paris) | https://www.cnil.fr/fr/adresser-une-plainte       |
| BfDI      | DE (federal) | de   | Web form (for federal bodies only) — most private-sector data broker complaints go to the **Land DPA** of the subject's Bundesland of residence | https://www.bfdi.bund.de/DE/Buerger/Inhalte/Allgemein/Datenschutz/BeschwerdeBeiDatenschutzbehoereden.html (BfDI). For the correct Land DPA: https://www.bfdi.bund.de/DE/Service/Anschriften/Laender/Laender-node.html |
| ICO       | UK      | en       | Online form                                          | https://ico.org.uk/make-a-complaint/              |
| (generic) | any EU member without shipped adapter | en | Each DPA's web form is published on the EDPB site | https://edpb.europa.eu/about-edpb/about-edpb/members_en |

For the generic path (subject residency codes `EU-ES, EU-NL, EU-BE, EU-AT, EU-IE, EU-PT,
EU-PL, EU-SE, EU-DK, EU-FI`): the rendered complaint is in English. The subject files it
with their own national DPA using the EDPB member list to find the right authority. The
subject must translate the working-language name and address block themselves if their DPA
prefers a local-language submission.

### 5. Record the filing

Once the complaint is filed with the DPA:

```bash
python3 scripts/pdd.py escalate <subject_id> <broker_id> --file
```

This:
- Records the filing timestamp in `dossier.preferences.dpa_complaint_filed_<broker_id>`.
- Records which DPA the complaint went to in `dossier.preferences.dpa_complaint_dpa_<broker_id>`.
- Transitions the case to `human_task_queued` with `reason="DPA complaint filed with <DPA name>"`.
- Adds a `dpa` field to the case for downstream reporting.

After this, `pdd.py next` will **stop** surfacing the escalation action for this broker — the
loop runs until the DPA responds or the next re-check window arrives.

### 6. Wait for the DPA

- **Garante**: typical response 3-6 months for the first contact, longer for substantive
  decisions. The PEC filing is timestamped; the Garante is bound by Art. 78(2) to respond
  within a reasonable time.
- **CNIL**: faster initial triage (typically 4-8 weeks); substantive decisions take months.
- **BfDI / Land DPAs**: variable; private-sector Land DPAs are typically faster than BfDI.
- **ICO**: typically 4-8 weeks for the first acknowledgement; substantive decisions take
  longer. The ICO's complaint procedure is well-documented.

If the DPA finds the complaint substantiated, it can order the controller to erase the data
under Article 58(2)(c) and impose administrative fines under Article 83. The subject is not
a party to the enforcement action but receives the outcome.

## When escalation does NOT make sense

- **The broker is in a non-EU/UK jurisdiction and refuses to engage.** DPA escalation is a
  lever against controllers subject to GDPR. For a US-domiciled broker with no EU presence,
  the DPA can still try (GDPR has extraterritorial scope under Art. 3(2)), but the chances of
  success drop substantially.
- **The broker has already erased the data.** `pdd.py next` will surface the broker as
  `confirmed_removed` once the re-scan re-check confirms the erasure; no escalation needed.
- **The subject wants faster action than a DPA timeline allows.** DPAs are slow. For
  time-sensitive removal, the right path is a private right of action under Article 79 in
  national court (which the unbroker skill does not currently automate).

## Operational notes

- **Audit trail.** Every transition through `pdd.py record` and `pdd.py escalate` writes to
  `audit.jsonl` with the timestamp, event, broker, and any disclosed fields. The DPA
  complaint and the prior Art. 17 request are both audit-logged.
- **Idempotency.** Running `pdd.py escalate --file` twice for the same broker is safe; it
  overwrites the timestamp with the most recent filing.
- **Re-filing after rejection.** If the DPA rejects the complaint (e.g. "you didn't exhaust
  the controller's complaint process"), the subject can re-record the Art. 17 request to the
  broker with a fresh timestamp, then re-run escalation after another 35 days.
- **Cumulative complaints.** Nothing prevents the subject from filing multiple DPA complaints
  about different brokers in parallel. Each one is independent.

## What this guide does NOT cover

- **Pre-litigation posture.** A privacy lawyer should review the complaint before filing if
  the exposure is non-trivial (sensitive data, ongoing harassment, defamation risk).
- **Class-action / collective complaints.** The skill is single-subject. Collective complaints
  under Article 80 (representation) are a different procedural path.
- **Cross-border complaints.** If the subject is in Italy and the broker is in Ireland (often
  the case for Big Tech), the "one-stop-shop" mechanism under Article 56 designates the
  Irish DPC as the lead authority. The skill's generic fallback covers this case but the
  optimal filing strategy is to contact both DPAs.

## Disclaimer

This is not legal advice. The procedures above are grounded in the GDPR text and the
published procedural guidance of each named DPA. Subject-specific strategy (especially
where sensitive data or pre-litigation considerations apply) should be reviewed by a privacy
lawyer in the subject's jurisdiction before filing.