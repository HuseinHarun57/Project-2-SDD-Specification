# Autonomous Robotics Management and Navigation Software Platform: skeleton

A runnable skeleton being implemented slice by slice (see `plan.md`). Functions that are not implemented yet raise `NotImplementedError`, and their routes return HTTP 501. See `contracts.md` for the function and route contracts.

## Start

macOS / Linux:

```bash
cd robot-platform
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt pytest
export STATUS_FRESHNESS_TIMEOUT_SECONDS=10   # optional; the spec leaves the value to the project team
uvicorn app.main:app --reload --port 8000
```

Windows PowerShell:

```powershell
cd robot-platform
python -m venv .venv
.venv\Scripts\Activate.ps1      # if scripts are disabled: Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
python -m pip install -r requirements.txt pytest
$env:STATUS_FRESHNESS_TIMEOUT_SECONDS = "10"
uvicorn app.main:app --reload --port 8000
```

On Windows, use `python`, not `python3`: the virtual environment only provides `python.exe`. Leave the environment with `deactivate`.

- Interactive docs: http://127.0.0.1:8000/docs
- Try a route: `curl -i http://127.0.0.1:8000/api/v1/dashboard` returns 501, because the running server has no identity source yet (see `contracts.md`, Access Control).

## Tests

pytest is used for the tests but is not in `requirements.txt`, so install it as shown above.

```bash
python -m pytest -q                                                    # full suite
python -m pytest tests/test_ac003_status.py::test_TP_003_08 -v         # Slice 1
```

## Layout

```
docs/spec.md                         spec with the definitions and FR-005 edit
docs/architecture.md                 architecture, with the Slice 1 storage note
plan.md                              implementation slices and their status
test_plan.md                         test scenarios (TP IDs)
tests/                               pytest suite, one file per acceptance criterion
app/main.py                          wiring + server + 501 handler
app/modules/access_control.py
app/modules/persistence.py           also holds the shared record types
app/modules/robot_integration.py
app/modules/robot_registry.py
app/modules/robot_status_telemetry.py
app/modules/sensor_information.py
app/modules/mission_assignment.py
app/modules/user_interface/          routes.py (one route per FR), schemas.py
```
