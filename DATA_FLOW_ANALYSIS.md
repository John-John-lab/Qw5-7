# Data Flow Analysis: Signal → Task → Tables

## Executive Summary

The application follows a **3-stage data pipeline**:
1. **Signal Parsing** → Creates `DownloadTask` objects
2. **Task Processing** → Enriches tasks with analysis results  
3. **Table Rendering** → Displays data from `golden_task_store_data`

---

## 1. DATA SOURCE HIERARCHY

### Primary Source: `golden_task_store_data` (Global Variable)
```python
# Line 349: Definition
golden_task_store_data = None

# Line 6320: Updated after task creation/recalculation
golden_task_store_data = list(tm.tasks.values())
```

**What it contains:** List of `DownloadTask` objects with ALL analysis results

**Where it's used:**
- Task Table (line 3741-3742)
- Summary Stats Table 1 (line 3513-3514)
- Summary Stats Table 2 (line 3741-3742, 4267-4268)

### Fallback Source: `tm.tasks` (TaskManager)
```python
# Line 2174: TaskManager class
class TaskManager:
    def __init__(self):
        self.tasks = {}  # task_id → DownloadTask
        self.lock = threading.Lock()
```

**When used:** If `golden_task_store_data` is None or empty

---

## 2. SIGNAL → TASK CREATION FLOW

### Trigger: "Create New Tasks from Signals" Button
**Callback:** `create_signal_tasks()` (line 3109)

#### Step-by-Step Process:

**A. Input Parameters:**
- `signals`: List of signal dicts from parsed JSON
- `period_type`: 'date' or 'hours'
- `start_date`, `end_date`: For date range mode
- `hours`: For hours-based mode
- `tf`: Timeframe (e.g., "1m", "5m")
- `ow`: Overwrite flag
- `beyond_val`: Analyze beyond period flag
- `strat_val`: Strategy enabled/disabled
- `imp_val`: Impulse enabled/disabled
- `pre_buffer`: Minutes before signal to include
- `event_log_val`: Enable event logging
- `hide_logs_val`: Hide logs in UI
- `autoclear_val`: Clear existing tasks first

**B. Task Creation Logic:**

For each signal:
```python
# Lines 3187-3219 (sync) / 3308-3340 (background)
symbol = sig['symbol']
signal_time = sig['time_ms']
signal_price = sig['price']
signal_direction = sig['direction']

# Calculate time range
if period_type == 'hours':
    pre_buffer_ms = pre_buf_min * 60 * 1000
    start_dt = datetime.fromtimestamp((signal_time - pre_buffer_ms) / 1000.0, tz=timezone.utc)
    end_dt = start_dt + timedelta(hours=hours)
else:  # date mode
    start_dt = datetime.fromisoformat(start_date)
    end_dt = datetime.fromisoformat(end_date)

# Create DownloadTask
task = DownloadTask(
    task_id=str(uuid.uuid4()),
    symbols=[symbol],
    timeframe=tf,
    mode='period',
    start_date=start_dt,
    end_date=end_dt,
    overwrite=ow_flag,
    signal_time=signal_time,
    signal_price=signal_price,
    signal_symbol=symbol,
    signal_direction=signal_direction,
    analyze_beyond=analyze_beyond,
    enable_strategy=not strat_disabled,
    enable_impulse=not imp_disabled,
    pre_buffer_minutes=int(pre_buffer),
    log_events=log_events,
    hide_logs=hide_logs
)
```

**C. Storage Location:**

**Small batches (<100 signals):**
```python
# Line 3220
tm.add_task(task)  # Adds to tm.tasks dict immediately
```

**Large batches (≥100 signals):**
```python
# Lines 3304, 3343, 3375-3379
local_tasks = {}  # Build locally first
local_tasks[tid] = task

# Atomic swap at end
with tm.lock:
    tm.tasks.update(local_tasks)

# Queue for processing
for task in local_tasks.values():
    tm.queue.put(task)
```

**D. Update Golden Store:**
```python
# Line 6320 (after all tasks created)
golden_task_store_data = list(tm.tasks.values())
```

---

## 3. TASK PROCESSING & ANALYSIS

### DownloadTask Attributes (Lines 602-698)

#### Initial State (from signal):
```python
self.signal_time           # ms timestamp
self.signal_price          # float
self.signal_symbol         # string
self.signal_direction      # 'resistance' or 'support'
self.analyze_beyond        # bool
self.enable_strategy       # bool
self.enable_impulse        # bool
```

