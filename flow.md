# Ontology Agent — Step-by-Step Code Trace

This traces one real request through the whole system, in the exact order
it executes, with the file + function + line number for every step. Read
top to bottom once; after that, use it as a map back into the code.

---

## Phase A — Happens once, when the backend process starts

### A1. Backend entrypoint loads environment variables
**File:** `backend/app/main.py:14`
```
load_dotenv(Path(__file__).resolve().parents[2] / ".env")
```
Loads `GOOGLE_CLOUD_PROJECT`, `GOOGLE_GENAI_USE_VERTEXAI`, `ONTOLOGY_MODEL`, etc.
from the project-root `.env` **before** anything else imports the agent —
required because the agent needs those variables the moment it's built.

### A2. main.py imports the routes, which imports the agent package
**File:** `backend/app/main.py:26` → `from .routes import router`
`routes.py:6` → `from . import runner_service`
`runner_service.py:20-22` inserts `agent/` onto `sys.path` so the sibling
`ontology_agent` package (not pip-installed) becomes importable, then
`runner_service.py:27` → `from ontology_agent import root_agent`.

### A3. Importing `ontology_agent` builds the agent object
**File:** `agent/ontology_agent/__init__.py:1` → `from .agent import root_agent`

This runs `agent.py` top to bottom:
- `agent.py:6` reads `settings.model_name` (from `config.py:24`, which read
  `ONTOLOGY_MODEL` off the environment at `config.py:20`).
- `agent.py:7` imports `ONTOLOGY_AGENT_INSTRUCTION` from `prompt.py:7-42`
  — the entire system prompt, as one string constant.
- `agent.py:9-16` constructs `root_agent = Agent(...)` — the ADK agent
  object itself. No tools, no `output_schema` (see the comment on
  `schema.py:4-9` for why).

### A4. Importing `ontology_agent.data` reads every data file off disk
**File:** `agent/ontology_agent/data.py`
- `data.py:14-16` compute `DATA_DIR`/`STANDARDS_DIR`/`SYNTHETIC_DIR` as
  paths relative to this file (`parents[2]` = project root).
- `data.py:52` → `_load_standards_text()` (defined at `data.py:23-31`)
  reads `source_registry.yaml` and `fleet_standards_profile.yaml` as raw
  text and concatenates them into `SYNTHETIC_ISO_STANDARD`.
- `data.py:53` → `_load_fleet_data_text()` (defined at `data.py:34-47`)
  reads `requirements.md` + all 6 synthetic CSVs as raw text and
  concatenates them into `SYNTHETIC_FLEET_DATA`.

**Nothing is parsed as YAML or CSV here** — both constants are just long
strings of file contents, read once at import time.

### A5. `runner_service.py` builds one shared ADK Runner
**File:** `backend/app/runner_service.py:34`
```
_runner = InMemoryRunner(agent=root_agent, app_name=APP_NAME)
```
One `Runner` instance for the whole process's lifetime. Sessions (created
per browser page load — see Phase B) are what keep different
conversations separate; the `Runner` itself is just the execution engine.

---

## Phase B — Happens once per browser page load

### B1. React app mounts and calls the backend
**File:** `frontend/src/App.jsx:31-40` (the `useEffect`)
Calls `getInitOntology()` (`App.jsx:19-22`), a module-level cached promise
— guarantees exactly one real network call even under React Strict Mode's
dev-only double-mount (explained in the comment at `App.jsx:11-17`).
That function calls `initOntology()`, defined in `frontend/src/api.js:4-8`,
which does:
```
POST http://localhost:8000/api/init
```

### B2. FastAPI route handles the request
**File:** `backend/app/routes.py:16-28` (`init_ontology`)
Calls `runner_service.generate_initial_ontology()` at line 23.

### B3. A brand-new ADK session is created
**File:** `backend/app/runner_service.py:66-77` (`generate_initial_ontology`)
- Line 70: `_create_session()` (defined at lines 48-52) asks ADK's
  `session_service` for a fresh session — this is what makes every page
  load start with zero conversation history, instead of one process-wide
  session growing forever (a bug this project hit and fixed — see the
  module docstring at lines 5-11).
- Lines 71-75: builds the **first message** by literally concatenating
  `"STANDARD:\n"` + `SYNTHETIC_ISO_STANDARD` + `"\n\nCLIENT FLEET DATA:\n"`
  + `SYNTHETIC_FLEET_DATA` + an instruction sentence.
- Line 76: calls `_ask(session_id, message)`.

### B4. The message is sent to Gemini through ADK
**File:** `backend/app/runner_service.py:55-63` (`_ask`)
- Line 56: wraps the message as a `types.Content` object.
- Lines 58-62: `_runner.run_async(...)` actually sends it — this is the
  ADK call that reaches Gemini (via Vertex AI, using the `.env` project
  config from A1). It streams back a sequence of `Event`s; the loop keeps
  overwriting `final_text` until `event.is_final_response()` is true.

### B5. **This is where the ontology is actually created** — inside the model
Not a function in this codebase — this is the LLM itself, given:
- The system instruction (`prompt.py:7-42`): ground everything in the
  standard/data, never drop a standard-required entity/attribute even if
  the client data omits it (rule 2, `prompt.py:31-34`), respond with
  *only* the JSON shape shown at `prompt.py:18-26`.
