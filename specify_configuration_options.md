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
- Uses OSS-Fuzz's default limits:
    - 2.5GB RAM per fuzzer
    - 30-second timeout per input
    - Single-core execution

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
With the enhanced system, researchers would have more control while maintaining compatibility with OSS-Fuzz defaults:

```yaml
project: "project_name"
language: "c++"
target_path: "/path/to/target"
fuzzing_config:
  engine: "libfuzzer"  # Default: libfuzzer (primary supported engine)
  sanitizer: "address"  # Default: address sanitizer
  options:
    dict_path: "/path/to/dict"  # Dictionary for fuzzing
    max_len: 1000               # Maximum input length
    timeout: 30                 # Seconds per test case
```

### How It Would Work: Benchmark Class Extension
The current Benchmark class lacks fields for fuzzing configuration. We'd extend it to store:

- Fuzzer engine (primarily libfuzzer, with extensible design for future engines)
- Sanitizer (primarily address, with support for others where compatible)
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
        '-len_control=0',
        '-timeout=30',
    ]
```

We'd need to make this more flexible while focusing on libFuzzer first:

1. Create a central method that handles the primary engine with extensibility:
```python
def _get_fuzzer_args(self) -> list[str]:
    """Get args for the chosen engine."""
    if self.benchmark.fuzzing_engine == 'libfuzzer' or not self.benchmark.fuzzing_engine:
        return self._libfuzzer_args()
    # Design for future extension to other engines
    # (Extended goal, not primary implementation)
```

2. Ensure robust support for libFuzzer first, with architecture allowing for future engines

## 3. Implementation Challenges in Plain English

### Engine-Sanitizer Compatibility
Not all combinations work well:

- AFL + MemorySanitizer = Limited support
- Some engines might not support certain sanitizers at all
- Need to warn users about problematic combinations

### Resource Management
Different environments handle resources differently:

- Docker has specific ways to limit CPU and memory
- Cloud environments have their own resource specification formats
- Some engines have built-in resource controls, others don't

### Logging and Results
Each fuzzing engine produces different logs:

- Coverage is reported differently (edges vs. blocks)
- Crashes might be counted differently
- Need a way to "normalize" metrics for fair comparison

## 4. Implementation Strategy

1. **Start Small: Core Framework**
   - Begin by extending the Benchmark class and YAML parsing
   - This lays the foundation without changing behavior

2. **One Engine at a Time**
   - Add support for each new engine one by one
   - Start with the most similar to LibFuzzer

3. **Resource Controls Next**
   - Add resource controls after basic engine support works

4. **Cloud Support Last**
   - Finally extend cloud support once everything works locally

## 5. Benefits to Researchers

### Flexibility in Fuzzing
Different projects benefit from different fuzzing approaches:

- Some projects find more bugs with AFL's genetic algorithms
- Others work better with LibFuzzer's coverage-guided approach
- Having a choice means better bug finding

### Resource Optimization
Controlling resources means:

- Running more fuzzers on the same hardware
- Giving more resources to complex targets
- Avoiding timeouts for slow code

### Better Experiment Design
Researchers can now:

- Compare different fuzzing engines on the same targets
- Test how resource allocation affects results
- Find the best sanitizer for each project

## 6. Future-Proofing
This design allows for:

- Adding new fuzzing engines as they're developed
- Supporting new sanitizers
- Adding new configuration options without breaking changes

By making these changes, OSS-Fuzz-Gen goes from a fixed system to a flexible research platform that can adapt to new fuzzing technologies and approaches. 