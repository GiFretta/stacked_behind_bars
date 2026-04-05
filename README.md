# Stacked Behind Bars — Simulation Notebook README

This notebook implements an object-oriented discrete-event simulation of overcrowding in the Brazilian prison system and adds an **interactive control panel** for scenario testing at the national level, for average/large comarcas, or for a specific comarca loaded from a parameter table. This is part of a larger project, the visual story "Stacked Behind Bars," developed by Gisele Fretta. Access the visual story here: https://lookerstudio.google.com/s/tmXwSlOA3UE

The file documented here is `simulation_code_with_user_interface.ipynb`. This Notebook won't render on GitHub because of the widgets used for the user interface. The best strategy is to open the gist at https://colab.research.google.com/gist/GiFretta/ac9f00c524cd6d6a167bf4c3b2a9c25d/simulation_code_with_user_interface.ipynb.

## What this notebook does

The notebook has three main layers:

1. **Core simulation engine**
   - Models arrests as stochastic arrivals.
   - Routes people through a pre-trial/litigation process.
   - Tracks queueing delays, decisions, convictions, sentence time, and incarceration totals.
   - Uses an event scheduler instead of step-by-step time simulation.

2. **Analysis and plotting layer**
   - Summarizes waiting times, queue lengths, incarceration dynamics, service times, and decision delays.
   - Runs policy-style experiments by varying arrest rates, number of litigation queues, and number of litigation stations.
   - Produces both single-run and multi-run plots, including confidence intervals for repeated simulations.

3. **User interface / control center**
   - Provides a one-cell widget-based control panel.
   - Lets the user load presets or comarca-specific parameter bundles.
   - Allows manual editing of capacities, arrest rates, sentence distributions, crime arrival probabilities, and experiment inputs.
   - Runs simulation outputs, experimental analyses, and theoretical utilization analysis without editing the core code.

---

## Recommended execution order

Run the notebook top to bottom.

1. Import packages.
2. Run all OOP class definitions.
3. Run helper, summary, analysis, and plotting functions.
4. Load `parameters_per_comarca`.
5. Run the **Simulation Control Panel Interface** cell.

The control panel assumes that the following have already been defined:

- `create_truncnorm`
- `Arrests`
- `calculate_exponential_scale`
- `run_multiple_simulations`
- `run_simulation`
- plotting functions
- experimental analysis functions
- `compute_mean_service_time`
- `parameters_per_comarca`

---

## Environment and dependencies

Core libraries used in the notebook:

- `numpy`
- `pandas`
- `scipy`
- `matplotlib`
- `ipywidgets`
- `IPython.display`
- `tqdm`

The UI was designed for a notebook environment that supports interactive widgets, such as **Google Colab** or **Jupyter Notebook / JupyterLab** with widget support enabled.

---

## Simulation architecture

The model represents the justice system as a queueing-style process:

- **Arrivals**: people enter through arrests.
- **Classification**: each person is assigned a crime profile and defense type.
- **Queueing**: people join a FIFO litigation queue.
- **Service**: litigation stations handle cases in parallel.
- **Post-service waiting**: after trial/litigation, each person waits for a sentencing/decision delay.
- **Outcome**: people are released or convicted.
- **Incarceration tracking**: the notebook records pre-trial and total incarcerated populations over time.

This event-driven structure makes it easy to study how system congestion changes under different policy assumptions.

---

## Class reference

### `Event`
Represents a scheduled simulation event.

**Purpose**
- Stores the time when an event occurs.
- Stores which function should run at that time.
- Makes events comparable so they can be sorted in a priority queue.

**Methods**
- `__init__(timestamp, function, *args, **kwargs)` — creates an event.
- `__lt__(other)` — allows chronological ordering in the heap.
- `run(schedule)` — executes the stored callback.

**Why it matters**
- This is the smallest unit of the discrete-event simulation.

### `Schedule`
Maintains the event calendar.

**Purpose**
- Stores future events in chronological order.
- Tracks the current simulation time.
- Advances the simulation one event at a time.

**Key attributes**
- `now` — current simulation time.
- `priority_queue` — heap of future events.

