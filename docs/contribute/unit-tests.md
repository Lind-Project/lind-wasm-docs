# wasmtestreport.py Documentation

## ⚙️ Command-Line Options

| **Option** | **Description** | **Example Usage** |
|------------|------------------|--------------------|
| `--skip <folder1> <folder2>` | Skip tests in the specified folders. | `./wasmtestreport.py --skip config_tests file_tests` |
| `--run <folder1> <folder2>` | Run only the tests in the specified folders. | `./wasmtestreport.py --run config_tests file_tests` |
| `--timeout <seconds>` | Timeout value in seconds for each test. Must be a positive integer. | `./wasmtestreport.py --timeout 10` |
| `--output <filename>` | Output JSON file name (default: `results.json`). | `./wasmtestreport.py --output newresult` |
| `--report <filename>` | Output HTML report file name (default: `report.html`). | `./wasmtestreport.py --report myreport.html` |
| `--pre-test-only` | Only prepare the test files in Lind FS, no test execution. | `./wasmtestreport.py --pre-test-only` |
| `--clean-testfiles` | Clean up the copied test files from Lind FS. | `./wasmtestreport.py --clean-testfiles` |
| `--clean-results` | Clean all result files and outputs. | `./wasmtestreport.py --clean-results` |

---

## Testing Workflow

1. **Test Case Collection:** Scans `unit-tests` folder for `.c` files.
2. **Filtering:** Applies include/exclude filters (`--run`, `--skip`, and `skip_test_cases.txt`).
3. **Execution:**
   - **Deterministic Tests:** Compare WASM output with native execution.
   - **Non-Deterministic Tests:** Validate only successful WASM execution.
4. **Result Recording:** All test outcomes stored with status, error type, and full output.
5. **Reporting:** JSON and HTML test report generated

---

## Output Files

- **JSON Report:** Detailed test summary in structured format.
- **HTML Report:** Human-readable visualization of test outcomes.

---

## Directory Structure

- `tests/unit-tests/`: Folder containing all `.c` test cases.
- `expected/`: Directory under each test folder for expected output files.
- `testfiles/`: Extra files needed by tests, copied into Lind FS.

---

## Dependencies

- Environment variables:
  - `LIND_WASM_BASE`: Root path of the Lind WASM project.
  - `LIND_FS_ROOT`: Path to the filesystem root used during tests.
- External Scripts:
  - `lindtool.sh compile_test`
  - `lindtool.sh run`

---

## Cleanup & Maintenance

- Temporary files generated (`.wasm`, `.cwasm`, `.o`) are deleted after each run.
- `--clean-testfiles` removes `testfiles` from Lind FS root.
- `--clean-results` clears both result and report files.

---

## Developer Notes

- Deterministic tests match outputs using `.output` reference files.
- Non-deterministic tests consider a run successful if it doesn't crash or timeout.

---

## Example Combined Usage

```bash
./wasmtestreport.py --generate-html --skip config_tests file_tests --timeout 10 --output results_json --report test_report
```

This will:
- Skip specified folders
- Use a 10-second timeout
- Save output as `results_json.json`
- Generate a report `test_report.html`

---
