Below is the current architecture traced from the actual files and current line numbers.

## 1. Complete agent and ontology-generation flow

```text
Browser opens
   ↓
Create session
   ↓
User uploads CSV/JSON + optional standards
   ↓
Backend profiles files locally
   ↓
LLM identifies domain
   ↓
Standard uploaded? ── Yes ──→ Generate ontology
   │
   No
   ↓
Search official standards
   ↓
Show references and request confirmation
   ↓
User confirms
   ↓
Generate validated ontology JSON
   ↓
React renders entities and relationships
```

### Step 1: Application starts

The backend starts from [main.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/main.py:21).

It:

- Loads the project `.env`.
- Configures Gemini/Vertex environment variables.
- Creates the FastAPI application at line 35.
- Registers all API routes at line 45.

The model name comes from [config.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/config.py:14):

```python
model_name=os.getenv("ONTOLOGY_MODEL", "gemini-2.5-flash")
```

### Step 2: Frontend creates an ontology session

When the UI opens, `App.jsx` calls `getInitOntology()` in [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:45).

That calls:

```text
POST /api/init
```

through [api.js](C:/Users/niketu/Downloads/ontology-agent/frontend/src/api.js:4).

The backend receives it in [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:26) and calls:

```python
runner_service.start_session()
```

`start_session()` is in [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:225).

It:

1. Creates an ADK session.
2. Creates a `SessionWorkflow`.
3. Sets the phase to `awaiting_upload`.
4. Returns the welcome message.
5. Does **not call the LLM yet**.

The session state structure is defined at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:88):

```python
SessionWorkflow:
    session_id
    phase
    files
    domain
    domain_summary
    references
    standards_confirmed
    ontology
    saved_graphs
```

Currently this session is stored in backend memory at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:100). Restarting the backend clears these active sessions.

---

### Step 3: User uploads files

The upload UI is implemented in [ChatPanel.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/components/ChatPanel.jsx:75).

It calls `onUpload()`, which is connected to `handleUpload()` in [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:77).

`handleUpload()` calls `uploadOntologyInputs()` in [api.js](C:/Users/niketu/Downloads/ontology-agent/frontend/src/api.js:20).

It builds multipart form data containing:

```text
session_id
domain_hint
files
```

and sends:

```text
POST /api/upload
```

The request reaches `upload_ontology_inputs()` in [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:36).

The route first calls:

```python
validate_upload_batch(...)
```

at [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:44).

---

### Step 4: Files are validated and profiled locally

File processing happens in [ingestion_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:230).

Supported file types are defined at [ingestion_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:19):

```text
CSV
JSON
PDF
TXT
Markdown
YAML
XML
```

Upload limits are defined by `upload_limits()` at [line 47](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:47).

Default limits:

- Maximum 20 files.
- Maximum 25 MB per file.
- Maximum 100 MB total.
- Maximum 50 CSV/JSON sample rows sent to the LLM.
- Maximum 120,000 extracted text characters per document.

#### CSV processing

`_csv_profile()` is at [ingestion_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:95).

It reads all CSV rows locally to calculate:

- Complete row count.
- Column names.
- Inferred data types.
- Null counts.
- Candidate ID/key columns.
- A bounded sample of up to 50 rows.

For example:

```text
FULL ROW COUNT: 500,000
COLUMNS: asset_id, depot_id, status
INFERRED TYPES: ...
NULL COUNTS: ...
CANDIDATE KEY COLUMNS: asset_id, depot_id
REPRESENTATIVE SAMPLE: only 50 rows
```

Therefore, a large CSV is fully analyzed locally, but the complete 500,000 records are not sent to the model.

#### JSON processing

`_json_profile()` is at [ingestion_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:159).

It determines whether the JSON is:

- A JSON Schema.
- A JSON object.
- A list of records.

For arrays, only a bounded sample is added to the prompt. Large JSON content is truncated to the configured character limit.

#### Standards documents

PDF processing is handled by `_pdf_profile()` at [line 186](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:186).

Text, Markdown, YAML and XML are processed by `_text_profile()` at [line 216](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:216).

