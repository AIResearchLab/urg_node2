# Overview
This package provides a ROS2 driver node for HOKUYO 2D LiDAR(SOKUIKI Sensor).

## Feature
  - Supports Ethernet and USB connections
  - Scanning data output
  - Multi-Echo Scanning data output (use `laser_proc` topic)
  - Output hardware diagnostic information (use `diagnostic_updater` topic)
  - Component Mounting
  - [Life cycle control](http://design.ros2.org/articles/node_lifecycle.html)support

The interfaces and some of the processes are based on [urg_node package](http://wiki.ros.org/urg_node), which is often used in ROS1.

[urg_library ver.1.2.5](https://github.com/UrgNetworks/urg_library/tree/ver.1.2.5) is used for communication with LiDAR.

## Life cycle control
The operation in each state of the life cycle is as follows

- Unconfigured $~~~~$ Startup state
- Inactive $~~~~~~~~~~~~$ LiDAR connection state（not delivered scanning data and diagnostic information）
- Active $~~~~~~~~~~~~~~$ LiDAR data delivery state（delivered scanning data and diagnostic information）
- Finalize $~~~~~~~~~~~~$ End state

# Supported models
Hokuyo's SCIP 2.2-compliant LiDAR
Tested models: `UTM-30LX-EW`, `UST-10LX`, `UTM-30LX`, `URG-04LX-UG01`, `UAM-05LP`, `UAM-05LPA`

Tested Environments: `foxy`, `galactic`, `humble`

# License
`Apache License 2.0`
The urg_libray C API is licensed under the `Simplified BSD License`.

# Publish

| Topic | Message | Description | depends on |
| ----- | ------- | ----------- | ---------- |
| /scan         | sensor_msgs::msg::LaserScan            | LiDAR scanning data            | `publish_multiecho`=false |
| /echoes       | sensor_msgs::msg::MultiEchoLaserScan   | LiDAR multi-echo scanning data | `publish_multiecho`=true |
| /first        | sensor_msgs::msg::LaserScan            | Nearest scanning data          | `publish_multiecho`=true |
| /last         | sensor_msgs::msg::LaserScan            | Farthest scanning data         | `publish_multiecho`=true |
| /most_intense | sensor_msgs::msg::LaserScan            | Most intense scanning data     | `publish_multiecho`=true |
| /diagnostics  | diagnostics_msgs::msg::DiagnosticArray | Diagnostic information         | |

# Parameters

| Parameter | Type   | Default | Description |
| --------- | ------ | ------- | ----------- |
| ip_address            | string  | " "            | IP address for Ethernet connection（Specify in the format "XX.XX.XX.XX.XX"), If the specified string is empty, a serial connection is selected|
| ip_port               | int     | 10940          | Port number for Ethernet connection |
| serial_port           | string  | `/dev/ttyACM0` | Serial device path to connect to |
| serial_baud           | int     | 115200         | Serial baud rate |
| frame_id              | string  | "laser"        | Frame_id of scanning data. The parameter `frame_id` is set in the message header "frame_id" of the scan data. |
| calibrate_time        | bool    | false          | Adjustment mode flags. If this flag is true, the discrepancy between the LiDAR time and the system time is measured at the start of the scan and added to the timestamp of the scan data as latency.|
| synchronize_time      | bool    | false          | Synchronous mode flags. If this flag is true, the system time, which is the reference for the timestamp of the scan data, is dynamically corrected using the discrepancy from the LiDAR time.|
| publish_intensity     | bool    | false          | Intensity output mode flag. If this flag is true, the intensity data of the scan data is output; if false, the intensity data is output empty. If LiDAR does not support intensity output, it works the same as the false setting.|
| publish_multiecho     | bool    | false          | Multi-echo mode flag. If this flag is true, multi-echo scan data (/echoes, /first, /last, /most_intense) is output; if false, scan data (/scan) is output. If LiDAR does not support multi-echo, it works the same as the false setting. |
| error_limit           | int     | 4 [count]      | Number of errors to perform reconnection. Reconnects the connection with LiDAR when the number of errors that occurred during data acquisition becomes larger than error_limit.  |
| error_reset_period    | double  | 5.0 [sec]      | Error reset cycle. Periodically resets the number of errors that occurred during data acquisition. To prevent reconnection in case of sporadic errors. |
| diagnostics_tolerance | double  | 0.05           | Allowable percentage of diagnostic information on frequency of scan data delivery relative to target delivery frequency. For example, if diagnostics_tolerance is 0.05, the normal range is 105% to 95% of the target delivery frequency. |
| diagnostics_window_time  | double   | 5.0 [sec]      | Period to measure the frequency of output delivery of diagnostic information on the frequency of scan data delivery. |
| error_limit        | int    | 4 [count]       | Number of errors to perform reconnection. Reconnects the connection with LiDAR when the number of errors that occurred during data acquisition becomes larger than error_limit.  |
| time_offset        | double | 0.0 [sec]       | User latency to be added to timestamp of scan data  |
| angle_min          | double | -pi [rad], range：-pi~pi | Minimum angle of scan data output range  |
| angle_max          | double |  pi [rad], range：-pi~pi | Maximum angle of scan data output range  |
| skip               | int    | 0 [count], range: 0~9    | Output thinning setting for scanned data. The frequency of scanned data delivery is multiplied by 1/(skip+1).  |
| cluster            | int    | 1 [count], range: 1~99   | Scan data grouping settings. The number of data in the scan data is multiplied by 1/cluster.  |


# Build

## Obtaining Source Code

```bash
mkdir -p workspace/src
cd workspace/src
git clone --recursive https://github.com/AIResearchLab/urg_node2.git
```

2. Installing Related Packages

```bash
cd workspace/
rosdep update
rosdep install -i --from-paths urg_node2
```

3. Build

```
workspace/
colcon build --symlink-install
```

# Use

## launch

1. Connect LiDAR via Ethernet or USB.
   
2. Set the connection destination (parameters). Edit `config/params_ether.yaml` (for Ethernet connections) or `config/params_serial.yaml` (for USB connections).
  
3. Execute one of the following commands to automatically transition to the Active state and begin distribution of scan data.
   ```bash
   # for ethernet connected LiDAR
   ros2 launch urg_node2 urg_node2_ether.launch.py

   # for serial connected LiDAR
   ros2 launch urg_node2 urg_node2_serial.launch.py
   ```
   If you do not want to automatically transition to the Active state, execute the following command. (After startup, it will enter the Unconfigured state.)
   ```
   # for ethernet connected LiDAR
   ros2 launch urg_node2 urg_node2_ether.launch.py auto_start:=false

   # for serial connected LiDAR
   ros2 launch urg_node2 urg_node2_serial.launch.py auto_start:=false
   ```

   > In `urg_node2.launch.py`, urg_node2 is launched as a standalone lifecycle node (not as a component). This is because there is no lifecycle control interface in launch when launched as a component and lifecycle node (not implemented in ROS2 at this time).

4. LaserScan messages are published with the `frame_id` parameter (default `laser`). If visualizing laser scan data with Rviz2, run the following command in a new terminal to startup a temporary static_transform_publisher

```bash
ros2 run tf2_ros static_transform_publisher 0 0 0 0.0 0.0 0.0 map laser
```

## Usage with Docker

Clone this reposiotory

```bash
git clone https://github.com/AIResearchLab/urg_node2.git && cd urg_node2/docker
```

Pull the Docker image and start compose (No need to run `docker compose build`)
```bash
docker compose pull
docker compose up
```

To clean the system,
```bash
docker compose down
docker volume rm docker_hakuyo
```