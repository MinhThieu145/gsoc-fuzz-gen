# API to retrieve results from custom fuzzing runs

## Current Implementation

The OSS-Fuzz-Gen system collects and tracks fuzzing metrics through several components:

### Result Collection and Processing

The `ExecutionStage` class runs fuzz targets and collects results:

```python
# From stage/execution_stage.py
class ExecutionStage(BaseStage):
    """Executes fuzz targets and build scripts. This stage takes a fuzz target
    and its build script, runs them locally or on the cloud with OSS-Fuzz infra,
    and outputs code coverage report and run-time crash information."""
```

This class orchestrates:
- Running targets locally or in the cloud
- Collecting fuzzing output logs
- Processing metrics from libFuzzer

### Core Metrics Tracked

Results are stored in the `RunResult` class with these key metrics:

```python
# From results.py
class RunResult(BuildResult):
    """The fuzzing run-time result info."""
    crashes: bool                       # Whether crashes were found
    run_error: str                      # Error details for run failures
    run_log: str                        # Complete fuzzing log
    coverage_summary: dict              # Detailed coverage information
    coverage: float                     # Overall coverage percentage
    line_coverage_diff: float           # New lines covered
    reproducer_path: str                # Path to crash test cases
    textcov_diff: Optional[textcov.Textcov]  # Line-by-line coverage
    log_path: str                       # Path to execution logs
    corpus_path: str                    # Path to corpus directory
    coverage_report_path: str           # Path to full coverage report
    cov_pcs: int                        # Program counters covered
    total_pcs: int                      # Total program counters
```

#### Coverage Metrics

The system tracks two different types of coverage:

1. **Edge Coverage** (`cov_pcs` and `total_pcs`):
   - Tracks which control-flow transitions were executed
   - `cov_pcs`: Number of edges (transitions between code blocks) executed
   - `total_pcs`: Total possible edges in the program
   - Extracted from LibFuzzer output lines like `#2 INITED cov: 1423 ft: 1423 corp: 5/10Kb`

2. **Line Coverage** (`coverage` and `coverage_summary`):
   - Maps directly to source code lines
   - More detailed than edge coverage
   - Includes file-level breakdown of covered lines
   - Stored as detailed line-by-line data in `textcov_diff`

#### Crash Detection

The system identifies and analyzes crashes through:

1. **Crash Detection** (`crashes` and `crash_info`):
   - `crashes`: Boolean flag indicating if any crashes were detected
   - `crash_info`: Detailed information about the crash (type, location, stack trace)
   - Extracted from sanitizer output in fuzzer logs

2. **Semantic Crash Analysis** (`semantic_check`):
   - Determines if a crash is a genuine bug or false positive
   - Categorizes crashes by type (NULL_DEREF, SIGNAL, OOM, etc.)
   - Helps filter out crashes in the fuzz harness vs actual bugs

#### Artifact Paths

The system also tracks file paths to detailed results:

- `log_path`: Full fuzzing execution logs
- `corpus_path`: Input files that trigger interesting behavior
- `reproducer_path`: Files that can reproduce any crashes
- `coverage_report_path`: Detailed coverage reports

### Metric Extraction Process

The key metrics are extracted via log parsing:

```python
# From experiment/builder_runner.py
def _parse_libfuzzer_logs(self, log_handle, project_name: str, check_cov_increase: bool = True):
    # ... code to parse log files ...
    for line in lines:
        # Extract coverage PCs
        m = LIBFUZZER_COV_REGEX.match(line)
        if m:
            cov_pcs = int(m.group(1))
            continue
            
        # Extract total PCs
        m = LIBFUZZER_MODULES_LOADED_REGEX.match(line)
        if m:
            total_pcs = int(m.group(2))
            continue
            
        # Detect crashes
        m = LIBFUZZER_CRASH_TYPE_REGEX.match(line)
        if m and not CRASH_EXCLUSIONS.match(line):
            crashes = True
            continue
```

This data is then used by the `Evaluator` to calculate final metrics like coverage percentage:

```python
# From experiment/evaluator.py
def check_target(self, ai_binary, target_path: str) -> Result:
    """Builds and runs a target."""
    # Calculate coverage percentage
    if run_result.total_pcs:
        coverage_percent = run_result.cov_pcs / run_result.total_pcs
    else:
        coverage_percent = 0.0

    # Analyze crashes
    run_result.triage = self.triage_crash(
        ai_binary,
        generated_oss_fuzz_project,
        target_path,
        run_result,
        dual_logger,
    )
```

## New Workflow

We'll create a more straightforward API for retrieving fuzzing results.

### The Cloud Jobs Problem

Right now, cloud jobs block until they finish:
```python
# From experiment/builder_runner.py
def build_and_run_cloud(self, generated_project, target_path, ...):
    # This code waits for the job to complete
    if not self._run_with_retry_control(...):
      return build_result, None
```

The monitoring system doc already outlines a solution for this in section 3.3 "Cloud Job Workflow":
- It proposes a `submit_cloud_job()` function that uses the `--async` flag
- It includes a `check_cloud_job_status()` function for non-blocking status checks
- It downloads results only when needed

We'll directly adopt this approach from monitoring_system.md rather than creating our own solution.

### The Complex Data Problem

The current data structure is too complicated:
- Results are spread across many nested objects
- Some fields are only in cloud jobs, others only in local jobs
- You need to know internal details to get simple metrics

