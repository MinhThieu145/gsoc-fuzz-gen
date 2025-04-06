## 1. Current Workflow Summary

The current OSS-Fuzz-Gen system tracks fuzzing jobs in two different ways, depending on where the job runs: locally or in the cloud.

### 1.1 Local Job Tracking

For jobs that run on the local machine:

1. **Directory Setup**: First, a working directory is created for each job:

```python
# From experiment/workdir.py
class WorkDirs:
    def __init__(self, base_dir, keep: bool = False):
        self._base_dir = os.path.realpath(base_dir)
        # Create various directories including status
        os.makedirs(self.status, exist_ok=True)
        # ...other directories...
    
    @property
    def status(self):
        return os.path.join(self._base_dir, 'status')
```

2. **Status Files**: The system creates JSON files in the status directory to track progress:

```python
# From pipeline.py
def _update_status(self, result_history: list[Result]) -> None:
    trial_result = TrialResult(benchmark=result_history[-1].benchmark,
                           trial=self.trial,
                           work_dirs=result_history[-1].work_dirs,
                           result_history=result_history)
    # Write to status file
    self.logger.write_result(
        result_status_dir=trial_result.best_result.work_dirs.status,
        result=trial_result)
```

3. **Progress Updates**: Each stage of the pipeline updates these status files:

```python
# From pipeline.py
def _execute_one_cycle(self, result_history: list[Result], cycle_count: int) -> None:
    # Run Writing Stage
    result_history.append(
        self.writing_stage.execute(result_history=result_history))
    self._update_status(result_history=result_history)  # Update status after writing
    
    # Skip remaining stages if build failed
    if (not isinstance(result_history[-1], BuildResult) or 
        not result_history[-1].success):
        self.logger.warning('Build failure, skipping remaining steps')
        return

    # Run Execution Stage
    result_history.append(
        self.execution_stage.execute(result_history=result_history))
    self._update_status(result_history=result_history)  # Update status after execution
    
    # Skip analysis if execution failed
    if (not isinstance(result_history[-1], RunResult) or 
        not result_history[-1].log_path):
        self.logger.warning('Run failure, skipping analysis')
        return

    # Run Analysis Stage
    result_history.append(
        self.analysis_stage.execute(result_history=result_history))
    self._update_status(result_history=result_history)  # Update status after analysis
```

4. **Status Checking**: To check status, you need to read these files from disk:

```python
# From module_for_researcher/define_and_submit_custom_fuzz_targets/status.py
def get_status(submission_id, submissions_dir):
    status_file = os.path.join(submissions_dir, submission_id, 'status.json')
    if os.path.exists(status_file):
        with open(status_file, 'r') as f:
            return json.load(f)
    # ...handle other cases...
```

### 1.2 Cloud Job Tracking

For jobs that run in Google Cloud:

1. **Job Submission**: When a cloud job starts, it gets a unique ID and paths in Google Cloud Storage:

```python
# From experiment/builder_runner.py
def build_and_run_cloud(self, generated_project, target_path, ...):
    # Generate unique ID
    uid = self.experiment_name + str(uuid.uuid4())
    
    # Set up storage paths in Google Cloud
    run_log_path = f'gs://{self.experiment_bucket}/{uid}.run.log'
    build_log_path = f'gs://{self.experiment_bucket}/{uid}.build.log'
    
    # Submit to Google Cloud Build
    command = [
        # Command to run the fuzzing job
    ]
```

2. **Blocking Execution**: The code waits for the cloud job to complete:

```python
# From experiment/builder_runner.py
def build_and_run_cloud(self, generated_project, target_path, ...):
    # ...setup code...
    
    # Wait for job to complete (blocking)
    result = self._run_with_retry_control(target_path, ...)
    
    # After completion, download results
    # ...download code...
```

3. **Results Retrieval**: Once the job finishes, results are downloaded from cloud storage:

