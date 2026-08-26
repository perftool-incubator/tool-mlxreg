# Tool-mlxreg

## Purpose
Crucible tool for monitoring Mellanox/NVIDIA ConnectX network adapter power consumption using Mellanox Firmware Tools (`mlxreg`) to query hardware voltage regulator and environmental power sensors.

## Languages
- Bash: start/stop and collection scripts (`mlxreg-start`, `mlxreg-stop`, `mlxreg-collect`)
- Python: post-processing and CDM metric emission (`mlxreg-post-process`)

## Key Files
| File | Purpose |
|------|---------|
| `mlxreg-start` | Validates parameters, resolves network interfaces to PCI addresses, and launches `mlxreg-collect` |
| `mlxreg-stop` | Sends SIGTERM to `mlxreg-collect` and compresses CSV files with xz |
| `mlxreg-collect` | Periodic hardware power sensor polling loop via `mlxreg` |
| `mlxreg-post-process` | Reads CSV logs and emits `power-watts` CDM metrics via `toolbox.cdm_metrics` |
| `rickshaw.json` | Rickshaw integration: collector scripts, blacklist/whitelist |
| `workshop.json` | Engine image build: installs MFT tools |
| `tool-metadata.json` | Machine-readable description and CDM-indexed status (consumed by `crucible tools list`) |
| `multiplex.json` | Parameter validation rules and `defaults` preset for multiplex (mirrors benchmark `multiplex.json`) |

## Configuration
- `--devices <list>` — Comma-separated list of PCI addresses (e.g. `0000:b5:00.0,0000:c3:00.0`) or network interface names (e.g. `ens7f0np0,ens8f0np0`)
- `--sensors <list>` — Comma-separated list of sensor indices to poll (default: `1,2,6,127`)
- `--interval <seconds>` — Polling interval in seconds (default: `2`)

## Architecture
- `mlxreg-start` — Validates device access (`mlxreg --device $device --reg_name MVCAP --get`), converts interface names to PCI addresses via sysfs (`/sys/class/net/$iface/device`), and launches `mlxreg-collect`
- `mlxreg-collect` — Queries each sensor on each device using `mlxreg --reg_name MVCR --indexes "sensor_index=$idx"`, parses hexadecimal power values into watts, and writes CSV records to `mlxreg-data/<device>.csv`
- `mlxreg-stop` — Sends SIGTERM to collector, waits for graceful exit, and compresses all CSV files to `.csv.xz`
- `mlxreg-post-process` — Reads `.csv.xz` files, constructs CDM metric samples with labels (`device`, `sensor_index`, `sensor_name`, `metric`), and logs `power-watts` metrics

## Testing
- Run post-processor locally: `cd <tool-data-dir> && TOOLBOX_HOME=/opt/crucible/subprojects/core/toolbox python3 /opt/crucible/subprojects/tools/mlxreg/mlxreg-post-process`
- Validate syntax: `python3 -c "import py_compile; py_compile.compile('mlxreg-post-process', doraise=True)"`
- Full integration: `crucible run <run-file.json>` with mlxreg configured

## Conventions
- Primary branch is `main`
- Standard Bash modelines and 4-space indentation
- Python code follows 4-space indentation with standard modelines