`ingest_file()` identifies whether a file appears to be a standard at [line 256](C:/Users/niketu/Downloads/ontology-agent/backend/app/ingestion_service.py:256).

It checks:

- Filename words such as `iso`, `standard`, `compliance`, `regulation`.
- Content such as `ISO 55001`, `normative references`, `shall provide`, or `compliance requirements`.

Every file becomes an `IngestedFile` containing:

```text
summary          → safe UI metadata
prompt_context   → bounded content for the LLM
is_standard      → true or false
```

---

### Step 5: The workflow analyzes the business domain

After profiling, the upload route calls `process_uploads()` at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:321).

It:

1. Stores the profiled files in session memory.
2. Clears any previous ontology.
3. Changes the phase to `analyzing_uploads`.
4. Calls `_assess_domain()`.

`_assess_domain()` is at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:232).

Before contacting the LLM, `_bounded_file_context()` at [line 200](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:200) limits the total prompt size. Its current default is 220,000 characters.

The agent receives something similar to:

```text
TASK: ANALYZE_DOMAIN

USER DOMAIN HINT: vehicle rental

UPLOADED FILE PROFILES:
FILE: vehicles.csv
COLUMNS: [...]
CANDIDATE KEY COLUMNS: [...]
REPRESENTATIVE SAMPLE: [...]
```

The expected model response is defined in [prompt.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/prompt.py:22):

```json
{
  "domain": "vehicle rental",
  "confidence": 0.95,
  "needs_clarification": false,
  "question": "",
  "summary": "The vehicle and booking schemas indicate a rental domain."
}
```

If the confidence is below `0.65`, or `needs_clarification=true`, `process_uploads()` changes the phase to `awaiting_domain` and asks the user a domain question at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:333).

---

### Step 6: Uploaded-standard and no-standard branches

#### Branch A: A standard was uploaded

This branch is at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:339).

If at least one uploaded file is classified as a standard:

```python
return await _generate_ontology(state)
```

It does not perform web research.

#### Branch B: No standard was uploaded

The workflow calls `_research_standards()` at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:251).

The agent is instructed to run:

```text
TASK: RESEARCH_STANDARDS
CONFIRMED DOMAIN: vehicle rental
```

The standards-research rules are defined in [prompt.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/prompt.py:34).

The agent must:

- Use Google Search.
- Search only using the confirmed domain.
- Prefer official standards organizations.
- Never put uploaded client records into a search query.
- Return supported references.
- Ask for confirmation.

Grounded search URLs are extracted from ADK events by `_grounding_sources()` at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:138).

The backend then filters results to permitted official domains in `_is_primary_source_domain()` at [line 217](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:217).

The workflow moves to:

```text
awaiting_standard_confirmation
```

The user must reply `yes`, `approve`, `proceed`, or another supported confirmation before ontology generation.

That confirmation is handled in `handle_message()` at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:372).

---

### Step 7: ADK agent configuration

The actual ADK agent is created in [agent.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/agent.py:10).

It is currently a **single-agent architecture**:

```python
root_agent = Agent(
    name="ontology_agent",
    model=settings.model_name,
    instruction=ONTOLOGY_AGENT_INSTRUCTION,
    planner=BuiltInPlanner(...),
    tools=[google_search],
)
```

Important points:

- The same specialist agent performs domain identification, research and ontology generation.
- The backend controls which task the agent performs using `TASK:` markers.
- Google Search is available but is permitted only for standards research.
- Internal reasoning/thought events are not sent to the UI.

The ADK runner is initialized in [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:39).

All model calls go through `_run_agent()` at [line 155](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:155).

---

### Step 8: Ontology is generated

Ontology generation happens in `_generate_ontology()` at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:294).

It separates:

```python
standard_files
client_files
```

Then it prepares:

```text
TASK: GENERATE_ONTOLOGY
CONFIRMED DOMAIN: ...

STANDARDS / BUSINESS RULES:
...

CLIENT DATA/SCHEMA PROFILES:
...
```