```python
# From experiment/builder_runner.py
def build_and_run_cloud(self, generated_project, target_path, ...):
    # ...job execution...
    
    # Download results
    storage_client = storage.Client()
    bucket = storage_client.bucket(self.experiment_bucket)
    
    # Download logs, corpus, coverage data
    with open(run_log_path, 'wb') as f:
        blob = bucket.blob(run_log_name)
        if blob.exists():
            blob.download_to_file(f)
```

### 1.3 Key Differences Between Local and Cloud Tracking

There are two major differences in how local and cloud jobs are tracked:

1. **Blocking vs. Non-blocking**:
   - **Local jobs** update status files throughout execution - you can check progress at any time
   - **Cloud jobs** block until completion - there's no way to check partial progress

2. **Storage Location**:
   - **Local jobs** store status and results in local files
   - **Cloud jobs** store logs and results in Google Cloud Storage, only downloading them after completion

These differences make it hard to monitor cloud jobs effectively, since you can't check their status until they finish. The current system doesn't provide a unified way to track both types of jobs.

## 2. New Monitoring System Design

The current monitoring system has different problems depending on whether we're running jobs locally or in the cloud. Let's look at what we could fix for each one.

### For Local Jobs

Our current local job monitoring is pretty basic - we just write status to files. Here are some ideas to make it better:

1. **Historical Status Tracking**: We could keep old versions of status files, kind of like Git does with commits. This would let us see how a job progressed over time.

   But honestly, I feel like this isn't a great idea. It would use up disk space for not much benefit. Plus, implementing this right would be more complicated than it's worth.

2. **Simpler Status Format**: Right now our status files contain a lot of nested data that's hard to parse. We could simplify this to make it easier to read and process (as I notice we expand and write upon the status file as we proceed)

### For Cloud Jobs

Cloud job monitoring has bigger problems that actually need fixing:

**Non-blocking Status Checks**: Currently, our code just waits for cloud jobs to finish. Instead, we should:
   - Submit the job to Google Cloud Build
   - Return immediately with the cloud job ID
   - Provide a way to check status later using that ID



## 3. New Workflow

The new monitoring system needs to handle two different types of fuzzing jobs:
- **Local jobs** that run on the same machine
- **Cloud jobs** that run on Google Cloud Build

The workflow for each is different, so I'll split them up below.

### 3.1 Shared Setup Functions

#### Initialize Monitoring System

```python
def initialize_monitoring(base_dir: str) -> MonitoringSystem:
    """Creates and initializes the monitoring system."""
```

This function sets up everything needed for monitoring:
- Creates directories for storing monitoring data
- Sets up a simple database for tracking jobs
- Prepares logging configuration

### 3.2 Local Job Workflow

Local jobs run directly on the machine and write status files during execution.

#### 1. Submit Local Job

```python
def submit_local_job(
    project: str,
    function: Dict[str, Any],
    options: Dict[str, Any] = None
) -> str:
    """Submits a local fuzzing job.
    
    Returns:
        submission_id to track the job
    """
```

This function:
- Generates a unique submission ID
- Creates working directories
- Initializes the status file
- Starts the fuzzing pipeline in the current process

#### 2. Update Local Status

```python
def update_local_status(
    submission_id: str,
    stage: str,
    progress: float,
    message: str = None,
    state: str = 'running'
) -> Dict[str, Any]:
    """Updates the status of a local job."""
```

This function:
- Writes the new status to the job's status file
- Called automatically by the pipeline after each stage

#### 3. Get Local Results

```python
def get_local_job_results(submission_id: str) -> Dict[str, Any]:
    """Gets results for a completed local job."""
```

This function:
- Reads the final status and result files
- Returns coverage, crashes, and other metrics

### 3.3 Cloud Job Workflow

Cloud jobs run on Google Cloud Build and need special handling to check status.

#### 1. Submit Cloud Job

```python
def submit_cloud_job(
    project: str,
    function: Dict[str, Any],
    options: Dict[str, Any] = None
) -> str:
    """Submits a job to Google Cloud Build without waiting.
    
    Returns:
        submission_id to track the job
    """
```

