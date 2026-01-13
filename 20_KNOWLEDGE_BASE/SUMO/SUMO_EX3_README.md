# Exercise 3: Real OSM Road Network Scenario

**Last Updated:** December 20, 2025
**SUMO Version:** 1.25.0+ (v1_25_0+0604)
**Status:** ✅ Verified working with GUI and CSV output

## Scenario Details

- **Network:** Real OpenStreetMap road network (curved highway)
- **Vehicles:** 3-vehicle convoy (V001, V002, V003)
- **Duration:** ~99 seconds (vehicles complete route)
- **Sampling Rate:** 10 Hz (0.1s intervals)
- **Output Format:** CSV (semicolon-delimited)

## Files

- `map.osm` - OpenStreetMap export (163KB)
- `network.net.xml` - SUMO network converted from OSM (31KB)
- `vehicles.rou.xml` - Vehicle definitions and routes
- `scenario.sumocfg` - Simulation configuration
- `fcd_output.csv` - Floating Car Data output (206KB, 2807 lines)

## Running the Scenario

### GUI Mode (Windows WSL2)
```bash
cd ~/projects/RoadSense2/sumo_learning/exercise3_osm

docker run --rm \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $(pwd):/data:Z \
  -w /data \
  ghcr.io/eclipse-sumo/sumo:main \
  sumo-gui -c scenario.sumocfg
```

### GUI Mode (Fedora Linux)
```bash
cd ~/projects/RoadSense2/sumo_learning/exercise3_osm

# Enable X11 access (one-time)
xhost +local:docker

# Run SUMO GUI
docker run --rm \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $(pwd):/data:Z \
  -w /data \
  --device=/dev/dri:/dev/dri \
  --security-opt label=type:container_runtime_t \
  ghcr.io/eclipse-sumo/sumo:main \
  sumo-gui -c scenario.sumocfg
```

### Headless Mode
```bash
cd ~/projects/RoadSense2/sumo_learning/exercise3_osm

docker run --rm \
  -v $(pwd):/data:Z \
  -w /data \
  ghcr.io/eclipse-sumo/sumo:main \
  sumo -c scenario.sumocfg --no-warnings
```

## CSV Output Format

```csv
timestep_time;vehicle_id;vehicle_x;vehicle_y;vehicle_angle;vehicle_type;
vehicle_speed;vehicle_pos;vehicle_lane;vehicle_edge;vehicle_slope;vehicle_acceleration
```

**Key Columns:**
- `timestep_time` - Simulation time in seconds
- `vehicle_id` - V001 (ego), V002 (middle), V003 (lead)
- `vehicle_x`, `vehicle_y` - Position coordinates
- `vehicle_angle` - Heading in degrees
- `vehicle_speed` - Speed in m/s
- `vehicle_acceleration` - Acceleration in m/s²

## Verification Results

✅ SUMO 1.25.0+ GUI working on Windows WSL2
✅ CSV output generated successfully (206KB)
✅ All 3 vehicles tracked throughout simulation
✅ Real OSM road geometry preserved (curved highway)
✅ 10 Hz sampling rate confirmed
✅ Acceleration data included in output

## Next Steps

1. Add SSM device configuration for hazard detection
2. Create parameter variations (Exercise 4)
3. Implement TraCI emergency braking events (Exercise 5)
4. Build ESP-NOW emulator based on real measurements
