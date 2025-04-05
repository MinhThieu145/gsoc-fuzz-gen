# API to retrieve results from custom fuzzing runs

## Current Implementation

The OSS-Fuzz-Gen system currently collects and processes fuzzing results in the following ways:

### Result Collection
The `ExecutionStage` class runs fuzz targets and collects results:

```python
# From stage/execution_stage.py
class ExecutionStage(BaseStage):
    """Executes fuzz targets and build scripts. This stage takes a fuzz target
    and its build script, runs them locally or on the cloud with OSS-Fuzz infra,
    and outputs code coverage report and run-time crash information."""
```

This class is responsible for:
- Running targets locally or in the cloud
- Managing the execution pipeline
- Collecting real-time results

### Result Storage
Results are stored in the `RunResult` class:

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

### Evaluation Process
The `Evaluator` class processes and analyzes the results:

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

## Proposed API for Result Retrieval

We'll create a more straightforward API for retrieving fuzzing results.

### 1. Result Data Structure

```python
@dataclass
class FuzzingResult:
    """Results from a completed fuzzing run."""
    submission_id: str                  # Unique identifier
    state: str                          # Current state (running, completed, error)
    
    # Build information
    build_success: bool                 # Whether build succeeded
    build_log_path: str                 # Path to build logs
    build_errors: List[str]             # Any build errors encountered
    
    # Coverage metrics
    coverage_percent: float             # Overall coverage percentage
    covered_pcs: int                    # Program counters covered
    total_pcs: int                      # Total program counters
    covered_lines: Dict[str, List[int]] # Covered lines by file
    
    # Crash information
    crashes_found: bool                 # Whether crashes were discovered
    crash_details: List[Dict]           # Details about each crash
    reproducers: List[str]              # Paths to crash reproducers
    
    # Artifacts
    corpus_path: str                    # Path to generated corpus
    log_path: str                       # Path to execution logs
    fuzz_target_path: str               # Path to the fuzz target source
    
    error: Optional[str] = None         # Error message if something failed
```

### 2. Status Tracking

```python
def get_fuzzing_status(submission_id: str) -> Dict[str, Any]:
    """Get current status of a fuzzing job.
    
    Args:
        submission_id: ID of the submitted fuzzing job
        
    Returns:
        Dictionary with current status information
    """
```

This function:
- Checks if the submission exists
- Reads the latest status information
- Returns a consistent status format
- Handles both local and cloud jobs

### 3. Result Retrieval

```python
def get_fuzzing_results(submission_id: str) -> FuzzingResult:
    """Get complete results for a finished fuzzing job.
    
    Args:
        submission_id: ID of the submitted fuzzing job
        
    Returns:
        FuzzingResult object with all result data
        
    Raises:
        ValueError: If job is not complete or doesn't exist
    """
```

This function:
- Verifies the job is complete
- Collects all result data (coverage, crashes, logs)
- Formats into the standard result structure
- Works for both local and cloud jobs

### 4. Source Code Access

```python
def get_fuzz_target_source(submission_id: str, target_name: str) -> str:
    """Get source code for a generated fuzz target.
    
    Args:
        submission_id: ID of the submitted fuzzing job
        target_name: Name of the specific target
        
    Returns:
        Source code as a string
    """
```

This allows:
- Examining the generated target
- Debugging execution issues
- Manually modifying targets

## Implementation Plan

1. Create consistent result data structures
2. Implement status checking for both local and cloud jobs
3. Add comprehensive result collection
4. Build source code access utilities
5. Create a simple client library

This API will provide a clean interface to access the detailed coverage and crash information that's already being collected, making it easier to analyze fuzzing performance without dealing with internal implementation details.