#### Analysis Results (filled during processing):
```python
# First event detection
self.first_event_time
self.first_event_type
self.first_event_is_pin
self.first_event_close

# Price movement
self.price_change_pct
self.reached_level         # Did price hit target?
self.reversed_direction    # Did direction flip?

# Hit tracking (1%, 1.5%, 2% levels)
self.hit_1, self.hit_1_5, self.hit_2
self.first_hit_1_expected, self.first_hit_1_expected_time
self.first_hit_1_5_expected, self.first_hit_1_5_expected_time
self.first_hit_2_expected, self.first_hit_2_expected_time
self.first_hit_1_opposite, self.first_hit_1_opposite_time
# ... (opposite direction hits)

# Drawdown metrics
self.max_adverse_move_pct
self.max_adverse_time
self.max_expected_move_pct
self.max_expected_time
self.drawdown_before_level
self.drawdown_before_1pct, self.drawdown_before_1pct_time
self.drawdown_before_1_5pct, self.drawdown_before_1_5pct_time
self.drawdown_before_2pct, self.drawdown_before_2pct_time

# Return-to-signal metrics
self.returned_to_signal
self.max_adverse_before_return_pct
self.max_adverse_before_return_time

# Signal-based metrics (alternative calculation)
self.max_adverse_sgnl_pct
self.max_expected_sgnl_pct
self.returned_to_sgnl

# Strategy results
self.strategy_signals      # List of signal dicts
self.strategy_log_summary  # Text summary
self.strategy_confidence   # Float 0-100

# Impulse results
# (stored in strategy_signals with type='impulse')
```

---

## 4. TABLE RENDERING DATA FLOW

### A. Task Table (Main Grid)

**Callback:** `update_task_table_only()` (line 3702)

**Data Retrieval:**
```python
# Lines 3741-3747
if golden_task_store_data is not None and len(golden_task_store_data) > 0:
    tasks = golden_task_store_data
else:
    with tm.lock:
        tasks = list(tm.tasks.values())
```

**Row Rendering:** `render_task_table_row(t)` (line 3814)

**Columns Displayed (52 total):**
1. Task ID (first 8 chars)
2. Status
3. Progress %
4. Symbols
5. Mode
6. **Direction** ← Uses `direction_display` (MISSING!)
7. **Signal Time** ← Uses `signal_time_display` (MISSING!)
8. **First Event** ← Uses `first_event_display` (MISSING!)
9. **Pin Bar** ← Uses `pin_display` (MISSING!)
10. **Price Change** ← Uses `price_change_display` (MISSING!)
11. **Reached Level** ← Uses `reached_display` (MISSING!)
12. Reversed Direction
13-27. Hit flags and timestamps
28-43. Drawdown/adverse move metrics
44-47. Strategy summary, confidence, impulse count
48. Log display
49. Action buttons

**🔴 CRITICAL BUG (Line 3814-3975):**
The function uses 6 undefined variables:
```python
direction_display      # Should be: t.signal_direction or "-"
signal_time_display    # Should be: fmt_time_ui(t.signal_time)
first_event_display    # Should be: f"{t.first_event_type} @ {fmt_time_ui(t.first_event_time)}"
pin_display            # Should be: "Yes" if t.first_event_is_pin else "No"
price_change_display   # Should be: fmt_dd_ui(t.price_change_pct)
reached_display        # Should be: "Yes" if t.reached_level else "No"
```

**Fix Required:** Add these definitions at line 3817, after the existing display variables.

---

### B. Summary Statistics Table 1

**Callback:** `update_summary_stats_only()` (line 3476)

**Data Retrieval:**
```python
# Lines 3498-3517
tasks = golden_task_store_data if golden_task_store_data else (list(tm.tasks.values()) if hasattr(tm, 'tasks') else [])

if golden_task_store_data is not None and len(golden_task_store_data) > 0:
    tasks = golden_task_store_data
else:
    with tm.lock:
        tasks = list(tm.tasks.values())
```

**Statistics Calculated (Lines 3519-3683):**
- Total tasks, completed, failed
- Hit rates (1%, 1.5%, 2%)
- Average/max/min price change
- Average/max adverse move
- Return-to-signal rate
- Strategy confidence distribution
- Impulse signal counts

**🔴 ISSUE: Duplicate Calculation**
Same statistics are calculated again in `render_signal_stats_table()` (line 4069).

---

### C. Summary Statistics Table 2 (Signal Stats)

**Callback:** Part of `update_task_table_only()` via `render_signal_stats_table()` (line 4069)

**Data Retrieval:** Same as Task Table (lines 4267-4272)

