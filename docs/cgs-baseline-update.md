# Record baseline provenance for CGS annotations

Use `collection_event: "Baseline"`, matching `sample.collection_event` and its
shared dictionary term. `sample_type` describes the specimen type, not timing.
This annotation does not create a sample link or assert a particular sample ID.
The case relationship and its cardinality are unchanged.

The field is optional to allow a nonbreaking deployment before backfilling.
There is deliberately no schema default: existing records must be updated
explicitly. Future CGS submissions should explicitly include this field. Do not
add it to the cohort builder's facet configuration.

## 1. Release and deploy the dictionary

Merge this dictionary change, publish a new release, and verify its versioned
CDN `schema.json` contains `cgs_risk_key_criteria.properties.collection_event`.
Update **dev only** in `mmrf_gen3` to that release URL. Wait for deployment and
check the dictionary served by Sheepdog, not just the CDN (see below).
No records need to be deleted or re-ingested.

## 2. Backfill existing dev records

Run the following cells in your existing authenticated ingestion notebook
environment (with the `gen3` SDK installed). Use a new notebook to retain the
history of prior migrations. Pause other CGS submissions during this update.
Backups contain project data: keep the audit directory in approved storage,
outside Git, and do not commit credentials or exported records.

Configure the project and credentials explicitly; this procedure only targets dev:

```python
import copy
import hashlib
import json
from datetime import datetime, timezone
from pathlib import Path
from gen3.auth import Gen3Auth
from gen3.submission import Gen3Submission

API = "https://dev-virtuallab.themmrf.org"
PROGRAM = "MMRF"
PROJECT = "COMMPASS-IA24"  # Confirm the project used by your previous notebook.
NODE = "cgs_risk_key_criteria"
CREDENTIALS = Path("/absolute/path/to/dev-credentials.json")
AUDIT = Path("cgs-baseline-" + datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%S%fZ"))
AUDIT.mkdir(exist_ok=False)
sub = Gen3Submission(API, Gen3Auth(API, refresh_file=str(CREDENTIALS)))

schema = sub.get_dictionary_all()
if isinstance(schema, str):
    schema = json.loads(schema)
assert schema[NODE]["properties"]["collection_event"]["enum"] == ["Baseline"]

def export(name):
    path = AUDIT / name
    sub.export_node(PROGRAM, PROJECT, NODE, "json", filename=str(path))
    data = json.loads(path.read_text())
    return data["data"] if isinstance(data, dict) else data

before = export("before.json")
assert before, "No CGS records exported; check the project."
assert all(r.get("id") and r.get("submitter_id") for r in before)
assert len({r["id"] for r in before}) == len(before)
assert len({r["submitter_id"] for r in before}) == len(before)
assert all(r.get("collection_event") in (None, "", "Baseline") for r in before), \
    "Unexpected collection event: investigate rather than overwriting it."

payload = []
for r in before:
    if r.get("collection_event") == "Baseline":
        continue
    assert r.get("cases") and r.get("cgs_risk_category")
    item = {k: copy.deepcopy(r[k]) for k in (
        "id", "submitter_id", "cases", "cgs_risk_category",
        "cgs_risk_criteria", "other_criteria"
    ) if k in r and r[k] is not None}
    item.update(type=NODE, collection_event="Baseline")
    payload.append(item)

(AUDIT / "payload.json").write_text(json.dumps(payload, indent=2, ensure_ascii=False))
(AUDIT / "manifest.json").write_text(json.dumps({
    "endpoint": API, "program": PROGRAM, "project": PROJECT,
    "exported": len(before), "updates": len(payload),
    "reason": "Team confirmed all CGS annotations derive from baseline samples",
    "sha256": {name: hashlib.sha256((AUDIT / name).read_bytes()).hexdigest()
               for name in ("before.json", "payload.json")},
}, indent=2))
print(f"Exported {len(before)} records; planned {len(payload)} updates. Audit: {AUDIT}")
```

Review the backup, payload, project, and count before enabling this next cell.
The update preserves the same record IDs, case links, and risk values. It does
not delete anything. If a batch fails, stop and inspect the saved response;
earlier batches may already have succeeded. A fresh run safely skips records
already marked Baseline.

```python
APPLY = False  # Change only after reviewing the plan above.
assert APPLY, "Review the plan, then explicitly enable APPLY."

def stable(records):
    # Ignore only server-maintained lifecycle metadata for drift detection.
    return {r["id"]: {k: v for k, v in r.items()
                       if k not in ("created_datetime", "updated_datetime", "state")}
            for r in records}

assert stable(export("preflight.json")) == stable(before), \
    "Records changed since planning; create a fresh plan."
for start in range(0, len(payload), 100):
    batch = payload[start:start + 100]
    response = sub.submit_record(PROGRAM, PROJECT, batch)
    if isinstance(response, str):
        response = json.loads(response)
    (AUDIT / f"response-{start:06d}.json").write_text(json.dumps(response, indent=2))
    assert response.get("code") == 200, response
    entities = response.get("entities", [])
    assert len(entities) == len(batch) and all(e.get("valid") for e in entities), response

after = export("after.json")
expected = copy.deepcopy(before)
for r in expected:
    r["collection_event"] = "Baseline"
assert stable(after) == stable(expected), "Post-update mismatch: inspect the audit files."
print(f"Verified {len(after)} records: Baseline added, other data unchanged.")
```

## 3. Propagate into search and the dev API

1. Install this dictionary release in the plaster environment, regenerate
   `mmrfdatamodel2`, review and commit the generated model.
2. In `mmrf-models`, install the updated dictionary and generated datamodel
   using your established installation procedure, then run `sync-models`.
   Review the generated mappings for
   `cgs_risk_key_criteria.properties.collection_event` and commit them.
   The CGS node is already registered in model sync; no new registration is needed.
3. On the ETL instance, install the updated model packages in **both** ESBuild
   and mutation-indexer environments. Rebuild any packaged dependency artifacts
   your launcher uses; pulling Git alone does not update installed packages.
4. Run ESBuild with a fresh prefix against the updated dev source data, then
   mutation indexer with those case/file inputs and a fresh output name. Do not
   reuse previously cached case data. This new field is a scalar string: it
   does not belong in mutation indexer's `include_as_arrays` lists.
5. Before transferring, inspect a populated case's `_source` and the mappings
   in both the ESBuild case index and mutation-indexer case-centric index.
   Confirm `cgs_risk_key_criteria.collection_event` is `Baseline`; cases without
   a CGS record should not acquire fabricated baseline annotations.
6. Transfer completed indices to dev and switch aliases using the established
   deployment procedure. Restart Guppy / the analysis API as needed to refresh
   their mapping-derived schemas and caches. Verify the new field in an API
   response for a known CGS case, plus an unchanged case count.

No frontend facet or match-mode changes are required. Sheepdog can expose the
stored field immediately after the backfill; Elasticsearch-backed APIs require
the refreshed indices (and any endpoint-specific field allowlist, if applicable).
Retain the previous release indices for rollback; do not touch prod in this rollout.