For example, to get coverage percentage you need to:
1. Check if total_pcs exists
2. Divide cov_pcs by total_pcs
3. Handle different formats for local vs. cloud

### Our Fix

We'll create a simpler way to get results:

1. Make a single function that works with both local and cloud jobs
2. Return just the useful data (coverage, crashes) in a simple format
3. Hide all the complex internal stuff

We'll take what's already in the status files (section 3.2.3 of monitoring_system.md):
```json
"run_result": {
  "crashes": false,
  "coverage": 0.75,
  "cov_pcs": 150,
  "total_pcs": 200
}
```

And make a clean function to get it:
```python
def get_fuzzing_results(submission_id):
    # Returns a simple dict with just what you need
```

### Implementation Steps

1. Use the monitoring system's status tracking
2. Add a simple results function that:
   - Finds the right status file
   - Extracts just the important metrics
   - Returns them in a consistent format

3. Make sure it works with both local and cloud jobs

When we're done, you'll be able to:
- Run a fuzzing job and get an ID
- Check progress at any time
- Get simple, easy-to-use results

## Detailed Implementation Plan

### Step 1: Adopt Non-blocking Cloud Job Handling from Monitoring System

For this step, we'll directly implement the approach described in monitoring_system.md section 3.3 "Cloud Job Workflow":

```python
def submit_cloud_job(project: str, function: Dict[str, Any], options: Dict[str, Any] = None) -> str:
    """Submits a job to Google Cloud Build without waiting.
    
    Returns:
        submission_id to track the job
    """
```

As described in monitoring_system.md, this function will:
- Create a unique ID for the job
- Set up Google Cloud Storage paths for logs and results
- Submit the job with the `--async` flag
- Store the job ID in the status file
- Return immediately with the job ID

Similarly, we'll implement the status checking function from monitoring_system.md:

```python
def check_cloud_job_status(job_id: str) -> Dict[str, Any]:
    """Checks the status of a cloud job without blocking.
    
    Returns:
        Status information including progress and state
    """
```

This matches the approach in monitoring_system.md section 3.3 to:
- Query the Google Cloud Build API to get current status
- Return a simple status dictionary
- Not block while waiting for completion

### Step 2: Create a Simple Results API

Building on the unified job status approach from monitoring_system.md section 3.4, we'll add a function to retrieve results:

```python
def get_fuzzing_results(self, submission_id: str) -> Dict[str, Any]:
    """Gets simplified results from a fuzzing run.
    
    Args:
        submission_id: ID of the fuzzing submission
        
    Returns:
        Dictionary with coverage and crash information
    """
```

This function will:
- First check if the submission exists using the status tracking from monitoring_system.md
- Read the status file directly from disk (following the format in section 3.2)
- Extract just the important metrics (coverage, crashes)
- Return them in a simple, consistent dictionary format
- Work for both local and cloud jobs

The returned dictionary will have this structure:
- `coverage`: Information about code coverage
  - `percentage`: Overall coverage as a float
  - `edges_covered`: Number of edges covered (cov_pcs)
  - `total_edges`: Total possible edges (total_pcs)
- `crashes`: Information about crashes
  - `found`: Boolean indicating if crashes were found
  - `types`: List of crash types detected
  - `reproducer_path`: Path to files that reproduce crashes
- `files`: Paths to various artifacts
  - `log`: Path to fuzzing logs
  - `corpus`: Path to corpus files
  - `coverage_report`: Path to detailed coverage report

### Step 3: Add Helper Functions

We'll add helper functions to process the complex data:

```python
def _extract_crash_types(self, run_result: Dict) -> List[str]:
    """Extracts types of crashes from run result data.
    
    Returns:
        List of crash type strings
    """
```

This function will:
- Check if crashes were found at all
- Parse the crash_info string to identify common crash types
- Return a list of crash types as strings
- Handle cases where crash types can't be identified

### Step 4: Update Status File Format

We need to ensure the status files include coverage and crash information, integrating with the format described in monitoring_system.md section 3.2:

```python
def update_status_with_results(status_file: str, run_result: RunResult) -> None:
    """Updates a status file with coverage and crash information.
    
    Args:
        status_file: Path to the status file
        run_result: RunResult object with metrics
    """
```

This function will:
- Read the existing status file
- Add coverage metrics (coverage, cov_pcs, total_pcs)
- Add crash information (crashes, crash_info)
- Add paths to relevant files (logs, corpus, etc.)
- Write the updated status back to disk

### Step 5: Create a Unified Status Interface

Following the unified approach in monitoring_system.md section 3.4, we'll implement:

```python
def get_job_status(submission_id: str) -> Dict[str, Any]:
    """Gets status for any fuzzing job (local or cloud).
    
    Args:
        submission_id: The submission ID
        
    Returns:
        Dictionary with status information
    """
```

This directly implements the function described in monitoring_system.md to:
- Determine if the job is local or cloud-based
- Read the appropriate status information
- Return a consistent format regardless of job type
- Include current progress and state

With these functions, researchers will be able to:
1. Submit a fuzzing job
2. Check its status at any time without waiting for completion
3. Get coverage and crash information in a simple format
4. Access detailed artifacts when needed

This implementation builds directly on the monitoring system approach while adding our specific enhancements for retrieving coverage and crash data.