**Methods**
- `add_event_at(timestamp, function, *args, **kwargs)` — schedule an event at an absolute time.
- `add_event_after(interval, function, *args, **kwargs)` — schedule an event relative to current time.
- `next_event_time()` — inspect the next event time.
- `run_next_event()` — pop and execute the next event.
- `print_events()` — debug helper.

**Why it matters**
- This class is the engine that makes the model event-driven rather than time-step driven.

### `Arrests`
Stores crime profiles and samples arrivals.

**Purpose**
- Defines how likely each crime group is.
- Stores conviction probabilities, detention distributions, and service-time distributions by crime group.
- Samples a crime type for each new person.

**Methods**
- `__init__(crime_profiles)`
- `sample_crime()` — returns a sampled crime type and its profile.
- `sample_detention_length(detention_distribution)` — samples sentence length.

**Why it matters**
- It connects empirical crime-group proportions to the simulation’s stochastic behavior.

### `Person`
Represents one individual moving through the system.

**Purpose**
- Keeps all person-level state in one object.
- Stores arrival time, defense type, crime type, sentence information, and stage transitions.

**Typical state tracked**
- arrival time
- service/trial start time
- decision wait time
- conviction status
- release time
- sentence time

**Methods**
- `__init__(crime_type, crime_profile, arrival_time, representation_type)`
- `determine_conviction()`
- `mark_trial_start(current_time)`
- `move_to_waiting_decision(decision_wait_time, current_time)`
- `set_conviction_outcome(convicted, sentence_time)`
- `mark_release(current_time)`

**Why it matters**
- This class carries the individual lifecycle that later feeds system-level metrics.

### `LitigationQueue`
Models the FIFO queue that feeds litigation/trial service.

**Purpose**
- Holds people waiting to be processed.
- Starts service when a litigation station becomes free.
- Records queue lengths and waiting times.

**Key ideas**
- FIFO discipline.
- Multiple parallel stations supported.
- Integrated post-trial decision scheduling.

**Methods**
- `__init__(service_dist, queue_id, num_litigation_station, is_print=True)`
- `enter_queue(schedule, person)`
- `try_start_trial(schedule)`
- `leave_service(schedule, person)`

**Why it matters**
- This is where system bottlenecks become visible through waiting-time and backlog behavior.

### `JudicialSystem`
Coordinates the full simulation.

**Purpose**
- Manages queues, arrests, decisions, and incarceration accounting.
- Chooses which queue receives an arrival.
- Tracks pre-trial and total incarceration against thresholds.

**Key responsibilities**
- schedule new arrests
- assign defense type
- choose shortest queue
- sample post-trial decision delays
- update conviction outcomes
- track total incarcerated population over time

**Methods**
- `__init__(arrest_rate_dist, num_queues, num_service_stations, capacity_threshold, pre_trial_capacity_threshold, prob_private_defense, is_print=True)`
- `choose_queue(queue_list)`
- `schedule_arrests(schedule)`
- `get_decision_wait_sampler(crime_type, defense_type)`
- `handle_decision(schedule, person)`
- `add_to_prison_population(person, sentence_length)`
- `track_incarceration(schedule)`
- `run(schedule)`

**Why it matters**
- This is the top-level state container for each simulation run.

---

## Function reference

## 1) Simulation runners

### `run_simulation(...)`
Runs one full simulation.

**What it does**
- Creates the system and schedule.
- Starts arrivals and incarceration tracking.
- Processes events until the chosen time horizon.
- Returns the resulting `JudicialSystem` object.

**Main inputs**
- arrival distribution
- `Arrests` object
- number of queues
- number of litigation stations
- total and pre-trial capacity thresholds
- probability of private defense
- simulation duration

### `run_multiple_simulations(num_trials, run_simulation_func, sim_params)`
Runs repeated trials under the same parameter set.

**What it does**
- Repeats the simulation many times.
- Returns a list of `JudicialSystem` outputs.
- Supports confidence intervals and averaged plots.

### `test_event_ordering()`
Basic test case to verify that the schedule processes events chronologically.

---

## 2) Helper functions

### `create_truncnorm(mean, std, lower, upper)`
Builds a truncated normal distribution.

### `calculate_exponential_scale(arrests_per_month)`
Converts an average arrest rate into the exponential scale parameter used by `scipy.stats.expon`.

---

## 3) Summary / metric functions

