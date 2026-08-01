# Appendix — Supporting Detail

**Companion to the Technical Assessment Response. Optional reading; the main response stands alone.**

This covers material your email indicated would come up in the final conversation — pivoting under resource constraints, and the schemas behind the design — plus the compliance posture the government Proof of Value work implies. Numbered sections (§0–§4) are in the main response, lettered ones (§A–§F) are here. Every reference is a link.

---

**Contents** — [Assumptions](#assumptions) · [A. Resource-Constrained Pivot Strategy](#a-resource-constrained-pivot-strategy) · [B. Provenance, Audit, and Compliance](#b-provenance-audit-and-canadian-compliance-posture) · [C. Data Model Sketch](#c-data-model-sketch) · [D. Store Implementation Notes](#d-store-implementation-notes) · [E. Validation Implementations](#e-validation-implementations) · [F. What Python 3.14 Changes](#f-what-python-314-actually-changes-here) · [Main response](Technical-Assessment-Response.md)

---

## Assumptions

[§0](Technical-Assessment-Response.md#0-assumptions) of the main response lists the assumptions that apply to both documents; those still hold here. These are additional, and the first one is the largest — [§A](#a-resource-constrained-pivot-strategy) is built on a scenario I invented, because your brief describes no constraints at all.

| Area | Assumption | Note |
|---|---|---|
| A | "Tight timeline, no infrastructure budget, small team" is hypothetical | Your brief states none of these. It's written to answer the pivot-under-constraints question your email flagged for the final conversation. Give me the real constraints and this table changes |
| A | ~15 s p95 is a tolerable synchronous wait behind a loading state | A chosen target, not a measured one |
| A | Deterministic checks are ~15% of the validation work | An estimate from comparable pipelines; not measured here |
| A | An LLM vendor with native multimodal document support is contractually available | Otherwise the OCR stage can't be skipped and that row of the table doesn't work |
| B | Canadian jurisdiction, with IRCC the likely PoV counterparty | Inferred solely from the IRPR reference in your screenshot. A provincial or non-Canadian counterparty changes the whole residency analysis |
| B | Protected B is the plausible classification ceiling | Stated nowhere; if it's higher, self-hosting stops being optional |
| B | Cloud hosting rather than on-premises | On-prem removes the residency question and substitutes an operations burden |
| B | The privilege discussion is engineering-informed, not legal advice | Needs counsel and law-society confirmation before it drives contract terms |
| B | A form is eventually exported or filed from the pre-filled draft | The [§3](Technical-Assessment-Response.md#3-data-pipeline--error-handling) pipeline stops at pre-fill; the `relied_upon` event assumes a discrete downstream step. If there isn't one, that event becomes "user accepted the draft" and the argument is unchanged |
| C | Multi-tenant, with user accounts (`org_id`, `created_by`) | Not stated; single-tenant would simplify the model |
| C | Five tables is a sketch, not a complete model | Omits auth, the form registry, field mapping, and billing |
| D | Next.js 16.1 App Router with server rendering | Hydration handling is unnecessary in a pure client-rendered SPA |
| D | Autosave payloads stay under the 64 KB `keepalive` cap | A 5,000-word draft is roughly 30 KB, so it fits; the code guards the flag rather than assuming it |
| E | Python 3.14, Pydantic v2, rapidfuzz; fuzzy threshold 90 | Dependency and tuning choices; [§F](#f-what-python-314-actually-changes-here) covers what 3.14 changes |
| F | The extraction worker's dependency stack can be audited and pinned | Free-threading is only safe where you control every C extension |
| E | `check_restoration` is illustrative | Claim construction is elided (`claims=[...]`) |

---

## A. Resource-Constrained Pivot Strategy

The architecture in the main response is the target state. Assume for this section the constraints named above — tight timeline, no infrastructure budget yet, small team — since dropped into that, I wouldn't build all of it before shipping a fix. Each corner below is cut deliberately, and each sits behind an interface (`fileId`, `CaseFieldsV1`, the file `status` enum) that doesn't change when the corner is later un-cut.

| Constraint | Cut | What it costs | Why the interface holds |
|---|---|---|---|
| No Redis/BullMQ | Run extraction synchronously in the upload handler behind a loading state; cap file size and page count to keep p95 under ~15s | Long documents rejected rather than queued | The frontend still polls `fileId` for status; introducing a queue is a backend-only change |
| No OCR/vision infrastructure | Send PDF/PNG directly to the LLM vendor's native multimodal document support | Higher per-call cost | Removes an entire pipeline stage; the extraction contract is unchanged |
| No object storage | Store bytes in a `bytea` column in the existing Postgres, with a hard size cap, keyed by the same `fileId` | Bloats the database, no CDN | Migrating to S3/GCS touches only the storage adapter |
| No time for cross-device sync | Ship the in-memory store alone; it fully fixes Defect 1. Server autosave lands as a fast-follow | Refresh loses the draft | `PATCH /cases/:id/draft` is additive; no schema change |
| No time for full validation | Ship deterministic checks only ([§4.4](Technical-Assessment-Response.md#44-methods-by-category), first bullet) — presence, types, dates, the s.182 rule | No semantic or judge-based detection | The severity taxonomy and `DiscrepancyFlag` are already N-source; semantic checks append flags to the same structure |

That last row is the one I'd defend hardest. Deterministic checks are perhaps fifteen percent of the validation work and catch the errors with the worst consequences, because statutory deadlines are arithmetic. Shipping them alone is a defensible interim product; shipping the LLM judge alone would not be.

The underlying discipline is what made the ingestion rescues at FoodLogiQ and Smile CDR survivable under similar pressure: **cut the corner that's cheapest to undo, never the interface.** A rushed synchronous extraction and a fully queued one look identical to the frontend. A rushed schema change never does.

---

## B. Provenance, Audit, and Canadian Compliance Posture

Provenance isn't an addition to this design — [§4.2](Technical-Assessment-Response.md#42-one-schema-with-provenance-on-every-field) attaches source, page, and verified quote to every extracted value for correctness reasons that have nothing to do with compliance. For a government Proof of Value, the same objects become the audit record with modest additions.

**Append-only, not latest-value-only.** An audit log is the easiest thing in a design like this to mistake for compliance scope creep, so it's worth naming what actually generates it. Three mechanisms already in the main response produce *different values for the same field over time*: [§3](Technical-Assessment-Response.md#3-data-pipeline--error-handling) retries extraction after a validation failure, a second upload re-runs it against a larger corpus, and [§A](#a-resource-constrained-pivot-strategy) above stages the extraction stack itself changing beneath live cases. The form a lawyer files is fixed at a moment; the extraction behind it isn't. Store only the current value and *"the form said 12 March when we filed and says 4 April now — which one was wrong?"* has no answer.

It is also the instrumentation whose absence let Defect 2 reach a user bug report. Nothing recorded what the model was actually given, so "the AI ignores my documents" stayed an anecdote rather than a diagnosable claim. [§1.3](Technical-Assessment-Response.md#13-confirming-this-in-the-first-hour) proposes logging the assembled prompt server-side during triage; this is the durable, per-case form of the same thing.

*What writes an event.* The `event_type` values in [§C](#c-data-model-sketch), in the order a case hits them:

| Event | Trigger | Payload carries |
|---|---|---|
| `uploaded` | File accepted or rejected at validation | MIME type, size, rejection reason |
| `extracted` | Extraction completes, per file per `model_id` | Assembled prompt and raw response |
| `judged` | A semantic or judge-based check returns ([§4.4](Technical-Assessment-Response.md#44-methods-by-category)) | Verdict and stated rationale |
| `flagged` | The gate raises a discrepancy ([§4.1](Technical-Assessment-Response.md#41-a-gate-not-a-report)) | The N competing claims, with provenance |
| `resolved` | A human overrides a blocking flag ([§4.6](Technical-Assessment-Response.md#46-surfacing-discrepancies)) | Resolver identity, chosen value |
| `relied_upon` | The form is exported or filed | Snapshot of every field as presented |

The last row matters most and is the one usually missed. The other events describe what the system believed; `relied_upon` is the only record of what a person acted on, and it's where an audit starts.

*Why not update in place.* `discrepancy_flags.resolved_by` holds the latest resolution only. A flag can be raised, resolved, re-raised when a later document arrives, then resolved differently — an `UPDATE` erases the first decision, which is precisely the one that gets asked about. Same argument for `extractions.is_current`: it preserves superseded values but not the prompt, model, and evidence that produced them.

*What it costs.* Prompt/response payloads dominate the volume — that's what `compression.zstd` in [§F](#f-what-python-314-actually-changes-here) is for — and it forces an explicit retention policy instead of an implicit keep-forever. Strip away the Proof of Value framing and the cheaper version is structured application logs at short retention: the debugging value survives, long-horizon per-case reconstruction doesn't. It partly pays for itself either way, since the `extracted` events are the labelled corpus the [§4.4](Technical-Assessment-Response.md#44-methods-by-category) regression harness needs.

**No opaque decisions.** Because extraction and validation are already schema-constrained with retry-on-malformed-output, raw prompt/response pairs are cheap to log and reviewable by a person without re-running the model. Combined with verified citations, every flag a reviewer sees can be traced to text in a specific file on a specific page.

**Residency and privilege — decided early, not discovered late.** Your placeholder text references IRPR s.182, so the case data is Canadian immigration matter, likely with IRCC as the eventual counterparty. Two consequences that a US-default architecture gets wrong:

- **Residency.** PIPEDA plus federal procurement expectations point at Canadian-region hosting (`ca-central-1` or Azure Canada Central) and, for anything at **Protected B**, alignment with the CCCS Medium cloud control profile. A public LLM API with unspecified processing geography is not a safe default. The options are a VPC-scoped endpoint with a signed no-training/no-retention agreement in a Canadian region, or a self-hosted model for the extraction step. This needs a decision before it becomes a blocker, not after the architecture assumes otherwise.
- **Privilege.** Distinct from privacy and more specific to a legal product: transmitting solicitor-client privileged material to a third-party processor raises waiver and law-society confidentiality questions that residency alone doesn't answer. It shapes the contract terms and the retention posture, and it's worth having a defensible answer before a government reviewer asks.

**Retention tied to the existing lifecycle.** The status model in [§3](Technical-Assessment-Response.md#3-data-pipeline--error-handling) already provides the hook: `Ready` is the point extraction is trusted and retained on the case clock; `Failed` and abandoned uploads purge on a much shorter one. Least-privilege access scopes per `fileId`, the same reference used everywhere else.

None of this requires infrastructure beyond what [§2](Technical-Assessment-Response.md#2-architectural-solution--state-management)–[§4](Technical-Assessment-Response.md#4-cross-reference-validation--anomaly-detection) already propose. It's the same objects logged rather than discarded, and the same status model given a retention policy rather than only a UI meaning.

---

## C. Data Model Sketch

Deliberately small. Five tables carry the whole design.

```sql
CREATE TABLE cases (
  case_id        UUID PRIMARY KEY,              -- UUIDv7, minted client-side
  org_id         UUID NOT NULL,
  analysis_role  TEXT NOT NULL,                 -- legal_analyst | counsel_applicant | ...
  prompt_text    TEXT,                          -- server-side home for the typed description
  created_by     UUID NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE case_files (
  file_id        UUID PRIMARY KEY,
  case_id        UUID NOT NULL REFERENCES cases,
  filename       TEXT NOT NULL,
  mime_type      TEXT NOT NULL,
  size_bytes     BIGINT NOT NULL,
  storage_uri    TEXT,                          -- object storage key, or NULL if inline
  sha256         BYTEA NOT NULL,                -- dedupe + tamper evidence
  status         TEXT NOT NULL,                 -- uploading | ... | ready | failed
  page_count     INT,
  uploaded_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON case_files (case_id, status);

-- Current best extraction per case. Superseded rows kept for audit.
CREATE TABLE extractions (
  extraction_id  UUID PRIMARY KEY,
  case_id        UUID NOT NULL REFERENCES cases,
  schema_version TEXT NOT NULL,                 -- 'CaseFieldsV1' — schema evolution
  fields         JSONB NOT NULL,                -- Extracted[T] per field: value+confidence+provenance
  model_id       TEXT NOT NULL,
  is_current     BOOLEAN NOT NULL DEFAULT true,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON extractions (case_id) WHERE is_current;

-- Append-only. Never updated, never deleted within the retention window.
CREATE TABLE extraction_events (
  event_id       BIGSERIAL PRIMARY KEY,
  case_id        UUID NOT NULL REFERENCES cases,
  file_id        UUID REFERENCES case_files,
  event_type     TEXT NOT NULL,                 -- uploaded | extracted | judged | flagged | resolved | relied_upon
  actor          TEXT NOT NULL,                 -- 'system:<model_id>' or 'user:<uuid>'
  payload        JSONB NOT NULL,                -- prompt/response pair, or the resolution
  occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON extraction_events (case_id, occurred_at);

CREATE TABLE discrepancy_flags (
  flag_id        UUID PRIMARY KEY,
  case_id        UUID NOT NULL REFERENCES cases,
  field          TEXT NOT NULL,
  severity       TEXT NOT NULL,                 -- contradiction | unsupported_claim | document_only | ...
  claims         JSONB NOT NULL,                -- [{source, value, provenance}] — N sources
  blocks_prefill BOOLEAN NOT NULL,
  resolved_by    UUID,                          -- NULL until a human decides
  resolved_value TEXT,
  resolved_at    TIMESTAMPTZ
);
CREATE INDEX ON discrepancy_flags (case_id) WHERE resolved_at IS NULL;
```

Three choices worth calling out. `schema_version` on `extractions` means a schema change doesn't invalidate historical records — necessary when an audit reads a case extracted under last year's schema. The partial unique index enforces exactly one current extraction per case while keeping superseded ones. And `discrepancy_flags.resolved_by` records *who* overrode a blocking flag, which is the question an auditor asks first — but only for the most recent resolution, which is why the `resolved` events in `extraction_events` carry the sequence when a flag is raised more than once.

---

## D. Store Implementation Notes

The main response shows the store shape. Two details matter in a Next.js 16 App Router codebase specifically.

**Hydration.** `sessionStorage` doesn't exist during server rendering, and rehydrating during render produces a mismatch between server and client markup. With `skipHydration: true`, rehydration is triggered from an effect and consumers gate on it:

```typescript
'use client';
// components/StoreHydration.tsx — rendered once in the root layout
export function StoreHydration() {
  useEffect(() => { useCaseDraftStore.persist.rehydrate(); }, []);
  return null;
}

// Consumers that must not render pre-hydration state:
const hydrated = useCaseDraftStore((s) => s._hasHydrated);
if (!hydrated) return <DraftSkeleton />;
```

**Debounced autosave**, subscribed at the store rather than wired into a component, so it survives navigation like the store does:

```typescript
const KEEPALIVE_LIMIT = 64 * 1024;   // browsers cap keepalive request bodies

const save = debounce((s: CaseDraftState) => {
  if (!s.caseId) return;
  const body = JSON.stringify({ promptText: s.promptText, analysisRole: s.analysisRole });
  fetch(`/api/cases/${s.caseId}/draft`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body,
    // Survives a tab close mid-flight, but only under the cap — over it the
    // request is rejected outright, so fall back rather than lose the save.
    keepalive: new Blob([body]).size < KEEPALIVE_LIMIT,
  });
}, 800);

useCaseDraftStore.subscribe(save);
```

`keepalive` matters more than it looks: without it, the final debounced save is cancelled when the tab closes, which reintroduces data loss at exactly the moment the user assumes their work is safe. The cap matters just as much in the other direction — a 5,000-word description is around 30 KB so it fits today, but setting `keepalive` on an oversized body fails the request entirely rather than degrading, so the flag is conditional rather than always on.

---

## E. Validation Implementations

**Citation verification.** The point made in [§4.4](Technical-Assessment-Response.md#44-methods-by-category): requiring a citation field catches an absent citation, not a fabricated one. Verification against the source text is what makes the guarantee real.

```python
from pydantic import BaseModel, model_validator
from rapidfuzz import fuzz

class DiscrepancyFlag(BaseModel):
    field: str
    severity: Severity
    claims: list[Claim]
    blocks_prefill: bool

    @model_validator(mode="after")
    def citations_must_exist(self, info):
        """Reject any flag whose cited span cannot be located in the source it names.

        Fuzzy rather than exact: OCR output drifts from the model's transcription
        of it by a character or two, and an exact match would reject valid flags.
        90 tolerates that without admitting a fabricated quote.
        """
        corpus = info.context["source_text_by_id"]
        for c in self.claims:
            source_text = corpus.get(c.provenance.source)
            if source_text is None:
                raise ValueError(f"unknown source: {c.provenance.source}")
            if fuzz.partial_ratio(c.provenance.quote, source_text) < 90:
                raise ValueError(f"cited span not present in {c.provenance.source}")
        return self
```

**The statutory deadline rule.** Deterministic, no model involved, and the highest-consequence check in the system.

```python
RESTORATION_WINDOW_DAYS = 90   # IRPR s.182(1); no officer discretion to extend

def restoration_days_remaining(
    status_expiry: date, refusal_date: date | None, today: date
) -> int:
    # The clock starts the day after the refusal decision when an extension was
    # filed and refused; otherwise the day after the printed expiry date. Which
    # of the two applies is precisely what a `document_only` flag surfaces when
    # the client uploads a refusal letter they never mentioned in the prompt.
    start = (refusal_date or status_expiry) + timedelta(days=1)
    return RESTORATION_WINDOW_DAYS - (today - start).days


def check_restoration(fields: CaseFieldsV1, today: date) -> DiscrepancyFlag | None:
    expiry = fields.status_expiry_date.value
    if expiry is None:
        return DiscrepancyFlag(
            field="status_expiry_date", severity="missing_field",
            claims=[], blocks_prefill=True,
        )
    remaining = restoration_days_remaining(expiry, fields.refusal_date.value, today)
    if remaining <= 0:
        # Out of time: restoration is unavailable and the form selection itself
        # is wrong. A TRP under IRPA s.24 is the remaining path.
        return DiscrepancyFlag(
            field="case_type", severity="contradiction",
            claims=[...], blocks_prefill=True,
        )
    return None
```

The reason this belongs in code rather than a prompt: the answer is arithmetic over two dates, the consequence of getting it wrong is an ineligible filing, and no amount of model quality makes a probabilistic answer preferable to a deterministic one when a deterministic one exists.

---

## F. What Python 3.14 Actually Changes Here

Targeting 3.14 rather than 3.12 is worth more than a version bump for this particular backend, though one of the headline features is a judgement call rather than a free win.

**Free-threading (PEP 779) — evaluate, don't assume.** This is the one that matters most and deserves the most caution. The extraction stage is genuinely CPU-bound, which is the workload the GIL has always punished: threads gave you nothing, so the answer was process pools and their memory cost. Free-threading is officially supported in 3.14 and reports near-linear scaling on CPU-bound work, with single-threaded overhead down to roughly 5–10% from ~40% in 3.13.

The caveats are real and specific:

- It's a **separate opt-in build** (`python3.14t`), not the default interpreter.
- If any imported C extension hasn't opted in, the interpreter **silently re-enables the GIL** — you pay the overhead and get none of the parallelism, with no error to tell you.
- The gain is uneven across this pipeline. `pdfplumber` is largely pure Python and would benefit; Tesseract runs as a subprocess and wouldn't care either way; Pillow and NumPy sit somewhere in between depending on version.
- Memory runs roughly 15–20% higher, since the free-threaded build uses mimalloc rather than pymalloc.

So the honest recommendation is not "switch the backend." It's that the **extraction worker is the right candidate** — it's already an isolated queue consumer in [§3](Technical-Assessment-Response.md#3-data-pipeline--error-handling) with a dependency stack small enough to audit and pin, which is exactly the profile the ecosystem guidance calls for. Benchmark it against the existing process pool, assert `sys._is_gil_enabled() is False` at worker startup so a silent GIL re-enable fails loudly instead of quietly halving throughput, and leave the API service on the default build where it would only pay the overhead. If it doesn't win, PEP 734 subinterpreters (`concurrent.interpreters`) are the other new option, with better isolation than threads and lower cost than processes.

**Things that are straightforwardly better, no trade-off:**

| Feature | Why it matters here |
|---|---|
| `uuid.uuid7()` native | The design keys everything on UUIDv7. In 3.12 that meant a third-party dependency; now it's stdlib, and the time-ordering gives better index locality on `cases` and `case_files` than random v4 |
| t-strings (PEP 750) | Prompt assembly keeps document-derived text structurally separate from instructions rather than concatenated — see [§3](Technical-Assessment-Response.md#3-data-pipeline--error-handling) of the main response |
| PEP 768 debugger interface | Attach to a running extraction worker and inspect in-flight asyncio tasks without restarting it. Directly useful for the triage in [§1.3](Technical-Assessment-Response.md#13-confirming-this-in-the-first-hour), and for a stuck job in production |
| `compression.zstd` (PEP 784) | `extraction_events` stores raw prompt/response pairs and has to be retained for years. Compressing payloads with a stdlib codec, no new dependency, is a meaningful storage win on the audit log |
| Deferred annotations (PEP 649) | Pydantic forward references resolve without `from __future__ import annotations`, and import cost drops — minor, but this codebase is schema-heavy |

**What I'd ignore for now:** the experimental JIT and the tail-call interpreter both need a source build and offer modest gains, which isn't a trade worth making on a system whose bottleneck is OCR and network calls to a model provider, not interpreter dispatch.
