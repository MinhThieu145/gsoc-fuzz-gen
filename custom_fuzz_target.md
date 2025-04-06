## 1. Current Workflow Summary

Right now, when you want to generate fuzz targets in OSS-Fuzz-Gen, here's what you have to do:

1. **Set up your benchmarks**: You need to create YAML files in the `benchmark-sets` directory. These files tell the system which projects to work with and which functions to target.
2. **Run the experiments**: You use `run_all_experiments.py` to process all your benchmark files one by one.
3. **Load the benchmarks**: The system reads your YAML files and creates `Benchmark` objects using `experiment/benchmark.py`.
4. **Set up directories**: `experiment/workdir.py` creates folders to store your generated targets and results.
5. **Generate targets**: `run_one_experiment.py` uses an LLM to create fuzz targets based on your function specs.
6. **Fix any issues**: If the generated targets don't compile, the system tries to fix them automatically or asks the LLM to try again.
7. **Test the targets**: `experiment/evaluator.py` builds and runs your targets, checking how much code they cover and if they cause any crashes.
8. **Get results**: Everything gets saved to your work directory and summarized in logs.

While this works, it's a bit clunky - you have to create YAML files by hand and use command-line tools.

## 2. New Workflow for Custom Fuzz Targets

We're making this whole process much simpler. Here's how it'll work:

1. **Browse projects**: You can look through available OSS-Fuzz projects and their functions using simple Python code.
2. **Pick functions**: Instead of writing YAML files, you just tell the system which functions you want to fuzz.
3. **Submit targets**: One API call is all you need to start the process.
4. **Let it run**: The system handles everything in the background - generating targets, building them, and testing them.
5. **Check progress**: You can check how your submission is doing at any time.
6. **Get results**: Once it's done, you get all the details - the generated code, build logs, coverage info, you name it.
7. **Look at the code**: You can check out any generated target's source code and modify it if you want.

This new approach means no more manual YAML file creation or command-line work - it's all done through a simple Python interface.

## 3. New Workflow with Function Signatures

### Step 1: Initialize the API

```python
def initialize() -> OssFuzzApi:
    """Creates and returns an API client instance."""
```

**What it does:**
- Gives you a clean way to work with OSS-Fuzz projects
- Sets up all the folders and settings you need
- Gets the API client ready to use

**How it works:**
- Makes sure you have the OSS-Fuzz code (downloads it if needed)
- Creates the right folders for your work
- Sets up logging so you can see what's happening
- Gets everything ready to use

**How it fits in:**
- Uses the existing OSS-Fuzz setup code
- Works with the current folder structure
- Plays nice with the rest of the system

### Step 2: Find Available Projects

```python
def get_projects() -> List[ProjectInfo]:
    """Retrieves a list of all available OSS-Fuzz projects."""
```

**What it does:**
- Shows you all the OSS-Fuzz projects you can work with
- Gives you the important details about each project
- Makes it easy to find what you're looking for

**How it works:**
- Looks through the OSS-Fuzz projects folder
- Pulls out info like what language each project uses and where its code lives
- Organizes everything in a way that's easy to use
- Handles any missing or broken info gracefully

**How it fits in:**
- Builds on the existing project listing code
- Works with all the languages we support
- Keeps project info consistent

### Step 3: Get Project Details

```python
def get_project_details(project_name: str) -> Dict[str, Any]:
    """Retrieves detailed information about a specific project."""
```

**What it does:**
- Gets you all the info about a specific project
- Shows you what functions you can fuzz
- Helps you pick the right targets

**How it works:**
- Talks to the Fuzz Introspector API to get function info
- Pulls out function signatures, return types, and parameters
- Organizes everything in a way that makes sense
- Handles any missing or incomplete data

**How it fits in:**
- Uses our existing Introspector API code
- Works with our current function data format
- Supports all types of functions and languages

### Step 4: Submit Custom Fuzz Target

```python
def create_fuzz_target(
    project: str,
    function: FunctionInfo,
    options: Optional[Dict[str, Any]] = None
) -> str:
    """Creates and submits a fuzz target for a function."""
```

**What it does:**
- Lets you easily submit new fuzz targets
- Handles everything in the background
- Keeps track of how things are going

**How it works:**
- Creates a unique ID to track your submission
- Sets up a Benchmark object with your function details
- Creates the folders it needs
- Configures the LLM and starts generating targets
- Runs everything using our existing code
- Creates a status file to track progress

**How it fits in:**
- Uses our existing Benchmark setup
- Works with our current experiment runner
- Fits into our target generation pipeline
- Adds a new way to track progress

### Step 5: Check Submission Status

```python
def get_submission_status(submission_id: str) -> SubmissionStatus:
    """Checks the status of a fuzz target submission."""
```

**What it does:**
- Shows you how your submission is doing
- Lets you keep an eye on progress
- Handles any problems that come up

**How it works:**
- Looks at the status file in your submission folder
- If it can't find that, checks if the job is done
- Gives you a clear picture of what's happening
- Handles all the different states (running, done, errors)

