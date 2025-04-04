# OSS-Fuzz-Gen Configuration System: Plain English Guide

## 1. How the Current System Works

### The Starting Point: YAML Files
Researchers create YAML files that tell the system what to fuzz. These files are simple and only contain basic information:

```yaml
project: "project_name"
language: "c++"
target_path: "/path/to/target" 
functions:
  - name: "function_name"
    signature: "function_signature"
    return_type: "return_type"
    params: [...]
```

The system reads these files using the `Benchmark.from_yaml()` method, which turns each function into a "benchmark" - a task to generate a fuzzer for.

### Behind the Scenes: Current Limitations
When we look at the current code, we find several fixed elements that can't be changed:

1. **Fixed Fuzzer Engine**: The system only uses LibFuzzer - there's no way to switch to other engines like AFL or Honggfuzz

2. **Hardcoded Fuzzer Settings**: In `builder_runner.py`, the settings are fixed:
```python
def _libfuzzer_args(self) -> list[str]:
    return [
        '-print_final_stats=1',
        f'-max_total_time={self.run_timeout}',
        '-len_control=0',
        '-timeout=30',
    ]
```

3. **Fixed Sanitizer**: The system uses AddressSanitizer by default:
```python
def build_target_local(self, generated_project: str, log_path: str,
                       sanitizer: str = 'address') -> bool:
```

### How It Runs

1. **Loading Phase**
    - System reads YAML file
    - Creates `Benchmark` objects for each function
    - Sets up work directories

2. **Building Phase**
    - `BuilderRunner` handles the build process
    - Currently hardcoded to use libFuzzer
    - Basic fuzzing options:
        ```python
        -print_final_stats=1
        -max_total_time=30
        -len_control=0
        -timeout=30
        ```

3. **Running Phase**
    - Default settings:
        - 3 parallel evaluations (`NUM_EVA`)
        - 2 samples per target (`NUM_SAMPLES`)
        - 30-second timeout (`RUN_TIMEOUT`)
        - 0.4 temperature for LLM (`TEMPERATURE`)

### Resource Management
The current implementation sets minimal resource constraints:

- **Shared Memory Limit**: The Docker container uses `--shm-size=2g` to set shared memory to 2GB:
```python
# From experiment/builder_runner.py
command = [
    'docker',
    'run',
    '--rm',
    '--privileged',
    '--shm-size=2g',
    '--platform',
    'linux/amd64',
    # ... other parameters ...
]
```
- **No CPU Limits**: No `--cpus` flag is used, allowing containers to use all available CPU cores
- **No Total Memory Limits**: No `--memory` flag is used to restrict overall container memory
- **Default Timeout**: 30-second timeout per input (set via LibFuzzer arguments)

### Cloud Execution
When running in the cloud using `CloudBuilderRunner`, the resource management differs:

```python
# From experiment/builder_runner.py
command = [
    f'./{oss_fuzz_checkout.VENV_DIR}/bin/python3',
    'infra/build/functions/target_experiment.py',
    f'--project={generated_project}',
    f'--target={self.benchmark.target_name}',
    # ... storage paths and other parameters ...
    f'--experiment_name={self.experiment_name}',
    f'--real_project={project_name}',
]
```

In cloud execution:
- Resources are managed by Google Cloud Build
- CPU and memory limits are defined in Google Cloud's configuration
- Our enhanced system would need mechanisms to specify these cloud resources

### The Runtime Flow
The system creates a BuilderRunner with a benchmark. When it's time to run:

```python
# From builder_runner.py
command = [
    'python3', 'infra/helper.py', 'run_fuzzer', 
    '--corpus-dir', corpus_dir,
    generated_project, self.benchmark.target_name, '--'
] + self._libfuzzer_args()
```

The fuzzer runs inside a Docker container with default resource limits.

## 2. The Enhanced System: Making It Flexible

### An Improved YAML Format

```yaml
project: "project_name"
language: "c++"
target_path: "/path/to/target"
fuzzing_config:
  engine: "libfuzzer"  # Default: libfuzzer (primary supported engine)
  sanitizer: "address"  # Default: address sanitizer
  resources:
    memory: "4g"        # Total container memory (--memory)
    cpus: "2"           # CPU limit (--cpus)
    shm_size: "2g"      # Shared memory size (--shm-size)
  cloud_resources:
    machine_type: "e2-standard-4"  # GCP machine type
    timeout: "1200s"               # Cloud build timeout  
  options:
    dict_path: "/path/to/dict"  # Dictionary for fuzzing
    max_len: 1000               # Maximum input length
    timeout: 30                 # Seconds per test case
```