This function:
- Generates a unique submission ID
- Creates Google Cloud Storage paths for logs and results
- **Submits the job using the --async flag so it returns immediately**
- Stores the Cloud Build ID for later status checking
- Returns the submission ID without waiting for the job to complete

#### 2. Check Cloud Job Status

```python
def check_cloud_job_status(submission_id: str) -> Dict[str, Any]:
    """Checks current status of a cloud job without blocking."""
```

This function:
- Looks up the Cloud Build ID associated with this submission
- Calls the Google Cloud Build API to check current status
- Translates Cloud Build status to our format
- Returns status without waiting for job completion

#### 3. Get Cloud Job Results

```python
def get_cloud_job_results(submission_id: str) -> Dict[str, Any]:
    """Downloads and processes results from a completed cloud job."""
```

This function:
- Checks if the cloud job is complete
- Downloads logs and artifacts from Google Cloud Storage
- Processes the results into the same format as local jobs
- Returns coverage, crashes, and other metrics

### 3.4 Unified Interface

These functions work for both local and cloud jobs by automatically detecting the job type.

#### Get Job Status

```python
def get_job_status(submission_id: str) -> Dict[str, Any]:
    """Gets status for any job (local or cloud)."""
```

This function:
- Determines if the job is local or cloud-based
- For local jobs: reads the status file
- For cloud jobs: calls check_cloud_job_status()
- Returns a consistent status format either way

#### Get Job Results

```python
def get_job_results(submission_id: str) -> Dict[str, Any]:
    """Gets results for any completed job (local or cloud)."""
```

This function:
- Determines if the job is local or cloud-based
- For local jobs: calls get_local_job_results()
- For cloud jobs: calls get_cloud_job_results()
- Returns a consistent results format either way

#### Handle Error

```python
def handle_job_error(
    submission_id: str,
    error: Exception,
    stage: str
) -> Dict[str, Any]:
    """Handles errors for any job (local or cloud)."""
```

This function:
- Records error details in the status system
- For cloud jobs: might attempt to cancel the cloud job
- Updates status to indicate failure

### 3.5 Key Differences in Implementation

The main differences between local and cloud monitoring are:

1. **Job Submission**
   - Local: Runs in same process, blocking until complete
   - Cloud: Submits with --async flag, returns immediately

2. **Status Checking**
   - Local: Read directly from status files
   - Cloud: Query Google Cloud Build API for current status

3. **Result Collection**
   - Local: Available directly in local files
   - Cloud: Must be downloaded from Google Cloud Storage

This approach gives us non-blocking cloud job monitoring while maintaining a consistent interface for users.

## 3.1 Monitoring System Flow Diagram

Here's how data flows through our monitoring system:

```
[OssFuzzApi] → creates → [Unique Submission ID]
                            ↓
[WorkDirs] → creates → [Directory Structure] 
                           ↓
                      [status.json] ← updated by ← [update_status()]
                           ↓
                  accessed by Pipeline
                           ↓
      ┌──────────┬─────────┴──────────┐
      ↓          ↓                    ↓
[Writing Stage] [Execution Stage] [Analysis Stage]
      ↓          ↓                    ↓
  [Results]   [Results]            [Results]
      └──────────┴─────────┬──────────┘
                           ↓
                  [Pipeline._update_status()]
                           ↓
                     [Logger.write_result()]
                           ↓
                      [status.json]
                           ↓
         accessed by [get_status()] & [get_submission_results()]
                           ↓
                     [OssFuzzApi] → returns to → [User]
```

This diagram shows how everything connects. Think of status files as the central hub of information - they let different parts of the system talk to each other without waiting for each other to finish. The submission ID works like a tracking number that follows your fuzzing job through the entire process.

## 3.2 What's in the Status Files: A Stage-by-Stage Look

Let's look at what information gets stored at each step of the process, along with the actual code that makes it happen.

### 3.2.1 Starting a Fuzzing Job

When someone submits a new fuzzing job, we create the first status file:

