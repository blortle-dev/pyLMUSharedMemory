# pyLMUSharedMemory — Usage Guide

A comprehensive guide to using the pyLMUSharedMemory library for reading real-time telemetry, scoring, and application state data from Le Mans Ultimate (LMU).

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Connecting to Shared Memory](#connecting-to-shared-memory)
  - [Using MMapControl (Recommended)](#using-mmapcontrol-recommended)
  - [Using SimInfo (Legacy)](#using-siminfo-legacy)
  - [Access Modes](#access-modes)
- [Data Structure Overview](#data-structure-overview)
- [Reading Generic & Application Data](#reading-generic--application-data)
  - [Game Version & Force Feedback](#game-version--force-feedback)
  - [Events](#events)
  - [Application State](#application-state)
- [Reading Path Data](#reading-path-data)
- [Reading Scoring Data](#reading-scoring-data)
  - [Session Information](#session-information)
  - [Per-Vehicle Scoring](#per-vehicle-scoring)
  - [Weather & Track Conditions](#weather--track-conditions)
  - [Server Information](#server-information)
- [Reading Telemetry Data](#reading-telemetry-data)
  - [Identifying the Player Vehicle](#identifying-the-player-vehicle)
  - [Vehicle Motion](#vehicle-motion)
  - [Engine & Drivetrain](#engine--drivetrain)
  - [Driver Inputs](#driver-inputs)
  - [Fuel & Pit Strategy](#fuel--pit-strategy)
  - [Aerodynamics](#aerodynamics)
  - [Damage](#damage)
  - [Hybrid / Electric Systems](#hybrid--electric-systems)
  - [Wheel & Tire Data](#wheel--tire-data)
- [Practical Examples](#practical-examples)
  - [Real-Time Telemetry Display](#real-time-telemetry-display)
  - [Lap Time Monitor](#lap-time-monitor)
  - [Track Position](#track-position)
  - [Live Race Standings](#live-race-standings)
- [Interpreting Encoded Values](#interpreting-encoded-values)
  - [Session Types](#session-types)
  - [Game Phases](#game-phases)
  - [Finish Status](#finish-status)
  - [Vehicle Control](#vehicle-control)
  - [Pit State](#pit-state)
  - [Yellow Flag States](#yellow-flag-states)
  - [Surface Types](#surface-types)
  - [Ignition / Starter](#ignition--starter)
  - [Rear Flap Legal Status](#rear-flap-legal-status)
  - [Electric Boost Motor State](#electric-boost-motor-state)
- [Platform Notes](#platform-notes)
- [Tips & Best Practices](#tips--best-practices)
- [Complete Data Reference](#complete-data-reference)
  - [LMUVect3](#lmuvect3)
  - [LMUWheel](#lmuwheel)
  - [LMUVehicleTelemetry](#lmuvehicletelemetry)
  - [LMUVehicleScoring](#lmuvehiclescoring)
  - [LMUScoringInfo](#lmuscoringinfo)
  - [LMUApplicationState](#lmuapplicationstate)
  - [LMUEvent](#lmuevent)

---

## Overview

Le Mans Ultimate (LMU) exposes a shared memory interface that allows external programs to read real-time simulation data while the game is running. The **pyLMUSharedMemory** library maps this shared memory into Python objects using `ctypes`, so you can access telemetry, scoring, and application state data directly from Python.

Key capabilities:

- **Telemetry** — position, speed, acceleration, engine RPM, tire temperatures, brake pressures, fuel level, and more for up to 104 vehicles.
- **Scoring** — race positions, lap times, sector times, gap to leader, pit stops, penalties, and driver/vehicle names.
- **Application state** — game version, window handle, screen resolution, events, and force feedback torque.
- **Path data** — file system paths used by the game (user data, plugins, profiles, etc.).

The library has **no external dependencies** — it uses only the Python standard library (`ctypes`, `mmap`, `logging`, `platform`).

---

## Installation

Clone or copy the repository into your project:

```bash
git clone https://github.com/blortle-dev/pyLMUSharedMemory.git
```

Or install directly from the repository:

```bash
pip install git+https://github.com/blortle-dev/pyLMUSharedMemory.git
```

Since the library uses only the Python standard library, no additional packages are required.

---

## Quick Start

Below is a minimal working example that connects to LMU's shared memory and prints basic information. **LMU must be running** for the data to contain meaningful values.

```python
from pyLMUSharedMemory.lmu_mmap import MMapControl
from pyLMUSharedMemory.lmu_data import LMUObjectOut, LMUConstants

# Create the shared memory connection
info = MMapControl(LMUConstants.LMU_SHARED_MEMORY_FILE, LMUObjectOut)
info.create()  # defaults to copy access mode

# Update the local buffer with the latest data from the game
info.update()

# Read some basic data
version = info.data.generic.gameVersion
track = info.data.scoring.scoringInfo.mTrackName.decode()
num_vehicles = info.data.scoring.scoringInfo.mNumVehicles

print(f"Game version: {version}")
print(f"Track: {track}")
print(f"Vehicles on track: {num_vehicles}")

# Clean up
info.close()
```

> **Note:** String fields (names, filenames) are returned as `bytes`. Call `.decode()` to convert them to Python strings.

---

## Connecting to Shared Memory

### Using MMapControl (Recommended)

`MMapControl` is the primary interface for accessing the shared memory. It handles platform differences, provides two access modes, and includes safety checks for data consistency.

```python
from pyLMUSharedMemory.lmu_mmap import MMapControl
from pyLMUSharedMemory.lmu_data import LMUObjectOut, LMUConstants

# Initialize (does not connect yet)
info = MMapControl(LMUConstants.LMU_SHARED_MEMORY_FILE, LMUObjectOut)

# Connect and create the memory mapping
info.create(access_mode=0)  # 0 = copy access (default), 1 = direct access

# In your main loop, call update() each iteration to refresh data
info.update()

# Access data through info.data
print(info.data.generic.gameVersion)

# When done, close the connection
info.close()
```

### Using SimInfo (Legacy)

`SimInfo` is a simpler, older interface that uses direct buffer access on Windows only. It is kept for backward compatibility.

```python
from pyLMUSharedMemory.lmu_data import SimInfo

info = SimInfo()

# Access data directly — no update() call needed (direct access)
print(info.LMUData.generic.gameVersion)
print(info.LMUData.scoring.scoringInfo.mTrackName.decode())

# Close when done
info.close()
```

> **Note:** `SimInfo` uses Windows-specific `mmap` with a `tagname` parameter and does not work on Linux. Use `MMapControl` for cross-platform support.

### Access Modes

`MMapControl.create()` accepts an `access_mode` parameter:

| Mode | Value | Description |
|------|-------|-------------|
| **Copy** | `0` (default) | Maintains a separate local buffer. When you call `update()`, the library checks that the game is actively updating data (via event flags) and that scoring and telemetry vehicle counts match, then copies the entire shared memory buffer atomically. This avoids reading partially-updated data. |
| **Direct** | `1` | Reads directly from the shared memory buffer. Faster, but you may read data that is in the middle of being written by the game, leading to inconsistent values within a single read. |

**Recommendation:** Use **copy access** (the default) for most applications. Use direct access only if you need maximum performance and can tolerate occasional data inconsistencies.

```python
# Copy access (safe, recommended)
info.create(access_mode=0)

# Direct access (fast, may have inconsistencies)
info.create(access_mode=1)
```

---

## Data Structure Overview

All data is accessed through `info.data`, which is an `LMUObjectOut` structure with four top-level sections:

```
info.data
├── generic          # LMUGeneric — game version, events, FFB, app state
│   ├── events       # LMUEvent — 16 event flags
│   ├── gameVersion  # int — game build version
│   ├── FFBTorque    # float — force feedback torque
│   └── appInfo      # LMUApplicationState — window, resolution, UI state
├── paths            # LMUPathData — 5 file system paths
├── scoring          # LMUScoringData — session + per-vehicle scoring
│   ├── scoringInfo  # LMUScoringInfo — session-level data
│   └── vehScoringInfo[0..103]  # LMUVehicleScoring — per-vehicle scoring
└── telemetry        # LMUTelemetryData — per-vehicle telemetry
    ├── activeVehicles    # int — number of active vehicles
    ├── playerVehicleIdx  # int — index of the player's vehicle
    ├── playerHasVehicle  # bool — whether the player has a vehicle
    └── telemInfo[0..103] # LMUVehicleTelemetry — per-vehicle telemetry
        └── mWheels[0..3] # LMUWheel — per-wheel data (FL, FR, RL, RR)
```

The shared memory can hold data for up to **104 vehicles** simultaneously. Check `scoring.scoringInfo.mNumVehicles` or `telemetry.activeVehicles` to know how many are currently active.

---

## Reading Generic & Application Data

### Game Version & Force Feedback

```python
info.update()

version = info.data.generic.gameVersion
ffb_torque = info.data.generic.FFBTorque

print(f"Game version: {version}")
print(f"FFB torque: {ffb_torque}")
```

- `gameVersion` is `0` when the game is not running.
- `FFBTorque` is the current force feedback torque value as a float.

### Events

The `events` field contains 16 flags that indicate which game events have occurred. These are useful for synchronization and knowing the current state of the game.

```python
events = info.data.generic.events

if events.SME_UPDATE_TELEMETRY:
    print("Telemetry data has been updated")

if events.SME_UPDATE_SCORING:
    print("Scoring data has been updated")

if events.SME_ENTER_REALTIME:
    print("Player entered the track")
```

| Event | Description |
|-------|-------------|
| `SME_ENTER` | Entered the simulation |
| `SME_EXIT` | Exited the simulation |
| `SME_STARTUP` | Game started up |
| `SME_SHUTDOWN` | Game shutting down |
| `SME_LOAD` | Session loaded |
| `SME_UNLOAD` | Session unloaded |
| `SME_START_SESSION` | Session started |
| `SME_END_SESSION` | Session ended |
| `SME_ENTER_REALTIME` | Player entered on-track (realtime) mode |
| `SME_EXIT_REALTIME` | Player left on-track mode (e.g. to monitor) |
| `SME_UPDATE_SCORING` | Scoring data was just updated |
| `SME_UPDATE_TELEMETRY` | Telemetry data was just updated |
| `SME_INIT_APPLICATION` | Application initialized |
| `SME_UNINIT_APPLICATION` | Application uninitialized |
| `SME_SET_ENVIRONMENT` | Environment set |
| `SME_FFB` | Force feedback update |

### Application State

```python
app = info.data.generic.appInfo

print(f"Window handle: {app.mAppWindow}")
print(f"Resolution: {app.mWidth}x{app.mHeight}")
print(f"Refresh rate: {app.mRefreshRate} Hz")
print(f"Windowed: {bool(app.mWindowed)}")
print(f"Options location: {app.mOptionsLocation}")  # 0=main UI, 1=track loading, 2=monitor, 3=on track
print(f"Options page: {app.mOptionsPage.decode()}")
```

---

## Reading Path Data

The path data section provides file system paths the game is using:

```python
paths = info.data.paths

print(f"User data:        {paths.userData.decode()}")
print(f"Custom variables: {paths.customVariables.decode()}")
print(f"Steward results:  {paths.stewardResults.decode()}")
print(f"Player profile:   {paths.playerProfile.decode()}")
print(f"Plugins folder:   {paths.pluginsFolder.decode()}")
```

Each path is up to 260 characters (the Windows `MAX_PATH` limit).

---

## Reading Scoring Data

### Session Information

Session-level data is in `info.data.scoring.scoringInfo`:

```python
info.update()
session = info.data.scoring.scoringInfo

track_name = session.mTrackName.decode()
session_type = session.mSession
elapsed_time = session.mCurrentET
end_time = session.mEndET
max_laps = session.mMaxLaps
lap_distance = session.mLapDist
num_vehicles = session.mNumVehicles
game_phase = session.mGamePhase
in_realtime = session.mInRealtime
player_name = session.mPlayerName.decode()

print(f"Track: {track_name}")
print(f"Session: {session_type}")  # See "Session Types" below
print(f"Elapsed: {elapsed_time:.1f}s")
print(f"Vehicles: {num_vehicles}")
print(f"Game phase: {game_phase}")  # See "Game Phases" below
print(f"Player: {player_name}")
```

### Per-Vehicle Scoring

Each vehicle's scoring data is in `info.data.scoring.vehScoringInfo[index]`:

```python
num_vehicles = info.data.scoring.scoringInfo.mNumVehicles

for i in range(num_vehicles):
    veh = info.data.scoring.vehScoringInfo[i]

    driver = veh.mDriverName.decode()
    vehicle = veh.mVehicleName.decode()
    vehicle_class = veh.mVehicleClass.decode()
    position = veh.mPlace
    total_laps = veh.mTotalLaps
    best_lap = veh.mBestLapTime
    last_lap = veh.mLastLapTime
    gap_to_leader = veh.mTimeBehindLeader
    in_pits = veh.mInPits
    is_player = veh.mIsPlayer

    best_lap_str = f"{best_lap:.3f}s" if best_lap > 0 else "No time"

    print(f"P{position:>2} | {driver:<20} | {vehicle_class:<10} | "
          f"Best: {best_lap_str:<12} | Gap: {gap_to_leader:>8.3f}s | "
          f"{'[PIT]' if in_pits else ''}")
```

**Sector times** are cumulative:

- `mBestSector1` — best sector 1 time.
- `mBestSector2` — best sector 1 + sector 2 combined. To get sector 2 alone: `mBestSector2 - mBestSector1`.
- `mBestLapTime` — full lap time. To get sector 3 alone: `mBestLapTime - mBestSector2`.
- The same applies to `mLastSector1`/`mLastSector2`/`mLastLapTime` and `mCurSector1`/`mCurSector2`.

**Sector numbering note:** The `mSector` field uses an unusual numbering: `0` = sector 3, `1` = sector 1, `2` = sector 2.

### Weather & Track Conditions

```python
session = info.data.scoring.scoringInfo

print(f"Ambient temp: {session.mAmbientTemp:.1f} °C")
print(f"Track temp:   {session.mTrackTemp:.1f} °C")
print(f"Cloud cover:  {session.mDarkCloud:.0%}")
print(f"Rain:         {session.mRaining:.0%}")
print(f"Wind:         x={session.mWind.x:.1f} y={session.mWind.y:.1f} z={session.mWind.z:.1f}")
print(f"Path wetness: min={session.mMinPathWetness:.0%} avg={session.mAvgPathWetness:.0%} max={session.mMaxPathWetness:.0%}")
```

- `mDarkCloud` and `mRaining` range from `0.0` (none) to `1.0` (maximum).
- Wetness values range from `0.0` (fully dry) to `1.0` (fully wet).

### Server Information

When connected to a multiplayer server:

```python
session = info.data.scoring.scoringInfo

if session.mGameMode in (1, 3):  # 1=server, 3=server+client
    print(f"Server name: {session.mServerName.decode()}")
    print(f"Server port: {session.mServerPort}")
    print(f"Max players: {session.mMaxPlayers}")
    print(f"Password protected: {session.mIsPasswordProtected}")
```

---

## Reading Telemetry Data

### Identifying the Player Vehicle

```python
info.update()
telem = info.data.telemetry

player_idx = telem.playerVehicleIdx
has_vehicle = telem.playerHasVehicle
active = telem.activeVehicles

if has_vehicle:
    player = telem.telemInfo[player_idx]
    print(f"Player vehicle: {player.mVehicleName.decode()}")
else:
    print("Player does not have a vehicle (spectating or in monitor)")
```

### Vehicle Motion

Position, velocity, and acceleration are provided as 3D vectors in the vehicle's **local coordinate system** (except `mPos` which is in world coordinates):

```python
player = info.data.telemetry.telemInfo[player_idx]

# World position (meters)
print(f"Position: x={player.mPos.x:.1f} y={player.mPos.y:.1f} z={player.mPos.z:.1f}")

# Local velocity (meters/sec) — x=left/right, y=up/down, z=forward/back
print(f"Velocity: x={player.mLocalVel.x:.1f} y={player.mLocalVel.y:.1f} z={player.mLocalVel.z:.1f}")

# Speed in km/h (from the forward velocity component)
import math
speed_ms = math.sqrt(
    player.mLocalVel.x**2 +
    player.mLocalVel.y**2 +
    player.mLocalVel.z**2
)
speed_kmh = speed_ms * 3.6
print(f"Speed: {speed_kmh:.1f} km/h")

# Local acceleration (meters/sec²)
lateral_g = player.mLocalAccel.x / 9.81
longitudinal_g = player.mLocalAccel.z / 9.81
print(f"Lateral G: {lateral_g:.2f}")
print(f"Longitudinal G: {longitudinal_g:.2f}")
```

**Orientation matrix** (`mOri`) is a 3×3 rotation matrix stored as three `LMUVect3` values (rows). You can use it to convert between local and world coordinates.

### Engine & Drivetrain

```python
player = info.data.telemetry.telemInfo[player_idx]

gear = player.mGear  # -1=reverse, 0=neutral, 1+=forward
rpm = player.mEngineRPM
max_rpm = player.mEngineMaxRPM
torque = player.mEngineTorque  # Nm
water_temp = player.mEngineWaterTemp  # Celsius
oil_temp = player.mEngineOilTemp  # Celsius
clutch_rpm = player.mClutchRPM
max_gears = player.mMaxGears
turbo = player.mTurboBoostPressure

gear_name = "R" if gear == -1 else ("N" if gear == 0 else str(gear))
print(f"Gear: {gear_name}")
print(f"RPM: {rpm:.0f} / {max_rpm:.0f}")
print(f"Torque: {torque:.1f} Nm")
print(f"Water temp: {water_temp:.1f} °C")
print(f"Oil temp: {oil_temp:.1f} °C")
if turbo > 0:
    print(f"Turbo boost: {turbo:.2f}")
```

### Driver Inputs

Two sets of inputs are available: **unfiltered** (raw driver input) and **filtered** (after game processing such as assists):

```python
player = info.data.telemetry.telemInfo[player_idx]

# Raw driver inputs
throttle = player.mUnfilteredThrottle    # 0.0 to 1.0
brake = player.mUnfilteredBrake          # 0.0 to 1.0
steering = player.mUnfilteredSteering    # -1.0 (left) to 1.0 (right)
clutch = player.mUnfilteredClutch        # 0.0 to 1.0

# Processed inputs (after assists, traction control, etc.)
f_throttle = player.mFilteredThrottle
f_brake = player.mFilteredBrake
f_steering = player.mFilteredSteering
f_clutch = player.mFilteredClutch

print(f"Throttle: {throttle:.0%} (filtered: {f_throttle:.0%})")
print(f"Brake:    {brake:.0%} (filtered: {f_brake:.0%})")
print(f"Steering: {steering:+.2f} (filtered: {f_steering:+.2f})")
print(f"Clutch:   {clutch:.0%} (filtered: {f_clutch:.0%})")

# Steering wheel range
visual_range = player.mVisualSteeringWheelRange    # degrees (visual)
physical_range = player.mPhysicalSteeringWheelRange  # degrees (physical)
ffb_torque = player.mSteeringShaftTorque             # Nm
rear_brake_bias = player.mRearBrakeBias              # fraction on rear
```

### Fuel & Pit Strategy

```python
player = info.data.telemetry.telemInfo[player_idx]

fuel = player.mFuel              # liters remaining
capacity = player.mFuelCapacity  # total capacity in liters
stops = player.mScheduledStops   # scheduled pit stops
limiter = player.mSpeedLimiter   # speed limiter active
limiter_avail = player.mSpeedLimiterAvailable

print(f"Fuel: {fuel:.1f} / {capacity:.1f} L ({fuel/capacity:.0%})")
print(f"Scheduled stops: {stops}")
print(f"Speed limiter: {'ON' if limiter else 'OFF'}")
```

### Aerodynamics

```python
player = info.data.telemetry.telemInfo[player_idx]

drag = player.mDrag
front_df = player.mFrontDownforce
rear_df = player.mRearDownforce
front_wing_h = player.mFrontWingHeight
front_ride = player.mFrontRideHeight
rear_ride = player.mRearRideHeight
front_flap = player.mFrontFlapActivated
rear_flap = player.mRearFlapActivated
rear_flap_legal = player.mRearFlapLegalStatus  # 0=disallowed, 1=detected, 2=allowed

print(f"Drag: {drag:.1f}")
print(f"Downforce: front={front_df:.1f} rear={rear_df:.1f}")
print(f"Ride height: front={front_ride:.4f}m rear={rear_ride:.4f}m")
```

### Damage

```python
player = info.data.telemetry.telemInfo[player_idx]

overheating = player.mOverheating
parts_detached = player.mDetached
headlights = player.mHeadlights
last_impact_time = player.mLastImpactET
last_impact_force = player.mLastImpactMagnitude

# Dent severity at 8 locations (0=none, 1=some, 2=severe)
dents = [player.mDentSeverity[i] for i in range(8)]

print(f"Overheating: {overheating}")
print(f"Parts detached: {parts_detached}")
print(f"Dent severity: {dents}")
if last_impact_force > 0:
    print(f"Last impact: force={last_impact_force:.1f} at t={last_impact_time:.1f}s")
    print(f"  Position: ({player.mLastImpactPos.x:.1f}, {player.mLastImpactPos.y:.1f}, {player.mLastImpactPos.z:.1f})")
```

### Hybrid / Electric Systems

For vehicles with hybrid or electric boost systems:

```python
player = info.data.telemetry.telemInfo[player_idx]

battery = player.mBatteryChargeFraction  # 0.0 to 1.0
motor_torque = player.mElectricBoostMotorTorque  # Nm (negative = regen)
motor_rpm = player.mElectricBoostMotorRPM
motor_temp = player.mElectricBoostMotorTemperature  # Celsius
motor_water_temp = player.mElectricBoostWaterTemperature
motor_state = player.mElectricBoostMotorState
# 0=unavailable, 1=inactive, 2=propulsion, 3=regeneration

state_names = {0: "Unavailable", 1: "Inactive", 2: "Propulsion", 3: "Regeneration"}

if motor_state > 0:
    print(f"Battery: {battery:.0%}")
    print(f"Motor state: {state_names.get(motor_state, 'Unknown')}")
    print(f"Motor torque: {motor_torque:.1f} Nm")
    print(f"Motor RPM: {motor_rpm:.0f}")
    print(f"Motor temp: {motor_temp:.1f} °C")
```

### Wheel & Tire Data

Each vehicle has 4 wheels indexed as: `0` = front-left, `1` = front-right, `2` = rear-left, `3` = rear-right.

```python
player = info.data.telemetry.telemInfo[player_idx]
wheel_names = ["FL", "FR", "RL", "RR"]

for i in range(4):
    w = player.mWheels[i]

    # Tire pressure (kPa)
    pressure = w.mPressure

    # Tire temperature — 3 readings across the tread width: left, center, right
    # (not inside, center, outside). Values are in Kelvin.
    temps_c = [w.mTemperature[j] - 273.15 for j in range(3)]

    # Brake temperature (Celsius)
    brake_temp = w.mBrakeTemp

    # Tire wear (0.0 = fully worn, 1.0 = new)
    wear = w.mWear

    # Suspension
    susp_deflection = w.mSuspensionDeflection  # meters
    ride_height = w.mRideHeight                # meters
    susp_force = w.mSuspForce                  # Newtons

    # Tire load and grip
    tire_load = w.mTireLoad      # Newtons
    grip = w.mGripFract          # fraction of contact patch sliding

    # Status
    is_flat = w.mFlat
    is_detached = w.mDetached
    surface = w.mSurfaceType     # 0=dry, 1=wet, 2=grass, 3=dirt, 4=gravel, 5=rumblestrip, 6=special
    terrain = w.mTerrainName.decode()

    print(f"\n{wheel_names[i]}:")
    print(f"  Pressure: {pressure:.1f} kPa")
    print(f"  Temp (L/C/R): {temps_c[0]:.1f} / {temps_c[1]:.1f} / {temps_c[2]:.1f} °C")
    print(f"  Brake temp: {brake_temp:.1f} °C")
    print(f"  Wear: {wear:.1%}")
    print(f"  Load: {tire_load:.0f} N | Grip: {grip:.2f}")
    print(f"  Ride height: {ride_height:.4f} m")
    print(f"  Surface: {terrain} (type {surface})")
    if is_flat:
        print(f"  *** FLAT TIRE ***")
    if is_detached:
        print(f"  *** WHEEL DETACHED ***")
```

**Tire temperature note:** The `mTemperature` array contains three readings across the tread width — left, center, and right (not inside, center, outside). All values are in **Kelvin**; subtract 273.15 to convert to Celsius.

**Additional tire data:**

```python
w = player.mWheels[0]  # front-left example

# Carcass temperature (Kelvin) — rough average of the tire carcass
carcass_temp_c = w.mTireCarcassTemperature - 273.15

# Inner layer temperatures (Kelvin) — 3 readings from rubber layer before carcass
inner_temps_c = [w.mTireInnerLayerTemperature[j] - 273.15 for j in range(3)]

# Tire compound
front_compound = player.mFrontTireCompoundName.decode()
rear_compound = player.mRearTireCompoundName.decode()

# Dynamics
camber = w.mCamber                     # radians
toe = w.mToe                           # radians
lateral_force = w.mLateralForce        # Newtons
longitudinal_force = w.mLongitudinalForce  # Newtons
rotation = w.mRotation                 # radians/sec
brake_pressure = w.mBrakePressure      # 0.0-1.0 (will be kPa in future)
```

---

## Practical Examples

### Real-Time Telemetry Display

A continuously updating telemetry readout:

```python
import time
from pyLMUSharedMemory.lmu_mmap import MMapControl
from pyLMUSharedMemory.lmu_data import LMUObjectOut, LMUConstants
import math

info = MMapControl(LMUConstants.LMU_SHARED_MEMORY_FILE, LMUObjectOut)
info.create()

try:
    while True:
        info.update()

        telem = info.data.telemetry
        if not telem.playerHasVehicle:
            print("Waiting for player vehicle...")
            time.sleep(1)
            continue

        player = telem.telemInfo[telem.playerVehicleIdx]

        # Calculate speed
        speed_ms = math.sqrt(
            player.mLocalVel.x**2 +
            player.mLocalVel.y**2 +
            player.mLocalVel.z**2
        )
        speed_kmh = speed_ms * 3.6

        gear = player.mGear
        gear_str = "R" if gear == -1 else ("N" if gear == 0 else str(gear))

        rpm = player.mEngineRPM
        max_rpm = player.mEngineMaxRPM
        throttle = player.mUnfilteredThrottle
        brake = player.mUnfilteredBrake
        fuel = player.mFuel

        print(f"\rGear: {gear_str} | "
              f"Speed: {speed_kmh:6.1f} km/h | "
              f"RPM: {rpm:7.0f}/{max_rpm:.0f} | "
              f"Throttle: {throttle:5.1%} | "
              f"Brake: {brake:5.1%} | "
              f"Fuel: {fuel:5.1f}L",
              end="", flush=True)

        time.sleep(0.05)  # ~20 Hz update rate

except KeyboardInterrupt:
    print("\nStopping...")
finally:
    info.close()
```

### Lap Time Monitor

Track lap times and sector splits:

```python
import time
from pyLMUSharedMemory.lmu_mmap import MMapControl
from pyLMUSharedMemory.lmu_data import LMUObjectOut, LMUConstants

def format_time(seconds):
    """Format time as M:SS.mmm"""
    if seconds <= 0:
        return "  —:——.———"
    mins = int(seconds // 60)
    secs = seconds % 60
    return f"{mins}:{secs:06.3f}"

info = MMapControl(LMUConstants.LMU_SHARED_MEMORY_FILE, LMUObjectOut)
info.create()

last_lap_count = -1

try:
    while True:
        info.update()

        telem = info.data.telemetry
        scoring = info.data.scoring

        if not telem.playerHasVehicle:
            time.sleep(1)
            continue

        player_idx = telem.playerVehicleIdx
        player_scoring = scoring.vehScoringInfo[player_idx]

        current_laps = player_scoring.mTotalLaps

        # Detect new lap completion
        if current_laps > last_lap_count and last_lap_count >= 0:
            last_lap = player_scoring.mLastLapTime
            best_lap = player_scoring.mBestLapTime

            s1 = player_scoring.mLastSector1
            s2 = player_scoring.mLastSector2 - player_scoring.mLastSector1
            s3 = last_lap - player_scoring.mLastSector2

            print(f"Lap {current_laps}: {format_time(last_lap)} "
                  f"(S1: {format_time(s1)} | S2: {format_time(s2)} | S3: {format_time(s3)}) "
                  f"{'** BEST **' if last_lap == best_lap and best_lap > 0 else ''}")

        last_lap_count = current_laps
        time.sleep(0.1)

except KeyboardInterrupt:
    print("\nStopping...")
finally:
    info.close()
```

### Track Position

Display the player's current position on track as a fraction of the total lap distance (0.0 at the start/finish line, approaching 1.0 at the end of the lap):

```python
import time
from pyLMUSharedMemory.lmu_mmap import MMapControl
from pyLMUSharedMemory.lmu_data import LMUObjectOut, LMUConstants

info = MMapControl(LMUConstants.LMU_SHARED_MEMORY_FILE, LMUObjectOut)
info.create()

try:
    while True:
        info.update()

        scoring = info.data.scoring
        session = scoring.scoringInfo
        track_length = session.mLapDist  # total track length in meters

        if track_length <= 0:
            print("No active session. Waiting...")
            time.sleep(1)
            continue

        num_vehicles = session.mNumVehicles
        for i in range(num_vehicles):
            veh = scoring.vehScoringInfo[i]
            if veh.mIsPlayer:
                # mLapDist is the vehicle's current distance around the track in meters
                lap_fraction = veh.mLapDist / track_length

                print(f"\rTrack position: {lap_fraction:.4f} "
                      f"({lap_fraction:.1%}) — "
                      f"{veh.mLapDist:.1f}m / {track_length:.1f}m",
                      end="", flush=True)
                break

        time.sleep(0.1)

except KeyboardInterrupt:
    print("\nStopping...")
finally:
    info.close()
```

- `scoringInfo.mLapDist` is the **total track length** in meters.
- `vehScoringInfo[i].mLapDist` is the **vehicle's current distance** around the track in meters.
- Dividing the two gives a value from `0.0` (start/finish line) to just under `1.0` (end of lap), representing how much of the lap distance the player has covered.

### Live Race Standings

Display a live leaderboard:

```python
import time
import os
from pyLMUSharedMemory.lmu_mmap import MMapControl
from pyLMUSharedMemory.lmu_data import LMUObjectOut, LMUConstants

def format_time(seconds):
    if seconds <= 0:
        return "—:——.———"
    mins = int(seconds // 60)
    secs = seconds % 60
    return f"{mins}:{secs:06.3f}"

info = MMapControl(LMUConstants.LMU_SHARED_MEMORY_FILE, LMUObjectOut)
info.create()

try:
    while True:
        info.update()

        scoring = info.data.scoring
        session = scoring.scoringInfo
        num_vehicles = session.mNumVehicles

        if num_vehicles == 0:
            print("No active session. Waiting...")
            time.sleep(2)
            continue

        # Clear screen
        os.system('cls' if os.name == 'nt' else 'clear')

        print(f"Track: {session.mTrackName.decode()}")
        print(f"Session: {session.mSession} | Phase: {session.mGamePhase} | "
              f"Vehicles: {num_vehicles}")
        print(f"Weather: {session.mAmbientTemp:.0f}°C | "
              f"Rain: {session.mRaining:.0%} | "
              f"Track: {session.mTrackTemp:.0f}°C")
        print("=" * 90)
        print(f"{'Pos':>3} {'Driver':<20} {'Class':<10} {'Laps':>4} "
              f"{'Best Lap':>12} {'Last Lap':>12} {'Gap':>10} {'Pits':>4}")
        print("-" * 90)

        # Collect and sort by position
        vehicles = []
        for i in range(num_vehicles):
            veh = scoring.vehScoringInfo[i]
            vehicles.append((veh.mPlace, i))
        vehicles.sort()

        for place, idx in vehicles:
            veh = scoring.vehScoringInfo[idx]
            driver = veh.mDriverName.decode().strip('\x00')
            vclass = veh.mVehicleClass.decode().strip('\x00')
            laps = veh.mTotalLaps
            best = format_time(veh.mBestLapTime)
            last = format_time(veh.mLastLapTime)
            gap = veh.mTimeBehindLeader
            pits = veh.mNumPitstops

            pit_str = "PIT" if veh.mInPits else str(pits)
            gap_str = "LEADER" if gap == 0 and place == 1 else f"+{gap:.3f}s"
            player_marker = " *" if veh.mIsPlayer else ""

            print(f"{place:>3} {driver:<20} {vclass:<10} {laps:>4} "
                  f"{best:>12} {last:>12} {gap_str:>10} {pit_str:>4}{player_marker}")

        time.sleep(1)

except KeyboardInterrupt:
    print("\nStopping...")
finally:
    info.close()
```

---

## Interpreting Encoded Values

Many fields use numeric codes to represent states. Here are the lookup tables:

### Session Types

`scoringInfo.mSession`:

| Value | Session |
|-------|---------|
| 0 | Test Day |
| 1–4 | Practice (1–4) |
| 5–8 | Qualifying (1–4) |
| 9 | Warmup |
| 10–13 | Race (1–4) |

### Game Phases

`scoringInfo.mGamePhase`:

| Value | Phase |
|-------|-------|
| 0 | Before session has begun |
| 1 | Reconnaissance laps (race only) |
| 2 | Grid walk-through (race only) |
| 3 | Formation lap (race only) |
| 4 | Starting-light countdown (race only) |
| 5 | Green flag |
| 6 | Full course yellow / safety car |
| 7 | Session stopped |
| 8 | Session over |
| 9 | Paused (heartbeat call) |

### Finish Status

`vehScoringInfo[i].mFinishStatus`:

| Value | Status |
|-------|--------|
| 0 | None (still running) |
| 1 | Finished |
| 2 | DNF (Did Not Finish) |
| 3 | DQ (Disqualified) |

### Vehicle Control

`vehScoringInfo[i].mControl`:

| Value | Controller |
|-------|-----------|
| -1 | Nobody (shouldn't appear) |
| 0 | Local player |
| 1 | Local AI |
| 2 | Remote (multiplayer) |
| 3 | Replay (shouldn't appear) |

### Pit State

`vehScoringInfo[i].mPitState`:

| Value | State |
|-------|-------|
| 0 | None |
| 1 | Request |
| 2 | Entering |
| 3 | Stopped |
| 4 | Exiting |

### Yellow Flag States

`scoringInfo.mYellowFlagState`:

| Value | State |
|-------|-------|
| -1 | Invalid |
| 0 | None |
| 1 | Pending |
| 2 | Pits closed |
| 3 | Pit lead lap |
| 4 | Pits open |
| 5 | Last lap |
| 6 | Resume |
| 7 | Race halt (not currently used) |

### Surface Types

`mWheel[i].mSurfaceType`:

| Value | Surface |
|-------|---------|
| 0 | Dry (paved) |
| 1 | Wet |
| 2 | Grass |
| 3 | Dirt |
| 4 | Gravel |
| 5 | Rumble strip |
| 6 | Special |

### Ignition / Starter

`telemInfo[i].mIgnitionStarter`:

| Value | State |
|-------|-------|
| 0 | Off |
| 1 | Ignition on |
| 2 | Ignition + starter |

### Rear Flap Legal Status

`telemInfo[i].mRearFlapLegalStatus`:

| Value | Status |
|-------|--------|
| 0 | Disallowed |
| 1 | Criteria detected, not yet allowed |
| 2 | Allowed |

### Electric Boost Motor State

`telemInfo[i].mElectricBoostMotorState`:

| Value | State |
|-------|-------|
| 0 | Unavailable |
| 1 | Inactive |
| 2 | Propulsion |
| 3 | Regeneration |

---

## Platform Notes

| | Windows | Linux |
|---|---------|-------|
| **Shared memory mechanism** | Named file mapping (`mmap` with `tagname`) | File-backed mapping (`/dev/shm/LMU_Data`) |
| **SimInfo class** | ✅ Supported | ❌ Not supported (uses Windows-only `tagname`) |
| **MMapControl class** | ✅ Supported | ✅ Supported |
| **Game requirement** | LMU must be running and have written to shared memory | LMU (or a bridge/proxy) must write to `/dev/shm/LMU_Data` |

On **Linux**, the shared memory file `/dev/shm/LMU_Data` is created automatically if it doesn't exist. If the game is running on Windows, you may need a bridge or proxy to forward the data to the Linux shared memory file.

---

## Tips & Best Practices

1. **Always check if the game is running.** A `gameVersion` of `0` means the game hasn't written to shared memory yet. Guard your code accordingly.

2. **Use copy access mode** (the default) to avoid reading partially-updated data. The small performance cost is negligible for most applications.

3. **Call `update()` at a reasonable rate.** Telemetry updates at the physics rate of the game (typically 90–400 Hz). For display purposes, 20–60 Hz is usually sufficient. There's no benefit to calling `update()` faster than the game updates.

4. **Decode byte strings.** All name fields (`mTrackName`, `mDriverName`, `mVehicleName`, etc.) are returned as `bytes`. Use `.decode()` to convert them to Python strings. They may contain null bytes, so use `.decode().strip('\x00')` if needed.

5. **Use the active vehicle count.** Don't iterate over all 104 vehicle slots. Use `scoring.scoringInfo.mNumVehicles` or `telemetry.activeVehicles` to know how many vehicles have valid data.

6. **Temperature units vary.** Engine temperatures (`mEngineWaterTemp`, `mEngineOilTemp`) and brake temperatures (`mBrakeTemp`) are in **Celsius**. Tire temperatures (`mTemperature`, `mTireCarcassTemperature`, `mTireInnerLayerTemperature`) are in **Kelvin** (subtract 273.15 to get Celsius).

7. **Sector times are cumulative.** `mBestSector2` is sector 1 + sector 2 time, not sector 2 alone. Subtract sector 1 to get the individual sector time. Also note the unusual `mSector` numbering: `0` = sector 3, `1` = sector 1, `2` = sector 2 (see [Per-Vehicle Scoring](#per-vehicle-scoring)).

8. **Close the connection when done.** Always call `info.close()` to properly release the memory mapping. Use a `try/finally` block to ensure cleanup.

9. **Logging.** The library uses Python's `logging` module. To see internal log messages, configure a handler on the root logger before creating the `MMapControl`:

   ```python
   import logging
   logging.basicConfig(level=logging.INFO)
   ```

10. **Type hints.** Import from `lmu_type` for IDE autocompletion and type checking assistance. The type hint classes mirror the ctypes structures but provide clean Python type annotations.

---

## Complete Data Reference

### LMUVect3

A 3D vector used for positions, velocities, accelerations, and rotations.

| Field | Type | Description |
|-------|------|-------------|
| `x` | float | X component |
| `y` | float | Y component |
| `z` | float | Z component |

### LMUWheel

Per-wheel data (4 per vehicle: FL, FR, RL, RR).

| Field | Type | Unit | Description |
|-------|------|------|-------------|
| `mSuspensionDeflection` | float | meters | Suspension deflection |
| `mRideHeight` | float | meters | Ride height |
| `mSuspForce` | float | Newtons | Pushrod load |
| `mBrakeTemp` | float | °C | Brake temperature |
| `mBrakePressure` | float | 0.0–1.0 | Brake pressure (currently driver input fraction; may change to kPa in a future game version) |
| `mRotation` | float | rad/s | Wheel rotation speed |
| `mLateralPatchVel` | float | m/s | Lateral velocity at contact patch |
| `mLongitudinalPatchVel` | float | m/s | Longitudinal velocity at contact patch |
| `mLateralGroundVel` | float | m/s | Lateral ground velocity |
| `mLongitudinalGroundVel` | float | m/s | Longitudinal ground velocity |
| `mCamber` | float | radians | Camber angle |
| `mLateralForce` | float | Newtons | Lateral tire force |
| `mLongitudinalForce` | float | Newtons | Longitudinal tire force |
| `mTireLoad` | float | Newtons | Vertical tire load |
| `mGripFract` | float | fraction | Fraction of contact patch sliding |
| `mPressure` | float | kPa | Tire pressure |
| `mTemperature[3]` | float×3 | Kelvin | Tread temperature: left, center, right |
| `mWear` | float | 0.0–1.0 | Tire wear (1.0 = new) |
| `mTerrainName` | bytes | — | Material name from TDF file |
| `mSurfaceType` | int | enum | Surface type (see [Surface Types](#surface-types)) |
| `mFlat` | bool | — | Whether tire is flat |
| `mDetached` | bool | — | Whether wheel is detached |
| `mStaticUndeflectedRadius` | int | cm | Tire radius |
| `mVerticalTireDeflection` | float | meters | Tire deflection from speed-sensitive radius |
| `mWheelYLocation` | float | meters | Wheel Y location relative to vehicle |
| `mToe` | float | radians | Current toe angle |
| `mTireCarcassTemperature` | float | Kelvin | Average carcass temperature |
| `mTireInnerLayerTemperature[3]` | float×3 | Kelvin | Inner rubber layer temperatures |

### LMUVehicleTelemetry

Per-vehicle telemetry data (up to 104 vehicles).

| Field | Type | Unit | Description |
|-------|------|------|-------------|
| `mID` | int | — | Slot ID |
| `mDeltaTime` | float | seconds | Time since last update |
| `mElapsedTime` | float | seconds | Game session time |
| `mLapNumber` | int | — | Current lap number |
| `mLapStartET` | float | seconds | Time this lap started |
| `mVehicleName` | bytes | — | Vehicle name |
| `mTrackName` | bytes | — | Track name |
| `mPos` | LMUVect3 | meters | World position |
| `mLocalVel` | LMUVect3 | m/s | Velocity in local coordinates |
| `mLocalAccel` | LMUVect3 | m/s² | Acceleration in local coordinates |
| `mOri[3]` | LMUVect3×3 | — | Orientation matrix (3 rows) |
| `mLocalRot` | LMUVect3 | rad/s | Rotation in local coordinates |
| `mLocalRotAccel` | LMUVect3 | rad/s² | Rotational acceleration |
| `mGear` | int | — | -1=reverse, 0=neutral, 1+=forward |
| `mEngineRPM` | float | RPM | Engine RPM |
| `mEngineWaterTemp` | float | °C | Engine water temperature |
| `mEngineOilTemp` | float | °C | Engine oil temperature |
| `mClutchRPM` | float | RPM | Clutch RPM |
| `mUnfilteredThrottle` | float | 0.0–1.0 | Raw throttle input |
| `mUnfilteredBrake` | float | 0.0–1.0 | Raw brake input |
| `mUnfilteredSteering` | float | -1.0–1.0 | Raw steering input (left to right) |
| `mUnfilteredClutch` | float | 0.0–1.0 | Raw clutch input |
| `mFilteredThrottle` | float | 0.0–1.0 | Processed throttle |
| `mFilteredBrake` | float | 0.0–1.0 | Processed brake |
| `mFilteredSteering` | float | -1.0–1.0 | Processed steering |
| `mFilteredClutch` | float | 0.0–1.0 | Processed clutch |
| `mSteeringShaftTorque` | float | Nm | Torque around steering shaft |
| `mFront3rdDeflection` | float | meters | Front 3rd spring deflection |
| `mRear3rdDeflection` | float | meters | Rear 3rd spring deflection |
| `mFrontWingHeight` | float | meters | Front wing height |
| `mFrontRideHeight` | float | meters | Front ride height |
| `mRearRideHeight` | float | meters | Rear ride height |
| `mDrag` | float | force | Aerodynamic drag |
| `mFrontDownforce` | float | force | Front downforce |
| `mRearDownforce` | float | force | Rear downforce |
| `mFuel` | float | liters | Fuel remaining |
| `mEngineMaxRPM` | float | RPM | Rev limit |
| `mScheduledStops` | int (0–255) | — | Scheduled pit stops |
| `mOverheating` | bool | — | Overheating icon shown |
| `mDetached` | bool | — | Parts detached (besides wheels) |
| `mHeadlights` | bool | — | Headlights on |
| `mDentSeverity[8]` | int×8 | 0–2 | Dent severity at 8 body locations |
| `mLastImpactET` | float | seconds | Time of last impact |
| `mLastImpactMagnitude` | float | — | Last impact magnitude |
| `mLastImpactPos` | LMUVect3 | meters | Last impact position |
| `mEngineTorque` | float | Nm | Current engine torque |
| `mCurrentSector` | int | — | Current sector (zero-based, pit in sign bit) |
| `mSpeedLimiter` | int (0–255) | — | Speed limiter active |
| `mMaxGears` | int (0–255) | — | Maximum forward gears |
| `mFrontTireCompoundIndex` | int (0–255) | — | Front tire compound index |
| `mRearTireCompoundIndex` | int (0–255) | — | Rear tire compound index |
| `mFuelCapacity` | float | liters | Fuel tank capacity |
| `mFrontFlapActivated` | int (0–255) | — | Front flap activated |
| `mRearFlapActivated` | int (0–255) | — | Rear flap activated |
| `mRearFlapLegalStatus` | int (0–255) | enum | Rear flap status (see [Rear Flap Legal Status](#rear-flap-legal-status)) |
| `mIgnitionStarter` | int (0–255) | enum | Ignition state (see [Ignition / Starter](#ignition--starter)) |
| `mFrontTireCompoundName` | bytes | — | Front tire compound name |
| `mRearTireCompoundName` | bytes | — | Rear tire compound name |
| `mSpeedLimiterAvailable` | int (0–255) | — | Speed limiter available |
| `mAntiStallActivated` | int (0–255) | — | Hard anti-stall activated |
| `mVisualSteeringWheelRange` | float | degrees | Visual steering wheel range |
| `mRearBrakeBias` | float | fraction | Fraction of brakes on rear |
| `mTurboBoostPressure` | float | — | Turbo boost pressure |
| `mPhysicsToGraphicsOffset[3]` | float×3 | — | CG to graphical center offset |
| `mPhysicalSteeringWheelRange` | float | degrees | Physical steering wheel range |
| `mDeltaBest` | float | seconds | Delta to best lap time |
| `mBatteryChargeFraction` | float | 0.0–1.0 | Battery charge level |
| `mElectricBoostMotorTorque` | float | Nm | Boost motor torque (negative = regen) |
| `mElectricBoostMotorRPM` | float | RPM | Boost motor RPM |
| `mElectricBoostMotorTemperature` | float | °C | Boost motor temperature |
| `mElectricBoostWaterTemperature` | float | °C | Boost motor coolant temperature |
| `mElectricBoostMotorState` | int (0–255) | enum | Motor state (see [Electric Boost Motor State](#electric-boost-motor-state)) |
| `mExpansion` | bytes | — | Reserved for future use |
| `mWheels[4]` | LMUWheel×4 | — | Wheel data: [0]=FL, [1]=FR, [2]=RL, [3]=RR |

### LMUVehicleScoring

Per-vehicle scoring/race data (up to 104 vehicles).

| Field | Type | Unit | Description |
|-------|------|------|-------------|
| `mID` | int | — | Slot ID |
| `mDriverName` | bytes | — | Driver name |
| `mVehicleName` | bytes | — | Vehicle name |
| `mTotalLaps` | int | — | Laps completed |
| `mSector` | int | — | Current sector: 0=sector 3, 1=sector 1, 2=sector 2 |
| `mFinishStatus` | int | enum | Finish status (see [Finish Status](#finish-status)) |
| `mLapDist` | float | meters | Distance around track |
| `mPathLateral` | float | meters | Lateral offset from center path |
| `mTrackEdge` | float | meters | Track edge distance from center |
| `mBestSector1` | float | seconds | Best sector 1 time |
| `mBestSector2` | float | seconds | Best sector 1 + sector 2 cumulative |
| `mBestLapTime` | float | seconds | Best lap time |
| `mLastSector1` | float | seconds | Last sector 1 time |
| `mLastSector2` | float | seconds | Last sector 1 + sector 2 cumulative |
| `mLastLapTime` | float | seconds | Last lap time |
| `mCurSector1` | float | seconds | Current sector 1 (if valid) |
| `mCurSector2` | float | seconds | Current sector 1 + sector 2 (if valid) |
| `mNumPitstops` | int | — | Number of pit stops made |
| `mNumPenalties` | int | — | Outstanding penalties |
| `mIsPlayer` | bool | — | Is this the player's vehicle |
| `mControl` | int | enum | Vehicle controller (see [Vehicle Control](#vehicle-control)) |
| `mInPits` | bool | — | Vehicle is in pit lane |
| `mPlace` | int | — | Race position (1-based) |
| `mVehicleClass` | bytes | — | Vehicle class name |
| `mTimeBehindNext` | float | seconds | Gap to car in next higher position |
| `mLapsBehindNext` | int | — | Laps behind car in next higher position |
| `mTimeBehindLeader` | float | seconds | Gap to leader |
| `mLapsBehindLeader` | int | — | Laps behind leader |
| `mLapStartET` | float | seconds | Time this lap was started |
| `mPos` | LMUVect3 | meters | World position |
| `mLocalVel` | LMUVect3 | m/s | Velocity in local coordinates |
| `mLocalAccel` | LMUVect3 | m/s² | Acceleration in local coordinates |
| `mOri[3]` | LMUVect3×3 | — | Orientation matrix |
| `mLocalRot` | LMUVect3 | rad/s | Local rotation |
| `mLocalRotAccel` | LMUVect3 | rad/s² | Local rotational acceleration |
| `mHeadlights` | int | — | Headlight status |
| `mPitState` | int | enum | Pit state (see [Pit State](#pit-state)) |
| `mServerScored` | int | — | Being scored by server |
| `mIndividualPhase` | int | — | Individual game phase |
| `mQualification` | int | — | Qualifying position (1-based, -1 = invalid) |
| `mTimeIntoLap` | float | seconds | Estimated time into current lap |
| `mEstimatedLapTime` | float | seconds | Estimated lap time |
| `mPitGroup` | bytes | — | Pit group (team name) |
| `mFlag` | int | — | Flag shown (0=green, 6=blue) |
| `mUnderYellow` | bool | — | Taken full-course caution flag |
| `mCountLapFlag` | int | — | 0=no count, 1=count lap only, 2=count lap+time |
| `mInGarageStall` | bool | — | In correct garage stall |
| `mUpgradePack[16]` | int×16 | — | Coded upgrade data |
| `mPitLapDist` | float | meters | Pit location in lap distance |
| `mBestLapSector1` | float | seconds | Sector 1 from best lap |
| `mBestLapSector2` | float | seconds | Sector 2 from best lap |
| `mSteamID` | int | — | Steam ID of current driver |
| `mVehFilename` | bytes | — | Vehicle file name (.veh) |
| `mAttackMode` | int | — | Attack mode state |
| `mFuelFraction` | int | 0x00–0xFF | Fuel/battery percentage (0x00=0%, 0xFF=100%) |
| `mDRSState` | bool | — | DRS (rear flap) active |

### LMUScoringInfo

Session-level scoring information.

| Field | Type | Unit | Description |
|-------|------|------|-------------|
| `mTrackName` | bytes | — | Track name |
| `mSession` | int | enum | Session type (see [Session Types](#session-types)) |
| `mCurrentET` | float | seconds | Current elapsed time |
| `mEndET` | float | seconds | Session end time |
| `mMaxLaps` | int | — | Maximum laps |
| `mLapDist` | float | meters | Track length |
| `mNumVehicles` | int | — | Number of vehicles |
| `mGamePhase` | int | enum | Game phase (see [Game Phases](#game-phases)) |
| `mYellowFlagState` | int | enum | Yellow flag state (see [Yellow Flag States](#yellow-flag-states)) |
| `mSectorFlag[3]` | int×3 | — | Local yellow in each sector |
| `mStartLight` | int | — | Start light frame |
| `mNumRedLights` | int | — | Number of red lights in start sequence |
| `mInRealtime` | bool | — | On track (vs. monitor mode) |
| `mPlayerName` | bytes | — | Player name |
| `mPlrFileName` | bytes | — | Player file name |
| `mDarkCloud` | float | 0.0–1.0 | Cloud darkness |
| `mRaining` | float | 0.0–1.0 | Rain severity |
| `mAmbientTemp` | float | °C | Ambient temperature |
| `mTrackTemp` | float | °C | Track surface temperature |
| `mWind` | LMUVect3 | m/s | Wind speed and direction |
| `mMinPathWetness` | float | 0.0–1.0 | Minimum path wetness |
| `mMaxPathWetness` | float | 0.0–1.0 | Maximum path wetness |
| `mGameMode` | int | — | 1=server, 2=client, 3=both |
| `mIsPasswordProtected` | bool | — | Server password protected |
| `mServerPort` | int | — | Server port |
| `mServerPublicIP` | int | — | Server public IP |
| `mMaxPlayers` | int | — | Maximum number of vehicles |
| `mServerName` | bytes | — | Server name |
| `mStartET` | float | seconds | Event start time (seconds since midnight) |
| `mAvgPathWetness` | float | 0.0–1.0 | Average path wetness |

### LMUApplicationState

Application window and UI state.

| Field | Type | Unit | Description |
|-------|------|------|-------------|
| `mAppWindow` | int (64-bit) | — | Window handle (HWND) |
| `mWidth` | int | pixels | Screen width |
| `mHeight` | int | pixels | Screen height |
| `mRefreshRate` | int | Hz | Refresh rate |
| `mWindowed` | int | bool | Windowed mode |
| `mOptionsLocation` | int | — | 0=main UI, 1=track loading, 2=monitor, 3=on track |
| `mOptionsPage` | bytes | — | Current options page name |

### LMUEvent

Event flags indicating game state transitions.

| Field | Type | Description |
|-------|------|-------------|
| `SME_ENTER` | int | Entered simulation |
| `SME_EXIT` | int | Exited simulation |
| `SME_STARTUP` | int | Game started |
| `SME_SHUTDOWN` | int | Game shutting down |
| `SME_LOAD` | int | Session loaded |
| `SME_UNLOAD` | int | Session unloaded |
| `SME_START_SESSION` | int | Session started |
| `SME_END_SESSION` | int | Session ended |
| `SME_ENTER_REALTIME` | int | Entered on-track mode |
| `SME_EXIT_REALTIME` | int | Left on-track mode |
| `SME_UPDATE_SCORING` | int | Scoring data updated |
| `SME_UPDATE_TELEMETRY` | int | Telemetry data updated |
| `SME_INIT_APPLICATION` | int | Application initialized |
| `SME_UNINIT_APPLICATION` | int | Application uninitialized |
| `SME_SET_ENVIRONMENT` | int | Environment set |
| `SME_FFB` | int | Force feedback update |
