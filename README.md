# simulated-city-template

This is a template repository.

Get started by reading [docs/setup.md](docs/setup.md).
See [docs/overview.md](docs/overview.md) for an overview of the base module content.

## Template for a project 

### Step 1: Define Your Simulation (Before Any Code)
This project is a simplified simulation model of train operator timetabling in Denmark. The geographical scope is limited to intercity rail services on the island of Zealand. The purpose of the simulation is to model how a single train operator moves along predefined train routes during the day while maintaining a closed circulation pattern — meaning the operator must end the simulation at the same station where they began.

The rail network is represented as a map of Zealand with selected intercity routes drawn as red lines. Stations along these routes are defined as discrete nodes. The operator is represented as a movable marker that travels between stations along these predefined paths.

In this model, one train trip is defined as a single movement from one station to the next adjacent station along a route. A full simulation consists of a configurable number (x) of consecutive train trips. The operator moves stepwise between stations according to the route structure, without considering infrastructure capacity, delays, or crew regulation constraints. Running times are assumed fixed and are not dynamically calculated.

The central constraint of the simulation is circulation closure: the sequence of station-to-station movements must be constructed so that after completing x trips, the operator returns to the original starting station. This reflects a fundamental principle in real-world operator and rolling stock planning, where daily diagrams typically begin and end at the same depot or base location.

The objective of the project is therefore not to optimize infrastructure usage, but to visualize and test operator circulation patterns on a simplified intercity network. The simulation demonstrates how sequential train movements can be structured into a closed operational loop within a fixed geographic system.


### My Smart City Project: Crew timetables and train operators 

#### 1. The Trigger (Who/What is moving?)
The agent in the simulation is a single train operator, represented as a movable marker on a map. The vehicle context is an intercity train operating on predefined routes across Zealand, Denmark. The surroundings consist of a simplified rail network map where stations are fixed nodes and train lines are predefined paths drawn in red. Time progresses in discrete steps, where each step represents one train trip — defined as a movement from one station to the next adjacent station along a route. The trigger event is the operator beginning a trip from a station and moving toward the next station in sequence.

#### 2. The Observer (What does the city see?)
The observer is the simulation system itself. It tracks the operator’s current station, the available connected stations along the selected intercity route, and the number of completed trips. The system continuously monitors three state variables: current position, trip counter, and starting position. It detects each completed movement from one station node to the next and updates the operator’s location accordingly.

#### 3. The Control Center (The Logic)
The control logic governs how the operator moves through the network. At each step, the system determines the next valid adjacent station along the predefined route. It increments the trip counter and updates the current position. The central constraint is circulation closure: after x trips, the operator must return to the original starting station. The logic therefore either selects a route sequence that naturally forms a closed loop or ensures that the final segment directs the operator back to the starting node. The system checks whether the trip count has reached x and whether the current station equals the starting station.

#### 4. The Response (What happens next?)
If the trip counter is below x, the operator marker moves to the next station along the red route line. The map visually updates to show this movement. When the operator completes x trips and arrives back at the starting station, the simulation cycle ends successfully. The result is a closed operational loop where the operator begins and ends at the same location, demonstrating a simplified model of daily circulation planning for an intercity operator on Zealand.

---

## Workflow: Document-Driven Development with AI

**This is how you work with any AI model (including GitHub Copilot, ChatGPT, Claude, etc.). The approach works regardless of which model your school uses.**

### Phase 1: Clarify Your Idea with AI (No Code Yet)

#### Copy this prompt into your AI chat:

```
I want to build a simulated city based on this outline. 
Please help me clarify it before I code.

[Paste your Project Template filled in above]

Please:
1. Rewrite the 4 components using clear, technical language
2. Identify the MQTT topics each agent will publish/subscribe to
3. List any configuration parameters (MQTT broker, locations, thresholds)
4. Point out any ambiguities or missing details

Do NOT write any code. Just clarify the design.
```

#### Review the AI's response
- Does it capture your idea correctly?
- Are the agents clearly separated?
- Are the MQTT topics clear?
- If not, refine and ask again