```python
# From module_for_researcher/define_and_submit_custom_fuzz_targets/api.py
def create_fuzz_target(self, project, function, options=None):
    # Generate a unique ID for tracking
    submission_id = str(uuid.uuid4())
    submission_dir = os.path.join(self.submissions_dir, submission_id)
    os.makedirs(submission_dir, exist_ok=True)
    
    # Create the first status update
    update_status(
        submission_id=submission_id,
        submissions_dir=self.submissions_dir,
        stage='starting',
        progress=0.1,
        message='Setting up benchmark'
    )
    
    # ...rest of function...
```

This creates a status file that looks like:

```json
{
  "submission_id": "550e8400-e29b-41d4-a716-446655440000",
  "stage": "starting",
  "progress": 0.1,
  "last_update": 1680123456.789,
  "message": "Setting up benchmark",
  "state": "running"
}
```

This tells us that the system has received the job and is starting to work on it.

### 3.2.2 Writing Stage: Generating Fuzz Target Code

The Writing Stage uses an LLM to create the fuzz target. After it's done:

```python
# From stage/writing_stage.py
class WritingStage(BaseStage):
    def execute(self, result_history: list[Result]) -> Result:
        # Generate fuzz target code
        last_result = result_history[-1]
        
        # Use agent to generate code
        fuzz_target_code = self.agent.generate_fuzz_target(last_result.benchmark)
        build_script_code = self.agent.generate_build_script(last_result.benchmark)
        
        # Create a build result with the generated code
        build_result = BuildResult(
            benchmark=last_result.benchmark,
            trial=self.trial,
            work_dirs=last_result.work_dirs,
            fuzz_target_source=fuzz_target_code,
            build_script_source=build_script_code,
            author=self,
            chat_history=last_result.chat_history,
        )
        
        return build_result
```

After this, the pipeline updates the status:

```python
# From pipeline.py
def _execute_one_cycle(self, result_history, cycle_count):
    # Run Writing Stage
    result_history.append(
        self.writing_stage.execute(result_history=result_history))
    self._update_status(result_history=result_history)  # Update status file
```

The status file now shows:

```json
{
  "submission_id": "550e8400-e29b-41d4-a716-446655440000",
  "stage": "writing_complete",
  "progress": 0.3,
  "last_update": 1680123466.789,
  "message": "Fuzz target code generated",
  "state": "running",
  "fuzz_target_source": "// Generated code...",
  "build_script_source": "// Build script code..."
}
```

Now we have the generated code saved in our status.

### 3.2.3 Execution Stage: Building and Running the Fuzzer

The Execution Stage tries to build and run the fuzz target:

```python
# From stage/execution_stage.py
class ExecutionStage(BaseStage):
  def execute(self, result_history: list[Result]) -> Result:
    last_result = result_history[-1]
    benchmark = last_result.benchmark
    
    # Set up the builder (local or cloud)
    builder_runner = builder_runner_lib.BuilderRunner(
        benchmark=benchmark,
        work_dirs=last_result.work_dirs,
        run_timeout=self.args.run_timeout,
    )

    # Run the fuzzer and get results
    try:
      build_result, run_result = builder_runner.build_and_run(
          generated_oss_fuzz_project,
          fuzz_target_path,
          0,
          benchmark.language)
      
      # Package results into a RunResult
      runresult = RunResult(
          benchmark=benchmark,
          trial=last_result.trial,
          compiles=last_result.compiles,
          crashes=run_result.crashes,
          coverage=coverage_percent,
          log_path=run_result.log_path,
          corpus_path=run_result.corpus_path,
          # ...other fields...
      )
    except Exception as e:
      # Handle errors
      self.logger.error('Exception %s occurred on %s', e, last_result)
      runresult = RunResult(
          # Minimal error result
      )

    return runresult
```

Again, the pipeline updates the status:

```python
# Update status after execution stage
result_history.append(
    self.execution_stage.execute(result_history=result_history))
self._update_status(result_history=result_history)
```

Now the status shows execution results:

