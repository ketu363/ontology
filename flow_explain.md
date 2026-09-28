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



**2. File: `runner_service.py`. Function: none yet, just the top of the file.**
This file imports those same two variables from `data.py`. Now
`runner_service.py` has access to "the standard" and "the client data"
as two plain strings.

**3. File: `runner_service.py`. Function: `generate_initial_ontology()`.**
This function is the one that actually kicks things off — it runs the
moment someone opens the app in their browser. It takes the two
variables from step 2 and glues them into ONE message, something like:
*"STANDARD: [all the yaml text] ... CLIENT DATA: [all the csv text] ...
now generate the ontology."*

**4. Same file, same function: `generate_initial_ontology()`.**
Before sending that message anywhere, this function first asks ADK
(Google's agent framework) to open a brand-new, empty conversation. This
gives us a `session_id` — think of it as a fresh chat thread with no
history yet.

**5. File: `runner_service.py`. Function: `_ask()`.**
`generate_initial_ontology()` now hands off the big message (from step
3) and the `session_id` (from step 4) to a second function called
`_ask()`. This function's only job is: send one message into one session
and get one answer back.

**6. Same file, same function: `_ask()`.**
This is the actual hand-off to the AI model. `_ask()` calls ADK's
runner, which sends our message to Gemini — and along with our message,
Gemini is also given its permanent instructions (these live in a
separate file, `prompt.py`, and were attached to the agent back in
`agent.py`). So Gemini receives: its instructions + the standard + the
client data, all at once.

**7. Outside our code — this is Gemini itself, not a file we wrote.**
Gemini reads all of that like a person reading a document, and works
out the entities, their attributes, and the relationships between them,
following the rules it was given. It writes its answer back as one
block of text that's supposed to be JSON.

**8. Back in `runner_service.py`. Function: `_ask()` again — same function, second half.**
`_ask()` receives Gemini's reply. Two things happen to it here:
- It's cleaned up slightly (in case Gemini wrapped it in ```json fences).
- It's checked against a strict format defined in `schema.py` — this
  step either produces a proper, trustworthy "Ontology" object, or fails
  loudly if Gemini's answer wasn't shaped correctly.

**9. File: `runner_service.py`. Function: `generate_initial_ontology()` — wrapping up.**
This function gets the validated ontology back from `_ask()`, and
returns it (together with the `session_id`) to whoever called it.

**10. File: `routes.py`. Function: `init_ontology()`.**
This is the function that originally called `generate_initial_ontology()`
in step 3. It now has the ontology + session_id, packages them as a plain
JSON response, and sends that back over the network to the browser.

**11. File: `api.js` (frontend). Function: `initOntology()`.**
This is the browser-side function that made the original request. It
receives the JSON response from step 10.

**12. File: `App.jsx` (frontend). Location: inside the `useEffect`.**
`App.jsx` takes what `initOntology()` got back, and stores two things in
the app's memory: the ontology itself, and the `session_id` (needed later
for chat). It also writes the first summary line into the chat panel.

**13. File: `GraphView.jsx` (frontend).**
`App.jsx` hands the ontology to this component. `GraphView.jsx` converts
the ontology's `entities` and `relationships` into the "nodes" and
"links" format a graph-drawing tool understands.

**14. Inside `GraphView.jsx`, but this part is a third-party library: `react-force-graph-2d`.**
This library takes the nodes/links from step 13 and actually draws the
circles and arrows on screen, and handles dragging/zooming/clicking —
none of that drawing logic is ours, we just feed it the right shape of
data.

---

## What happens when you type a chat message (the update loop)

**15. File: `ChatPanel.jsx`. Function: `submit()`.**
When you type a message and hit Send, this function fires and passes
your text up to `App.jsx`.

**16. File: `App.jsx`. Function: `handleSend()`.**
This calls `sendChatMessage()` in `api.js`, sending your text **plus**
the `session_id` we saved back in step 12.

**17. File: `routes.py`. Function: `chat()`.**
Receives your message + session_id, and calls `update_ontology()` in
`runner_service.py`.

**18. File: `runner_service.py`. Function: `update_ontology()`.**
This function is almost empty on purpose — it just calls `_ask()` again
(the exact same function from steps 5–8), but using the **existing**
session_id instead of creating a new one. That's the whole trick: because
it's the same session, Gemini still remembers the original standard and
data and everything said before.

**19. Everything from step 6 onward repeats.**
Gemini reasons over your new message plus the full history, returns the
full updated ontology, it gets validated the same way, sent back to the
browser, and `GraphView.jsx` redraws the graph with the new result.

---

## One line to say out loud in the meeting

*"`data.py` turns our standard and CSV files into two text variables.
`runner_service.py`'s `generate_initial_ontology()` glues those into one
message and passes it to `_ask()`, which sends it to Gemini through ADK.
Gemini's reply gets validated against `schema.py`, sent back through
`routes.py` to the browser, where `GraphView.jsx` turns it into the graph
you see on screen. Every chat message repeats that same `_ask()` step on
the same session, so the graph keeps updating."*
