# tool-mlxreg

Power telemetry collection tool for NVIDIA BlueField-3 DPUs using the `mlxreg` utility from Mellanox Firmware Tools (MFT).

## Purpose

Collects real-time power consumption data directly from BF-3 DPU hardware via PCIe register access using the `mlxreg` tool. This provides an alternative to Redfish-based power monitoring and works even when BF-3 BMC sensors show `Absent` status (e.g., uninitialized IPMB).

## Features

- ✅ Direct PCIe register access via `mlxreg` (no IPMB dependency)
- ✅ Multi-device support (multiple DPUs per worker node)
- ✅ Configurable sensor selection
- ✅ CSV output format with timestamps
- ✅ Automatic compression with xz
- ✅ Worker node deployment via rickshaw

## Architecture

### Deployment Model

- **Runs on:** Worker nodes (where BF-3 DPU hardware is present)
- **Execution:** Inside privileged container with MFT tools installed
- **Access:** Direct PCI device access to BF-3 DPU registers

### Tool Build and Dependencies

This tool follows the standard Crucible tool build process. The build and dependency installation is defined in `workshop.json`, which handles:

- MFT (Mellanox Firmware Tools) installation
- Required system packages and dependencies
- Userenv container preparation

See `workshop.json` for the complete build specification and MFT installation details.

## Hardware Requirements

- NVIDIA NIC with MFT (Mellanox Firmware Tools) support and PCIe access
  - Primarily tested with NVIDIA BlueField-3 DPU
  - Should work with other NVIDIA NICs that support mlxreg utility
- Container must run with `--privileged` flag

### Finding Device Addresses

You can specify devices using either **PCI addresses** or **network interface names**.

#### Option 1: Using Network Interface Names (Recommended)

Simply use the network interface name (e.g., `ens7f0np0`):

```json
{
  "arg": "devices",
  "val": "ens7f0np0"
}
```

The tool will automatically resolve the interface name to its PCI address at runtime.

**For multiple DPUs:**
```json
{
  "arg": "devices",
  "val": "ens7f0np0,ens8f0np0"
}
```

#### Option 2: Using PCI BDF Addresses

Find the PCI address on the worker node (or in a privileged container):

```bash
lspci | grep -i mellanox
```

Example output:
```
0000:b5:00.0 Ethernet controller: Mellanox Technologies MT2894 Family [ConnectX-6 Lx]
0000:b5:00.1 Ethernet controller: Mellanox Technologies MT2894 Family [ConnectX-6 Lx]
0000:b5:00.2 System peripheral: Mellanox Technologies MT2894 Family [BlueField-3 SoC Management Interface]
```

Use the first PCI function (e.g., `0000:b5:00.0`) as the `--devices` parameter:

```json
{
  "arg": "devices",
  "val": "0000:b5:00.0"
}
```

**Important:**
- Use only **one PCI address per DPU** (typically `.0` function)
- Both ports (e.g., `ens7f0np0` and `ens7f1np1`) share the same DPU, so monitoring one address captures total DPU power
- Power is measured at the **DPU level**, not per-port

## Usage

### Rickshaw Configuration

**Example 1: Using interface names (recommended):**

```json
{
  "tool-params": [
    {
      "tool": "mlxreg",
      "params": [
        {
          "arg": "devices",
          "val": "ens7f0np0"
        },
        {
          "arg": "interval",
          "val": "2"
        },
        {
          "arg": "sensors",
          "val": "1,2,6"
        }
      ]
    }
  ]
}
```

**Example 2: Using PCI addresses:**

```json
{
  "tool-params": [
    {
      "tool": "mlxreg",
      "params": [
        {
          "arg": "devices",
          "val": "0000:b5:00.0"
        },
        {
          "arg": "interval",
          "val": "2"
        },
        {
          "arg": "sensors",
          "val": "1,2,6"
        }
      ]
    }
  ]
}
```

### Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `--devices` | **Yes** | - | Comma-separated device identifiers. Can be either:<br>• Network interface names: `ens7f0np0` or `ens7f0np0,ens8f0np0`<br>• PCI BDF addresses: `0000:b5:00.0` or `0000:b5:00.0,0000:c3:00.0`<br>The tool automatically converts interface names to PCI addresses |
| `--interval` | No | `2` | Collection interval in seconds |
| `--sensors` | No | `1,2,6` | Comma-separated MVCR sensor indices to collect |

### Sensor Indices

The tool queries the `MVCR` (Voltage/Current/Power) register from the BF-3 DPU. The BF-3 DPU exposes 21 power sensors (indices 1-21), but only a subset provides actual power consumption data:

#### Active Power Sensors (Default Collection: 1,2,6)

| Sensor Index | Sensor Name | Description | Typical Value | Notes |
|--------------|-------------|-------------|---------------|-------|
| 1 | `Vr0Pwr` | Voltage Regulator 0 power | ~17W | Core power supply |
| 2 | `Vr1Pwr` | Voltage Regulator 1 power | ~28W | I/O power supply |
| 6 | `PwrEnv` | Environmental/Total power | ~124W | **Total device power** |

