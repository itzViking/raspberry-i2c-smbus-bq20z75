# Raspberry Pi I2C/SMBus Tool for TI bq20z75 Battery Packs

A Python utility for communicating with laptop battery packs built around the Texas Instruments bq20z75 family of smart battery fuel-gauge controllers using a Raspberry Pi SMBus/I2C interface.

The script can read pack information and status data, issue manufacturer-access commands, change security state, clear selected stored values, write selected data-flash parameters, and start the Impedance Track algorithm.

> **Warning**
>
> Several menu options write persistent battery-gauge data or change the gauge security state. Incorrect commands, keys, values, or data-flash contents can make a battery pack unusable or leave its fuel-gauge calibration in an invalid state.
>
> Verify all commands against the exact gauge, firmware, battery design, and TI documentation for the pack you are working with before using any write operation.

## Intended setup

The script is configured for:

- Raspberry Pi SMBus/I2C bus `1`
- Smart Battery address `0x0B`
- Python 3
- `smbus`
- A compatible TI bq20z-series battery gauge

The relevant defaults in the script are:

```python
DEVICE_ADDRESS = 0x0B
BUS_NUMBER = 1

new_capacity = 4400

unseal_key_1 = 0x0414
unseal_key_2 = 0x3672

full_access_key_1 = 0xFFFF
full_access_key_2 = 0xFFFF

pf_clear_key_1 = 0x2673
pf_clear_key_2 = 0x1712
```

These values are pack-specific assumptions from the current script. Do not assume they are valid for another battery pack.

## Requirements

Install the SMBus Python package:

```bash
sudo apt install python3-smbus
```

The Raspberry Pi I2C interface must also be enabled and the battery must be connected correctly to the Pi's SMBus/I2C lines.

The script opens:

```text
/dev/i2c-1
```

through `smbus.SMBus(1)`.

## Important hardware note

Laptop smart-battery interfaces normally include at least:

```text
SMBus clock
SMBus data
Ground
```

Do not connect a battery pack based only on connector position, wire color, or assumptions from another pack.

Confirm the pinout and electrical levels for the specific battery before connecting it to a Raspberry Pi.

## Running the tool

Start the script with:

```bash
python3 raspberry-i2c-smbus-bq20z75.py
```

At startup it continuously checks address `0x0B` until the device responds.

Once communication succeeds, the script displays:

```text
Raspberry Smart Battery
Several utilities for working with TI bq20z... IC
Checking communication with the device at address 0x0B...
The device was found !!!
```

It then presents an interactive menu.

There is currently no menu option to exit, so use `Ctrl+C` when finished.

## Menu operations

### 1. Read pack info

Reads a collection of Smart Battery and manufacturer-specific values.

The current implementation includes:

| Data | Command/register |
|---|---:|
| Design Capacity | `0x18` |
| Full Charge Capacity | `0x10` |
| Cycle Count | `0x17` |
| Manufacture Date | `0x1B` |
| Design Voltage | `0x19` |
| Manufacturer Name | `0x20` |
| Device Name | `0x21` |
| Serial Number | `0x1C` |
| Charging Current | `0x14` |
| Charging Voltage | `0x15` |
| Device Chemistry | `0x22` |
| Temperature | `0x08` |
| Voltage | `0x09` |
| Current | `0x0A` |
| Relative State of Charge | `0x0D` |
| Absolute State of Charge | `0x0E` |
| Remaining Capacity | `0x0F` |
| Cell 4 Voltage | `0x3C` |
| Cell 3 Voltage | `0x3D` |
| Cell 2 Voltage | `0x3E` |
| Cell 1 Voltage | `0x3F` |
| Specification Info | `0x1A` |
| Battery Status | `0x16` |
| Operation Status | `0x54` |

The script also reads additional manufacturer/status values when the pack is unsealed.

The manufacture date is decoded from the Smart Battery packed-date format using 1980 as the base year.

Temperature is displayed using the calculation:

```text
raw / 10 - 273
```

### 2. Pack Reset

Writes:

```text
ManufacturerAccess (0x00) <- 0x0041
```

and waits one second.

### 3. Unseal pack

Writes the two configured unseal keys to ManufacturerAccess:

```text
0x0414
0x3672
```

These keys are hard-coded in the current script.

### 4. Move pack to Full Access mode

Writes the configured full-access key pair:

```text
0xFFFF
0xFFFF
```

These values are also hard-coded and must be validated for the target pack.

### 5. Clear Permanent Failure

Writes the configured PF-clear key sequence:

```text
0x2673
0x1712
```

This changes pack state and should only be used when you understand the reason the Permanent Failure condition was set and have validated that clearing it is appropriate.

### 6. Clear CycleCount

Writes zero to command `0x17`:

```text
CycleCount <- 0
```

This modifies persistent pack data.

### 7. Set current date

Reads the Raspberry Pi system date, packs it into the Smart Battery date format, and writes it to command `0x1B`.