The LLM is required to return the structure defined in [prompt.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/prompt.py:45):

```json
{
  "entities": [
    {
      "name": "Asset",
      "attributes": ["AssetID", "VIN"],
      "rationale": "Why Asset is required",
      "evidence": ["assets.csv"]
    }
  ],
  "relationships": [
    {
      "source": "Asset",
      "type": "LOCATED_AT",
      "target": "Depot",
      "rationale": "DepotID connects assets to depots",
      "evidence": ["assets.csv.DepotID"]
    }
  ],
  "reasoning_summary": "...",
  "notes": ["..."],
  "message": "..."
}
```

Relationship identification is based on:

- Matching `_id` columns.
- Candidate primary and foreign keys.
- Similar column names between files.
- Uploaded standard rules.
- Confirmed public standards references.
- Schema structure and record samples.

The modeling rules are at [prompt.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/prompt.py:70).

The result is parsed by `_parse_ontology_response()` at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:134).

It is validated against the `Ontology` Pydantic model in [schema.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/schema.py:33).

The validated ontology is stored in:

```python
state.ontology
```

at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:312).

---

### Step 9: Ontology is returned and rendered

The API response structure is defined in [workflow_models.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/workflow_models.py:26):

```text
session_id
phase
message
ontology
files
references
```

`App.jsx` receives it through `applyWorkflowResponse()` at [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:36).

It stores the ontology in React state:

```javascript
setOntology(data.ontology || null);
```

When `ontology` exists, the UI renders `GraphViewNetwork` at [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:172).

`GraphViewNetwork` starts at [GraphViewNetwork.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/components/GraphViewNetwork.jsx:75).

It converts:

- `ontology.entities` into visual nodes at [line 148](C:/Users/niketu/Downloads/ontology-agent/frontend/src/components/GraphViewNetwork.jsx:148).
- `ontology.relationships` into visual links at [line 95](C:/Users/niketu/Downloads/ontology-agent/frontend/src/components/GraphViewNetwork.jsx:95).

`ForceGraph2D` renders the graph at [line 431](C:/Users/niketu/Downloads/ontology-agent/frontend/src/components/GraphViewNetwork.jsx:431).

---

### Step 10: Later chat messages

Messages are sent through `handleSend()` at [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:64), then:

```text
POST /api/chat
```

through [api.js](C:/Users/niketu/Downloads/ontology-agent/frontend/src/api.js:10).

The backend endpoint is [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:64), which calls `handle_message()` at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:345).

Its behavior depends on the current phase:

- Greeting such as `hi` → normal greeting, no LLM ontology update.
- No files → asks the user to upload files.
- `awaiting_domain` → treats the message as domain clarification.
- `awaiting_standard_confirmation` → accepts or rejects standards.
- `ontology_ready` → sends `TASK: UPDATE_ONTOLOGY` to the LLM.

For an update, the current complete ontology and user request are sent at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:391).

The model must return the complete ontology again—not only the changed node.

---

# 2. Complete BigQuery save flow

```text
Click “Save Ontology to BigQuery”
   ↓
POST /api/bigquery/sync
   ↓
Read ontology from session memory
   ↓
Convert entities into node rows
   ↓
Convert relationships into edge rows
   ↓
Append rows to three physical tables
   ↓
Create entity and relationship views
   ↓
Create uniquely named BigQuery Property Graph
   ↓
Return graph name and preview query to chat
```

## Step 1: User clicks the save button

The button is connected to `handleBigQuerySync()` in [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:94).

The button itself is at [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:156).

It is disabled when:

- BigQuery is not configured.
- No session exists.
- No ontology exists.
- A save is already running.

`handleBigQuerySync()` calls `syncBigQuery()` in [api.js](C:/Users/niketu/Downloads/ontology-agent/frontend/src/api.js:46).

That sends:

```http
POST /api/bigquery/sync

{
  "session_id": "..."
}
```

It does **not send the ontology again from the browser**. The backend retrieves the validated ontology from session memory.

---

## Step 2: Backend obtains the cached ontology

The endpoint is `sync_bigquery()` in [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:109).

