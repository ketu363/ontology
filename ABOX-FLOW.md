## ABox data flow

```text
Synthetic CSV files
        ↓
      data.py
        ↓
knowledge_graph.py
        ↓
   /api/abox API
        ↓
      api.js
        ↓
     App.jsx
        ↓
ABoxGraphView.jsx
        ↓
Record-level knowledge graph
```

Important: the ABox is created directly by Python from CSV records. Gemini is not involved in creating ABox nodes.

# 1. Synthetic CSV data

The ABox starts with files inside:

[data/synthetic](C:/Users/niketu/Downloads/ontology-agent/data/synthetic)

For example:

```text
vehicles.csv
drivers.csv
depots.csv
driver_assignments.csv
maintenance_events.csv
fault_records.csv
```

Example vehicle:

```csv
vehicle_id,make,model,depot_id
V-1001,Ford,F-750,DEPOT-N
```

Example assignment:

```csv
vehicle_id,driver_id,assigned_from,assigned_to
V-1001,D-201,2026-09-01,2026-09-07
```

# 2. `data.py` selects the dataset

[data.py](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/data.py:29) checks:

```python
dataset_name = os.getenv(
    "ONTOLOGY_DATASET",
    "synthetic"
)
```

Because no other dataset is selected, it uses:

```text
data/synthetic
```

The selected folder is stored in:

```python
FLEET_DATA_DIR
```

# 3. `load_fleet_tables()` parses the CSV files

[`load_fleet_tables()`](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/data.py:58) opens every CSV using `csv.DictReader`.

A vehicle row becomes a Python dictionary:

```python
{
    "vehicle_id": "V-1001",
    "vin": "1FTRW14W71KB12345",
    "make": "Ford",
    "model": "F-750",
    "depot_id": "DEPOT-N"
}
```

All parsed tables are returned approximately as:

```python
{
    "vehicles.csv": [
        {"vehicle_id": "V-1001", ...},
        {"vehicle_id": "V-1002", ...}
    ],
    "drivers.csv": [
        {"driver_id": "D-201", ...}
    ],
    "depots.csv": [...],
    "driver_assignments.csv": [...],
    "maintenance_events.csv": [...],
    "fault_records.csv": [...]
}
```

# 4. `knowledge_graph.py` creates the complete ABox

[`_complete_graph()`](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:22) calls:

```python
tables = load_fleet_tables()
```

It prepares:

```python
nodes = {}
links = []
```

## Vehicles become Asset nodes

At [knowledge_graph.py line 27](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:27):

```python
for row in tables["vehicles.csv"]:
```

Each vehicle becomes:

```python
ABoxNode(
    id="V-1001",
    type="Asset",
    label="V-1001 · Ford F-750",
    properties={
        "vin": "...",
        "make": "Ford",
        "model": "F-750",
        "year": "2021"
    }
)
```

The `depot_id` creates:

```text
V-1001 ──LOCATED_AT──> DEPOT-N
```

## Drivers become Driver nodes

At [line 39](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:39):

```python
for row in tables["drivers.csv"]:
```

This creates:

```python
ABoxNode(
    id="D-201",
    type="Driver",
    label="D-201 · Maria Chen"
)
```

## Depots become Depot nodes

At [line 48](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:48), a depot row becomes:

```text
DEPOT-N : Depot
```

The `org_unit` value creates an organizational-unit node and relationship:

```text
DEPOT-N ──BELONGS_TO──> ORG-MIDWEST-OPERATIONS
```

## Assignments become relationships

At [line 71](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:71):

```python
for row in tables["driver_assignments.csv"]:
```

This creates:

```text
D-201 ──ASSIGNED_TO──> V-1001
```

Assignment dates are stored as relationship properties:

```json
{
  "assigned_from": "2026-09-01",
  "assigned_to": "2026-09-07"
}
```

## Maintenance records become nodes

At [line 82](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:82):

