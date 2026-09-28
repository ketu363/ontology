# Fleet Ontology Agent — Architecture & Flow

This document explains what every file does and how a request flows through
the whole system, end to end. It reflects the state of the project as a
proof of concept: it generates and lets you interactively refine a fleet
**ontology** (schema) and renders it as a **knowledge graph** in the browser.
It does **not** yet populate the graph with real instance records (see
[Current scope vs. roadmap](#8-current-scope-vs-roadmap)).

## 1. What this project does

Given (a) a fleet-management industry-standard profile and (b) a client's
raw fleet data, an ADK agent proposes a fleet ontology (entities,
attributes, relationships), validates it against the standard, and lets a
user refine it conversationally in a chat UI while watching the resulting
knowledge graph update live.

## 2. Project structure

```
ontology-agent/
├── data/
│   ├── standards/                     # the RULES (industry standard, synthetic)
│   │   ├── fleet_standards_profile.yaml
│   │   └── source_registry.yaml
│   └── synthetic/                     # the CLIENT DATA (synthetic, for now)
│       ├── requirements.md
│       ├── vehicles.csv
│       ├── drivers.csv
│       ├── depots.csv
│       ├── driver_assignments.csv
│       ├── maintenance_events.csv
│       └── fault_records.csv
├── agent/                             # the ADK agent (pure agent logic)
│   └── ontology_agent/
│       ├── schema.py                  # Ontology/Entity/Relationship Pydantic models
│       ├── data.py                    # loads data/ files from disk into text
│       ├── prompt.py                  # the agent's system instruction
│       ├── config.py                  # env-driven settings (model name)
│       └── agent.py                   # the ADK Agent() definition -- wiring only
├── backend/                           # FastAPI service between agent and browser
│   └── app/
│       ├── runner_service.py          # wraps ADK Runner + sessions
│       ├── routes.py                  # POST /api/init, POST /api/chat
│       └── main.py                    # FastAPI app, CORS, env setup
└── frontend/                          # React + Vite
    └── src/
        ├── api.js                     # fetch calls to the backend
        ├── App.jsx                    # top-level state + layout
        ├── components/
        │   ├── ChatPanel.jsx          # chat bubbles + input
        │   └── GraphView.jsx          # renders the ontology as an interactive graph
        └── styles.css
```

## 3. The data layer: rules vs. client data

Two folders, kept deliberately separate:

- **`data/standards/`** -- what the ontology is *required* to contain.
  `fleet_standards_profile.yaml` lists required entities, their required
  attributes, and cardinality rules, each tagged with a `source` id.
  `source_registry.yaml` maps those ids to the real standards they're
  modeled after (ISO 55001, ISO 15143-3, ISO 14224) -- for traceability,
  not because the clause text itself is copied (it isn't; it's an
  original summary in the same style, since the real standards are
  copyrighted).
- **`data/synthetic/`** -- what a *specific client* actually has: CSVs of
  vehicles, drivers, depots, assignments, maintenance events, fault
  records. Fabricated for this PoC, but structured so a real client
  export with the same column names drops in with zero code changes.

## 4. Feeding data to the agent (`agent/ontology_agent/data.py`)

`data.py` reads every file above off disk as **plain text** at import
time -- no CSV/YAML parsing happens in Python. It concatenates the two
standards files into `SYNTHETIC_ISO_STANDARD` and all the CSVs (plus
`requirements.md`) into `SYNTHETIC_FLEET_DATA`. These are the same two
variable names the code used before real files existed (originally
hardcoded strings), so nothing downstream had to change when this became
file-based.

## 5. The agent itself (`agent.py` + `prompt.py` + `schema.py` + `config.py`)

- **`config.py`** -- a `Settings` dataclass reading `ONTOLOGY_MODEL` from
  the environment (default `gemini-2.5-flash`). No hardcoded constants
  elsewhere in agent code.
- **`prompt.py`** -- the actual "brain": `ONTOLOGY_AGENT_INSTRUCTION`
  tells the model to (1) ground everything in the standard/data, never
  invent unrelated concepts, (2) **never drop a standard-required
  entity/attribute**, even if the client data doesn't mention it -- explain
  the conflict in `notes` instead, (3) treat every later chat message as an
  instruction to return the FULL updated ontology again, and (4) respond
  with ONLY a JSON object matching a specific shape.
- **`agent.py`** -- wires an ADK `Agent` with that model + instruction.
  No tools yet, and deliberately no `output_schema` (ADK's structured-output
  parameter) -- an agent with `output_schema` can't also use tools, and a
  document-parsing tool is a natural next step, so validation is done
  manually instead (see below).
- **`schema.py`** -- `Entity` / `Relationship` / `Ontology` Pydantic
  models describing the exact JSON shape the prompt asks for. The model's
  raw text response gets validated against these with
  `Ontology.model_validate_json(...)` -- this is the actual correctness
  check: if the LLM's output doesn't match, this raises instead of quietly
  passing bad data downstream.

## 6. The backend (`backend/app/`)

- **`runner_service.py`** -- wraps ADK's `InMemoryRunner`. Two entry
  points:
  - `generate_initial_ontology()` -- creates a **brand-new ADK session**
    per call, sends the standard + client data as the first message,
    returns `(session_id, ontology)`.
  - `update_ontology(session_id, message)` -- sends a follow-up chat
    message on an *existing* session.

  Important fix made during testing: this used to keep **one single
  shared session for the whole process**, so every page refresh silently
  kept appending to the same ever-growing conversation. Now every page
  load gets its own fresh session, and the frontend must pass the
  `session_id` back on every chat message.
- **`routes.py`** -- `POST /api/init` (no body; returns the ontology +
  `session_id`) and `POST /api/chat` (`{message, session_id}`; returns the
  updated ontology). Both wrap agent/parsing failures as HTTP 502 with the
  real error message, so the frontend can surface it.
- **`main.py`** -- loads `.env` from the project root, bridges
  `GEMINI_API_KEY`/`GOOGLE_API_KEY` naming, sets CORS to allow the Vite
  dev server origin, mounts the routes.

## 7. The frontend (`frontend/src/`)

- **`api.js`** -- two functions, `initOntology()` and
  `sendChatMessage(message, sessionId)`, thin wrappers over `fetch`.
- **`App.jsx`** -- on mount, calls `initOntology()` once and stores the
  returned ontology + `session_id` in state; every chat send calls
  `sendChatMessage` with that same `session_id` and replaces the ontology
  in state with the response. Uses a **module-scoped cached promise**
  (`getInitOntology()`), not component state, to guarantee the init call
  fires exactly once even under React 18 Strict Mode's dev-only
  mount→unmount→remount cycle (a `useRef` guard does *not* survive that
  cycle, since remounting creates a brand-new ref; a module-level variable
  does, since the module isn't re-evaluated).
- **`ChatPanel.jsx`** -- renders the `messages` list as chat bubbles
  (right-aligned/blue for the user, left-aligned/gray for the agent), an
  animated "typing" indicator while a request is in flight, and the input
  form.
- **`GraphView.jsx`** -- reshapes `{entities, relationships}` into the
  `{nodes, links}` shape `react-force-graph-2d` expects, renders it with
  custom canvas drawing (colored circles + labels, colored consistently
  per entity name via a hash), and shows a details sidebar (entity count,
  relationship count, and the selected node's attributes on click). All
  interactivity (drag, zoom, pan, click) comes from the library --
  nothing custom-built there.

## 8. End-to-end request flow

**On page load:**
1. `App.jsx` calls `POST /api/init`.
2. `routes.py` calls `runner_service.generate_initial_ontology()`.
3. A new ADK session is created; the first message (standard text +
   client data text, concatenated) is sent to the Gemini model via ADK's
   `Runner`.
4. The model's final text response is stripped of any markdown code
   fences and validated with `Ontology.model_validate_json()`.
5. The validated ontology + the new `session_id` are returned as JSON.
6. `App.jsx` stores both in state; `GraphView` renders the graph;
   `ChatPanel` shows a one-line summary + any clarifying `notes`.

**On every chat message:**
1. `App.jsx` calls `POST /api/chat` with `{message, session_id}`.
2. `runner_service.update_ontology()` sends that message on the *same*
   session -- so the model sees the full prior conversation (including the
   original standard + data) as context.
3. Per `prompt.py`'s rule, the model treats the message as an instruction
   to return the FULL updated ontology again (not a diff), re-validates
   every requirement against the standard, and responds with the same
   JSON shape.
4. Same validation + response path as above; the graph re-renders with
   the new nodes/edges.

## 9. Current scope vs. roadmap

The most important thing to understand about the current state: this
produces a **T-box (ontology/schema)** -- *"a `Vehicle` has attributes
VIN, Make, Model..."* -- not an **A-box (instance data)** -- actual nodes
like `V-1001` linked to `Depot North`. The meeting notes describe these as
two separate steps: *Generate Ontology → visual preview → approve → Graph
Generation*. This project currently implements the first. Populating the
graph with real rows from `vehicles.csv`/`drivers.csv`/etc. as actual
nodes is the natural next milestone, and should likely be **deterministic
code** (loop over CSV rows, create a node per row against the *approved*
schema) rather than another LLM call -- the LLM's value is reasoning over
ambiguous prose/standards, not mapping known columns to known fields.

Other not-yet-built pieces: real document upload/parsing (PDF/DOCX --
currently only the fixed synthetic files are used), an approval/versioning
step before anything is persisted, and a persistence target (BigQuery
Property Graph, per the meeting notes).

## 10. Running it locally

```powershell
# Terminal 1
cd ontology-agent\backend
..\.venv\Scripts\Activate.ps1
uvicorn app.main:app --reload --port 8000

# Terminal 2
cd ontology-agent\frontend
npm run dev
```

Open the frontend's printed URL (normally `http://localhost:5173`).
Requires `gcloud auth application-default login` once, and the project
fields in `.env` filled in (Vertex AI via a GCP sandbox project -- no
Gemini API key needed).