These functions extract interpretable outputs from a completed `JudicialSystem` object.

### `summarize_waiting_times(court_system)`
Returns waiting-time summaries for the queue/litigation stage.

### `summarize_queue_lengths(court_system)`
Summarizes queue length trajectories across time.

### `summarize_incarceration(court_system)`
Summarizes incarceration trajectories, including total, pre-trial, and convicted populations.

### `summarize_service_times(court_system)`
Summarizes litigation/trial durations.

### `summarize_decision_wait_times(court_system)`
Summarizes the post-trial wait before a decision is issued.

---

## 4) Experimental analysis functions

These functions automate repeated experiments and return results ready for plotting.

### `analyze_waiting_time_vs_queues(...)`
Varies the number of litigation queues and measures waiting-time changes.

### `analyze_queues_vs_incarceration(...)`
Varies the number of litigation queues and measures incarceration outcomes.

### `analyze_arrival_rate_vs_pretrial_ratio(...)`
Varies arrest rate and measures the share of incarceration that remains pre-trial.

### `analyze_stations_vs_incarceration(...)`
Varies the number of litigation stations per queue and measures incarceration outcomes.

### `analyze_waiting_time_vs_stations(...)`
Varies the number of litigation stations per queue and measures waiting-time changes.

---

## 5) Theoretical analysis

### `compute_mean_service_time(crime_profiles)`
Computes the weighted mean service time across crime groups.

**Why it matters**
- Used in the utilization formula \(\rho = (\lambda / N_q) \cdot S\).
- Supports the notebook’s theoretical stability analysis.

---

## 6) Plotting functions

### Single-run plots
- `plot_incarceration(...)` — incarceration over time for one run.
- `plot_queue_lengths(...)` — queue lengths over time.
- `plot_time_before_sentence(...)` — histogram of time in system before sentence/decision outcome.

### Multiple-run plots
- `plot_incarceration_multiple(...)` — mean incarceration trajectory with confidence intervals.
- `plot_incarceration_by_crime_multiple(...)` — incarceration trajectory by crime type across repeated runs.

### Experiment plots
- `plot_waiting_times_vs_queues(...)`
- `plot_incarceration_vs_queues(...)`
- `plot_arrival_rate_vs_pretrial_ratio(...)`
- `plot_waiting_times_vs_stations(...)`
- `plot_incarceration_vs_stations(...)`
- `plot_utilization_curve(...)`

---

## 7) Control-panel helper functions

These functions power the user interface.

### Parsing and validation
- `parse_number_list(text)` — parses comma-separated numeric input.
- `parse_int_list(text)` — parses comma-separated integers.
- `safe_float(x, default=0.0)` — safe numeric conversion.
- `safe_int(x, default=0)` — safe integer conversion.
- `normalize_text(text)` — removes accents and normalizes search text.

### Widget ↔ data conversion
- `get_sentence_counts_from_widgets()`
- `get_arrival_probabilities_from_widgets()`
- `set_sentence_count_widgets(sentence_counts)`
- `set_arrival_probability_widgets(ap)`
- `normalize_probabilities(prob_dict)`

### Model-input builders
- `build_sentence_distribution(sentence_counts)` — converts sentence-bin counts into a discrete distribution.
- `build_crime_profiles(arrival_probabilities, sentence_counts)` — constructs the `crime_profiles` dictionary used by `Arrests`.
- `build_sim_params_from_controls()` — reads the current UI state and constructs the final simulation inputs.

### Presets and comarca logic
- `row_to_local_parameters(row)` — converts one `parameters_per_comarca` row into a preset dictionary usable by the UI.
- `apply_parameter_preset(preset_dict, preset_name="")` — loads a preset into the widgets.
- `refresh_comarca_search_results(_=None)` — searches the comarca table.
- `load_selected_comarca(_=None)` — loads the selected comarca’s parameters into the control panel.
- `update_parameter_preset_visibility(change=None)` — toggles UI behavior when preset mode changes.

### Main UI runner
- `run_selected_analysis(_=None)` — executes whichever analyses are currently selected in the interface.

### Small UI helper
- `section_title(text)` — formats control-panel section headers.

---

## Parameter-per-comarca table

The notebook includes a dataframe named `parameters_per_comarca`.

