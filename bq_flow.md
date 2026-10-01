Every time you click **Save Ontology to BigQuery**, the application saves the current **TBox ontology** as a new version and updates `ontology_pg` to display that latest version.

### Complete flow

1. **UI button click**

[App.jsx](C:/Users/niketu/Downloads/ontology-agent/frontend/src/App.jsx:103) runs:

```javascript
handleBigQuerySync()
```

It passes the current browser `sessionId`.

2. **Frontend API call**

[api.js](C:/Users/niketu/Downloads/ontology-agent/frontend/src/api.js:33) sends:

```http
POST /api/bigquery/sync

{
  "session_id": "current-session-id"
}
```

3. **Backend receives the request**

[routes.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/routes.py:83) runs:

```python
sync_bigquery()
```

It gets the latest ontology generated for that browser session using:

```python
runner_service.get_cached_ontology(session_id)
```

That function is in [runner_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/runner_service.py:107).

4. **TBox is converted into nodes and edges**

[bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:179) runs:

```python
save_tbox(ontology)
```

Entities become nodes:

```json
{
  "id": "Vehicle",
  "type": "Class",
  "label": "Vehicle",
  "properties": {
    "attributes": ["VehicleID", "Make", "Model"]
  }
}
```

Relationships become edges:

```json
{
  "source": "Vehicle",
  "type": "IS_OF_TYPE",
  "target": "VehicleType"
}
```

5. **A new graph version is created**

[bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:244) runs `_save_graph()`.

Every click creates a unique version:

```text
tbox_a248b75f97014f9d8bb1496be1aefd5f
```

Previous versions are not deleted.

### Where the data is stored

The project and dataset come from:

```dotenv
ONTOLOGY_BQ_PROJECT=...
ONTOLOGY_BQ_DATASET=ontology_graph
```

Three permanent tables are used:

| BigQuery table | What it stores |
|---|---|
| `graph_nodes` | One row per ontology class |
| `graph_edges` | One row per ontology relationship |
| `graph_versions` | One row describing the complete saved version |

#### `graph_nodes`

Written at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:288).

Important columns:

```text
graph_version_id
graph_type
node_id
node_type
label
properties
created_at
```

Example:

```text
graph_version_id: tbox_abc123
graph_type: TBOX
node_id: Vehicle
node_type: Class
label: Vehicle
properties: {"attributes": [...]}
```

#### `graph_edges`

Written at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:289).

Important columns:

```text
graph_version_id
edge_id
source_id
relationship_type
target_id
properties
created_at
```

Example:

```text
source_id: Vehicle
relationship_type: IS_OF_TYPE
target_id: VehicleType
```

#### `graph_versions`

Written last at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:292).

It contains:

```text
graph_version_id
graph_type
session_id
node_count
edge_count
metadata
created_at
```

Writing this table last means the version is recorded as complete only after the nodes, edges, and graph schema are successfully created.

### How the visual BigQuery graph is created

After storing the rows, [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:349) runs:

```python
_replace_tbox_property_graph()
```

It creates:

- One `_ontology_pg_node_*` view for every class.
- One `_ontology_pg_edge_*` view for every relationship.

For example:

```text
_ontology_pg_node_vehicle_...
_ontology_pg_node_booking_...
_ontology_pg_edge_vehicle_is_of_type_vehicletype_...
```

Those views are filtered to the newly generated `graph_version_id`.

Finally, at [bigquery_service.py](C:/Users/niketu/Downloads/ontology-agent/backend/app/bigquery_service.py:468), it runs:

```sql
CREATE OR REPLACE PROPERTY GRAPH ontology_pg
```

Therefore:

```text
Permanent tables
├── Keep every historical version
│
Filtered _ontology_pg_* views
├── Point to the latest version
│
ontology_pg
└── Shows the latest classes and relationships in BigQuery Graph
```

Important: the current Save button stores only the **TBox**. The `save_abox()` function exists, but `/api/bigquery/sync` does not currently call it. Therefore, individual records such as `V-1001` are not saved by this button.