**Statistics Calculated (Lines 4089-4222):**
- Identical to Summary Table 1
- Grouped by direction (resistance/support)
- Grouped by timeframe

---

## 5. BUTTON ACTIONS & CALLBACKS

### "Create Tasks from Signals" Button
**Callback:** `create_signal_tasks()` (line 3109)
**Inputs:** Signals from store, user settings
**Outputs:** Updates `tm.tasks`, then `golden_task_store_data`

### "Recalculate" Button
**Callback:** Triggers re-analysis of existing tasks
**Process:**
1. Sets lock state
2. Re-runs `analyze_signal()` on each task
3. Updates `golden_task_store_data`
4. Triggers `analysis-complete-trigger`

### "Load Selected" Button
**Callback:** Loads specific tasks from database
**Process:**
1. Queries database for task IDs
2. Creates `DownloadTask` objects
3. Adds to `tm.tasks`
4. Updates `golden_task_store_data`

### Task Action Buttons (per row):
- **Stop:** Sets `task.stop_event`
- **Pause/Resume:** Toggles `task.pause_event`
- **Chart:** Opens modal with Plotly chart
- **Details:** Opens modal with full task info
- **Impulse:** Opens impulse analysis modal
- **Re-run Strategy:** Re-executes strategy logic
- **Re-run Impulse:** Re-executes impulse detection
- **TV:** Opens TradingView link

---

## 6. WEAK POINTS & TROUBLE SPOTS

### 🔴 Critical Issues:

1. **Broken Task Table Row (Line 3814)**
   - 6 undefined variables cause crash
   - Table won't render without fix

2. **Duplicate Statistics Calculation**
   - `update_summary_stats_only()` (line 3476)
   - `render_signal_stats_table()` (line 4069)
   - Same logic, different locations
   - Risk of inconsistency

3. **Business Logic Mixed with UI**
   - Heavy calculations inside `update_summary_stats_only()`
   - Should be in separate module
   - Makes testing difficult

4. **Thread Safety Workarounds**
   - `np_local_global`, `bisect_local_global` (lines 107-108)
   - Indicates import issues in background threads
   - Potential race conditions

5. **Global State Dependencies**
   - `golden_task_store_data` accessed everywhere
   - Hard to track when/where it changes
   - Risk of stale data

### 🟡 Moderate Issues:

6. **Callback Spaghetti**
   - 50+ callbacks without clear grouping
   - Difficult to trace data flow
   - High cognitive load

7. **Large Callback Functions**
   - `update_task_table_only()`: 100+ lines
   - `update_summary_stats_only()`: 200+ lines
   - Should be broken into smaller helpers

8. **Missing Error Handling**
   - No try/catch in `render_task_table_row()`
   - Single bad task could crash entire table

9. **Inconsistent Data Access Patterns**
   - Sometimes uses `golden_task_store_data`
   - Sometimes uses `tm.tasks`
   - Logic for choosing is duplicated

10. **Performance Bottlenecks**
    - Full stats recalculation on every page change
    - No memoization for expensive calculations
    - Could benefit from caching layer

---

## 7. RECOMMENDED REFACTORING PLAN

### Phase 1: Fix Critical Bugs (Immediate)
1. **Fix `render_task_table_row()`** - Add missing variable definitions
2. **Extract `calculate_signal_statistics()`** - Remove duplication
3. **Update both callbacks** - Use extracted function

### Phase 2: Separate Logic from UI
4. **Create `calculations.py` module** - Move all business logic
5. **Create `ui_helpers.py` module** - Move all rendering logic
6. **Update imports** - Point to new modules

### Phase 3: Improve Data Flow
7. **Add intermediate stats store** - Cache calculated statistics
8. **Unify data access pattern** - Single source of truth
9. **Add data validation layer** - Ensure data integrity

### Phase 4: Thread Safety & Performance
10. **Fix numpy/bisect imports** - Remove workarounds
11. **Add proper locking** - Protect shared state
12. **Implement caching** - Reduce redundant calculations

---

## 8. KEY TAKEAWAYS

**Data Source:** `golden_task_store_data` → List of enriched `DownloadTask` objects

**Creation Path:** Signals → `create_signal_tasks()` → `DownloadTask` → `tm.tasks` → `golden_task_store_data`

**Rendering Path:** `golden_task_store_data` → Callbacks → HTML tables

**Critical Fix Needed:** Add 6 missing variable definitions in `render_task_table_row()` before any other refactoring

**Biggest Architectural Issue:** Business logic embedded in UI callbacks makes maintenance difficult