```text
WO-5001 : MaintenanceEvent
```

It creates:

```text
WO-5001 ──REFERENCES──> V-1001
```

## Fault records become nodes

At [line 94](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:94):

```text
F-9003 : FaultRecord
```

It creates:

```text
F-9003 ──REFERENCES──> V-1001
```

# 5. Complete internal ABox

The resulting graph looks like:

```text
D-201 ──ASSIGNED_TO────┐
                       ▼
                    V-1001 ──LOCATED_AT──> DEPOT-N
                       ▲                       │
                       │                       │ BELONGS_TO
WO-5001 ──REFERENCES───┤                       ▼
WO-5002 ──REFERENCES───┤            ORG-MIDWEST-OPERATIONS
F-9003  ──REFERENCES───┘
```

# 6. `get_abox_graph()` selects a small neighborhood

The complete ABox has many records, so the application does not return everything to the browser.

[`get_abox_graph()`](C:/Users/niketu/Downloads/ontology-agent/agent/ontology_agent/knowledge_graph.py:116) accepts:

```python
query
limit
```

For example:

```text
query = V-1001
limit = 100
```

It finds `V-1001` and returns its connected records:

- Drivers assigned to it
- Its depot
- The depot’s organizational unit
- Its maintenance events
- Its faults

This keeps the graph small and readable.

# 7. Backend exposes `/api/abox`

[routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:45) provides:

```text
GET /api/abox
```

Example request:

```text
GET /api/abox?query=V-1001&limit=100
```

Example response:

```json
{
  "dataset": "synthetic",
  "query": "V-1001",
  "nodes": [
    {
      "id": "V-1001",
      "type": "Asset",
      "label": "V-1001 · Ford F-750",
      "properties": {
        "make": "Ford",
        "model": "F-750"
      }
    }
  ],
  "links": [
    {
      "source": "V-1001",
      "type": "LOCATED_AT",
      "target": "DEPOT-N"
    }
  ],
  "total_nodes": 33,
  "total_links": 35,
  "truncated": true
}
```

# 8. `api.js` calls the backend

[`getABoxGraph()`](C:/Users/niketu/Downloads/ontology-agent/frontend/src/api.js:20) sends:

```javascript
fetch(
  "http://localhost:8000/api/abox?query=V-1001&limit=100"
)
```

It converts the HTTP response into JavaScript JSON.

# 9. `App.jsx` stores the ABox

On initial loading, [`getInitialABox()`](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:26) requests the first ABox neighborhood.

When the user searches, `handleABoxSearch()` calls:

```javascript
setABox(await getABoxGraph(aboxQuery));
```

React stores the returned nodes and links in:

```javascript
const [abox, setABox] = useState(null);
```

# 10. `ABoxGraphView` renders the graph

[ABoxGraphView.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/components/ABoxGraphView.jsx:17) converts the API response into force-graph data:

```javascript
nodes: graph.nodes.map(...)
links: graph.links.map(...)
```

It assigns different colors based on record type:

```text
Asset              → green
Driver             → blue
Depot              → light green
MaintenanceEvent   → light blue
FaultRecord        → red
OrganizationalUnit → dark green
```

When you click `V-1001`, the right sidebar displays its CSV properties.

## Complete ABox call sequence

```text
data/synthetic/*.csv
        ↓
data.py: load_fleet_tables()
        ↓
knowledge_graph.py: _complete_graph()
        ↓
knowledge_graph.py: get_abox_graph("V-1001")
        ↓
routes.py: GET /api/abox
        ↓
api.js: getABoxGraph()
        ↓
App.jsx: setABox()
        ↓
ABoxGraphView.jsx
        ↓
Interactive record graph
```

In one sentence: **Python reads every synthetic CSV row, converts master records into ABox nodes, converts foreign-key rows into facts/relationships, returns only the searched node’s neighborhood through `/api/abox`, and React renders those records as the interactive ABox graph.**