**How it fits in:**
- Works with our existing folder structure
- Fits with our current result format
- Gives you clear status updates

### Step 6: Retrieve Results

```python
def get_submission_results(submission_id: str) -> SubmissionResult:
    """Retrieves results for a completed fuzz target submission."""
```

**What it does:**
- Gets you all the results from your submission
- Shows you how well the targets worked
- Lets you analyze and compare results

**How it works:**
- Checks if your submission is done
- Reads the results from your files
- Organizes all the metrics (build success, coverage, etc.)
- Handles any missing or incomplete data

**How it fits in:**
- Uses our existing results code
- Works with our current metrics format
- Gives you nicely organized results

### Step 7: View Target Code

```python
def get_target_code(submission_id: str, target_filename: str) -> str:
    """Retrieves the source code for a specific generated target."""
```

**What it does:**
- Lets you look at the generated target code
- Makes it easy to check and modify the code
- Helps you customize things if needed

**How it works:**
- Finds the target file in your fixed_targets folder
- Reads and shows you the code
- Handles any missing files
- Works with both fixed and raw targets

**How it fits in:**
- Works with our existing folder setup
- Fits with our current file organization
- Makes it easy to get at the code

## Summary of New Implementation Needed

To build this API, we'll need to:

1. **Use What We Already Have**:
    - The OSS-Fuzz setup and management code
    - Our Benchmark class for configuring targets
    - The Introspector API for getting function info
    - Our target generation and testing pipeline

2. **Build New Stuff**:
    - A way to track submission progress
    - A clean API wrapper around our existing code
    - A system to monitor long-running jobs
    - A nice way to format results for the API

## Core Encapsulated Function for Fuzz Target Execution

The new workflow requires a standalone function in a new file (e.g., `executor.py`) that handles the complete process of building and running fuzz targets. This function will serve as the execution engine for generated fuzz targets.

```python
def run_fuzz_target(
    project_name: str,
    fuzz_target_code: str,
    build_script: str,
    target_name: str = "fuzz_target",
    run_in_cloud: bool = False,
    max_retries: int = 3
) -> Dict[str, Any]:
    """
    Builds and runs a fuzz target for a specific project and returns the results.
    
    Args:
        project_name: Name of the OSS-Fuzz project to use
        fuzz_target_code: Source code for the fuzz target
        build_script: Build script content to compile the target
        target_name: Name for the target (default: "fuzz_target")
        run_in_cloud: Whether to run on Google Cloud Build
        max_retries: Maximum number of retry attempts
        
    Returns:
        Dictionary containing build results, coverage data, and crash information
    """
```

### Function Design Details

1. **Project Environment Setup**
   - Clone the OSS-Fuzz repository using `oss_fuzz_checkout.clone_oss_fuzz()` from `experiment/oss_fuzz_checkout.py`
   - Create a copy of the project directory from `OSS_FUZZ_DIR/projects/{project_name}` to a temporary location
   - Set up output directories for build artifacts, logs, and corpus using a structure similar to `WorkDirs` in `experiment/workdir.py`

2. **Fuzz Target Integration**
   - Write fuzz target code to `{project_dir}/{target_name}.c` (or appropriate extension based on language)
   - Create build script at `{project_dir}/build.sh` with executable permissions
   - Modify the project's Dockerfile to include the new target using techniques from `experiment/evaluator.py:create_ossfuzz_project()`

3. **Execution Management**
   - Create a BuilderRunner instance from `experiment/builder_runner.py`:
     - `BuilderRunner` for local execution
     - `CloudBuilderRunner` for cloud execution
   - Build and run the target with retry logic similar to `experiment/evaluator.py:check_target()`
   - Handle transient failures (network issues, resource limits) by retrying failed operations

4. **Result Collection**
   - Gather build results from the `BuildResult` object returned by the builder
   - Extract coverage data using techniques from `experiment/evaluator.py` and `experiment/textcov.py`
   - Check for crashes in the fuzzer output logs
   - Collect all artifacts into a standardized directory structure for easy access

### Integration with Existing Code

The function should leverage these existing components:
- Project setup code from `experiment/oss_fuzz_checkout.py`
- Build and run processes from `experiment/builder_runner.py`
- Result collection from `experiment/evaluator.py`
- Coverage analysis from `experiment/textcov.py`

### Return Value Structure

```python
# Result structure returned by the function
{
    "build_success": bool,  # Whether build succeeded
    "build_logs": str,      # Path to build logs
    "build_errors": list,   # Any errors during build
    
    "run_success": bool,    # Whether fuzzing ran successfully
    "coverage": {
        "covered_lines": int,
        "total_lines": int,
        "coverage_percent": float
    },
    "crashes": bool,        # Whether any crashes were found
    "crash_info": str,      # Details about crashes if any
    
    # Paths to important artifacts
    "coverage_report_path": str,
    "log_path": str,
    "corpus_path": str
}
```