- The user message built in B3: the full standards YAML text + the full
  CSV text.

The model reads standard-rule sentences (e.g. "every Asset SHALL be
assigned to exactly one Depot") and CSV column names (e.g. `depot_id`)
as plain text and infers entities, attributes, and relationship edges
from them. No code here builds the graph structure — it's entirely the
model's reasoning over text.

### B6. The response text is cleaned and validated
**File:** `backend/app/runner_service.py:63` calls into:
- `_strip_code_fences` (`runner_service.py:37-45`) — defensively removes
  any ` ```json ` wrapper the model added despite being told not to.
- `Ontology.model_validate_json(...)` — `Ontology` is defined at
  `agent/ontology_agent/schema.py:26-29` (with `Entity` at lines 15-17 and
  `Relationship` at lines 20-23). This is the actual correctness gate: if
  the model's JSON doesn't match this shape, this line raises instead of
  passing bad data downstream.

### B7. Result flows back up to the HTTP response
**File:** `backend/app/runner_service.py:77` returns `(session_id, ontology)`
→ `backend/app/routes.py:23` receives it → lines 26-28 turn it into a
plain dict and add `session_id` into it → returned as the JSON HTTP
response body.

### B8. The browser stores it and renders the graph
**File:** `frontend/src/App.jsx:32-39`
- Line 34: `setOntology(data)` — stores the ontology in React state.
- Line 35: `setSessionId(data.session_id)` — stores the session id, needed
  for every future chat message.
- Line 36: builds the first chat-panel message text via `summarize()`
  (`App.jsx:6-9`).
- Line 73: `<GraphView ontology={ontology} />` renders it.

### B9. GraphView turns the ontology JSON into an actual picture
**File:** `frontend/src/components/GraphView.jsx`
- Lines 22-36: reshapes `{entities, relationships}` into the
  `{nodes, links}` shape the graph library expects (`useMemo`, so this
  only recomputes when `ontology` changes).
- Lines 41-82: hands that to `<ForceGraph2D>` — a third-party library
  doing all the physics layout, drag, zoom, and click handling. The only
  custom part is `nodeCanvasObject` (lines 55-75), which draws each node
  as a colored circle + label (color picked deterministically per entity
  name by `colorFor()`, lines 12-16).
- Lines 91-104: clicking a node (`onNodeClick`, line 52) shows its
  attribute list in the sidebar.

---

## Phase C — Happens every time the user sends a chat message

### C1. User submits the input box
**File:** `frontend/src/components/ChatPanel.jsx:11-16` (`submit`) calls
the `onSend` prop, which is `handleSend` passed down from `App.jsx:71`.

### C2. App.jsx sends the message + the existing session id
**File:** `frontend/src/App.jsx:42-54` (`handleSend`)
- Line 46: `sendChatMessage(text, sessionId)` — defined at
  `frontend/src/api.js:10-18` — posts `{message, session_id}` to
  `POST /api/chat`.

### C3. Backend route dispatches on the existing session
**File:** `backend/app/routes.py:31-40` (`chat`)
Calls `runner_service.update_ontology(request.session_id, request.message)`
at line 35 — **note**: no new session is created here, unlike B3. The
same session id from the original `/api/init` call is reused, so the
model still has the entire original standard + data + every prior turn as
context.

### C4. Same ask/validate path as B4–B6
**File:** `backend/app/runner_service.py:80-82` (`update_ontology`) just
calls `_ask(session_id, user_message)` — the exact same function used in
B4. Because `prompt.py`'s rule 4 (`prompt.py:37-39`) says every later
message is an instruction to return the **full** updated ontology (not a
diff), the response shape is identical to the initial one.

### C5. Response replaces state and the graph re-renders
Same as B7-B9, except `App.jsx:47` (`setOntology(data)`) replaces the
*existing* ontology rather than setting it for the first time — this is
what makes the graph visibly update after each chat message.

---

## One-paragraph summary

Files on disk (`data/standards/*.yaml`, `data/synthetic/*.csv`) are read as
raw text once at import time (`data.py`). A browser page load triggers a
fresh ADK session and one big first message combining that text
(`runner_service.py: generate_initial_ontology`). The LLM — guided by
`prompt.py`'s instruction — reasons over that text and returns ontology
JSON, which gets validated against `schema.py`'s Pydantic models before
being trusted. The browser renders that JSON as a graph
(`GraphView.jsx`). Every subsequent chat message repeats the same
ask-and-validate step on the *same* session, so the model always has the
full history, and returns the full updated ontology again each time.

## Where phase 2 work plugs in

- **Real document upload** (PDF/DOCX instead of fixed synthetic files) —
  replaces Phase A4's file-reading with a tool the agent calls per
  request; would need a new function in a `tools.py`, wired into
  `agent.py`'s `tools=[...]` (currently empty, `agent.py:14`).
- **Instance population ("Graph Generation" in the meeting notes)** — a
  new, deterministic (non-LLM) step that reads the same CSVs row-by-row
  and creates actual instance nodes conforming to the *approved* ontology
  — this is new code, not an extension of B5/B6.
- **Approval/versioning** — would sit between B8 and B9: a gate + stored
  history before a graph is treated as "current."
- **Persistence (BigQuery Property Graph)** — a new write path taking the
  same `{entities, relationships}` shape `GraphView.jsx` already consumes.
