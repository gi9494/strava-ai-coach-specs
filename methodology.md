# strava-ai-coach

A running coach built on your own training data.

It pulls your runs from Strava and corrects them for the weather. From them it
works out your threshold, predicts race times, and forecasts how fit you will
be on race day. A coach agent then builds each week's plan, which you approve
or decline day by day.

![How it works](docs/architecture.png)

## Demo

**The app at a glance** — goal race card, the Mon–Sun week, charts, coach chat

![Demo 1](docs/demo1.gif)

**Planning with the coach** — ask for a week or a change, review the proposal,
approve or decline each day

![Demo 2](docs/demo2.gif)

---

## Contents

1. [Demo](#demo)
2. [Quick start](#quick-start)
3. [How it fits together](#how-it-fits-together)
4. [Methodology](#methodology)
   - [Making runs comparable: weather](#1-making-runs-comparable-weather)
   - [Best efforts](#2-best-efforts)
   - [Threshold](#3-threshold)
   - [Session labels and HR zones](#4-session-labels-and-hr-zones)
   - [Race prediction: Riegel k](#5-race-prediction-riegel-k)
   - [Fitness forecast: race-day range](#6-fitness-forecast-race-day-range)
   - [Goal race and training phase](#7-goal-race-and-training-phase)
5. [The coach agent](#the-coach-agent)
6. [Defaults at a glance](#defaults-at-a-glance)
7. [Assumptions and known limits](#assumptions-and-known-limits)
8. [Roadmap (after v1)](#roadmap-after-v1)
9. [Reference](#reference): layout, tables, several athletes, fresh start

---

## Quick start

Always run commands from the project root.

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Copy `.env.example` to `.env` in the root and fill it in. Never commit it.

```
STRAVA_CLIENT_ID=...
STRAVA_CLIENT_SECRET=...
STRAVA_REFRESH_TOKEN=...          # needs scope activity:read_all

LLM_PROVIDER=ollama               # chat: local, no quota (or: gemini)
OLLAMA_MODEL=qwen2.5:14b          # or qwen3:14b
OLLAMA_CONTEXT=16384
PLANNER_PROVIDER=gemini           # weekly planner: one Gemini call per week
GEMINI_MODEL=gemini-2.5-flash
GOOGLE_API_KEY=...
```

For the local chat: `ollama pull qwen2.5:14b` once, and keep Ollama running.

To get a refresh token with the right scope, run
`python get_new_refresh_token.py`. A token with only `read` authenticates
fine, then fails with a 401 error on activities.

To run the app:

```bash
streamlit run src/ui/app.py
```

Every time the app is opened (a new tab or a reload) it runs `refresh()`,
which does the rest; the **↻ Refresh** button runs it again without reloading.
It is incremental, so it only fetches what is new, and the chat is kept. To
refresh without the UI, run `python src/pipeline.py`.

---

## How it fits together

There are four blocks, one per column of the diagram.

| Block | What it does | Where |
|---|---|---|
| **Sources** | Strava (runs and per-second streams), Open-Meteo (weather), an LLM (Gemini or Ollama) | `src/ingestion/`, `src/agent/llm.py` |
| **Pipeline** | `refresh()`: sync, weather, threshold, labels, race times, forecast, plan matching | `src/pipeline.py`, `src/features/`, `src/models/` |
| **Coach agent** | A LangGraph agent with tools, plus a planner subagent | `src/agent/` |
| **App** | Streamlit: goal race card, week table with approve/decline, charts, chat | `src/ui/app.py` |

Everything is stored in one DuckDB file. Per-run streams are Parquet files.

`refresh()` runs these steps in order:

```
sync → runs_weather → runs_adjusted → best_efforts → threshold → run_labels
     → pace_features → decay (k) → race_predictions
     → agent tables → match runs to plans → fitness_forecast
```

---

## Methodology

### 1. Making runs comparable: weather

The same effort is slower in heat. Every run therefore gets a second pace:
what it would have been in neutral weather.

**Weather source.** Open-Meteo, free and with no API key. Coordinates are
rounded to 0.1° (about 11 km), so one request covers one area.
`start_date_local` is already local time even though it ends in `Z`: the `Z`
is stripped, not converted.

**Why dew point.** Dew point says whether sweat can evaporate. Relative
humidity cannot say that on its own: 90% at 5 °C is harmless, and 90% at 28 °C
is brutal.

**Adjustment.** The adjustment is set by **temperature + dew point** in °C.
Within each band, the midpoint is used.

| T + Td (°C) | ≤20 | 20–25.6 | 25.6–31.1 | 31.1–36.7 | 36.7–42.2 | 42.2–47.8 | 47.8–53.3 | 53.3–58.9 | 58.9–64.4 | >64.4 |
|---|---|---|---|---|---|---|---|---|---|---|
| slower by | 0 | 0–0.5% | 0.5–1% | 1–2% | 2–3% | 3–4.5% | 4.5–6% | 6–8% | 8–10% | don't train hard |

```
adjusted_pace = actual_pace / (1 + adjustment)
```

Example: 24.8 °C with a dew point of 22.1 °C gives a sum of 46.9, which is
3.75% slower. So 5:44/km in those conditions is about 5:31/km neutral.

**The one rule about heat: correct it once.** Heat is applied only when
*judging* a run (its adjusted pace) and when *predicting* a race in given
weather. Prescribed paces are never slowed for the forecast. A plan says
"threshold 4:37"; on a hot day you run by feel, and afterwards the run's
adjusted pace is compared with 4:37. Adjusting both sides would count the
heat twice.

**Future weather** is used for race day and for upcoming sessions:
- **7 days or less away:** the Open-Meteo forecast at your usual hour and
  place.
- **Further away:** climatology. The last **3 years** of archive data, ±7 days
  around the date, at the same hour. This gives a typical value plus a cool
  case (10th percentile) and a warm case (90th percentile).

### 2. Best efforts

There are no races in the data, so performances are taken from training. For
every run, a two-pointer scan of the stream finds the fastest continuous
stretch at 1, 2, 5, 10, 15 and 20 km, the half marathon, and beyond. Each time
is scaled to the exact distance (a "5 km" window is often 5003 m). Pauses
count, so repeats with the watch stopped never pass for a continuous 5 km.
Times are weather-corrected before any fit.

### 3. Threshold

Threshold pace is the fastest pace you can hold for about 30 minutes. It is
the reference for everything else: session labels, pace bands, zones and race
times. It is found without a test:

1. For each run, slide a **7-minute window** over every point of the run and
   keep the **fastest** one. Its HR is the mean over the window's **last 3
   minutes**, because HR lags pace by a minute or two. Time is elapsed time,
   so a window with a stop or a jog in it is slow and never wins: in
   3 × 8' at 4:40 the window lands inside one repeat. A 3–5' VO2max repeat
   can't fill 7 minutes on its own, so it doesn't pass for threshold.
2. **Max HR** is the highest 30-second-smoothed peak seen *up to that date*,
   never with hindsight.
3. Keep windows with mean HR **≥ 88% of max HR**. There is no upper limit.
4. Correct them for weather, rank them by neutral pace, and **average the top
   3**. The result is the threshold pace and the threshold HR (LTHR).
5. Report nothing if there are fewer than **6** qualifying efforts.

The threshold is recomputed **every 7 days over a 120-day window, plus
today**, so every run is judged against the threshold of its own day and a
run counts on the day it's done.

**Window length.** `WINDOW_MIN` in `src/features/threshold.py` (7 by default,
6 also tested). Each row of `effort_windows` stores its length, so changing it
recomputes every run on the next refresh.
`python src/features/threshold.py test` compares 6 and 7 minutes and shows the
three efforts behind each result, without saving anything.

**Why not the 10 minutes before the HR peak** (the first version). It gave
4:48 at 182 bpm, but it couldn't read intervals: in 3 × 8' at 4:40 the window
held 8' of work plus 2' of jog and came out at 5:00, so a threshold session
never counted. With the sliding window, the reference athlete's threshold is
**4:37/km at 182 bpm** (max HR 201, HR floor 177).

### 4. Session labels and HR zones

**Labels** describe what a run *turned out* to be, not what was planned. The
first matching rule wins:

| Label | Rule |
|---|---|
| long | distance above the 85th percentile of your runs in the previous 90 days |
| hard | avg HR > 95% of LTHR **and** pace no slower than 1.25 × threshold pace |
| medium | avg HR > 88% of LTHR |
| easy | everything else |

HR leads and pace only vetoes: a high HR on a slow jog (heat, fatigue) is not
called hard. Runs from before the first threshold estimate stay unlabelled.

**HR zones** (Friel, as % of LTHR, in `src/features/zones.py`):

| Zone | % LTHR | Meaning |
|---|---|---|
| Z1 | < 85 | recovery |
| Z2 | 85–89 | aerobic |
| Z3 | 90–94 | tempo |
| Z4 | 95–99 | threshold |
| Z5 | ≥ 100 | above threshold |

Time in zones comes from the per-second stream. A gap between samples of more
than 10 s counts as a pause, not running time. The coach always gives zones in
bpm, because watches often use % of max HR, and "zone 2" can then mean two
different things.

### 5. Race prediction: Riegel k

```
T₂ = T₁ × (D₂ / D₁)^k          log T = log a + k · log D
```

`k` is how much your pace fades with distance. It is fitted per athlete as the
slope of a line through **all** best efforts on a log-log scale, not from two
distances. It is refit weekly on the previous 120 days, plus today.

- **Guards:** at least 4 distinct distances, and the longest at least 2× the
  shortest. If either fails, the population value k = 1.06 is used and the
  table flags `is_default`.
- **Race time** = training equivalent × **race factor 0.96** × weather
  adjustment for race day. The 0.96 reflects that training efforts have no
  taper and no competition. It is a guess until one real race calibrates it.
- **Reference athlete:** k = 1.061 with R² > 0.999.

This is **today's fitness**. The UI card shows it in neutral weather ("half
marathon today · neutral weather").

### 6. Fitness forecast: race-day range

The forecast answers one question: *how will threshold pace move between now
and the race?* Race time is assumed to move by the same percentage.

```
race-day time = today's prediction (race-day weather) × (1 + forecast change)
```

**Horizon.** H = days to the goal race, clipped to 7–112 days. With no goal
race, H = 28.

**Training examples.** For every weekly threshold value in the past, the
system records the features known on that date and the target: the % change
of threshold pace **H days later**. That later value must exist within
±10 days; otherwise the example is dropped.

| Feature | Meaning |
|---|---|
| `km_week` | average weekly km, last 4 weeks |
| `quality_share` | share of those km in hard or medium runs |
| `longest_km` | longest run, last 4 weeks |
| `trend` | the recent trend projected over H (see below) |
| `gap_to_best` | how far today's threshold is from its best so far |

**The trend.** It is a weighted straight line through the threshold estimates
of the **last 16 weeks**. Weights halve every **28 days**, so this week counts
twice as much as a week 4 weeks ago. After a **break** (more than 21 days with
no estimate), only the weeks since the break are used. The trend becomes a
projection with `change per week × weeks left ÷ today's pace`.

**Four methods compete:**

| Method | Prediction |
|---|---|
| `no_change` | the threshold stays where it is (the baseline) |
| `trend` | the weighted trend continues all the way to the race |
| `trend_half` | half of that: improvement slows near a ceiling |
| `ridge` | Ridge regression (standardised, α = 1) on the five features. Features are clipped to the training range so it never extrapolates volume. Needs at least 15 past examples. |

**Choosing the method.** The backtest walks forward: each past date is
predicted using only what came before it. The method with the lowest **MAE on
the last 12 predictions** wins. Recent predictions are used because you change
over time (a comeback, a plateau). A method that needs a feature missing today
can't be chosen.

**Correcting the winner with its own errors.** The method is picked by MAE
(error *without* sign, so errors in opposite directions can't cancel out).
Its signed errors then correct it (error = real change − predicted change):

```
centre = prediction + median of its past errors
range  = prediction + 10th … 90th percentile of its past errors   (80%)
```

A method that has always been too pessimistic is shifted by how much it
usually was, and the centre always sits inside the range. Example (22 Sep,
26 days out): `no_change` wins with MAE 6.8 s/km, but all 12 of its errors
are negative (you always improved more than "no change"); median −1.06%, so
the centre is 0% − 1.06% = −1.1%, range −5.1% … −0.2%. The other methods,
each corrected the same way, land between −0.6% and −2.4%.

**Safety cap.** A forecast is never more optimistic than the full trend
carrying on unchanged; if it is, the centre is set to the trend and the range
moves with it. The method is then labelled "(capped at trend)".

**Too little history** (fewer than 10 backtest points): the method is
`no_change`, so the race-day time equals today's prediction, with the spread of
past changes as the range.

The forecast is rebuilt on every refresh and saved to `fitness_forecast`. To
inspect it:

```bash
python src/models/fitness/race_predictions.py        # full backtest at the goal-race horizon
python src/models/fitness/race_predictions.py 60     # try 60 days; saves nothing
```

### 7. Goal race and training phase

You set the goal race from the chat ("I've entered the Rome half on
15 November"), and the app's race card updates. The distance is one of `5k`, `10k`, `half` or
`marathon`. The **phase** comes from the number of weeks to the race:

| Race distance | build starts | specific starts | taper starts |
|---|---|---|---|
| up to 10 km | 12 weeks out | 5 | 1 |
| up to half | 12 | 6 | 2 |
| marathon | 16 | 8 | 3 |

Before the build starts, the phase is base. With no goal race, the coach
trains a rolling base.

| Phase | Focus |
|---|---|
| base | easy volume, strides, a growing long run |
| build | 1–2 quality sessions a week |
| specific | work at and around goal pace |
| taper | volume −40–60%, same intensity and number of runs |

The race card shows the countdown, the phase, and the race-day prediction from
§6 **in neutral weather**, with its range. Under it, in smaller type, is the
same prediction in the expected race-day weather and what the weather costs.
The last figure is the weather itself: the forecast if the race is 7 days or
less away, otherwise climate.

---

## The coach agent

The coach is a LangGraph `StateGraph` with three nodes: **model**, **tools**
and **summarise**. The chat is saved in SQLite (`chat.sqlite`), so it survives
restarts. After 40 messages, older ones are folded into a summary and the last
16 are kept.

### What the model sees every turn

A **snapshot**, rebuilt from the database on each message. It contains:
- today's date, the goal race and phase;
- threshold pace and HR, and your zones in bpm;
- training load and the last run;
- **the real Mon–Sun week** (this week and next), with statuses and plan ids;
- any open proposal, and the facts you asked it to remember.

The snapshot is the source of truth. The model is told to trust it over
anything said earlier in the chat. This is what stops it hallucinating a plan.

### Tools (17)

| Area | Tools |
|---|---|
| Reading | `get_last_run` (incl. time in zones), `get_run_detail` (inside one run, see below), `get_recent_runs`, `get_training_load`, `get_paces_for_session`, `get_conditions`, `predict_race_time` (today + race-day range), `get_guidelines`, `get_plans` |
| Goal race | `set_goal_race`, `clear_goal_race` |
| Planning | `plan_week`, `edit_plan` |
| Bookkeeping | `confirm_planned_run`, `skip_plan`, `remember`, `forget` |

### Inside a run: `get_run_detail`

For questions about what happened *during* a run, the coach gets a summary of
the second-by-second stream (`src/features/run_detail.py`), never the raw
~3,000 points per metric: they wouldn't fit a local model's context, and an
LLM is poor at arithmetic over thousands of numbers. The code measures, the
model reads and explains. Only measurements, no guessing about the structure:

- **summary**: distance, time, average and peak HR (30-second average),
  cadence (Strava's per-leg value × 2);
- **splits** per km: pace, average HR, elevation up/down;
- **laps** recorded by the watch (lap button or structured workout), as
  Strava returns them: time, distance, pace, average/max HR. Downloaded the
  first time and cached as `data/streams/<id>.laps.json`, so one API call per
  run. Auto-laps every km are skipped (they repeat the splits);
- **HR drift** on runs of 40'+: speed per heartbeat, first vs second half
  after a 10' warm-up. Only meaningful on a steady-pace run; under 5% steady,
  over 10% the effort got much harder;
- time in zones.

An earlier version found the repeats from the pace (30-second blocks faster
than 1.12 × threshold). It cut repeats short (7'30" for 8') and invented one
from a faster stretch of the cool-down, so it was replaced by the watch's own
laps. `python src/features/run_detail.py [YYYY-MM-DD]` prints the summary.

### Act, don't describe: intent check and forced tool choice

LLMs like to *describe* a plan change instead of making it. So before the
model answers, a small classifier (a regex filter, then the LLM) labels the
message as `none`, `edit` or `new_week`. For `edit` and `new_week`, the first
model step is **forced** (`tool_choice="any"`) to call `edit_plan` or
`plan_week`. That way a change is always a tool call, never just text. Some
local models ignore `tool_choice`: if the forced step comes back without a
tool call, it is asked once more, plainly, to call the tool.

### Two ways to change a plan

- **`edit_plan`**: a JSON-patch-style list of operations (`set`, `remove`,
  `move`), touching only the days that change. This is the default. A rest
  day is a `remove`, never a 0-minute session. Every operation is validated
  before any is applied. If a proposal is already open, the edits merge into
  it.
- **`plan_week`**: a whole Mon–Sun week, for a new week or an explicit
  "redo it". It keeps runs already done this week, and any sessions you liked
  (`keep_plan_ids`).

### The planner subagent

`plan_week` calls a **separate LLM call with its own prompt** (the "agent as a
tool" pattern). Code does what code does best, and the model only plans:

1. **Gather** (code): phase, threshold, pace bands, zones, load for the last
   4 weeks, longest run for the last 30 days, fixed days (done or kept), your
   preferences and notes, and the weather per day.
2. **Plan** (LLM): returns a structured `WeekPlan`. Each session is a list of
   phases from a fixed English vocabulary (warm-up, easy, long, steady, race
   pace, threshold, intervals, hills, cool-down). Each phase has minutes, pace,
   repeats, recovery and target HR. This gives the same format every time.
3. **Check** (code), `check_rules` over the whole week, fixed days included:
   - no hard day right after another hard day (the long run counts as hard);
   - at most 2 quality sessions (hard or medium);
   - at most 1 long run;
   - no run longer than 1.10 × the longest of the last 30 days;
   - weekly km no more than 1.20 × the biggest of the last 4 weeks;
   - no phase of 0 minutes.
4. **Retry** up to 3 times, sending the errors back. If you asked for it
   anyway (**force**), rule errors (spacing, volume) are accepted and those
   sessions are marked **forced**, shown in grey. Structural errors (0-minute
   phases, a malformed week) are never accepted.

The result is saved as a **proposal**, never as the plan itself.

### Proposals: nothing changes until you approve

- Proposed sessions are **drafts** linked to a `plan_proposals` row. The week
  table shows them next to the current plan, with **approve ✓ / decline ✗ per
  day**, plus **Approve all / Decline all**.
- Approving a day turns its draft into `planned` and retires what it replaces.
  Declining drops the draft.
- A proposal closes automatically when no day is left open. Starting a new
  proposal declines the previous one.
- Moving a hard session next to another hard one marks it forced.

### After you run

On the next refresh, each new run is matched to that day's plan:

| Mark | Meaning |
|---|---|
| ✓ | done |
| ≠ | done, but different from the plan |
| ✗ | past planned session with no run: skipped |

An unplanned run is *not* turned into a plan. The week table shows the pace
you actually ran, with the weather-adjusted one in brackets when the weather
changed it (`11.0 km · 5:23 (adj. 5:18)`). When reviewing a run, the coach
compares its **weather-adjusted** pace with the **prescribed** pace.

### Behaviour principles (in `src/agent/prompt.py`)

- **The athlete decides.** The guidelines are defaults, not gates. When you
  ask for something against them, the coach gives its view with numbers in one
  or two sentences, then proposes it anyway. It never refuses and never says
  "ask another coach".
- It replies in your language. Plan content is always in English.
- Its numbers come from tools, never from memory. It says when a prediction is
  an extrapolation.
- It uses its general running knowledge freely. The guidelines document
  (`src/agent/knowledge/training_guidelines.md`, with rules tagged
  [strong], [moderate] or [convention]) settles the points where consistency
  matters.
- It doesn't diagnose. Pain means a physio or a doctor.

### Pace bands (× threshold pace, neutral weather)

| Session | Band | Quality sessions use |
|---|---|---|
| easy | 1.20–1.30 | |
| long | 1.15–1.25 | |
| medium | 1.08–1.15 | |
| threshold | 1.00–1.02 | the fast end: threshold pace itself |
| intervals | 0.93–0.98 | the fast end |

### LLM choice

Set in `src/agent/llm.py` from `.env`:

| Setting | Options | Default |
|---|---|---|
| `LLM_PROVIDER` | `gemini`, `ollama` | `gemini` |
| `GEMINI_MODEL` | | `gemini-2.5-flash` (free tier, rate-limited) |
| `OLLAMA_MODEL` | `qwen3:14b` suggested for 18 GB | `qwen2.5:14b` |
| `OLLAMA_CONTEXT` | | `16384` |
| `PLANNER_PROVIDER` | a different provider for the planner only | same as `LLM_PROVIDER` |

**Local chat, Gemini planner** (no chat quota, one Gemini call per planned
week):

```
LLM_PROVIDER=ollama
OLLAMA_MODEL=qwen2.5:14b          # or qwen3:14b
PLANNER_PROVIDER=gemini
GOOGLE_API_KEY=...
```

`ollama pull qwen2.5:14b` once, and keep Ollama running. The chat, the
intent check and the chat summary run locally; only `plan_week` goes to
Gemini. With Ollama, structured output uses `json_schema`, and qwen3's
`<think>` blocks are stripped from replies.

---

## Defaults at a glance

| What | Value | Where |
|---|---|---|
| Weather correction | T + Td bands, band midpoint | `features/adjusted_pace.py` |
| Future weather | forecast ≤ 7 days, otherwise 3-year climate ±7 days (p10/p90) | `ingestion/forecast.py` |
| Threshold | every 7 days + today, 120-day window, HR ≥ 88% max, top 3, needs ≥ 6 efforts | `features/threshold.py` |
| Effort window | fastest 7-min sliding window per run (`WINDOW_MIN`); HR over its last 3 min | `features/threshold.py` |
| Labels | long > p85 of 90 days; hard > 95% LTHR and pace ≤ 1.25×; medium > 88% LTHR | `features/classify.py` |
| HR zones | 85 / 90 / 95 / 100% of LTHR | `features/zones.py` |
| Riegel k | weekly + today, 120 days, ≥ 4 distances, max/min ≥ 2, otherwise 1.06 | `models/predictions/race.py` |
| Race factor | 0.96 | `models/predictions/race.py` |
| Forecast horizon | days to race, clipped 7–112; 28 with no race | `models/fitness/race_predictions.py` |
| Trend | 16 weeks, half-life 28 days, break > 21 days | `models/fitness/race_predictions.py` |
| Method choice | MAE on the last 12 backtest points; ridge needs ≥ 15 examples | `models/fitness/race_predictions.py` |
| Centre and range | prediction + median error; p10–p90 of the method's past errors (80%) | `models/fitness/race_predictions.py` |
| Phases | see §7 | `agent/context.py` |
| Planner rules | ≤ 2 quality, ≤ 1 long, no hard back-to-back, run ≤ 1.10× longest in 30 days, week ≤ 1.20× max of last 4 | `agent/planner.py` |
| Planner retries | 3 | `agent/planner.py` |
| Chat memory | summary after 40 messages, keep 16 | `agent/graph.py` |
| Refresh | every app opening (new tab or reload), or ↻ Refresh | `ui/app.py` |

---

## Assumptions and known limits

**Weather**
- The heat bands are coaching heuristics. They are consistent with Ely et al.
  (2007) for non-elite runners, but they are not fitted to you.
- Sun, wind, rain and heat acclimatisation are ignored.
- The correction is the same at every intensity.
- HR is not weather-corrected, so summer runs lean towards harder labels.

**Threshold**
- It needs some near-threshold running in the last 120 days. Without it,
  nothing is reported.
- Max HR is the highest *observed* value. The 88% floor partly compensates.
  The floor does not change with training level or sex.
- Threshold repeats shorter than 7 minutes aren't read.
- A race or an all-out 5 km can hold a 7-minute window faster than threshold,
  which pulls the top 3 down. `threshold.py test` shows which runs are behind
  the number.

**Best efforts and k**
- They come from training, so they are submaximal. That is why the race
  factor exists, and that factor is the least certain number in the chain.
- Riegel is optimistic for the marathon.
- Interval segments can enter the fit, and course elevation is not modelled.

**Forecast**
- It assumes race time moves by the same % as threshold pace.
- A few seasons of history make a small backtest, so the range is wide on
  purpose.
- A comeback after a break improves fast, which the capped trend and the
  break handling are there to contain.

**Labels**
- One label per run.
- Intervals are not told apart from continuous hard runs.

**The coach**
- An LLM can still misread a request. The snapshot, forced tool choice, patch
  edits, code-side rule checks and approve-first proposals keep mistakes
  visible and reversible.
- It is not a medical tool.

---

## Roadmap (after v1)

**Calibrate with real results**
- After the 18 Oct half: replace the race factor 0.96 with the one the real
  result gives (the least certain number in the chain).
- Check the 4:37 threshold against the next Garmin estimate, and look at the
  three efforts behind it (`python src/features/threshold.py test`).

**Models**
- Labels: use the watch laps so an interval session is labelled hard, not
  medium (3 × 8' came out "≠ ran medium").
- Threshold HR floor (88% of max) fixed for everyone: scale it with training
  level and sex.
- Forecast report: one sign convention for bias and errors.
- Predict `k` from training composition, and expected vs actual per run
  (pace and HR a run *should* have had).
- Elevation, wind and heat acclimatisation in the weather correction.

**Coach and app**
- Several goal races at once (a tune-up plus the main race): the nearest is
  already the one used; `set_goal_race` still replaces the old one.
- Plans approved before the threshold change still carry the old paces: ask
  the coach to refresh them, or re-derive paces on approval.
- Check that the local model calls tools reliably (forced edits and
  `get_run_detail`); switch to qwen3:14b if not.

**Access**
- Second athlete (`ATHLETE=...`, own `.env` and data folder): set up, not
  tested yet.
- Phone access: Tailscale to the Mac, or a free cloud VM with Gemini.

---

## Reference

### Layout

```
src/
  paths.py                  all paths; ATHLETE switch; loads .env
  pipeline.py               refresh()
  ingestion/  strava_client.py · sync.py · weather.py · forecast.py (future weather)
  features/   adjusted_pace.py · best_efforts.py · threshold.py · classify.py · zones.py · run_detail.py
  models/
    pace_model/             per-run features
    predictions/race.py     k and race times
    fitness/race_predictions.py  race-day fitness forecast
  agent/
    graph.py                LangGraph: intent check, forced tools, memory
    prompt.py               the coach's instructions
    context.py              snapshot, phase, week text
    tools.py                the 17 tools
    planner.py              planner subagent + check_rules
    store.py                facts, plans, proposals, goal race
    llm.py                  Gemini / Ollama
    knowledge/training_guidelines.md
  ui/app.py                 Streamlit
docs/architecture.png
docs/demo1.gif, demo2.gif
.env.example
```

### Tables (DuckDB)

| Table | Contents |
|---|---|
| `activities` | one row per run |
| `runs_weather`, `runs_adjusted` | conditions; pace corrected to neutral |
| `best_efforts`, `effort_windows` | fastest stretch per distance; fastest 7-min window per run |
| `threshold` | threshold pace and HR, one row per week plus today |
| `run_labels`, `pace_features` | session labels; per-run features |
| `decay`, `race_predictions` | k per week; predicted times |
| `fitness_forecast` | the forecast change, its range, method, horizon |
| `goal_races` | the goal race |
| `training_plans` | sessions: planned / done / skipped / moved / draft, `forced`, `proposal_id` |
| `plan_proposals` | proposals and their status |
| `athlete_facts` | things the coach remembers (injuries, preferences) |

### Several athletes

Set `ATHLETE` to give each person their own data folder and `.env`:

```bash
# .env.marco holds Marco's STRAVA_* keys (his refresh token)
ATHLETE=marco python get_new_refresh_token.py
ATHLETE=marco streamlit run src/ui/app.py --server.port 8502
```

His data goes to `data/marco/`. Leave `ATHLETE` unset to keep the original
`data/` layout. A Strava app allows one athlete until you raise its capacity in
the Strava API settings.

### Fresh start for the chat

Stop the app, then delete the chat database **and its two side files**
(leaving them behind gives `disk I/O error` on the next start):

```bash
rm -f data/chat.sqlite data/chat.sqlite-wal data/chat.sqlite-shm
```

For a second athlete, the same three files under `data/<athlete>/`.
Plans, the goal race and remembered facts are in DuckDB and stay.

### References

- Ely, Cheuvront, Roberts, Montain (2007). Impact of weather on
  marathon-running performance. *Med Sci Sports Exerc* 39(3).
- Riegel (1977). Time predicting. *Runner's World*.
- Blythe, Király (2016). Prediction and quantification of individual athletic
  performance of runners. *PLOS ONE* 11(6).
- Jones et al. (2024). Fixed intensity anchors to estimate lactate thresholds
  in recreational runners. *Eur J Appl Physiol*.
- Friel. *The Triathlete's Training Bible*: HR zones from LTHR.