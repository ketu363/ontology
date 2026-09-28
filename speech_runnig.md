## Short data flow

```text
Data files → data.py → runner_service.py → Gemini Agent
→ schema.py validation → FastAPI → React → GraphView
```

### 1. Data folder

The `data` folder contains two types of information:

- **Standards data:** [data/standards](C:/Users/niketu/Downloads/ontology-agent/data/standards)  
  Synthetic ISO-style rules defining required entities, attributes, relationships, and cardinality rules. For example, an `Asset` must have `VIN`, `Make`, `Model`, and status.

- **Synthetic data:** [data/synthetic](C:/Users/niketu/Downloads/ontology-agent/data/synthetic)  
  Fake fleet records used for testing, including vehicles, drivers, depots, maintenance events, faults, and driver assignments.

### 2. `data.py` loads and concatenates data

[data.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/data.py:19) reads the files as text.

- `_read()` reads one file.
- `_load_standards_text()` combines both YAML standards files.
- `_load_fleet_data_text()` combines all CSV fleet files.
- The results are stored in:

```python
SYNTHETIC_ISO_STANDARD
SYNTHETIC_FLEET_DATA
```

### 3. `prompt.py` defines the agent rules

[prompt.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/prompt.py:7) tells Gemini to:

- Understand fleet-management data.
- Follow the ISO-style standard.
- Identify entities, attributes, and relationships.
- Return only ontology JSON.

### 4. `agent.py` creates the Gemini agent

[agent.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/agent.py:9) creates the Google ADK agent and connects it to Gemini using the instructions from `prompt.py`.

### 5. Frontend requests the initial ontology

When the application opens:

```text
App.jsx → api.js → POST /api/init
```

The request reaches [`init_ontology()`](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:17).

### 6. `runner_service.py` prepares the AI message

[`generate_initial_ontology()`](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:66):

1. Creates a new conversation session.
2. Combines the standards text and synthetic fleet text:

```text
STANDARD:
[combined YAML]

CLIENT FLEET DATA:
[combined CSV files]
```

3. Sends this message to Gemini through `_ask()`.

### 7. Gemini creates the ontology

Gemini examines the rules and CSV columns and produces something like:

```json
{
  "entities": [
    {"name": "Asset", "attributes": ["AssetID", "VIN", "Make"]},
    {"name": "Driver", "attributes": ["DriverID", "Name"]},
    {"name": "Depot", "attributes": ["DepotID", "City"]}
  ],
  "relationships": [
    {"source": "Driver", "type": "OPERATES", "target": "Asset"},
    {"source": "Asset", "type": "LOCATED_AT", "target": "Depot"}
  ],
  "notes": []
}
```

### 8. `schema.py` validates the result

[schema.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/schema.py:15) checks that:

- Every entity has a name and attributes.
- Every relationship has a source, type, and target.
- The result is valid JSON.

### 9. Backend returns it to React

`routes.py` sends the validated ontology JSON back to `App.jsx`.

`App.jsx` stores it and passes it to:

```jsx
<GraphView ontology={ontology} />
```

### 10. `GraphView.jsx` renders the knowledge graph

[GraphView.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/components/GraphView.jsx:18) converts:

- Ontology entities into graph nodes.
- Ontology relationships into graph arrows.

```text
Driver ──OPERATES──> Asset ──LOCATED_AT──> Depot
                           │
                           └──HAS_FAULT──> FaultRecord
```

`react-force-graph-2d` then draws the interactive graph.

In one sentence: **the application loads ISO-style rules and fake fleet CSV data, sends both to Gemini to generate ontology JSON, validates it, and converts the entities and relationships into a visual graph.**