**Power Breakdown:**
- **Total BF-3 Power (PwrEnv):** ~124W
- **VR Power (Vr0 + Vr1):** ~45W (17W + 28W)
- **Other Components:** ~79W (PCIe, memory, networking, etc.)

**Important:** `PwrEnv` (sensor 6) represents the **total device power consumption** of the BF-3 DPU, which includes:
- Voltage regulator power (Vr0Pwr + Vr1Pwr)
- PCIe interface power
- Memory subsystem power
- Network controller power
- Other internal components

For most use cases, collecting sensors **1, 2, and 6** provides both total power and component-level breakdown.

#### Additional Sensors (Status/Diagnostic)

| Sensor Index | Sensor Name | Type | Description |
|--------------|-------------|------|-------------|
| 3 | `AtxAvail` | Status | ATX power availability |
| 4 | `PwrTcnt` | Counter | Power throttle count |
| 5 | `TmpTcnt` | Counter | Temperature throttle count |
| 7 | `PwrTStat` | Status | Power throttle status |
| 8 | `TmpTStat` | Status | Temperature throttle status |
| 9 | `TStat` | Status | General throttle status |
| 10-21 | Various | VR Details | Per-phase voltage/current/power (typically 0 or unused) |

**Note:** Sensors 10-21 provide per-phase voltage regulator details (current, voltage, power for VR phases) but typically read as 0 on BF-3 DPUs, suggesting they may be disabled or not used in the current firmware configuration.

Power values are reported in **watts** (automatically converted from 1/10 watt units).

## Output Format

### CSV Structure

Data is written to CSV files per device in the `mlxreg-data/` directory:

**Filename:** `mlxreg-data/<device-bdf>.csv` (with colons removed, e.g., `0000b50000.csv`)

**CSV Fields:**
```csv
timestamp,date,device,sensor_index,sensor_name,power_watts,status
1715180000.123456,2026-05-08 12:00:00,0000:b5:00.0,1,Vr0Pwr,18.0,OK
1715180000.234567,2026-05-08 12:00:00,0000:b5:00.0,2,Vr1Pwr,28.0,OK
1715180000.345678,2026-05-08 12:00:00,0000:b5:00.0,6,PwrEnv,124.0,OK
```

**Field Descriptions:**
- `timestamp` - Unix timestamp with nanosecond precision
- `date` - Human-readable date/time (YYYY-MM-DD HH:MM:SS)
- `device` - PCI BDF address of the DPU
- `sensor_index` - MVCR sensor index (1, 2, 6, etc.)
- `sensor_name` - Decoded sensor name from register (Vr0Pwr, Vr1Pwr, PwrEnv)
- `power_watts` - Power consumption in watts
- `status` - Collection status (`OK`, `ERROR`, or error description)

### Compression

CSV files are compressed with `xz -3 -T 0` during `mlxreg-stop`, producing `.csv.xz` files.

## Scripts

| Script | Description |
|--------|-------------|
| `mlxreg-start` | Parse command-line arguments, validate device access, launch mlxreg-collect in background |
| `mlxreg-collect` | Main collection loop: query MVCR sensors from each device at specified interval |
| `mlxreg-stop` | Stop mlxreg-collect process and compress CSV output files |
| `mlxreg-post-process` | Generate metric data for OpenSearch indexing |

## Querying Metrics

After running a benchmark with mlxreg and indexing the results, you can query the power metrics using Crucible tools.

### View Available Metrics

```bash
crucible get result --run <run-uuid>
```

This shows all collected metrics, including:
```
source: mlxreg
  types: power-watts
```

### Query Power Data with Breakdown

To view the three power sensors separately (total, VR0, VR1), use the `--breakout metric` option:

```bash
crucible get metric --source mlxreg --type power-watts \
  --period <period-id> --breakout metric --resolution 2
```

**Example output:**
```
                                09-05-2026 09-05-2026
 source        type      metric   03:54:04   03:54:09
-----------------------------------------------------
 mlxreg power-watts total-power     247.95     247.95
 mlxreg power-watts   vr0-power      34.99      34.99
 mlxreg power-watts   vr1-power      27.99      27.99
```

**Metric Breakdown:**
- `total-power` - Total BF-3 DPU power consumption (PwrEnv sensor)
- `vr0-power` - Voltage Regulator 0 power (Vr0Pwr sensor)
- `vr1-power` - Voltage Regulator 1 power (Vr1Pwr sensor)

**Additional Breakout Options:**

You can also break down by device, hostname, or other fields:

```bash
# Break down by device (useful for multi-DPU configurations)
crucible get metric --source mlxreg --type power-watts \
  --period <period-id> --breakout device,metric --resolution 2

# Break down by sensor name
crucible get metric --source mlxreg --type power-watts \
  --period <period-id> --breakout sensor_name --resolution 2
```

Available breakout fields: `device`, `sensor_index`, `sensor_name`, `metric`, `hostname`, `csid`, `engine-id`

