# Appendix — Supporting Detail

**Companion to the Technical Assessment Response. Optional reading; the main response stands alone.**

This covers material your email indicated would come up in the final conversation — pivoting under resource constraints, and the schemas behind the design — plus the compliance posture the government Proof of Value work implies. Section references point back to the main document.

---

## A. Resource-Constrained Pivot Strategy

The architecture in the main response is the target state. Dropped into this as immediate rescue work — tight timeline, no infrastructure budget yet, small team — I wouldn't build all of it before shipping a fix. Each corner below is cut deliberately, and each sits behind an interface (`fileId`, `CaseFieldsV1`, the file `status` enum) that doesn't change when the corner is later un-cut.

| Constraint | Cut | What it costs | Why the interface holds |
|---|---|---|---|
| No Redis/BullMQ | Run extraction synchronously in the upload handler behind a loading state; cap file size and page count to keep p95 under ~15s | Long documents rejected rather than queued | The frontend still polls `fileId` for status; introducing a queue is a backend-only change |
| No OCR/vision infrastructure | Send PDF/PNG directly to the LLM vendor's native multimodal document support | Higher per-call cost | Removes an entire pipeline stage; the extraction contract is unchanged |
| No object storage | Store bytes in a `bytea` column in the existing Postgres, with a hard size cap, keyed by the same `fileId` | Bloats the database, no CDN | Migrating to S3/GCS touches only the storage adapter |
| No time for cross-device sync | Ship the in-memory store alone; it fully fixes Defect 1. Server autosave lands as a fast-follow | Refresh loses the draft | `PATCH /cases/:id/draft` is additive; no schema change |
| No time for full validation | Ship deterministic checks only (§4.4, first bullet) — presence, types, dates, the s.182 rule | No semantic or judge-based detection | The severity taxonomy and `DiscrepancyFlag` are already N-source; semantic checks append flags to the same structure |

That last row is the one I'd defend hardest. Deterministic checks are perhaps fifteen percent of the validation work and catch the errors with the worst consequences, because statutory deadlines are arithmetic. Shipping them alone is a defensible interim product; shipping the LLM judge alone would not be.

The underlying discipline is what made the ingestion rescues at FoodLogiQ and Smile CDR survivable under similar pressure: **cut the corner that's cheapest to undo, never the interface.** A rushed synchronous extraction and a fully queued one look identical to the frontend. A rushed schema change never does.

---

## B. Provenance, Audit, and Canadian Compliance Posture

Provenance isn't an addition to this design — §4.2 attaches source, page, and verified quote to every extracted value for correctness reasons that have nothing to do with compliance. For a government Proof of Value, the same objects become the audit record with modest additions.

**Append-only, not latest-value-only.** Persist every extraction, every judge verdict, and every human resolution to an immutable `extraction_events` log keyed by `caseId` (schema in §C). "What did the system claim, from which source, when, and who overrode it" has to be answerable months later, not only at request time.

**No opaque decisions.** Because extraction and validation are already schema-constrained with retry-on-malformed-output, raw prompt/response pairs are cheap to log and reviewable by a person without re-running the model. Combined with verified citations, every flag a reviewer sees can be traced to text in a specific file on a specific page.

**Residency and privilege — decided early, not discovered late.** Your placeholder text references IRPR s.182, so the case data is Canadian immigration matter, likely with IRCC as the eventual counterparty. Two consequences that a US-default architecture gets wrong:

- **Residency.** PIPEDA plus federal procurement expectations point at Canadian-region hosting (`ca-central-1` or Azure Canada Central) and, for anything at **Protected B**, alignment with the CCCS Medium cloud control profile. A public LLM API with unspecified processing geography is not a safe default. The options are a VPC-scoped endpoint with a signed no-training/no-retention agreement in a Canadian region, or a self-hosted model for the extraction step. This needs a decision before it becomes a blocker, not after the architecture assumes otherwise.
- **Privilege.** Distinct from privacy and more specific to a legal product: transmitting solicitor-client privileged material to a third-party processor raises waiver and law-society confidentiality questions that residency alone doesn't answer. It shapes the contract terms and the retention posture, and it's worth having a defensible answer before a government reviewer asks.

**Retention tied to the existing lifecycle.** The status model in §3 already provides the hook: `Ready` is the point extraction is trusted and retained on the case clock; `Failed` and abandoned uploads purge on a much shorter one. Least-privilege access scopes per `fileId`, the same reference used everywhere else.

None of this requires infrastructure beyond what §2–§4 already propose. It's the same objects logged rather than discarded, and the same status model given a retention policy rather than only a UI meaning.

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
  event_type     TEXT NOT NULL,                 -- uploaded | extracted | judged | flagged | resolved
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

Three choices worth calling out. `schema_version` on `extractions` means a schema change doesn't invalidate historical records — necessary when an audit reads a case extracted under last year's schema. The partial unique index enforces exactly one current extraction per case while keeping superseded ones. And `discrepancy_flags.resolved_by` records *who* overrode a blocking flag, which is the question an auditor asks first.

---

## D. Store Implementation Notes

The main response shows the store shape. Two details matter in a Next.js codebase specifically.

**Hydration.** `sessionStorage` doesn't exist during server rendering, and rehydrating during render produces a mismatch between server and client markup. With `skipHydration: true`, rehydration is triggered from an effect and consumers gate on it:

```typescript
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
const save = debounce((s: CaseDraftState) => {
  if (!s.caseId) return;
  fetch(`/api/cases/${s.caseId}/draft`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ promptText: s.promptText, analysisRole: s.analysisRole }),
    keepalive: true,          // survives a tab close mid-flight
  });
}, 800);

useCaseDraftStore.subscribe(save);
```

`keepalive` matters more than it looks: without it, the final debounced save is cancelled when the tab closes, which reintroduces data loss at exactly the moment the user assumes their work is safe.

---

## E. Validation Implementations

**Citation verification.** The point made in §4.4: requiring a citation field catches an absent citation, not a fabricated one. Verification against the source text is what makes the guarantee real.

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
