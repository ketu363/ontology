# How the Ontology Agent Works — The Simple Story

This is the plain, talk-through version of the flow — say this out loud
while walking someone through the code.

---

**1. We start with two kinds of files.**
One kind is the raw fleet data — CSV files like `vehicles.csv`,
`drivers.csv`, `depots.csv`, `driver_assignments.csv`,
`maintenance_events.csv`, `fault_records.csv`. This is just what a real
client's fleet looks like: their vehicles, their drivers, their
maintenance history.

The other kind is the ISO standard — two YAML files,
`fleet_standards_profile.yaml` (the actual rules: what entities and
attributes a fleet ontology must have) and `source_registry.yaml` (which
real ISO standard each rule is based on).

**2. Both of these go into `data.py`.**
`data.py` doesn't try to understand the CSVs or the YAML — it just opens
every file and reads it as plain text. Then it glues the two YAML files
together into one variable called `SYNTHETIC_ISO_STANDARD`, and glues all
the CSVs together into another variable called `SYNTHETIC_FLEET_DATA`.
So after this step, we just have two long strings of text sitting in
memory — one is "the rules," one is "the client's data."

**3. Those two variables get combined into one message and sent to the agent.**
When someone opens the app, the backend takes `SYNTHETIC_ISO_STANDARD` +
`SYNTHETIC_FLEET_DATA`, glues them into a single message, and sends that
as the first message to the ADK agent — basically saying *"here's the
standard, here's the client's data, build me an ontology."* Alongside
that message, the agent also carries its instructions (`prompt.py`) —
its permanent system prompt that tells it exactly how to behave: ground
everything in the standard and the data, never drop something the
standard requires, and always answer in one specific JSON format.

**4. The agent (really, the LLM) reads all of that and creates the ontology.**
This is the actual "thinking" step. The model reads the standard text and
the CSV text like a person would, and works out:
- what entities should exist (`Vehicle`, `Driver`, `Depot`,
  `MaintenanceEvent`, ...),
- what attributes each one has,
- and what relationships connect them (e.g. it sees a `depot_id` column
  in the vehicle data and a rule saying every asset belongs to one depot,
  and turns that into a `Vehicle -> Depot` connection).

It sends all of that back as one JSON object: a list of entities, a list
of relationships, and a list of notes (things it flagged, like "the
standard needs a telematics reading but the client data didn't include
one").

**5. The backend double-checks the agent's answer.**
Before trusting that JSON, the backend runs it through a schema check
(`schema.py`) — basically making sure it actually has the shape we asked
for (entities/relationships/notes, with the right fields). If the agent
returned something broken, this step catches it instead of passing junk
forward.

**6. The clean ontology JSON is sent to the browser.**
At this point it's just data traveling over the network — the backend
hands the validated ontology JSON to the React frontend.

**7. The frontend turns that JSON into an actual picture — the knowledge graph.**
The React app takes the `entities` and `relationships` and reshapes them
into "nodes" and "links" — the format a graph-drawing library
understands. That library (`react-force-graph-2d`) is what actually draws
the circles, the arrows, and lets you drag, zoom, and click on a node to
see its attributes. None of the graph *drawing* is custom code — we just
hand the library the right shape of data and it does the rest.

**8. Every chat message repeats steps 3–7, but on the same conversation.**
When you type something like "add an InsurancePolicy entity," that
message goes through the exact same path — sent to the agent, agent
reasons over it (still remembering the original standard + data because
it's the same session), returns the full updated ontology again, gets
validated the same way, and the graph on screen redraws with the new
result.

---

**One-line version:** CSV + YAML files → read as plain text → glued into
two variables in `data.py` → sent to the LLM as one big message along
with its instructions → LLM reasons out entities/attributes/relationships
as JSON → backend validates the JSON shape → frontend reshapes it into
nodes/links → graph library draws it → every chat message repeats this on
the same conversation so the graph keeps updating.