## Technical Details

### Power Register (MVCR)

The tool queries the **MVCR (Management Voltage/Current Register)** which provides:
- Real-time voltage regulator power readings
- Power envelope/budget information
- Per-sensor status

**Register query command:**
```bash
mlxreg --device <BDF> --reg_name MVCR --indexes "sensor_index=<N>" --get
```

**Raw output format:**
```
Field Name                  | Data
=========================================
sensor_index                | 0x00000002
power_sensor_value          | 0x00000118  # 280 decimal → 28.0W
sensor_name_hi              | 0x56723150  # "Vr1P"
sensor_name_lo              | 0x77720000  # "wr\0\0"
=========================================
```

**Conversion:**
- `power_sensor_value` is in units of **1/10 Watt**
- Divide by 10 to get actual watts: `0x118 = 280 → 28.0W`
- Sensor name is encoded as ASCII bytes across two 32-bit fields

### Sensor Map

From `MVCAP` register: `sensor_map_lo = 0x003ffffe` → sensors 1–21 are valid.

Commonly used sensors:
- **Sensor 1 (Vr0Pwr):** VR0 power consumption
- **Sensor 2 (Vr1Pwr):** VR1 power consumption
- **Sensor 6 (PwrEnv):** Power envelope/budget

## Deployment

The tool is configured for **profiler collector** deployment via rickshaw.json:

```json
{
  "collector": {
    "whitelist": [
      {
        "endpoint": "remotehosts",
        "collector-types": ["profiler"]
      },
      {
        "endpoint": "kube",
        "collector-types": ["profiler"]
      }
    ]
  }
}
```

### Why Profiler Collector Type?

- Runs on worker nodes where DPU hardware is physically present
- Direct PCI access is required (not available remotely)
- Runs once per node (not per client/server benchmark instance)
- Tool must run in privileged container on the same host as the DPU

## Integration with Existing Tools

This tool complements existing power monitoring approaches:

- **tool-power (Redfish):** Collects from BMC endpoints via network API
- **tool-mlxreg (this tool):** Collects from DPU hardware via local PCIe access
- **iDRAC/BMC Redfish:** Host-side total power consumption

### When to Use tool-mlxreg

- BF-3 BMC sensors show `Absent` status
- IPMB not initialized on BMC
- Need direct hardware access without network dependencies
- Want per-VR power breakdown

### When to Use tool-power

- BMC Redfish API is available and working
- Collecting from multiple remote endpoints
- Network-based collection preferred
- Running from profiler/controller node

## Example: Multi-Device Configuration

For a worker node with two BF-3 DPUs, you can use either interface names or PCI addresses:

**Using interface names:**

```json
{
  "tool-params": [
    {
      "tool": "mlxreg",
      "params": [
        {
          "arg": "devices",
          "val": "ens7f0np0,ens8f0np0"
        },
        {
          "arg": "interval",
          "val": "1"
        },
        {
          "arg": "sensors",
          "val": "1,2,6"
        }
      ]
    }
  ]
}
```

**Using PCI addresses:**

```json
{
  "tool-params": [
    {
      "tool": "mlxreg",
      "params": [
        {
          "arg": "devices",
          "val": "0000:b5:00.0,0000:c3:00.0"
        },
        {
          "arg": "interval",
          "val": "1"
        },
        {
          "arg": "sensors",
          "val": "1,2,6"
        }
      ]
    }
  ]
}
```

This will create two CSV files (using PCI addresses in filenames):
- `mlxreg-data/0000b50000.csv.xz`
- `mlxreg-data/0000c30000.csv.xz`

Each collecting sensors 1, 2, and 6 every second.

## Troubleshooting

### Device Access Errors

**Error:** `Cannot access device 0000:b5:00.0`

**Solutions:**
1. Verify container is running with `--privileged` flag
2. Check device exists: `lspci | grep -i mellanox`
3. Verify MFT tools installed: `which mlxreg`
4. Test manual access: `mlxreg --device 0000:b5:00.0 --reg_name MVCAP --get`

### No Mellanox Devices Found

**Symptoms:** `lspci` shows no Mellanox devices

**Solutions:**
1. Ensure you're running on the correct worker node
2. Verify BF-3 hardware is installed
3. Check if device is visible from host OS
4. Verify container has access to `/sys/bus/pci/devices/`

### mlxreg Command Not Found

**Symptoms:** `ERROR: mlxreg command not found`

**Solutions:**
1. Install MFT package in container image (see Container Image section)
2. Verify MFT RPMs installed: `rpm -qa | grep mft`
3. Check PATH includes `/usr/bin`

## References

- [Mellanox Firmware Tools (MFT)](https://www.nvidia.com/en-us/networking/ethernet-software/firmware-tools/)
- [tool-power](https://github.com/perftool-incubator/tool-power) - Redfish-based power collection
- [Rickshaw Tool Development](https://github.com/perftool-incubator/rickshaw)

## License

See [LICENSE](LICENSE) file.