### How It Would Work: Benchmark Class Extension
The current Benchmark class lacks fields for fuzzing configuration. We'd extend it to store:

- Fuzzer engine (primarily libfuzzer, with extensible design for future engines)
- Sanitizer (primarily address, with support for others where compatible)
- Resource specifications (for both local Docker and cloud execution)
- Engine-specific options (dictionary, timeouts, etc.)

Instead of writing a complete implementation, we'd need to:

1. Add new fields to store these settings
2. Make sure the YAML parser can read the new format
3. Add validation to ensure the settings make sense together
4. Provide defaults for backward compatibility with OSS-Fuzz expectations

### The Heart of the Change: BuilderRunner Improvements
Currently, BuilderRunner has a method that only knows about LibFuzzer:

```python
def _libfuzzer_args(self) -> list[str]:
    return [
        '-print_final_stats=1',
        f'-max_total_time={self.run_timeout}',
        # Without this flag, libFuzzer only consider short inputs in short
        # experiments, which lowers the coverage for quick performance tests.
        '-len_control=0',
        # Timeout per testcase.
        '-timeout=30',
    ]
```

The enhanced BuilderRunner would need changes in several key areas:

#### 1. Engine-Specific Arguments
Create a central method that handles different fuzzing engines:

```python
def _get_fuzzer_args(self) -> list[str]:
    """Get args for the chosen engine."""
    # Would select appropriate arguments based on the configured engine
```

#### 2. Resource Management in Local Docker
Modify `build_target_local` to apply resource limits from the benchmark configuration:

```python
def build_target_local(self, generated_project: str, log_path: str,
                      sanitizer: str = None) -> bool:
    """Builds a target with OSS-Fuzz."""
    # Would add resource constraints from configuration to Docker command
    # Including memory limits, CPU limits, and shared memory size
```

#### 3. Cloud Resource Management
Enhance `build_and_run_cloud` to support cloud-specific resource configurations:

```python
def build_and_run_cloud(self, generated_project: str, target_path: str, ...):
    """Builds and runs in the cloud."""
    # Would add cloud resource specifications like machine type and timeouts
    # to the cloud build command
```

## 3. Implementation Challenges in Plain English

### Engine-Sanitizer Compatibility
Not all combinations work well:

- AFL + MemorySanitizer = Limited support
- Some engines might not support certain sanitizers at all
- Need to warn users about problematic combinations

### Resource Management
Different environments handle resources differently:

- **Local Docker Resources**:
  - CPU cores/shares via the `--cpus` flag
  - Memory limits via the `--memory` flag
  - Shared memory size via the `--shm-size` flag (already implemented)

- **Cloud Resources**:
  - Machine types determine CPU and memory allocation
  - Timeout settings control maximum build duration
  - Worker pools can provide specialized hardware

### Logging and Results
Each fuzzing engine produces different logs:

- Coverage is reported differently (edges vs. blocks)
- Crashes might be counted differently
- Need a way to "normalize" metrics for fair comparison

## 4. Implementation Strategy

1. **Core Framework**
   - Begin by extending the Benchmark class and YAML parsing
   - This lays the foundation without changing behavior

2. **Local Resource Controls**
   - Implement Docker resource limits for local execution:
   ```python
   # Proposed modification to build_target_local
   if self.benchmark.resources.get('memory'):
       command.extend(['--memory', self.benchmark.resources['memory']])
   if self.benchmark.resources.get('cpus'):
       command.extend(['--cpus', self.benchmark.resources['cpus']])
   ```

3. **One Engine at a Time**
   - Add support for each new engine one by one
   - Start with the most similar to LibFuzzer

4. **Cloud Resource Controls**
   - Pass cloud resource specifications to Cloud Build:
   ```python
   # Function signature only
   def build_and_run_cloud(self, generated_project: str, target_path: str, ...):
       """Builds and runs in the cloud with resource specifications."""
   ```


