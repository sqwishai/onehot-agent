---
name: onehot
description: Use OneHot from code or through its MCP tools. Run decisions (a probability for each allowed outcome of a question) with POST /v1/decide, check questions with the linter, tune a decision's wording on labelled rows, train families of models on datasets that grow in versions, read the gate, and switch, keep warm or roll back families and models. Use for OneHot API, SDK and MCP work, not for general text generation or training on other platforms.
---

# OneHot

A decision is one question with a fixed set of allowed outcomes, answered with a
probability for each outcome. One request can carry several decisions about the same
context. The server's `/openapi.json` is the contract and `/llms.txt` is a short map of it.
Do not invent fields.

Install the Python client with `pip install sqwishai`. Then `from sqwishai import OneHot`
and connect with
`OneHot(os.environ["SQWISH_BASE_URL"], api_key=os.environ["SQWISH_API_KEY"])`
(`AsyncOneHot` has the same methods). Install the TypeScript client with
`npm install sqwishai` and `import { OneHot } from "sqwishai"`; it reads the same
`SQWISH_BASE_URL` and `SQWISH_API_KEY`. If the OneHot
MCP server is connected, its tools do the same jobs: `decide`, `estimate_decide`,
`lint_decision`, `list_recipes`, `create_dataset`, `add_dataset_rows`,
`list_dataset_versions`, `start_prompt_tuning`, `get_prompt_tuning`, `start_fine_tune`,
`estimate_fine_tune`, `continue_family`, `estimate_continue`, `get_fine_tune`,
`list_families`, `get_family`, `switch_family`, `keep_family_warm`, `roll_back_family`,
`delete_family`, `list_models`, `get_model`, `model_ready`, `switch_model`,
`keep_model_warm` and `delete_model`.
Setup is in the Agents guide, https://docs.sqwish.ai/agent-skills.

Every resource reads one item with `get`. Most list calls return one page, newest first;
pass `limit` and the page's `next_cursor` as `cursor` for the next. To go through all of
them, use `iter()`, as in `for job in client.fine_tuning.iter():` (`async for` in the async
client and TypeScript). `client.models.list()`, `client.families.list()` and
`client.datasets.versions()` take no cursor and return the whole list.

## Integrate a decide call

```python
decisions = [
    {"id": "team", "question": "Which team should handle this ticket?",
     "outcomes": {"billing": "Charges, invoices and refunds.",
                  "technical": "Anything that is broken.", "other": "Anything else."},
     "policy": {"abstain_below": 0.6}},
    {"id": "urgent", "kind": "binary", "question": "Does the ticket state a deadline within one day?"},
]
print(client.lint("A sample ticket.", decisions)["warnings"])  # fix these before shipping
answer = client.decide(ticket_text, decisions, model="kettei-core")
team = answer["decisions"]["team"]  # probabilities, top, p_top; policy.action: "abstain" or a team
```

- `kind` is `single` (the default), `binary` or `ordinal`. An ordinal decision uses outcomes
  `["0", "1", ...]` and a `rubric` with one line per level.
- Give a way out, such as `"other"`, and ask questions the positive way. The linter flags both.
- Keep outcomes and JSON keys in their original order, because order changes what the model
  reads. Do not sort keys when you retry or move examples into training.
- Use the probabilities as returned. Do not ask for explanations, parse text or round before
  a threshold. The base models' calibration is not measured yet (a trained version reports
  `ece` on its held-out rows), so pick thresholds on your own labelled data.
- `GET /v1/models` (or `list_models`) shows which models this server runs. `list_recipes`
  has complete requests for common jobs.
- `POST /v1/playground/decide` serves the console's playground only, behind Cloudflare's
  browser check, so an agent calls `/v1/decide` with a key instead.
- A version that hasn't been used for a while may still be loading. With
  `fallback="none"` that is a 503 `model_warming` error with `Retry-After: 5` that charged
  nothing; its idempotency key keeps the failure, so send the call again after 5 seconds with a
  new key. `wait_for_warm=30` on `decide` does this for you. Keep warm keeps a model loaded
  for steady latency.
- Code written for TypeSafe's Jev (`state`, `questions`) can move to `/v1/systemone` with a
  OneHot key and model id. Write new code against `decide`, and do not copy Jev's
  `confidence` into it. `/llms.txt` shows both shapes side by side.

## Tune the wording before you train

With 50 to 1,000 training rows asking a decision, and 5 or more for each outcome, prompt
tuning rewrites the question, outcome descriptions and rubric, then tests the result once on
rows it never saw. Outcome ids never change. It tunes for a base model, or for a fine-tuned
model while it answers calls (`model="router@3"`).

```python
data = client.datasets.upload("tickets.jsonl", idempotency_key="tickets-v1")
job = client.prompt_tuning.create(data["id"], decision="team", idempotency_key="tune-team")
job = client.prompt_tuning.wait(job["id"], timeout=3600)  # raises JobFailed if it fails
result = job["result"]
if result["verdict"] == "improved":
    decisions[0] = result["tuned"]  # use it only when the verdict is "improved"
```

For `inconclusive` or `no_improvement`, keep the user's wording and show them
`result["candidate"]` and `result["label_issues"]` rather than using either. A job's fee
is held while it runs. A failed or cancelled job releases it, and so does a first test of a
dataset's rows that proves no improvement; a rerun on rows tested before is charged when its
test runs, whatever the verdict. `job["charge"]` says which. `cancel` is never repeated
automatically; call `prompt_tuning.get` to see whether the job stopped.