It obtains the current ontology using:

```python
runner_service.get_cached_ontology(session_id)
```

at [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:113).

`get_cached_ontology()` is defined at [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:403).

The endpoint then calls:

```python
graph_store.save_tbox(...)
```

at [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:121).

Currently, the UI saves only the **TBox ontology**:

- Classes/entities.
- Class attributes.
- Relationships.
- Rationale.
- Evidence.
- Modeling notes.

It does not save individual ABox records through this button.

---

## Step 3: `save_tbox()` converts the ontology

The BigQuery implementation is in [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:128).

`save_tbox()` begins at [line 190](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:190).

Each ontology entity becomes a node:

```python
{
    "id": entity.name,
    "type": "Class",
    "label": entity.name,
    "properties": {
        "attributes": entity.attributes,
        "rationale": entity.rationale,
        "evidence": entity.evidence
    }
}
```

This conversion is at [lines 198–210](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:198).

Each relationship becomes an edge:

```python
{
    "source": relationship.source,
    "type": relationship.type,
    "target": relationship.target,
    "properties": {
        "rationale": relationship.rationale,
        "evidence": relationship.evidence
    }
}
```

This conversion begins at [line 211](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:211).

Ontology-level metadata is also passed:

```text
reasoning_summary
notes
message
```

at [lines 231–235](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:231).

---

## Step 4: A unique graph version and graph name are created

`_save_graph()` begins at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:270).

It creates a unique version ID:

```python
tbox_<UUID>
```

at line 285.

It creates a unique Property Graph name using `_new_property_graph_name()` at [line 148](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:148).

Example:

```text
ontology_global_mobility_fleet_operations_20261006_061244_0ee8ca4c
```

The name contains:

```text
ontology
+ domain
+ UTC timestamp
+ random identifier
```

---

## Step 5: BigQuery dataset and tables are created

`_ensure_storage()` is at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:366).

It creates the configured dataset if it does not already exist.

It then creates three physical tables:

```text
graph_versions
graph_nodes
graph_edges
```

The schemas are defined by `_schemas()` at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:546).

All three tables are:

- Partitioned by `created_at`.
- Clustered by `graph_type` and `graph_version_id`.

That configuration is at [lines 377–385](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:377).

---

## What each BigQuery table stores

### `graph_nodes`

One row is saved for every ontology entity.

Schema starts at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:563).

| Column | Stored value |
|---|---|
| `graph_version_id` | Unique ontology save ID |
| `graph_type` | `TBOX` |
| `node_id` | Entity name, such as `Asset` |
| `node_type` | `Class` |
| `label` | Display name |
| `properties` | JSON containing attributes, rationale and evidence |
| `created_at` | Save timestamp |

Example:

```json
{
  "graph_version_id": "tbox_abc123",
  "graph_type": "TBOX",
  "node_id": "Asset",
  "node_type": "Class",
  "label": "Asset",
  "properties": {
    "attributes": ["AssetID", "VIN", "Status"],
    "rationale": "Represents managed fleet assets",
    "evidence": ["assets.csv"]
  }
}
```

### `graph_edges`

One row is saved for every ontology relationship.

Schema starts at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:571).

| Column | Stored value |
|---|---|
| `graph_version_id` | Ontology save ID |
| `graph_type` | `TBOX` |
| `edge_id` | Generated unique relationship ID |
| `source_id` | Source entity |
| `relationship_type` | Relationship label |
| `target_id` | Target entity |
| `properties` | JSON containing rationale and evidence |
| `created_at` | Save timestamp |

Example:

```json
{
  "graph_version_id": "tbox_abc123",
  "graph_type": "TBOX",
  "edge_id": "tbox_abc123:e:0",
  "source_id": "Asset",
  "relationship_type": "LOCATED_AT",
  "target_id": "Depot",
  "properties": {
    "rationale": "DepotID connects assets to depots",
    "evidence": ["assets.csv.DepotID"]
  }
}
```

### `graph_versions`

One row is saved for every complete ontology save.

Schema starts at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:553).