```json
{
  "submission_id": "550e8400-e29b-41d4-a716-446655440000",
  "stage": "execution_complete",
  "progress": 0.7,
  "last_update": 1680123566.789,
  "message": "Fuzzing executed with 75% coverage",
  "state": "running",
  "build_result": {
    "compiles": true,
    "binary_exists": true,
    "is_function_referenced": true
  },
  "run_result": {
    "crashes": false,
    "coverage": 0.75,
    "cov_pcs": 150,
    "total_pcs": 200,
    "log_path": "/path/to/logs/run.log",
    "corpus_path": "/path/to/corpus"
  }
}
```

This tells us if the code worked, how much of the target function was covered, and if any crashes were found.

### 3.2.4 Analysis Stage: Understanding the Results

The Analysis Stage looks at how well the fuzzer performed:

```python
# From stage/analysis_stage.py
class AnalysisStage(BaseStage):
  def execute(self, result_history: list[Result]) -> Result:
    last_result = result_history[-1]
    
    # Analyze the results
    semantic_result = self._check_semantic(last_result)
    
    if last_result.crashes:
      # Analyze crashes
      crash_result = self._analyze_crash(last_result)
    else:
      crash_result = None
      
    if last_result.coverage < COVERAGE_THRESHOLD:
      # Analyze low coverage
      coverage_result = self._analyze_coverage(last_result)
    else:
      coverage_result = None
    
    # Create analysis result
    analysis_result = AnalysisResult(
        author=self,
        run_result=last_result,
        semantic_result=semantic_result,
        crash_result=crash_result,
        coverage_result=coverage_result,
    )
    
    return analysis_result
```

And the pipeline updates the status again:

```python
# Update status after analysis
result_history.append(
    self.analysis_stage.execute(result_history=result_history))
self._update_status(result_history=result_history)
```

The status now includes analysis insights:

```json
{
  "submission_id": "550e8400-e29b-41d4-a716-446655440000",
  "stage": "analysis_complete",
  "progress": 0.9,
  "last_update": 1680123666.789,
  "message": "Analysis complete - coverage 75%, no crashes",
  "state": "running",
  "analysis_result": {
    "success": true,
    "recommendations": "Try adding more edge cases to improve coverage",
    "suggested_improvements": [
      "Add null input tests",
      "Test boundary conditions"
    ]
  }
}
```

This gives feedback about how the fuzz target could be improved.

### 3.2.5 Job Complete: Final Results

When all cycles are finished (either we hit the limit or achieved great results):

```python
# From pipeline.py
def _terminate(self, result_history: list[Result], cycle_count: int) -> bool:
    # Check if we've reached termination conditions
    if cycle_count > 5:
      self.logger.info('[Cycle %d] Terminate after 5 cycles: %s', cycle_count,
                       result_history)
      return True

    last_result = result_history[-1]
    if isinstance(last_result, AnalysisResult) and last_result.success:
      self.logger.info('[Cycle %d] Generation succeeds: %s', cycle_count,
                       result_history)
      return True
      
    # ...other termination conditions...
    
    return False
```

The final status looks like:

```json
{
  "submission_id": "550e8400-e29b-41d4-a716-446655440000",
  "stage": "completed",
  "progress": 1.0,
  "last_update": 1680123766.789,
  "message": "Fuzzing complete with 82% final coverage",
  "state": "completed",
  "final_results": {
    "cycles_completed": 3,
    "best_coverage": 0.82,
    "crashes_found": 0,
    "best_result_cycle": 3,
    "total_execution_time": 310.5
  }
}
```

This summary shows the final outcome of all our fuzzing efforts.

## 3.3 The Glue That Holds It All Together

Five key parts connect everything in our monitoring system:

1. **Submission ID**: Works like a tracking number that follows each job through the entire process.

```python
# From api.py
submission_id = str(uuid.uuid4())  # Generate unique ID
```

2. **Status Files**: Store all the information about a job's progress where any part of the system can find it.

```python
# From status.py
def update_status(submission_id, submissions_dir, stage, progress, message, state='running'):
    status_file = os.path.join(submissions_dir, submission_id, 'status.json')
    with open(status_file, 'w') as f:
        json.dump(status, f, indent=2)
```