### What it represents
Each row is a **comarca-level parameter bundle** that can be used to pre-fill the control panel.

A comarca is treated as a localized judicial unit, so the row works as a ready-made scenario with:

- arrest intensity
- prison capacity thresholds
- sentence-duration composition
- crime-group composition
- supporting metadata such as number of cities and prisons covered

### Main groups of columns

#### 1. Identifier columns
- `cod_mun_ibge_comarca`
- `nome_comarca`
- `Endereço Comarca`

These identify the comarca and support search/display in the UI.

#### 2. Sentence-count columns
Examples:
- `count per sentence time - 0_6mo`
- `count per sentence time - 13mo_2yr`
- `count per sentence time - 9_15yr`
- `count per sentence time - gt100`

These are used to build the detention/sentence distribution. The UI maps them through `SENTENCE_COL_MAP`.

#### 3. Capacity and flow columns
- `new_admissions_per_semester`
- `arrests_per_month`
- `total_capacity_threshold`
- `pre_trial_capacity_threshold`

These define the size and pressure of the local system.

#### 4. Crime-count columns
Examples:
- `count per crime group - Against Property`
- `count per crime group - Drug-related`
- `count per crime group - Against the person`

These are raw counts behind the local crime composition.

#### 5. Crime-arrival-probability columns
Examples:
- `arrival probability - Against Property`
- `arrival probability - Drug-related`
- `arrival probability - Firearms`

These are the normalized crime-group arrival probabilities used in simulation sampling. The UI maps them through `ARRIVAL_COL_MAP`.

#### 6. Coverage / quality metadata
- `count of cities covered`
- `count of prisons covered`
- `sum arrival probabilities`
- `total count per sentence time`

These fields help the user assess whether a row looks complete and how broad the comarca aggregation is.

### How the notebook uses this table

`row_to_local_parameters(row)` transforms a dataframe row into a parameter preset.

That function:

1. reads sentence-bin counts,
2. reads local crime arrival probabilities,
3. reads arrest/capacity values,
4. fills in UI defaults for fields not present in the table,
5. normalizes probabilities when needed,
6. falls back to national/default values when row data is missing or zero.

### Important fallback behavior
- If sentence counts sum to zero, the notebook falls back to `DEFAULT_SENTENCE_COUNTS`.
- If arrival probabilities sum to zero, it falls back to `DEFAULT_ARRIVAL_PROBABILITIES`.
- Invalid or missing capacity/arrival values are replaced by default benchmark values.

This design makes the UI robust when some comarca rows are incomplete.

---

## Default presets in the control panel

The notebook defines three built-in presets:

### `national`
A large aggregated scenario intended to approximate the full system.

### `average_comarca`
A benchmark local scenario using average comarca-style values.

### `large_comarca`
A larger local scenario with higher arrest rate, higher capacity, and more litigation stations.

### `search_for_comarca`
Not a numerical preset by itself; it activates the comarca search/load workflow.

---

## Guide to the parameter control center

The last notebook cell creates the interactive simulation control panel.

## Step 1 — Run all prerequisite cells
Before opening the UI, make sure:
- the classes are defined,
- the helper/analysis/plotting functions are defined,
- `parameters_per_comarca` has been loaded.

## Step 2 — Choose a parameter source
Use the **preset dropdown** to choose one of the following:
- `national`
- `average_comarca`
- `large_comarca`
- `search_for_comarca`

If you choose a built-in preset, the widgets populate immediately.

## Step 3 — Load a comarca (optional)
If you choose `search_for_comarca`:

1. Type a comarca name in the search box.
2. Click **Search**.
3. Select a match from the results dropdown.
4. Click **Load comarca parameters**.

Once loaded, the notebook displays metadata such as:
- comarca code
- address
- number of cities covered
- number of prisons covered
- raw sentence count total
- raw arrival probability sum

## Step 4 — Review or edit general parameters
The main simulation controls include:

- **Arrests/month**
- **Capacity threshold**
- **Pre-trial capacity**
- **Litigation queues**
- **Litigation stations**
- **Probability of private defense**
- **Simulation duration**
- **Number of trials**
- **Summary start / summary end**

These values determine both the run itself and the averaging window used in repeated-trial summaries.