---

### Phase 2: Get an Implementation Plan (Still No Code)

#### Once you agree on the design, use this prompt:

```
Based on the design we just clarified:

[Paste the clarified design from Phase 1]

Please propose a phased implementation plan:
- Phase 1: Single basic agent (smallest working notebook)
- Phase 2: Add configuration file
- Phase 3: Add MQTT publishing
- Phase 4: Add second agent with MQTT subscription
- Phase 5: Add dashboard visualization

For each phase:
1. List what new notebook files will be created
2. List what tests/verifications I should run
3. Say exactly what I should investigate/understand before moving to the next phase

Do NOT write code yet. Just show the phases.
```

#### Review and approve the plan
- Does each phase test one new thing?
- Can you run and understand each phase?
- Are there gaps?
- Ask AI to adjust if needed

---

### Phase 3: Implement ONE Phase at a Time

#### For the FIRST phase only, use this prompt:

```
Implement ONLY Phase 1 from the plan above:
[Paste Phase 1 description]

Remember these rules (from .github/copilot-instructions.md):
- Use anymap-ts for mapping (NOT folium)
- Each notebook is ONE agent (NOT monolithic)
- Load config via simulated_city.config.load_config()
- Use mqtt.publish_json_checked() for publishing
- Add all dependencies to pyproject.toml (NOT !pip install in notebooks)

Only implement Phase 1. Do NOT jump ahead to Phase 2.
Include comments explaining each section.
```

#### After you get the code:
```bash
python scripts/verify_setup.py      # Check dependencies
python -m pytest                     # Run tests
python -m jupyterlab                # Open the notebook and RUN IT
```

#### Investigate before moving forward
- Does the notebook actually run without errors?
- Can you explain what each cell does?
- Does it match the design from Phase 1?
- If something is wrong, ask AI to fix it before moving to Phase 2

---

### Phase 4: Move to the Next Phase

Once Phase 1 works, use this prompt:

```
Good! Phase 1 works. Now implement ONLY Phase 2:
[Paste Phase 2 description]

The Phase 1 notebooks/code are:
[List what was created in Phase 1]

Implement only Phase 2. Do NOT modify Phase 1 code unless necessary.
```

**Repeat this cycle for each phase.**

---

## Key Rules to Remember

✅ **DO** enforce these in every AI prompt:
1. Two separate MQTT topics are better than one shared variable
2. Each agent notebook is independent and can restart anytime
3. Configuration comes from `config.yaml`, not hardcoded values
4. All dependencies go in `pyproject.toml` first, then `pip install -e ".[notebooks]"`
5. Dependencies must be approved: `anymap-ts` ✅, `folium` ❌

❌ **DO NOT** let AI:
- Skip the documentation/planning phases
- Create one giant notebook with all logic
- Jump to implementation without a clear, approved design
- Install packages inside notebooks with `!pip install`
- Use `folium`, `matplotlib`, or `plotly` for real-time maps

---

## If the AI Skips Steps

If you ask for implementation and the AI writes code without clarifying the design first, respond with:

> "No code yet. I need to clarify the design first. Please rewrite my outline using the Phase 1 prompt above, then we'll get a plan before any implementation."

If the AI proposes all 5 phases at once instead of letting you implement one at a time:

> "I need only Phase 1 implementation. We'll do the other phases after I test Phase 1. Just give me Phase 1 code."

If the AI installs `folium` or uses `!pip install`:

> "No, use anymap-ts and add dependencies to pyproject.toml. Also, re-read .github/copilot-instructions.md for the full list of rules."

---

## Testing Your Work

After each phase, run:

```bash
# Check environment
python scripts/verify_setup.py

# Run existing tests
python -m pytest

# Try your new notebook
python -m jupyterlab
# Open the notebook and run all cells
```

Before submitting a pull request, include this in your description:

```
Docs updated: yes/no
Phases completed: [e.g., "Phase 1 and Phase 2"]
Tests passing: yes/no
```