| Column | Stored value |
|---|---|
| `graph_version_id` | Unique ontology version |
| `graph_type` | `TBOX` |
| `dataset_name` | Currently `NULL` for TBox |
| `source` | `ontology-ui` |
| `session_id` | UI/backend session |
| `node_count` | Number of entities |
| `edge_count` | Number of relationships |
| `metadata` | Notes, reasoning, message and Property Graph name |
| `created_at` | Save timestamp |

---

## Step 6: Rows are appended

Inside `_save_graph()`:

```python
self._append_rows("graph_nodes", node_rows)
self._append_rows("graph_edges", edge_rows)
```

are executed at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:318).

`_append_rows()` is defined at [line 527](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:527).

It uses:

```python
WRITE_APPEND
```

at line 535.

Therefore:

- Existing table rows are not replaced.
- Every save adds another version.
- `graph_version_id` separates different ontology saves.

The `graph_versions` row is deliberately written last at [line 327](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:327), after nodes, edges and the Property Graph are successfully created.

---

## Step 7: Supporting views are created

After writing nodes and edges, `_save_graph()` calls `_create_tbox_property_graph()` at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:320).

The function itself begins at [line 389](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:389).

It calls `_tbox_property_graph_statements()` at [line 407](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:407).

For every entity, it creates a filtered node view at [line 453](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:453):

```sql
CREATE VIEW `_ontology_pg_node_asset_...` AS
SELECT ...
FROM `graph_nodes`
WHERE graph_version_id = 'tbox_abc123'
  AND node_id = 'Asset'
```

For every relationship, it creates a filtered edge view at [line 490](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:490):

```sql
CREATE VIEW `_ontology_pg_edge_asset_located_at_depot_...` AS
SELECT ...
FROM `graph_edges`
WHERE graph_version_id = 'tbox_abc123'
  AND source_id = 'Asset'
  AND relationship_type = 'LOCATED_AT'
  AND target_id = 'Depot'
```

These views do not copy the physical rows. They expose filtered subsets of `graph_nodes` and `graph_edges`.

They are needed because BigQuery’s schema graph requires separate node and edge definitions to display:

```text
Asset --LOCATED_AT--> Depot
```

instead of displaying one generic `Entity` node with one generic self-loop.

---

## Step 8: BigQuery Property Graph is created

The `CREATE PROPERTY GRAPH` statement is built at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:511).

It registers:

- Every entity view as a node table.
- Every relationship view as an edge table.
- Entity names as node labels.
- Relationship types as edge labels.
- Source and destination key mappings.

The complete SQL script is executed at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:404).

Current behavior:

- Every save creates a new graph.
- Every save creates new supporting views.
- Old graphs and views remain.
- Historical rows remain in the three physical tables.

---

## Step 9: Saved graph name is returned to chat

The backend returns:

```json
{
  "stored": true,
  "dataset": "project.ontology_graph",
  "version": {
    "graph_version_id": "...",
    "property_graph_name": "...",
    "node_count": 50,
    "edge_count": 72,
    "gql_preview_query": "..."
  }
}
```

This response is constructed in [routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:139).

`App.jsx` receives it and prints the exact graph name into chat at [App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:98).

The preview query is generated by `_gql_preview_query()` at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:153):

```sql
GRAPH `project.dataset.unique_graph_name`
MATCH p = (a)-[r]->(b)
WHERE a.graph_version_id = 'tbox_...'
RETURN TO_JSON(p) AS graph_path
```

## Short summary

```text
Uploaded files
    ↓
ingestion_service.py profiles them
    ↓
runner_service.py controls domain/standard/confirmation phases
    ↓
agent.py executes the Gemini ADK agent
    ↓
prompt.py defines the required ontology output
    ↓
schema.py validates the ontology
    ↓
App.jsx receives it
    ↓
GraphViewNetwork.jsx renders it
    ↓
BigQuery button calls routes.py
    ↓
bigquery_service.py saves:
    - graph_nodes
    - graph_edges
    - graph_versions
    - node views
    - edge views
    - Property Graph
```