## Train a family's first model

Each example is a decide request without `model`, plus `targets` (a probability for every
outcome, summing to 1) or `rewards` (a reward for every outcome, never a single logged
reward). A family starts on 100 to 10,000 training rows, and its gate needs 30 held-out decisions
to judge, so give it enough rows: about a fifth of the contexts are held out.

```python
data = client.datasets.upload("tickets.jsonl", idempotency_key="tickets-v1")
job = client.fine_tuning.create(data["id"], family="router", idempotency_key="router-v1")
job = client.fine_tuning.wait(job["id"], timeout=6 * 3600)  # raises JobFailed on failure
```

No labels yet? Upload the real queries with `labeling.sources.upload`, create a
`labeling.jobs` run with `teacher_models`, review the results page by page, then `finalize`
the row indices you accept into a dataset. Teacher votes are labels, not calibrated
confidence, and teachers must not invent questions or outcomes.

`family="router"` names a new family: a name already taken gets `family_exists`. Pass
`start_from="router@2"` to start it from one of your models instead of core; then only
decisions about contexts that model's line didn't train on count toward the 30. Too few gets
`not_enough_evaluation_data`.

## Add rows and train its next model

Add the new rows to the family's dataset as its next version, then continue the family:

```python
client.datasets.add_rows(data["id"], new_rows, family="router", idempotency_key="tickets-v2")
# or client.datasets.upload_rows(data["id"], "more.csv", family="router") for JSONL or CSV
job = client.fine_tuning.continue_family("router", idempotency_key="router-next")
job = client.fine_tuning.wait(job["id"], timeout=6 * 3600)
```

The rows must ask the dataset's own decisions, worded the same way, or they get
`task_mismatch`: a different task needs a new dataset and a new family. A context the dataset
holds keeps its side of the split, so evaluation rows never enter training.
`client.datasets.versions(data["id"])` lists the versions, and `download(data["id"], version=1)`
returns one as JSONL.

It trains from the family's latest model on the rows added since the family last trained,
with as many earlier rows mixed in to reduce forgetting. Without new rows it gets
`no_new_data`, and with fewer than 30 new held-out decisions `not_enough_evaluation_data`.

It trains on the dataset's newest version. To train on an earlier one, name it:
`client.fine_tuning.continue_family("router", dataset_version=2)`, or `dataset_version` on the
`continue_family` tool. The rows added after it stay new for the next model. It must be newer
than the version the family's last finished job trained on, or it gets `no_new_data`.

## Read the gate

`job["metrics"]["gate"]` compares the new model with its parent, or its base when the family
started from one, on the same held-out decisions, split into earlier and new rows. `passed`
is the verdict and `reasons` says what failed. A model that fails joins nothing: the job's
`outcome` is `gate_failed`, and nothing can call it. A model that passes joins the family as
its next number. `client.families.get("router")` shows every model's gate, and `jobs` shows
each fine-tune with how it ended and what it cost. Report the gate
to the user as it is; do not describe a version as better unless it passed.

## See which model a family serves

```python
family = client.families.get("router")
family["served"]["ref"]  # such as "router@2": what calls to "router" get; family["latest"] is the newest
```

A family that is switched on serves its latest model. The server loads and probes the latest
model, then moves the family to it at its next check, every 30 seconds, so a model that just
joined may not answer `router` yet: wait until `served` is the latest. If it stays behind,
read `family["move"]["waiting_for"]`: `place` means the account has no place left for it, so
tell the user, who can switch a model off or move to a higher tier; `warm_place` means a
kept-warm family's move would keep more models warm than the tier allows, so tell the user,
who can turn keep warm off on a model or move to a higher tier; `warm_capacity` means the
model's size has no kept-warm place free on our engines, so the move waits until one frees;
`engines` means the model loads or waits to load again. A family that is switched off doesn't
move.

## Switch, keep warm and roll back

`client.families.switch("router", False)` stops a family answering, and
`client.models.switch("router@1", True)` makes one model answer calls to its number. Name a
model by its number, never by an id. `keep_warm` on either keeps the model loaded, billed by the
hour where it is billed. Past the tier's limit these raise `live_limit` or `warm_limit`, whose
`details` list what answers or is kept warm, least recently used first: show the user, and pass
`switch_off=` only with what they agreed to switch off.
`client.families.rollback("router", "router@1", ["router@2"])` discards every newer model for
good. `client.models.delete("router@1")` deletes one model once it is neither its family's latest
nor answering, such as router@1 while router serves router@3. Confirm both with the user first.

## When something fails

Errors are `{"error": {code, message, retryable, request_id, details}}`. Retry only when
`retryable` is true. On a 422, fix the fields in `details` rather than resending the same
body. Reuse the same `idempotency_key` when you retry a decision or creating a dataset or job,
and use a new one for a new run. If you passed none, the error names the key the call sent
(`idempotency_key` on the SDK's `APIError`, on its `TimeoutError` or httpx error when a call
got no answer, and in a tool's error): send that one, because a retry without it is a second
operation and may be charged as well. A job that is still running after a wait timed out is
still running; poll it with `fine_tuning.get` or `prompt_tuning.get`. Do not claim a trained
model, a tuned wording, a gate result or a latency figure unless the job or response shows it.
