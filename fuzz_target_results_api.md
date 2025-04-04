## 1. Current Workflow Summary

The current workflow for retrieving fuzz target results in OSS-Fuzz-Gen operates as follows:

1. **Result Collection**: The `ExecutionStage` class handles running fuzz targets and collecting results:
```python
# From stage/execution_stage.py
class ExecutionStage(BaseStage):
    """Executes fuzz targets and build scripts. This stage takes a fuzz target
    and its build script, runs them locally or on the cloud with OSS-Fuzz infra,
    and outputs code coverage report and run-time crash information."""
```

**Explanation:**
- This is the main entry point for running fuzz targets
- It can run targets either locally or in the cloud
- It manages the entire execution pipeline
- Results are collected in real-time during execution

2. **Result Storage**: Results are stored in structured classes that capture different aspects:
```python
# From results.py
class RunResult(BuildResult):
    """The fuzzing run-time result info."""
    crashes: bool
    run_error: str
    run_log: str
    coverage_summary: dict
    coverage: float
    line_coverage_diff: float
    reproducer_path: str
    textcov_diff: Optional[textcov.Textcov]
    log_path: str
    corpus_path: str
    coverage_report_path: str
    cov_pcs: int
    total_pcs: int
```

**Explanation:**
- `crashes`: Tracks if the target caused any crashes
- `coverage`: Stores code coverage metrics
- `line_coverage_diff`: Shows new lines covered
- `reproducer_path`: Path to crash test cases
- `cov_pcs` and `total_pcs`: Program counter coverage stats
- `textcov_diff`: Detailed line-by-line coverage info

3. **Evaluation Process**: The `Evaluator` class processes and analyzes results:
```python
# From experiment/evaluator.py
def check_target(self, ai_binary, target_path: str) -> Result:
    """Builds and runs a target."""
    # Calculate coverage percentage
    if run_result.total_pcs:
        coverage_percent = run_result.cov_pcs / run_result.total_pcs
    else:
        coverage_percent = 0.0

    # Track crashes
    run_result.triage = self.triage_crash(
        ai_binary,
        generated_oss_fuzz_project,
        target_path,
        run_result,
        dual_logger,
    )
```

**Explanation:**
- Calculates coverage percentage from program counters
- Tracks and analyzes crashes
- Provides detailed crash information
- Logs execution details for debugging

**Current Limitations:**
1. No easy way to track submission status
2. Results are scattered across different files
3. No simple API to access results
4. Requires understanding internal data structures
5. Manual file navigation needed to find results

This workflow is powerful but requires direct access to the codebase and knowledge of internal data structures.

## 2. New Workflow Design

The new workflow for retrieving fuzz target results will be simpler:

1. **API Interface**: Users interact with a clean Python API to get results
2. **Status Tracking**: Results are tracked with unique submission IDs
3. **Asynchronous Access**: Users can check status and get results at any time
4. **Structured Response**: Results are returned in a consistent, easy-to-use format

The workflow follows these steps:
1. Submit fuzz target (returns submission ID)
2. Check status using submission ID
3. Get results when complete
4. Access detailed data (coverage, crashes, etc.)

## 3. New Implementation Steps

### Step 1: Result Data Structure

```python
@dataclass
class SubmissionResult:
    """Results of a completed fuzz target."""
    submission_id: str
    state: str
    build_success_rate: float
    build_success_count: int
    max_coverage: float
    crash_rate: float
    best_target: str
    targets: List[str]
    error: Optional[str] = None
```

**Core Logic:**
- `submission_id`: Unique identifier for tracking the submission
- `state`: Current state (running, completed, error)
- `build_success_rate`: Percentage of successful builds
- `build_success_count`: Number of successful builds
- `max_coverage`: Highest code coverage achieved
- `crash_rate`: Percentage of runs that caused crashes
- `best_target`: Name of the target with best coverage
- `targets`: List of all generated targets
- `error`: Optional error message if something went wrong

**Usage:**
- Used to store all result data in one place
- Makes it easy to convert to/from JSON for API responses
- Provides a consistent format for all result data
- Includes error handling for failed submissions

### Step 2: Status Tracking

```python
def update_status(
    submission_id: str,
    submissions_dir: str,
    stage: str,
    progress: float,
    message: str = None,
    state: str = 'running'
) -> Dict[str, Any]:
```

**Core Logic:**
- `submission_id`: ID of the submission to update
- `submissions_dir`: Directory where submissions are stored
- `stage`: Current stage (e.g., 'building', 'running', 'analyzing')
- `progress`: Progress percentage (0.0 to 1.0)
- `message`: Optional status message
- `state`: Current state (running, completed, error)

**Usage:**
- Updates status file with new information
- Tracks progress and current stage
- Handles state transitions
- Provides real-time updates to users
- Stores status history for debugging

### Step 3: Result Retrieval

```python
def get_submission_results(submission_id: str) -> Dict[str, Any]:
```

**Core Logic:**
- Takes a submission ID
- Returns a dictionary with all result data

**Process:**
1. Check if submission exists
2. Verify submission is complete
3. Read result files from submission directory
4. Parse coverage and crash data
5. Format data for API response
6. Handle missing or incomplete results

**Error Handling:**
- Returns error message if submission not found
- Handles incomplete submissions
- Provides clear error messages
- Gracefully handles missing data

### Step 4: Target Code Access

```python
def get_target_code(submission_id: str, target_filename: str) -> str:
```

**Core Logic:**
- Takes submission ID and target filename
- Returns source code as string

**Process:**
1. Validate submission exists
2. Look for target in fixed_targets directory
3. If not found, check raw_targets directory
4. Read and return file contents
5. Handle file not found errors

**Features:**
- Supports both fixed and raw targets
- Provides clear error messages
- Handles file system errors
- Returns formatted source code

### Step 5: Error Handling

```python
def handle_submission_error(
    submission_id: str,
    error: Exception,
    stage: str
) -> Dict[str, Any]:
```

**Core Logic:**
- Takes submission ID, error object, and stage
- Returns updated status with error info

**Process:**
1. Record error details
2. Update submission status
3. Log error for debugging
4. Provide user-friendly error message
5. Update status file

**Features:**
- Detailed error tracking
- Stage-specific error handling
- Error recovery options
- Clear error reporting

