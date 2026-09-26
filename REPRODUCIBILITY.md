# Reproducibility guide

## Snapshot purpose

This archive supports review of the original BioHub Cell Tracking research
logic, tests, methods, campaign records, data-lineage notes, and lightweight
receipts. It excludes data, generated artifacts/output, submissions, and the
upstream implementation clone.

## Entry points

- `CAMPAIGN.md` is the historical campaign record.
- `DATA_LINEAGE.md` describes input and derivative boundaries.
- `audit_campaign.py` cross-checks the full original campaign evidence.
- `tests/` contains the source-level validator and finalizer tests.
- `THIRD_PARTY_NOTICES.md` and `THIRD_PARTY_LICENSES/` record dependency
  notices retained in the snapshot.

After reconstructing the documented Python environment, a code-level test
entry from the repository root is:

```bash
python3 -m unittest discover -s tests -v
```

The full campaign audit is not a standalone public-snapshot smoke test: it
references excluded `data/`, `artifacts/`, output, and remote-run material.
Supply only inputs obtained through authorized sources, preserve their hashes,
and do not copy the upstream clone into a public derivative.

## Claim boundary

A passing unit test demonstrates only the tested code path in the reconstructed
environment. It does not recreate a leaderboard result, establish current
competition status, resolve personal eligibility, or clear the third-party
license issues recorded in the campaign archive.