The encoding is:

```text
(year - 1980) << 9
month << 5
day
```

### 8. Write DesignCapacity, QMAX, Update Status, and Ra table

This is the most invasive operation in the current script.

It:

1. writes `new_capacity` to command `0x18`
2. attempts to read subclass `82`
3. modifies capacity-related bytes and update-status data
4. writes the modified subclass
5. writes fixed data blocks to subclasses `88` through `95`

The default configured capacity is:

```python
new_capacity = 4400
```

#### Current implementation limitation

The uploaded script calls:

```python
read_block_smb(...)
write_smb_subclass(...)
```

but those functions are not defined in the current source file.

As provided, menu option 8 is therefore incomplete and is expected to fail when it reaches those calls unless the missing helper functions are added.

Do not rely on option 8 until the data-flash read/write implementation has been completed and validated.

### 9. Begin Impedance Track algorithm

Writes:

```text
ManufacturerAccess (0x00) <- 0x0021
```

This is intended to begin the gauge's Impedance Track process.

Before using it, confirm the required gauge state and learning-cycle procedure for the exact battery design.

## Security and access state

The script checks `OperationStatus` and uses one of its bits to decide whether the pack appears sealed.

When it considers the pack unsealed, it additionally reads:

- MaxError
- SafetyStatus
- PFStatus
- ChargingStatus
- device type
- firmware version
- hardware version
- manufacturer status
- chemistry ID
- battery mode

This can be useful when diagnosing gauge state before attempting write operations.

## Status decoding limitation

The helper function `print_status()` currently decodes only the lowest eight bits of a value:

```text
0x80 ... 0x01
```

Some of the status values passed to it are 16-bit words and some of the supplied label lists contain more than eight entries.

As a result, the current script does **not** decode all possible high-byte status flags.

The raw hexadecimal value printed before the decoded labels should therefore be treated as authoritative when investigating status registers.

## Current-value limitation

The script reads Current (`0x0A`) using `read_word_data()` and prints the returned integer directly.

If the target gauge represents discharge current as a signed 16-bit value, the script does not currently convert the raw unsigned Python value into a negative signed current.

This should be considered when interpreting discharge-current readings.

## I2C clock setting

The source defines:

```python
CLOCK_FREQUENCY = 50000
```

but does not apply this value to the Raspberry Pi I2C controller.

Changing that variable alone does not change the actual SMBus/I2C bus frequency.

Bus speed must be configured separately at the Raspberry Pi / operating-system level if required.

## Customization

Before using the script with another pack, review at minimum:

```python
DEVICE_ADDRESS
BUS_NUMBER
new_capacity
unseal_key_1
unseal_key_2
full_access_key_1
full_access_key_2
pf_clear_key_1
pf_clear_key_2
```

Also verify every manufacturer command and data-flash subclass against documentation for the exact gauge firmware.

## Recommended workflow

For an unfamiliar pack, start with read-only operations.

A safer workflow is:

```text
1. Confirm I2C communication
2. Read pack information
3. Record raw status values
4. Confirm device / firmware identity
5. Verify security state
6. Compare commands and keys with documentation for that exact pack
7. Only then consider write operations
```

Keep a record of original values before modifying persistent data.

## Known limitations

The current script:

- assumes address `0x0B`
- assumes Raspberry Pi bus `1`
- contains hard-coded security/access keys
- contains a fixed default design capacity of `4400`
- has no command-line arguments or configuration file
- has no menu exit option
- decodes only eight status bits in `print_status()`
- does not convert Current to signed 16-bit form
- defines, but does not apply, `CLOCK_FREQUENCY`
- contains incomplete data-flash support for menu option 8 because helper functions referenced there are missing
- has limited validation before issuing destructive or persistent write commands

## Troubleshooting

### Device is not responding

The script will repeatedly print:

```text
The device is not responding.
```

Check:

- battery connector pinout
- common ground
- SDA/SCL wiring
- Raspberry Pi I2C enablement
- `/dev/i2c-1`
- expected address `0x0B`
- whether the pack electronics are awake

You can also check visible I2C devices with tools such as `i2cdetect` if appropriate for the hardware.

### Permission denied when opening the I2C bus

Run with suitable permissions or ensure the user has access to the Raspberry Pi I2C device.

### Option 8 raises `NameError`

The current source references data-flash helper functions that are not defined in the uploaded script.

That operation needs to be completed before option 8 can work reliably.

## Safety

Smart battery packs contain lithium-ion cells and protection electronics.

This project communicates with the battery-management electronics directly and includes operations that can alter protection state, calibration data, capacity values, security state, and Permanent Failure state.

Use the tool only when you understand the battery design and the consequences of the command being issued.

Do not treat clearing an error flag as evidence that the underlying electrical or cell-level fault has been corrected.

## License

No license is currently declared by the original project.

If you intend other people to reuse, modify, or redistribute the code, consider adding a license such as MIT.