## Step 5 — Review or edit crime arrival probabilities
The panel exposes one widget per crime group. These values define how likely each crime type is at arrival.

The notebook normalizes the set automatically when building the simulation parameters, so the values do not need to sum exactly to 1 in the widgets.

## Step 6 — Review or edit sentence counts
The sentence-count widgets control the empirical sentence distribution used to build detention lengths.

Higher counts in a bin mean that that sentence-duration interval receives more probability mass.

## Step 7 — Choose which analyses to run
The `Run:` selector supports multiple selections:

- **A. Simulation Results**
  - Runs repeated simulations with the current parameter set.
  - Plots incarceration over time and incarceration by crime type.

- **B1. Experiment 1: Arrest Rate**
  - Uses the arrest-rate list widget.
  - Shows how pre-trial share changes as arrests/month changes.

- **B2. Experiment 2: Number of Litigation Queues**
  - Uses the queue-count list widget.
  - Shows waiting-time and incarceration effects of changing the number of queues.

- **B3. Experiment 3: Number of Litigation Stations**
  - Uses the station-count list widget.
  - Shows waiting-time and incarceration effects of changing service capacity per queue.

- **C. Theoretical Analysis**
  - Computes utilization using the weighted average service time.
  - Can use the automatically computed service time or a manual override.

## Step 8 — Configure experiment-specific inputs
Additional text widgets accept comma-separated lists for:

- arrest-rate experiments
- queue-count experiments
- station-count experiments
- theoretical-analysis queue values

Examples:
- Arrest rates: `10, 20, 30, 40`
- Queue counts: `1, 2, 4, 6, 8`
- Station counts: `1, 2, 3, 4`

## Step 9 — Run the panel
Click the **Run selected analysis** button.

The output area will:
- print the name of each selected analysis,
- execute them sequentially,
- show the relevant plots,
- report utilization values for the theoretical analysis.

---

## How the control panel builds simulation inputs

Internally, the UI follows this pipeline:

1. Read widget values.
2. Build sentence counts and crime probabilities.
3. Normalize arrival probabilities.
4. Build the sentence distribution.
5. Build `crime_profiles`.
6. Create an `Arrests` object.
7. Convert `arrests_per_month` into an exponential inter-arrival distribution.
8. Assemble `sim_params_base`.
9. Pass those parameters into the selected simulation/analysis function.

This is the key function chain:

`build_sim_params_from_controls()`  
→ `build_crime_profiles()`  
→ `Arrests(crime_profiles)`  
→ `calculate_exponential_scale()`  
→ `run_simulation()` / experiment runners

---

## Interpreting the main parameters

### `arrests_per_month`
Controls demand entering the system. Larger values generally increase congestion.

### `num_litigation_queues`
Controls how many parallel queues exist. In the experiments, this acts as one possible capacity lever.

### `num_litigation_stations`
Controls service capacity per queue. Increasing this typically reduces waiting time and incarceration accumulation.

### `capacity_threshold`
Total incarceration threshold used as an overcrowding benchmark.

### `pre_trial_capacity_threshold`
Threshold for pre-trial incarceration only.

### `prob_private_defense`
Changes the share of people assigned private defense, which affects post-trial decision waits and can influence outcomes.

### `sentence_counts`
Shape the sentence/disposition duration distribution used when people are convicted.

### `arrival_probabilities`
Shape the local crime mix at entry.

---

## Notes and implementation details

- The notebook uses the same sentence distribution across crime groups, but crime groups differ in arrival probability, conviction probability, service time, and decision-wait distributions.
- Arrival processes are modeled with an exponential inter-arrival distribution.
- Some labels in the notebook use “litigation” terminology to make the UI easier to interpret for a broader audience.
- Multi-trial plots are the best choice when you want smoother, more reliable comparisons.
- Single-run plots are useful for debugging or demonstrating one simulated trajectory.

---

## Suggested future improvements

A few natural extensions for this notebook are:

- export control-panel configurations to JSON or CSV,
- save simulation outputs directly from the UI,
- support state-level presets in addition to comarca presets,
- add sensitivity analysis for conviction probabilities and decision waits,
- add policy-to-staff translation logic using comarca staffing assumptions,
- package the notebook logic into Python modules for easier maintenance.

---

## File documented

- `simulation_code_with_user_interface.ipynb`
